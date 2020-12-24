---
title: "k8s 1.20の更新後のEKSのコンテナランタイムについて"
date: 2020-12-24T14:55:09+09:00
draft: false
toc: true
tags:
  - k8s
  - docker
  - containerd
---
## 背景
k8s 1.20よりdocker用のCRIであるdockershimがサポートされなくなるので  
それに伴いeksではなにをすればいいのか、コンテナランタイムの代替えとしては何を利用するのかを調べた  

## コンテナランタイム
文字通りコンテナを実行するモジュール  
現在の選択肢としてはdocker, containerd, CRI-Oなどがある  
k8sではコンテナランタイムに対してCRIという規約に従ったgrpcサーバを要求する  

## CRI(ContainerRuntimeInterface)
k8sがコンテナランタイムに要求するインタフェースの規約  
コンテナランタイムはこれに則ったgrpcサーバを用意する必要がある  
k8s自体元々(1.12より前)は利用できるコンテナランタイムがdockerのみだったが  
他のランタイムも利用できるようにするためにこの規約を定めた  

が、当然docker自身はこの規約に則ったインタフェースは持っていないため  
docker用のCRIとして**k8s側が**dockershimというものを用意していた  

## k8s 1.20
[Dockershimが非推奨になった](https://github.com/kubernetes/kubernetes/pull/94624)  
以後Dockershimのメンテがされなくなるので他のコンテナランタイムを利用してね、と公式からのアナウンスがある  
containerdやCRI-OはCRIがネイティブサポートされている  

## on EKS
今回の本題  
EKSではどう対応されるのかを調べる  
2020/12/24時点  
[AmazonLinux最適化AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)ではコンテナランタイムは引き続きdockerだった  

```sh
$ kube get nodes -o wide
NAME                                               STATUS   ROLES    AGE    VERSION              INTERNAL-IP     EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-xxx-xxx-xxx-xxx.ap-northeast-1.compute.internal    Ready    <none>   114m   v1.18.9-eks-d1db3c   xxx.xxx.xxx.xxx    <none>        Amazon Linux 2   4.14.209-160.335.amzn2.x86_64   docker://19.3.6
```

[Bottlerocket最適化AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami-bottlerocket.html)の場合containerdらしいけどこちらは  
cluster managed node groupを利用している場合は現在選択できない  

Fargateの場合はcontainerdが採用されているようだ  
```sh
$ kube get nodes -o wide
NAME                                                      STATUS     ROLES    AGE    VERSION              INTERNAL-IP     EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
fargate-ip-xxx-xxx-xxx-xxx.ap-northeast-1.compute.internal   Ready      <none>   28s    v1.18.8-eks-7c9bda   xxx.xxx.xxx.xxx    <none>        Amazon Linux 2   4.14.209-160.335.amzn2.x86_64   containerd://1.3.2
```

現時点でのAmazonLinux最適化AMIでのcontainerdに関する声明はとくにないが  
eksでのlatest k8s versionが 1.18系であることから焦って対応する必要がなく、様子見でよさそうだ
