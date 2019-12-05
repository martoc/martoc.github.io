---
title: Nitro
subtitle: Deep dive
layout: post
author: martoc
image: https://martoc.london/blog/images/aws.png
---

It based on cards, there are four different card types depending of the
functionality.

# Nitro Cards

## Nitro Card for VPC

* ENA Controller: it's an abstraction for different network drivers.

* VPC data plane: the card implement security groups, limiters, routing and
encapsulation.

## Nitro Card for EBS

* NVMe Controller: interface with the OS.

* EBS data plane: encryption, NVM to remote storage protocol.

## Nitro Card for Instance Storage

* NVMe controller: interface with the OS.

* Instance Storage data plane: transparent encryption, limiters, drive
monitoring.

## Nitro Card Controller

* System Control: provides passive API endpoint, coordinates all the other
cards, Nitro Hypervisor and security chip.

* Hardware root of trust: provides measurement and attestation.

# Nitro Security Chip

It's a microcontroller that provides security to the bare metal instance.
therefore customer instances cannot update the flash code that lives in the
motherboard.

# Nitro Hypervisor

It's based on KVM hypervisor but with a minimum number of features, the
hypervisor runs only when the instance requires it.
