---
title: istioctl installが失敗する
tags:
  - kubernetes
  - Calico
  - istio
  - istioctl
private: false
updated_at: '2022-08-01T11:22:54+09:00'
id: fd8edd3a41ac2908fc2f
organization_url_name: null
slide: false
ignorePublish: false
---
[Calico公式の手順](https://projectcalico.docs.tigera.io/security/app-layer-policy)に従って、Istioをインストールしようとしたら、`Install Istio`の手順で次のエラーメッセージが出て困った話。
```bash
$ istioctl install -y
✔ Istio core installed
✘ Istiod encountered an error: failed to wait for resource: resources not ready after 5m0s: timed out waiting for the condition
```

## 環境
- Debian 11.4
- Kubernetes 1.21-13-00
- Calico 3.23.3
- Istio 1.13.2
- シングルノード環境

## 解決方法
マスターノードへのtaintを外す必要がありました。
次のコマンドでマスターノードへのtaintを外します。
```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```
taintとは汚れの意味で、nodeに紐づきます。
nodeにtaintが付いているとPodが起動できないことがあり、マスターノードにはデフォルトでtaintが付いています。
これによって、`csi-node-driver`という名前のPodが起動せず、`istioctl install`が失敗するようでした。
[シングルノード環境でのCalicoインストール手順(公式)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)にもtaintを外すように書いてありますが、見落としていました。

## 参考
[シングルノード環境でのCalicoインストール手順(公式)](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)
[timed out waiting for the condition Deployment/istio-system/istiod](https://github.com/istio/istio/issues/22677)
[TaintとToleration](https://kubernetes.io/ja/docs/concepts/scheduling-eviction/taint-and-toleration/)

