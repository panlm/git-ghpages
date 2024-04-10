---
title: eksdemo
description: 使用 eksdemo 快速搭建 eks 集群以及其他所需组件
created: 2023-07-15 09:44:34.470
last_modified: 2024-04-09
tags:
  - aws/container/eks
  - aws/cmd
---
> [!WARNING] This is a github note
# eksdemo

## install
```sh
curl --location "https://github.com/awslabs/eksdemo/releases/latest/download/eksdemo_$(uname -s)_x86_64.tar.gz" |tar xz -C /tmp
sudo mv -v /tmp/eksdemo /usr/local/bin

```

## create-eks-cluster-
```sh
CLUSTER_NAME=ekscluster2
eksdemo create cluster ${CLUSTER_NAME} \
    --instance m5.large \
    --nodes 3 \
    --version 1.26 \
    --vpc-cidr 10.10.0.0/16
```

## addons-
- externaldns ([[../../EKS/addons/externaldns-for-route53#install-with-eksdemo-]])
- aws load balancer controller ([[git/git-mkdocs/EKS/addons/aws-load-balancer-controller#install-with-eksdemo-]])

- 2 certificates, one for each domain name in original region
![[../awscli/acm-cmd#^kresvp]]

refer: [[../awscli/acm-cmd#create-certificate-with-eksdemo-]]

