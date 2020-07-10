---
id: k8s-faq
title: K8S 常见问题
---

# K8S 常见问题
## K3S 常见问题
* [#396](https://github.com/rancher/k3s/issues/396) - Initializing eviction metric for zone

## Unable to connect to the server: x509: certificate is valid for 10.10.1.2, 10.43.0.1, 127.0.0.1, 192.168.1.10, not 123.123.123.123

签发的证书不包含 IP，初始化的时候可以添加，建议新建子账号管理。

```bash
# 原先的位于
# /etc/kubernetes/pki/apiserver.*
kubeadm init --apiserver-cert-extra-sans=123.123.123.123
```

## Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create a sandbox for pod "cron-1594372200-lmkcb": operation timeout: context deadline exceeded

测试是否正常

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: alpine-pwd
spec:
  template:
    spec:
      containers:
      - name: alpine
        image: wener/base
        command: ["pwd"]
      restartPolicy: Never
  backoffLimit: 4
```

一个主节点异常，重启解决。

## error: specifying a root certificates file with the insecure flag is not allowed

`insecure-skip-tls-verify` 不能用于根证书，新建子账号。

## node_lifecycle_controller.go:1433 Initializing eviction metric for zone
* 当有新的 zone 时会初始化 metric
* 参考
  * [node_lifecycle_controller.go:1433](https://github.com/kubernetes/kubernetes/blob/d01cc01ab4455d1c0db84d2cc79d963a1b15d292/pkg/controller/nodelifecycle/node_lifecycle_controller.go#L1429)
  * [多可用区实践](https://kubernetes.io/docs/setup/best-practices/multiple-zones/)

## running "VolumeBinding" filter plugin for pod "web-0": pod has unbound immediate PersistentVolumeClaims

