---
title: Kafka Auth Handler Goes Multi-Cloud
subtitle: Extending SASL/OAUTHBEARER Support from GCP to AWS MSK
layout: post
author: martoc
image: https://martoc.github.io/blog/images/aws.png
---

Back in December 2024, I wrote about [gcp-kafka-auth-handler](/2024/12/09/gcp-kafka-auth-handler/), a utility I created to bridge the authentication gap between Apache Beam Dataflow and GCP Managed Kafka. Since then, the project has evolved significantly as part of our broader multi-cloud journey. Today, I'm pleased to announce that the library has been renamed to [kafka-auth-handler](https://github.com/martoc/kafka-auth-handler) and now supports both GCP and AWS MSK.

## The Journey to Multi-Cloud

As outlined in my recent post on [building an open deployment framework](/2026/01/03/open-deployment-framework/), we've been working towards supporting multiple cloud providers across our tooling. The kafka-auth-handler is a natural extension of this effort.

When we began migrating workloads to AWS, we encountered the same authentication challenge with Amazon MSK that we'd previously solved for GCP Managed Kafka. MSK supports IAM authentication, but Apache Beam's Kafka connector still requires an external OAuth token endpoint. Rather than creating a separate tool, it made sense to extend the existing solution.

## What's Changed

### Repository Rename

The project has been renamed from `gcp-kafka-auth-handler` to `kafka-auth-handler` to reflect its multi-cloud capabilities. The Docker image and Go module paths have been updated accordingly:

- **Repository**: [github.com/martoc/kafka-auth-handler](https://github.com/martoc/kafka-auth-handler)
- **Docker Image**: `martoc/kafka-auth-handler:latest`
- **Go Module**: `github.com/martoc/kafka-auth-handler`

### Provider Selection

The handler now supports a `PROVIDER` environment variable to select the cloud platform:

| Provider | Value | Additional Configuration |
|----------|-------|-------------------------|
| GCP | `gcp` (default) | Uses `GOOGLE_APPLICATION_CREDENTIALS` or Workload Identity |
| AWS | `aws` | Requires `REGION` environment variable, uses IRSA or AWS credential chain |

### Token Format

Both providers return tokens in the same format, ensuring compatibility with the Kafka OAUTHBEARER mechanism:

```json
{
  "access_token": "<header>.<claims>.<token>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The key difference is in the algorithm claim within the token header:
- **GCP**: Uses `GOOG_OAUTH2_TOKEN`
- **AWS**: Uses `AWS_MSK_IAM`

## Running the Handler

### For GCP (unchanged from before)

```bash
docker run -p 14293:14293 martoc/kafka-auth-handler:latest
```

Or with explicit provider selection:

```bash
docker run -p 14293:14293 \
  -e PROVIDER=gcp \
  martoc/kafka-auth-handler:latest
```

### For AWS MSK

```bash
docker run -p 14293:14293 \
  -e PROVIDER=aws \
  -e REGION=eu-west-1 \
  martoc/kafka-auth-handler:latest
```

On EKS, ensure your pod has the appropriate IAM role attached via IRSA (IAM Roles for Service Accounts).

## Usage in Apache Beam Dataflow

### GCP Configuration

The GCP configuration remains identical to the [original implementation](/blog/2024/12/09/gcp-kafka-auth-handler/):

```python
events = pipeline | "Read from Kafka" >> ReadFromKafka(
    consumer_config={
        "bootstrap.servers": "managed-kafka.example.com:9092",
        "security.protocol": "SASL_SSL",
        "sasl.mechanism": "OAUTHBEARER",
        "sasl.oauthbearer.token.endpoint.url": "http://localhost:14293/",
        "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        "sasl.jaas.config": 'org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required clientId="unused" clientSecret="unused";',
    },
    topics=["my-topic"],
)
```

### AWS MSK Configuration

For AWS MSK with IAM authentication, note the different bootstrap server port:

```python
events = pipeline | "Read from Kafka" >> ReadFromKafka(
    consumer_config={
        "bootstrap.servers": "b-1.msk-cluster.abc123.kafka.eu-west-1.amazonaws.com:9098",
        "security.protocol": "SASL_SSL",
        "sasl.mechanism": "OAUTHBEARER",
        "sasl.oauthbearer.token.endpoint.url": "http://localhost:14293/",
        "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        "sasl.jaas.config": 'org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required clientId="unused" clientSecret="unused";',
    },
    topics=["my-topic"],
)
```

**Important**: AWS MSK uses port **9098** for IAM-authenticated connections, not the standard 9092.

## Go Library Integration

The library API has been updated to support both providers:

### GCP Handler

```go
import "github.com/martoc/kafka-auth-handler/handler"

gcpHandler := handler.NewGCPAuthHandlerBuilder().Build()
http.Handle("/", gcpHandler)
```

### AWS Handler

```go
import "github.com/martoc/kafka-auth-handler/handler"

awsHandler := handler.NewAWSAuthHandlerBuilder().
    WithRegion("eu-west-1").
    Build()
http.Handle("/", awsHandler)
```

Both handlers implement the standard `http.Handler` interface and work with popular Go web frameworks including gorilla/mux, chi, and gin.

## Migration from gcp-kafka-auth-handler

If you're currently using the GCP-only version, migration is straightforward:

1. **Update Docker image**: Change from `martoc/gcp-kafka-auth-handler:latest` to `martoc/kafka-auth-handler:latest`
2. **Update Go imports**: Change from `github.com/martoc/gcp-kafka-auth-handler/handler` to `github.com/martoc/kafka-auth-handler/handler`
3. **No configuration changes required**: The default provider is `gcp`, so existing deployments will continue to work without modification

## Conclusion

The evolution from gcp-kafka-auth-handler to kafka-auth-handler reflects our ongoing journey towards a truly multi-cloud architecture. What started as a workaround for GCP Managed Kafka authentication has grown into a unified solution supporting both major cloud providers.

This update is part of a broader effort to ensure our tooling works consistently across cloud platforms, complementing the [open deployment framework](/blog/2026/01/03/open-deployment-framework/) we've built for CI/CD.

The project remains open source under the MIT licence. Check out the [repository](https://github.com/martoc/kafka-auth-handler) for the latest updates and feel free to contribute.
