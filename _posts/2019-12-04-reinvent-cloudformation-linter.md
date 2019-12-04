---
title: Using Cloudformation Linter
subtitle: cfn-lint
layout: post
author: martoc
image: https://martoc.london/blog/images/aws.png
---

This tool provides static code analysis into your Cloudformation templates,
enforcing these checks in your code will allow infrastructure developers to
create better templates.

Install the tool

```bash
pip install cfn-lint
```

Then you can run it to check your templates, it supports `YAML` and `JSON`
templates

```bash
cfn-lint my-template.json
```

The result is output to the console and the command returns a non zero code
what is perfect to integrate it in a pipeline to check pull requests. I've
integrated this linter into my NAT project <https://github.com/martoc/nat>, when
I ran it locally I've got many violations

```
(aws) C02XH2HSJGH6:nat martoc$ cfn-lint nat.json
W2506 Parameter ImageId should be of type [AWS::EC2::Image::Id, AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>]
nat.json:18:7

W2030 You must specify a valid allowed value for InstanceType (m5.medium).
Valid values are [u'a1.2xlarge', u'a1.4xlarge', u'a1.large', u'a1.medium', u'a1.metal', u'a1.xlarge', u'c1.medium', u'c1.xlarge', u'c3.2xlarge', u'c3.4xlarge', u'c3.8xlarge', u'c3.large', u'c3.xlarge', u'c4.2xlarge', u'c4.4xlarge', u'c4.8xlarge', u'c4.large', u'c4.xlarge', u'c5.12xlarge', u'c5.18xlarge', u'c5.24xlarge', u'c5.2xlarge', u'c5.4xlarge', u'c5.9xlarge', u'c5.large', u'c5.metal', u'c5.xlarge', u'c5d.12xlarge', u'c5d.18xlarge', u'c5d.24xlarge', u'c5d.2xlarge', u'c5d.4xlarge', u'c5d.9xlarge', u'c5d.large', u'c5d.metal', u'c5d.xlarge', u'c5n.18xlarge', u'c5n.2xlarge', u'c5n.4xlarge', u'c5n.9xlarge', u'c5n.large', u'c5n.metal', u'c5n.xlarge', u'cc2.8xlarge', u'cr1.8xlarge', u'd2.2xlarge', u'd2.4xlarge', u'd2.8xlarge', u'd2.xlarge', u'f1.16xlarge', u'f1.2xlarge', u'f1.4xlarge', u'g2.2xlarge', u'g2.8xlarge', u'g3.16xlarge', u'g3.4xlarge', u'g3.8xlarge', u'g3s.xlarge', u'g4dn.12xlarge', u'g4dn.16xlarge', u'g4dn.2xlarge', u'g4dn.4xlarge', u'g4dn.8xlarge', u'g4dn.metal', u'g4dn.xlarge', u'h1.16xlarge', u'h1.2xlarge', u'h1.4xlarge', u'h1.8xlarge', u'hs1.8xlarge', u'i2.2xlarge', u'i2.4xlarge', u'i2.8xlarge', u'i2.xlarge', u'i3.16xlarge', u'i3.2xlarge', u'i3.4xlarge', u'i3.8xlarge', u'i3.large', u'i3.metal', u'i3.xlarge', u'i3en.12xlarge', u'i3en.24xlarge', u'i3en.2xlarge', u'i3en.3xlarge', u'i3en.6xlarge', u'i3en.large', u'i3en.metal', u'i3en.xlarge', u'm1.large', u'm1.medium', u'm1.small', u'm1.xlarge', u'm2.2xlarge', u'm2.4xlarge', u'm2.xlarge', u'm3.2xlarge', u'm3.large', u'm3.medium', u'm3.xlarge', u'm4.10xlarge', u'm4.16xlarge', u'm4.2xlarge', u'm4.4xlarge', u'm4.large', u'm4.xlarge', u'm5.12xlarge', u'm5.16xlarge', u'm5.24xlarge', u'm5.2xlarge', u'm5.4xlarge', u'm5.8xlarge', u'm5.large', u'm5.metal', u'm5.xlarge', u'm5a.12xlarge', u'm5a.16xlarge', u'm5a.24xlarge', u'm5a.2xlarge', u'm5a.4xlarge', u'm5a.8xlarge', u'm5a.large', u'm5a.xlarge', u'm5ad.12xlarge', u'm5ad.24xlarge', u'm5ad.2xlarge', u'm5ad.4xlarge', u'm5ad.large', u'm5ad.xlarge', u'm5d.12xlarge', u'm5d.16xlarge', u'm5d.24xlarge', u'm5d.2xlarge', u'm5d.4xlarge', u'm5d.8xlarge', u'm5d.large', u'm5d.metal', u'm5d.xlarge', u'm5dn.12xlarge', u'm5dn.16xlarge', u'm5dn.24xlarge', u'm5dn.2xlarge', u'm5dn.4xlarge', u'm5dn.8xlarge', u'm5dn.large', u'm5dn.metal', u'm5dn.xlarge', u'm5n.12xlarge', u'm5n.16xlarge', u'm5n.24xlarge', u'm5n.2xlarge', u'm5n.4xlarge', u'm5n.8xlarge', u'm5n.large', u'm5n.metal', u'm5n.xlarge', u'p2.16xlarge', u'p2.8xlarge', u'p2.xlarge', u'p3.16xlarge', u'p3.2xlarge', u'p3.8xlarge', u'p3dn.24xlarge', u'r3.2xlarge', u'r3.4xlarge', u'r3.8xlarge', u'r3.large', u'r3.xlarge', u'r4.16xlarge', u'r4.2xlarge', u'r4.4xlarge', u'r4.8xlarge', u'r4.large', u'r4.xlarge', u'r5.12xlarge', u'r5.16xlarge', u'r5.24xlarge', u'r5.2xlarge', u'r5.4xlarge', u'r5.8xlarge', u'r5.large', u'r5.metal', u'r5.xlarge', u'r5a.12xlarge', u'r5a.16xlarge', u'r5a.24xlarge', u'r5a.2xlarge', u'r5a.4xlarge', u'r5a.8xlarge', u'r5a.large', u'r5a.xlarge', u'r5ad.12xlarge', u'r5ad.24xlarge', u'r5ad.2xlarge', u'r5ad.4xlarge', u'r5ad.large', u'r5ad.xlarge', u'r5d.12xlarge', u'r5d.16xlarge', u'r5d.24xlarge', u'r5d.2xlarge', u'r5d.4xlarge', u'r5d.8xlarge', u'r5d.large', u'r5d.metal', u'r5d.xlarge', u'r5dn.12xlarge', u'r5dn.16xlarge', u'r5dn.24xlarge', u'r5dn.2xlarge', u'r5dn.4xlarge', u'r5dn.8xlarge', u'r5dn.large', u'r5dn.metal', u'r5dn.xlarge', u'r5n.12xlarge', u'r5n.16xlarge', u'r5n.24xlarge', u'r5n.2xlarge', u'r5n.4xlarge', u'r5n.8xlarge', u'r5n.large', u'r5n.metal', u'r5n.xlarge', u't1.micro', u't2.2xlarge', u't2.large', u't2.medium', u't2.micro', u't2.nano', u't2.small', u't2.xlarge', u't3.2xlarge', u't3.large', u't3.medium', u't3.micro', u't3.nano', u't3.small', u't3.xlarge', u't3a.2xlarge', u't3a.large', u't3a.medium', u't3a.micro', u't3a.nano', u't3a.small', u't3a.xlarge', u'u-18tb1.metal', u'u-24tb1.metal', u'x1.16xlarge', u'x1.32xlarge', u'x1e.16xlarge', u'x1e.2xlarge', u'x1e.32xlarge', u'x1e.4xlarge', u'x1e.8xlarge', u'x1e.xlarge', u'z1d.12xlarge', u'z1d.2xlarge', u'z1d.3xlarge', u'z1d.6xlarge', u'z1d.large', u'z1d.metal', u'z1d.xlarge']
nat.json:30:13

E3012 Property Resources/ServerGroup/Properties/MinSize should be of type String
nat.json:121:13

E3012 Property Resources/ServerGroup/Properties/MaxSize should be of type String
nat.json:122:13

E3012 Property Resources/ServerGroup/Properties/Tags/0/PropagateAtLaunch should be of type Boolean
nat.json:132:19

E3012 Property Resources/ServerGroup/Properties/Tags/1/PropagateAtLaunch should be of type Boolean
nat.json:139:19

E3012 Property Resources/ServerGroup/Properties/Tags/2/PropagateAtLaunch should be of type Boolean
nat.json:146:19

E3002 Property SecurityGroups should be of type List for resource LaunchConfig
nat.json:198:13

E3008 Property "SecurityGroups" has no valid Refs to Resources at Resources/LaunchConfig/Properties/SecurityGroups/Ref
nat.json:199:16

E3012 Property Resources/LaunchConfig/Properties/AssociatePublicIpAddress should be of type Boolean
nat.json:201:13

E3012 Property Resources/LaunchConfig/Properties/InstanceMonitoring should be of type Boolean
nat.json:208:13
```

I fixed these violations and integrated the tool using GitHub Actions. This is
the action file, I'm using an action created by scottbrenner.

```yaml
name: Cloudformation
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v1
      - name: Validate Cloudformation template
        uses: scottbrenner/cfn-lint-action@1.2.1
        with:
          args: nat.json
```
