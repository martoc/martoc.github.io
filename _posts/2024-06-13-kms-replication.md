---
title: AWS KMS Key Replication
subtitle: Understanding Its Implications`
layout: post
author: martoc
image: https://martoc.github.io/blog/images/aws.png
---

When architecting cloud-based solutions, one key principle I follow is to isolate resources within their respective regions and avoid sharing or replicating them across regions. This approach consistently provides a more secure and compliant framework for business continuity. Recently, AWS has introduced replication capabilities for various resources. In this post, I will delve into AWS Key Management Service (KMS) and assess whether adopting replication for KMS keys offers tangible benefits.

## Understanding KMS Key Replication

At its core, replication in AWS KMS creates a duplicate key with the same key material and ID across regions. This enables applications that rely on the key ID to function seamlessly across multiple regions. However, the recommended best practice is to use aliases for keys, which simplifies key management. Unfortunately, not all AWS services currently support this feature and require passing the Amazon Resource Name (ARN) of the key instead. If you choose not to replicate your keys, managing and maintaining different ARNs for each region can become cumbersome.

## Limitations of KMS Key Replication

One significant limitation of KMS key replication is the lack of support for policy replication and I guess the reason is that AWS does not automatically rewrite policies to comply with regional requirements. For example, if a KMS key in region-1 is permitted to be used by a certain AWS resource, replicating this key to region-2 will not adjust the policy to match the new region. Consequently, manual adjustment of these policies is necessary, which could lead to configuration drift between regions. The image below illustrates how the replication process includes a warning about access policy differences. In a specific example, I created a key in eu-west-1 and replicated it to eu-west-2, where I modified the policy in the replication configuration, resulting in different access policies for the same key across two regions.

![KMS Key Replication](/blog/images/kms-replication-updated.png)

To address this challenge, the recommended approach is to utilise Infrastructure as Code (IaC) tools such as AWS CloudFormation or Terraform. These tools allow you to create templates or modules that configure your KMS keys consistently across all regions, ensuring a single source of truth for your policies. This method guarantees that your policies remain uniform and compliant, effectively reducing the risk of configuration drift.

Implementing IaC for managing KMS keys offers another critical advantage: it reduces the potential impact of security breaches by isolating keys within individual regions. Should a key become compromised, only data within that specific region is affected, bolstering overall security measures.

## Why Did AWS Implement KMS Key Replication?

I assume AWS implemented KMS key replication to speed up data replication and reduce associated KMS costs. When copying encrypted data across regions, the process is consistent across all services. In the source region, the data is decrypted and then re-encrypted using the key from the target region before transfer. AWS may skip this decryption and encryption process, directly transferring the data. However, it's important to note that AWS documentation lacks clarity on this matter, and it appears that AWS services continue to perform decryption and encryption even when using the same key. The following code demonstrates that this behaviour is possible.

Convert your secret to base64:

```bash
$ echo "Sensitive Information" | base64
U2Vuc2l0aXZlIEluZm9ybWF0aW9uCg==
```

Encrypt your secret using the key in region-1:

```bash
$ aws kms encrypt --key-id arn:aws:kms:eu-west-1::**********::key/mrk-c9dbd65ce07f4d878e20557cdfc19dda --plaintext "U2Vuc2l0aXZlIEluZm9ybWF0aW9uCg==" --region eu-west-1
{
    "CiphertextBlob": "AQICAHjbqMkwi7zkXkCCQy07CINpe3MbFx8tr7ZYmMVblojKMAE4evln/lRkw4A2MVLq3sAeAAAAdDByBgkqhkiG9w0BBwagZTBjAgEAMF4GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMvboDoLAfBuFHyAJgAgEQgDFBFxciV9cKWgY6w2zfGvVwgf1WzjDFmphyuOAFz82/paMRcUIPJ/+ZhSbVVhAkri27",
    "KeyId": "arn:aws:kms:eu-west-1:**********:key/mrk-c9dbd65ce07f4d878e20557cdfc19dda",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
```

Decrypt your secret using the key in region-2:

```bash
$ aws kms decrypt --key-id arn:aws:kms:eu-west-2::**********::key/mrk-c9dbd65ce07f4d878e20557cdfc19dda --ciphertext-blob AQICAHjbqMkwi7zkXkCCQy07CINpe3MbFx8tr7ZYmMVblojKMAE4evln/lRkw4A2MVLq3sAeAAAAdDByBgkqhkiG9w0BBwagZTBjAgEAMF4GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMvboDoLAfBuFHyAJgAgEQgDFBFxciV9cKWgY6w2zfGvVwgf1WzjDFmphyuOAFz82/paMRcUIPJ/+ZhSbVVhAkri27 --region eu-west-2
{
    "KeyId": "arn:aws:kms:eu-west-2::**********::key/mrk-c9dbd65ce07f4d878e20557cdfc19dda",
    "Plaintext": "U2Vuc2l0aXZlIEluZm9ybWF0aW9uCg==",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
```

Decode your secret:

```bash
$ echo "U2Vuc2l0aXZlIEluZm9ybWF0aW9uCg==" | base64 -d 
Sensitive Information
```

Technically, the KMS key replication feature is not designed to enhance security but rather to facilitate data replication across regions, as demonstrated in the example above. The feature does not provide any additional security benefits, as the key material is the same across regions. I'm not sure that the AWS Services apply this approach, but it's worth considering when evaluating the use of KMS key replication for Business Continuity and Disaster Recovery (BCDR) purposes.

## Conclusion

Despite the apparent convenience of AWS KMS key replication, the absence of policy replication and potential configuration drift diminish its appeal. By leveraging IaC tools like AWS CloudFormation or Terraform, organisations can ensure consistent and secure management of KMS keys across multiple regions. This approach not only simplifies administrative tasks but also fortifies security measures by minimising the impact radius of encryption key exposures. Therefore, opting for region-specific key management practices is advisable over relying solely on KMS key replications.

