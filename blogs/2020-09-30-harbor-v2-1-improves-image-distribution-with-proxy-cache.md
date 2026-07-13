---
title: "Harbor v2.1 improves image distribution with Proxy Cache and P2P support"
url: "https://goharbor.io/blog/harbor-2.1/"
date: "2020-09-30"
author: ""
feed_url: "https://goharbor.io/blog/index.xml"
---
We’re excited to announce the Harbor v2.1 GA release which focuses on scalable image distribution through Proxy Caching and Peer-To-Peer (P2P) solutions like Uber’s Kraken and Alibaba’s Dragonfly, Non-Blocking Garbage Collection, Sysdig Secure Scanner support, and support for Machine Learning on Kubernetes data models. Let’s dive in. Proxy Cache There are plenty of scenarios where clusters of container hosts are attempting to pull artifacts from some registry (herein referred to as the target registry) but have limited, intermittent network connectivity or even no connectivity at all.
