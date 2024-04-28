---
title: cert-manager
description: 证书管理插件
created: 2023-07-31 15:36:34.121
last_modified: 2024-04-02
tags:
  - kubernetes
  - aws/container/eks
---

# cert-manager
## install
### install with eksdemo
- https://github.com/awslabs/eksdemo/blob/main/docs/install-cert-manager.md
```sh
echo ${CLUSTER_NAME}
echo ${AWS_DEFAULT_REGION}
eksdemo install cert-manager -c ${CLUSTER_NAME}

kubectl get clusterissuer
# default name is letsencrypt-prod
```

### install with helm
- https://cert-manager.io/docs/installation/helm/

### install manually
- https://cert-manager.io/docs/installation/
- install newest version 
```sh
kubectl create ns cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml \
-n cert-manager
# CM_VERSION=v1.12.3 (2023/07)
# https://github.com/cert-manager/cert-manager/releases/download/${CM_VERSION}/cert-manager.yaml

```

## issuer-certificates-
```
TEST_DOMAIN=thanos-gateway.poc1109.aws.panlm.xyz
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${TEST_DOMAIN}
  namespace: monitoring
spec:
  secretName: thanos-gateway-tls
  dnsNames:
    - ${TEST_DOMAIN}
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
EOF

```


## newest version v1.12.3 (2024/04)
https://cert-manager.io/docs/releases/

![[../../git-attachment/cert-manager-png-1.png]]


