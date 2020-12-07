---
title: "Helm Tips"
date: 2020-12-06T13:49:40+09:00
draft: true
toc: true
tags:
  - k8s
  - helm
---
この記事は [FOLIO Advent Calendar 2020](https://adventar.org/calendars/5553) 8日目の記事です  

FOLIOでは現在k8sを利用した新しいアプリケーション基盤を検証しています  
その中でk8sマニフェストを管理する上で欠かせないツール [Helm](https://helm.sh/) についての知見を少しだけ共有しようと思います  

## Helmって？
https://helm.sh/  
k8s上で動くパッケージマネージャ  
パッケージマネージャとはあるinfraの上で動くアプリケーションの  
インストール/アンインストールの管理などを行ってくれるソフトウェアのことを指します  

k8sというinfraの上でhelmというパッケージマネージャーが扱う「アプリケーション」は  
マニフェストファイルの集合となります  

helmの世界ではパッケージはChartと呼ばれ、Chartリポジトリと呼ばれるhttp(s) serverにて管理されます  
標準では [stableリポジトリ](https://charts.helm.sh/stable/index.yaml) が用意されていますが  
ユーザ側で用意した独自リポジトリも使用することが可能です  

HelmのChart構造やtemplate構文については以下の記事を参照ください  
https://keisunagawa.github.io/blog/posts/helm-template/  

## Local Chart
helmはChartリポジトリからChartをインストールする他、localに存在するChartをディレクトリ指定で  
直接インストールすることも可能です  
この方法は検証段階や、privateなChartリポジトリを建てる環境が存在しない場合などに役に立ちます  

## Install Tips
helm Chartは `helm install <release名> <chart名>` コマンドによってk8sクラスタ上へインストールします  
またインストール済みのChartについては `helm upgrade` コマンドでversionを更新することができます  
`<release名>` が違う場合は同じChartを複数インストールすることも可能です、ただしこの場合別名Chart同士でresource名の  
コンフリクトが発生しないことを前提としています  
versionというのはChart自身のversionのことでdocker imageのversionなどはChartのversionもしくはinstall時に  
引き渡す `values.yaml` の値によって決定されます、つまりChartの内容次第ということになります  

`helm upgrade --install` とすることでinstall操作とupgrade操作を同時に行えます  
これは新規Chartの場合install、インストール済みChartの場合はupgradeを行うというコマンドになります  

Local Chartは `helm install <release名> <chart dir>` とすることでインストールすることができます  

以下install(upgrade)時によく使う引数をいくつかpickupします  

|flag|  |  
|--|--|  
|`--create-namespace`|`-n`で指定したnamespaceが存在しない場合自動的に作成します|  
|`--atomic`|何れかのマニフェスト適用に失敗した場合、install(upgrade)コマンド発行前の状態に戻します|  
|`--timeout`|install(upgrade)タイムアウトまでの時間を設定します</br>外部リソースと連動したk8s resourceを発行するChartなどはタイムアウト値を伸ばすことを推奨します|  
|`--wait`|Podリソースなどが稼働状態になるのまでwaitします</br>ただし、timeout時間までに稼働状態にならなかった場合はfailとなります|  
|`--values` `-f`|利用する`values.yaml`を引き渡します</br>複数引き渡すことも可能で、その場合同一キーは後勝ちとなります|  

## 依存Chart
Chartは他のChartへ依存することができ、そのChartをインストールした場合は依存Chartも一緒にインストールされます  
依存定義はChart.yamlに記載されます(古いChartであれば `requirements.yaml` というファイルに記載されていますが、こちらは現在 `deprecated` です)  
何も指定しない場合、依存Chartは常にインストールされますが、`condition` もしくは `tags` キーを利用することでユーザ側に利用の有無を移譲することができます  

依存chartの条件については少し複雑なので以下のソースを参照しながら説明します  
よければvalues.yamlの値を書き換え、実際の挙動を確認してください  
https://github.com/keiSunagawa/helm-chart-playground/tree/master/dependency-example  

### condition
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/Chart.yaml#L9  
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/values.yaml#L2  
```yaml
dependencies:
- name: sub-chart-1
  version: "0.1.0"
  repository: "file://../sub-chart-1"
  condition: subChart1.enabled
```
valuesの値がboolであるキーを指定することで依存の利用有無を制御できます  
純粋なbool値のみを利用できnotやor/and演算を利用することはできません  
キーに紐づく値がnullの場合、キーが存在しない場合はtrueとして判定されます  

また、カンマ区切りで複数の値を指定することができ、その場合は先頭から順に検査され  
**nullでない一番最初の値** が利用されます  
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/Chart.yaml#L18  
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/values.yaml#L7  
```yaml
condition: subChart3.enabled,subChart3.dev
```
この場合は `subChart3.enabled` がnullである場合は `subChart3.dev` が代わりに参照されます  
どちらのキーもnullだった場合はtrueとして判定されます  

### tags
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/Chart.yaml#L13  
https://github.com/keiSunagawa/helm-chart-playground/blob/master/dependency-example/main-chart/values.yaml#L4  
```yaml
- name: sub-chart-2
  version: "0.1.0"
  repository: "file://../sub-chart-2"
  tags:
  - subChart2
```
valuesの値がboolであるキーを配列で渡すことで `condition` より比較的複雑な条件分岐ができます  
`values.yaml` の `tags.xxx` のキーを参照し、対象キーがnullまたはtrueであった場合に依存Chartがインストールされます  
複数指定することもでき、複数指定した場合は以下の条件に従いインストールされます  
(公式ドキュメントが見当たらなかったため、`v3.1.2` で検証した結果を載せています)  
- インストールされる
  - 何れかのキーがtrueの場合
  - すべてのキーがnullの場合
- インストールされない
  - falseのキーが存在し、他のキーがnullの場合
  - 全てのキーがfalseの場合

## マニフェストの検証
`helm template` コマンドを利用することでk8sクラスタへChartをインストールすることなくマニフェストの検証をすることができます  
`-f` で本番環境で利用する `values.yaml` を引き渡せば本番適用時のものとほぼ同様のマニフェストを取得することができます  
`helm template` コマンドはtemplateが正しいyamlに展開されなかった場合にfailします  
この場合は `--debug` 引数を与えてあげることで不正なyamlを確認することができます  
出力は純粋なyamlになるので、[yq](https://github.com/mikefarah/yq) などを利用して検証を行うことができます  
```shell
# Kindの一覧を取得する
$ helm template . | yq r -d'*' --printMode pv - 'kind' | uniq
kind: ServiceAccount
kind: Service
kind: Deployment
kind: Pod
```

----

## 終わりに
helmはk8sを使う上で欠かせないツールです  
今回紹介した機能の他にもライフサイクルhookとなる `helm hook` や、インストールしたChartをロールバックする  
機能などが提供されており、単なるパッケージマネージャ以上の働きを期待することができます  

その他、[FOLIO Advent Calendar 2020](https://adventar.org/calendars/5553) 20日目にてArgoCDというk8sで動くCDツールの話を  
書く予定ですが、ArgoCDとhelmを連携させることでさらに強力なツールとなるので、よければぜひそちらの記事も読んでいただけると幸いです。  
