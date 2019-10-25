---
title: Installing Minikube and Hyperkit
subtitle: bye bye Virtualbox
layout: post
author: martoc
image: https://martoc.london/blog/images/k8s.png
---

I've got some problems with Minikube and Virtualbox, everytime I close my laptop
it stops working and I need to restart, delete, start, reboot, etc. it's not
very stable for this use case. I've installed the HyperKit driver and configured
Minikube to use this instead the default Virtualbox.

# Steps

* First, install the HyperKit driver (<https://github.com/moby/hyperkit>)

```bash
brew install hyperkit
```

For more information about Homebrew please visit <https://brew.sh/>

* If you don't have Minikube installed yet

```bash
brew cask install minikube
```
* Configure Minikube to use the HyperKit driver by default

```bash
minikube config set vm-driver hyperkit
```

If you don't want to configure this as the default driver you can run Minikube
setting the drive like this

```bash
minikube start --vm-driver=hyperkit
```
If you were running Virtualbox before like me wipe out your `~/.minikube` and
cluster configuration

```bash
minikube delete
rm -rf ~/.minikube
```
