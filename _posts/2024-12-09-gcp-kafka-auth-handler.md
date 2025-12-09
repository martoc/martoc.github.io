---
title: GCP Managed Kafka Authentication Handler
subtitle: A Workaround for SASL/OAUTHBEARER in Apache Beam Dataflow
layout: post
author: martoc
image: https://martoc.github.io/blog/images/gcp.png
---

When working with Google Cloud Platform's Managed Service for Apache Kafka, you'll quickly discover that authentication can be surprisingly challenging, especially when using Apache Beam Dataflow pipelines. In this post, I'll share a utility I created called [gcp-kafka-auth-handler](https://github.com/martoc/gcp-kafka-auth-handler) that bridges this gap.

## The Problem

GCP Managed Kafka recommends SASL/OAUTHBEARER as the authentication method. According to [Google's documentation](https://docs.cloud.google.com/managed-service-for-apache-kafka/docs/authn-types-kafka), this is "the most seamless and secure method for clients within Google Cloud" because it leverages Application Default Credentials (ADC) to automatically discover the service account identity.

For Java clients, Google provides a `GcpLoginCallbackHandler` class that handles obtaining a JSON Web Token (JWT) from Google's identity provider. For Python clients using the Confluent Kafka library, you can make internal calls to a Python function to obtain the token from IAM directly.

However, here's where things get tricky: **Apache Beam's Dataflow Kafka connector uses the Apache Kafka Java library rather than the Confluent library**. This means you cannot simply inject a Python callback function to handle token retrieval. The connector requires an external OAuth token endpoint URL.

## The Solution

I created [gcp-kafka-auth-handler](https://github.com/martoc/gcp-kafka-auth-handler), a lightweight HTTP server written in Go that acts as a sidecar service. It exchanges GCP default credentials for Kafka-compatible OAuth2 tokens and exposes them via an HTTP endpoint that the Kafka client can call.

### How It Works

The handler listens on port 14293 and converts GCP default credentials into JWT-formatted access tokens suitable for Kafka authentication. When a Kafka client requests a token, the handler:

1. Uses Application Default Credentials (ADC) to obtain a GCP access token
2. Formats it as a JWT-compatible response with the required claims (exp, iss, iat, sub)
3. Returns the token in a format the Kafka OAUTHBEARER mechanism understands

The response format is:

```json
{
  "access_token": "<header>.<jwt>.<gcp-access-token>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### Running the Handler

You can run the handler as a Docker container alongside your Dataflow job:

```bash
docker pull martoc/gcp-kafka-auth-handler:latest
docker run -p 14293:14293 martoc/gcp-kafka-auth-handler:latest
```

Or build and run from source:

```bash
make build
./target/builds/gcp-kafka-auth-handler-darwin-arm64 serve
```

## Usage in Apache Beam Dataflow

Here's how to configure your Dataflow pipeline to use the authentication handler:

```python
events = pipeline | "Read from Kafka" >> ReadFromKafka(
    consumer_config={
        "group.id": args.kafka_consumer_group_id.get(),
        "enable.auto.commit": "true",
        "bootstrap.servers": args.kafka_bootstrap_server.get(),
        "security.protocol": "SASL_SSL",
        "sasl.mechanism": "OAUTHBEARER",
        "sasl.oauthbearer.token.endpoint.url": args.kafka_auth_endpoint.get(),
        "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        "sasl.jaas.config": 'org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
                            clientId="unused" \
                            clientSecret="unused";',
    },
    topics=[args.kafka_topic.get()],
    commit_offset_in_finalize=True,
)
```

The key configuration points are:

- **sasl.oauthbearer.token.endpoint.url**: Points to your running gcp-kafka-auth-handler instance (e.g., `http://localhost:14293/`)
- **sasl.login.callback.handler.class**: Uses the standard Apache Kafka `OAuthBearerLoginCallbackHandler`
- **clientId and clientSecret**: Set to "unused" as the handler relies on ADC rather than client credentials

## Why This Works

The `OAuthBearerLoginCallbackHandler` in the Apache Kafka library expects an OAuth2 token endpoint that returns tokens in a specific format. By deploying gcp-kafka-auth-handler as a sidecar, we provide exactly that endpoint, bridging the gap between GCP's IAM-based authentication and Kafka's OAUTHBEARER mechanism.

## Integration as a Go Library

If you're building Go applications, you can also integrate the handler directly as a library:

```go
import "github.com/martoc/gcp-kafka-auth-handler/handler"

authHandler := handler.NewAuthHandlerBuilder().Build()
http.Handle("/oauth/token", authHandler)
```

This approach works well with popular Go web frameworks including gorilla/mux, chi, and gin.

## Conclusion

While GCP Managed Kafka's SASL/OAUTHBEARER authentication is the recommended approach, the lack of native IAM integration in Apache Beam's Kafka connector creates a gap. The gcp-kafka-auth-handler provides a simple workaround by exposing GCP credentials as an OAuth2 token endpoint.

This is admittedly a hack, but it works reliably and might help anyone facing similar challenges with Dataflow and GCP Managed Kafka integration. The project is open source under the MIT licence, and contributions are welcome.

Check out the [repository](https://github.com/martoc/gcp-kafka-auth-handler) for more details and the latest updates.
