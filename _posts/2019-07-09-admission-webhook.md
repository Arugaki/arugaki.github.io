---
layout: post
title: Kubernetes Admission Webhook 实践
tags:
  - Kubernetes
  - Admission
---

Strimzi kafka operator 介绍及其部署

### Kubernetes Admission Webhook 实践

#### 简介

<https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/>

#### 实践

* 开启, 检查 apiserver 启动参数中是否有 

```
--enable-admission-plugins=ValidatingAdmissionWebhook,MutatingAdmissionWebhook
```

* 创建namespace

```
kubectl create ns admission
```

* 创建service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: admission-webhook
  name: admission-webhook
  namespace: admission
spec:
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    webhook: "true"
  sessionAffinity: None
  type: ClusterIP
```

* 创建 MutatingWebhookConfiguration, 其中 ca 为 kubernetes 的 rootCA, 从任一 serviceaccount 中获取即可

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: admission-webhook
webhooks:
- clientConfig:
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNU1EVXdOVEE0TVRRME5Gb1hEVEk1TURVd01qQTRNVFEwTkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS2d6CkN3MEtxZlY0bVdCaGVzY0RZOW1CMXVIS3M0SzdWQm1kOWhFdGRORkNSaEg0WVYxSHNtMEM4a2ZzUC9lcUp3SmMKRUNrODJjWlF4MDRsMkF3WGJ5NGw5Q2lkaW0yRzJBcldKdUxEd2YxeDNHU2RSclZBY1lGZlFyYnhOampaaXpydwowWCt6dDk4QW9pYU5mUUlEN3QxaTdobXZMeERFMDFuNDUxYTJ3bXdOK2pmc2QzMlRiS1pxSmJHYXVWdGtXb3JnCnVFd2hyeWF0ckliRFlnRktnTUR3S1o4dndNRGNDemdZb3lIMnprZzFydGEvTTF0Vjd0QjNWdlJyQWhTVmRLREkKdVFiVWp3MDZ1Z3ZMRUdySEl4bkxlSklPS2d1Nlpua2J6UnNKb3VXdWhrVFdYb2w0UEw1aVZsTlZWd09BTi9mUwpQa1dkY291ajY0RHFWYy94aXZNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFGVWpManFra2R1bW1EQ25zUU5XUnFEQ1ZGUkcKdTN5eTYwWjJDaHRRMDFKLzBmbUUxdHNqV29TT1M3Vm5zSUN4TytFT1BidURBSjB5bUZwVlBCRFpydmw0dEEwbgo0OHJDbzZpSFhWU2pYV1JSQW1MYnZNYlM4amVZRFhUYWQ2YXM2REF4RXp4SHJmSERyNEtudG8wRWI2ZU5oR0xBCjIyQVhmTURITGpabjQ1WFMyKytGVjRMMW1RQVRNSHp2a0RuaTdwdFhOZ3NiSGZJZExMc2tsTVVjWEJPODdOOTcKcTRUSjVjbkZCbW8zMnF2QzU0WDEvdlJ3L2F6QnRwa0R6QW1EUUZ1VmJGZU5aUTFCZXdUZUs2M1NsdDVTMENWego1Y2JLeVlDdGpDL3hSSGJtYi9uTStRSUkvWDBkcURkckI3SEh3dW1zbS9uY0o3dUN4dzRRbHpKam44cz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    service:
      name: admission-webhook
      namespace: admission
      path: /mutating-pods
  failurePolicy: Fail
  name: adding-hostport-label.k8s.io
  namespaceSelector: {}
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
```

* 生成证书, 需要 <a href="https://pkg.cfssl.org/"> 安装</a> cfssl 与 cfssljson

```
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "admission-webhook.admission.svc.cluster.local",
    "admission-webhook.admission.svc",
    "admission-webhook.admission",
    "admission-webhook",
    "admission-webhook ClusterIP"
  ],
  "CN": "admission-webhook.admission.svc.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```

* 创建 CertificateSigningRequest

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: admission-webhook.admission
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

* Approve CertificateSigningRequest <a href="https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/">详情</a>

```
kubectl get scr
kubectl certificate approve admission-webhook.admission
```

* 创建 secret 给 webhook 使用

```
kubectl create secret tls admission-secret --cert=server.crt --key=server-key.pem -n admission
```

* 创建 admission-webhook

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: admission-webhook
    webhook: "true"
  name: admission-webhook
  namespace: admission
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admission-webhook
      webhook: "true"
  template:
    metadata:
      labels:
        app: admission-webhook
        webhook: "true"
    spec:
      containers:
      - args:
        - --tls-cert-file=/webhook.local.config/certificates/tls.crt
        - --tls-private-key-file=/webhook.local.config/certificates/tls.key
        image: daocloud.io/daocloud/admission:test-v7
        imagePullPolicy: IfNotPresent
        name: admission-webhook
        volumeMounts:
        - mountPath: /webhook.local.config/certificates
          name: webhook-certs
          readOnly: true
      volumes:
      - name: webhook-certs
        secret:
          defaultMode: 420
          secretName: admission-secret
```

* 测试

```
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
  namespace: admission
spec:
  containers:
    - name: influxdb
      image: influxdb
      ports:
        - containerPort: 8086
          hostPort: 8086
```
