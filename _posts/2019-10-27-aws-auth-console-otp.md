---
title: Using aws-cli and oathtool
subtitle: Seamlessly two-factor authentication for the AWS CLI
layout: post
author: martoc
image: https://martoc.london/blog/images/aws.png
---

Using two factor authentication with the AWS CLI is sometimes a pain, you need
to get a new token every N minutes then parse the result of this operation and
create the corresponding environment variables, I've installed the oath-toolkit
and configured the AWS CLI to get these OTP dynamically.

# Steps

* Install oath-toolkit

```bash
brew install oath-toolkit
```
For more information about Homebrew please visit <https://brew.sh/>

* This guide assumes that you have installed the AWS CLI, otherwise please check
this documentation <https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html>.

* Configure the MFA for the account <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html>,
store the OTP key in a file at `~/.aws-mfa`.

* Configuring MFA-Protected API Access <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_configure-api-require.html>.
in my account, my account user is assigned to a group and this group has the
following policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "*"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```
* Create an API access key id and secret key <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey>.

* Configure the access key id and secret key calling this and follow the
instructions.

```bash
aws configure
```

* Now to access your API you need to create a new set of credentials calling the
AWS STS service, in the following command replace the `ACCOUNT_ID` and
`USERNAME`.

```bash
aws sts get-session-token --region us-east-1 --serial-number arn:aws:iam::<ACCOUNT_ID>:mfa/<USERNAME> --token-code $(oathtool --base32 --totp $(cat ~/.aws-mfa))
```

* The output of the previous command returns a credential for accessing the AWS
API.
