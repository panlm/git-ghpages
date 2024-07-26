---
title: Fargate on EKS
description: 在 EKS 集群中使用 Fargate
created: 2022-06-25 09:17:05.835
last_modified: 2024-02-03
locale: zh
tags:
  - aws/container/eks
  - aws/container/fargate
---

# Fargate on EKS
## 环境准备
- 登录你的实验环境 ([LINK](https://dashboard.eventengine.run/login))，并且打开 `AWS Console` 
- 进入 aws cloud9 ([LINK](https://console.aws.amazon.com/cloud9))，打开已经准备好的 `EKSLabIDE` 桌面，点击 `+` 新建一个 `Terminal` 窗口
- 安装必要的软件
```sh
aws s3 cp s3://ee-assets-prod-us-east-1/modules/bd7b369f613f452dacbcea2a5d058d5b/v6/eksinit.sh . 
chmod +x eksinit.sh
./eksinit.sh 
source ~/.bash_profile 
source ~/.bashrc

kubectl get nodes

```

- 如果可以正常显示节点信息，表示环境已经就绪。

## create fargate profile
### use eksctl
- 在现有集群中添加 fargate 支持
```sh
CLUSTER_NAME=eksworkshop-eksctl
AWS_REGION=${AWS_DEFAULT_REGION}
NAMESPACE=game-2048

# pods in namespace called `game-2048` will be deployed to fargate profile
eksctl create fargateprofile \
    --cluster ${CLUSTER_NAME} \
    --region ${AWS_REGION} \
    --name ${NAMESPACE} \
    --namespace ${NAMESPACE}

eksctl get fargateprofile \
    --cluster ${CLUSTER_NAME} \
    -o yaml

```

- 你可以登录 eks 管理界面确认 fargate profile 创建成功 
- 截个图

### using eksdemo
refer: [[git/git-mkdocs/CLI/linux/eksdemo#fargate-profile-]]

## install aws load balancer controller
- 接下来我们将安装一个应用，并且对外发布，这里需要用到 aws load balancer controller (refer [[aws-load-balancer-controller#install-with-eksdemo-]])

## deploy game 2048 
- 部署应用
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml

```

- 观察到有 `fargate-` 开头的节点被加入到集群中
```sh
kubectl get all -n $NAMESPACE -o wide

```

- 观察应用运行在 fargate 节点上，且状态已经为 `running`
```sh
kubectl get no  -o wide

```

- 打开浏览器访问应用URL
```sh
kubectl get ing -n ${NAMESPACE} -o=custom-columns="URL":.status.loadBalancer.ingress[*].hostname

```

## other fargate labs
- [immersion workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/76a5dd80-3249-4101-8726-9be3eeee09b2/en-US/fargate)
- [eksworkshop](https://www.eksworkshop.com/beginner/180_fargate/)


## monitoring pods on fargate 


