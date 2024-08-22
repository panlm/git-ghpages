---
title: eks-loggroup-description
description: eks 日志类型分析
created: 2022-03-18 13:22:59.039
last_modified: 2024-04-18
tags:
  - aws/container/eks
  - aws/mgmt/cloudwatch
---
# eks-loggroup-description
## loggroup
### /aws/eks/cluster_name/cluster
eks control panel logging
- kube-apiserver-audit
![[attachments/eks-loggroup-description/IMG-eks-loggroup-description.png]]
- kube-apiserver
![[attachments/eks-loggroup-description/IMG-eks-loggroup-description-1.png]]
- kube-scheduler
- kube-controller-manager
![[attachments/eks-loggroup-description/IMG-eks-loggroup-description-2.png]]
- cloud-controller-manager
![[attachments/eks-loggroup-description/IMG-eks-loggroup-description-3.png]]
- authenticator
![[attachments/eks-loggroup-description/IMG-eks-loggroup-description-4.png]]
> 如果使用[[lambda-cwl-opensearch]]，不需要分开成多个index；如果使用firehose to opensearch，可能需要考虑多个firehose

https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html


### /aws/containerinsights/cluster_name/application 
All applications logs stored under `/var/log/containers/*.log` are streamed into the dedicated log group. 当 Kubernetes Pod 被驱逐或删除时，与该 Pod 关联的日志将从工作节点中永久删除。 All non-application logs such as kube-proxy and aws-node logs are excluded by default. However, additional Kubernetes add-on logs, such as CoreDNS logs, are also processed and streamed into this log group.
- log rotate in k8s refer: [link](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- **`containerLogMaxSize`** default 10MB
- **`containerLogMaxFiles`** default 5


### /aws/containerinsights/cluster_name/dataplane
EKS already provides control plane logs. With Fluent Bit integration in Container Insights, the logs generated by EKS data plane components, which run on every worker node and are responsible for maintaining running pods are captured as data plane logs. These logs are also streamed into a dedicated CloudWatch log group under this log group. **kube-proxy**, **aws-node**, and Docker runtime logs are saved into this log group. In addition to control plane logs, having data plane logs stored in CloudWatch Logs helps to provide a complete picture of your EKS clusters.


### /aws/containerinsights/cluster_name/host
system logs for each EKS worker node are streamed into the log group. These system logs include the contents of `/var/log/messages, /var/log/dmesg, /var/log/secure` files. Considering the stateless and dynamic nature of containerized workloads, where EKS worker nodes are often being terminated during scaling activities, streaming those logs in real time with Fluent Bit and having those logs available in CloudWatch logs, even after the node is terminated, are critical in terms of observability and monitoring health of EKS worker nodes. It also enables you to debug or troubleshoot cluster issues without logging into worker nodes in many cases and analyze these logs in more systematic way.


### /aws/containerinsights/cluster_name/performance
performance metrics used by container insight


### /aws/containerinsights/cluster_name/prometheus
prometheus metrics 
- [LINK1](https://www.eksworkshop.com/advanced/330_servicemesh_using_appmesh/add_nodegroup_fargate/cloudwatch_setup/#enable-prometheus-metrics-in-cloudwatch)
- [LINK2](https://catalog.us-east-1.prod.workshops.aws/workshops/31676d37-bbe9-4992-9cd1-ceae13c5116c/en-US/containerinsights/eks/prometheusmonitoring)


### /EKS/cluster_name/Windows
可以定制
log for windows pod
[[logging-windows-container]]


## query with logs insights
- https://repost.aws/knowledge-center/eks-get-control-plane-logs
- https://aws.github.io/aws-eks-best-practices/security/docs/detective/#analyze-logs-with-log-insights


## enable container insight
- [[git/git-mkdocs/EKS/solutions/monitor/eks-container-insights]]


## solutions
```expander
( file:POC-PCF-EKS-DRAFT  section:/Metric-指标-/)
$matchline:+27
```
### ==Metric-指标-==
- 使用开源 prometheus 监控 ([[../monitor/TC-prometheus-ha-architect-with-thanos.zh|TC-prometheus-ha-architect-with-thanos.zh]])
- cloudwatch原始监控ec2 metric
- 使用cloudwatch container insight ([[git/git-mkdocs/EKS/solutions/monitor/eks-container-insights|eks-container-insights]])
    - Enable Prometheus Metrics in CloudWatch ([[git/git-mkdocs/EKS/solutions/monitor/enable-prometheus-in-cloudwatch|enable-prometheus-in-cloudwatch]])

### Logging-日志-
- eks相关日志组描述 ([[git/git-mkdocs/EKS/solutions/logging/eks-loggroup-description|eks-loggroup-description]])
- 转发cloudwatch日志到splunk ([[cloudwatch-firehose-splunk]])
    - 控制平面：
        - 启用eks日志发送到Cloudwatch ([LINK](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html#enabling-control-plane-log-export))
        - 转发cloudtrail中相关eks日志到splunk ([LINK](https://docs.splunk.com/Documentation/AddOns/released/AWS/CloudTrail))
    - 数据平面
        - 推荐EFK解决方案作为日志收集 ([[eks-fluentbit-opensearch]])
        - Windows节点组日志收集 ([[logging-windows-container]])
        - ALB logging could be enabled to S3, how to forward to splunk ([LINK](https://splunkbase.splunk.com/app/1274/))
- 快速检索 Amazon EKS 控制平面日志 ([LINK](https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-get-control-plane-logs/))
- 转发cloudwatch日志到opensearch ([[eks-control-panel-log-cwl-firehose-opensearch]])
- eks控制平面日志组包含多个日志流，使用[[lambda-cwl-opensearch]]，将日志流发送到opensearch时，基于log group name和log stream name组合创建独立index文件
    - 或者将日志通过 kinesis firehose 注入到 s3 并且使用，然后使用 glue crawler 爬取表结构便于 athena 进行查询，另外可以使用 glue databrew 进行二次处理 ([[git/git-mkdocs/EKS/solutions/logging/stream-k8s-control-panel-logs-to-s3]])
- 转发eks控制平面日志到s3 ([[eks-logging-cost-compare]])

### Tracing-跟踪-
- 使用app mesh ([[aws-appmesh-RETIRED]])
- 使用x-ray ([[x-ray-lab]])
- 使用ADOT ([[adot-aws-distro-open-telemetry]])

<-->
