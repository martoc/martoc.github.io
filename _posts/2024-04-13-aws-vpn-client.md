---
title: AWS VPN Client
subtitle: Elastic VPN Client for AWS
layout: post
author: martoc
image: https://martoc.github.io/blog/images/aws.png
---

Amazon Web Services (AWS) offers a VPN Client that is particularly advantageous for organizations seeking scalable and secure connectivity solutions compared to traditional VPN services like NordVPN. This distinction is largely due to the inherent flexibility and elasticity of cloud-based services provided by AWS, tailored to meet the dynamic requirements of modern businesses.

The AWS VPN Client is a component of AWS's broader suite of networking services, designed to facilitate secure and private connections between the remote workforce and AWS environments. It provides an encrypted tunnel from the client's computer directly to the AWS environment, ensuring that sensitive data remains protected over the internet.

One of the primary advantages of AWS VPN Client over a solution like NordVPN lies in its scalability. AWS services are designed to grow seamlessly with the needs of a business. This means that as an organization expands, its networking infrastructure can adapt without the need for significant overhauls or the procurement of additional hardware. This is not only cost-effective but also reduces the complexity of network management for IT departments.

Scalability in AWS is largely driven by its pay-as-you-go pricing model, which allows businesses to pay only for the resources they use without requiring long-term commitments. This contrasts with traditional VPN solutions like NordVPN, which typically operate on fixed subscription plans. While NordVPN offers different pricing tiers, adjusting the scale of service can often involve manual changes to subscription plans and can incur fixed costs regardless of actual usage.

Moreover, AWS VPN Client integrates directly with other AWS services, providing a more cohesive and streamlined infrastructure. For organizations heavily invested in the AWS ecosystem, using AWS VPN Client means fewer compatibility issues and a unified platform for managing both their data and network security. This integration facilitates better automation and centralized management, reducing the administrative burden and enhancing overall efficiency.

Another significant benefit of AWS VPN Client is its elasticity. AWS allows users to dynamically adjust their network configurations to accommodate varying workloads and changing business conditions. This capability is particularly valuable in scenarios such as remote work surges or project-based increases in data traffic. Traditional VPN clients like NordVPN, while offering robust security features, do not inherently provide the same level of flexibility in network configuration and rapid scalability.

Security is also a critical consideration, and here AWS offers distinct advantages through its comprehensive compliance and security governance framework. AWS environments are designed to meet the requirements of the most stringent regulatory standards, making them suitable for industries like healthcare, finance, and public services. NordVPN also prioritizes security but does not inherently offer the same level of integration with an organization’s governance and compliance protocols, particularly those already using AWS services.

Finally, the global infrastructure of AWS ensures that its VPN services are supported by a vast network of data centers around the world. This global presence helps in reducing latency and improving the reliability of connections for users distributed across various geographical locations. NordVPN also provides a wide network of servers globally, but the performance can vary significantly, and they lack the direct control over the network that AWS can provide its users.

In conclusion, while traditional VPN services like NordVPN offer strong security and privacy features, AWS VPN Client surpasses them in terms of elasticity, scalability, and integration, particularly for organizations already utilizing AWS services. This makes AWS VPN Client a more suitable choice for businesses looking for a scalable, secure, and integrated networking solution.

The following example [martoc/vpn-client](https://github.com/martoc/vpn-client) demostrates how to create a VPN client in AWS:

You need a VPC in AWS and a VPN client to connect to the VPC. This document describes how to create a VPC and VPN client configuration in AWS.

1. Create VPC (Optional)

An existing VPC could be used to create the VPN client. If you don't have a VPC, you can create one using the following command:

```bash
aws cloudformation create-stack --stack-name vpn-vpc --template-body file://src/cloudformation/vpn-vpc.yaml --region us-east-2
```

2. Create certificates for multual TLS

This document describes how to create certificates for mutual TLS. The same certificate could be used in multiple regions.

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
src/scripts/generate.sh
```

3. Import certificates into AWS ACM

```bash
aws acm import-certificate --certificate fileb://workdir/server.crt --private-key fileb://workdir/server.key --certificate-chain fileb://workdir/ca.crt --region us-east-2
```

4. Create vpn-client stack

```bash
aws cloudformation create-stack --stack-name vpn-client --template-body file://src/cloudformation/vpn-client.yaml --parameters "ParameterKey=ServerCertificateArn,ParameterValue=arn:aws:acm:*******:************:certificate/*********-****-****-****-*************" --region us-east-2
```

5. Configure AWS VPN Client

* Download the configuration file from the AWS Console
* Update the configuration file with the client certificate `workdir/client.crt` and client key `workdir/client.key` adding this section to the configuration file below the `<ca></ca>` section
```
<cert>
-----BEGIN CERTIFICATE-----
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
....
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
....
-----END PRIVATE KEY-----
```
* In your local install the [AWS VPN client](https://aws.amazon.com/vpn/client-vpn-download/)
* Configure the OpenVPN client with the configuration file
* Connect to the VPN

6. Connect your iOS device to the VPN (Optional)

* Download [OpenVPN Connect](https://apps.apple.com/us/app/openvpn-connect-openvpn-app/id590379981) from the App Store
* Share the configuration file with the iOS device and open it with OpenVPN Connect
* Provide a name to the connection and save it
* Connect to the VPN



