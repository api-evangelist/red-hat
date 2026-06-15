---
title: "An overview of confidential containers on OpenShift bare metal"
url: "https://developers.redhat.com/articles/2026/06/04/overview-confidential-containers-openshift-bare-metal"
date: "2026-06-04"
author: "Pradipta Banerjee, Leonardo Milleri, Emanuele Giuseppe Esposito, Pei Zhang"
feed_url: "https://developers.redhat.com/blog/feed"
---
Confidential containers integrate trusted execution environments into Kubernetes to provide hardware-backed workload isolation using Kata Containers as the runtime and Trustee for remote attestation and secret provisioning. The solution enables a zero-trust execution model where pods operate within isolated VM boundaries and secrets are released only after cryptographic verification of runtime integrity on OpenShift bare metal.
