---
title: "Helm Template Tips"
date: 2020-12-06T17:31:55+09:00
draft: false
toc: true
tags:
  - k8s
  - helm
---

この記事は過去に [Qiita](https://qiita.com/keiSunagawa/items/db0db26579d918c81457) で書いたものの転載です  

## Helm  
https://helm.sh/  
Helmはk8sにおけるパッケージマネージャーのようなもです  
k8sのService, Deployment, ConfigMap, etcをひとつのまとまり(Chartと呼ばれる)として構成し、それらのk8sクラスタへのinstall/update/deleteなどの責務を担当してくれます  
公式や誰かの作ったChartをinstallするほか、自分たちの作ってるアプリケーションのChartを作成することでマニフェストファイルの共通化やdeploy/rollbackを簡単にすることができます  

当記事では後者の用途を前提として進めます  

Helmの基礎を理解頂いてる方は [Template](#template) までスキップして頂けると、今回の本題はそちらですので  

## Init
Helmのインストール手順は公式を参照、以下はHelm/k8sをinstallしている前提で進めます  

### Chartのスケルトン生成

```sh
$ helm create first-app
Creating first-app
$ tree first-app/
first-app/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 8 files
```

`helm create <chart name>` でChartのスケルトンを生成できます  

### 生成されるファイル/ディレクトリ
- Chart.yaml  
Chartの概要、対象のk8s API versionなどが記載されている  

- charts  
Subchart(自身のChartが依存するChart)が含まれるディレクトリ  
Subchartについての説明はこの記事では行いません(自分もまだ使ったことのない機能なので)  

- templates  
今回の本題、マニフェストファイルのTemplateが格納されているディレクトリ  
マニフェストファイルについての詳しい説明は後述  

- values.yaml  
Templateで使える値のyaml  
このファイルではデフォルト値を定義しておき、install/update時に環境に応じたファイルを渡すことでここで定義した値を差し替えることができる  

### デバッグ  
`$ helm install . --debug --dry-run`  
とすることで作成中のTemplateに変数を展開したマニフェストファイルをみることができます  

## Template
Chartを構成するk8sマニフェストはTemplateと呼ばれるファイルによって生成されます  
Templateはその名の通りマニフェストファイルのテンプレートで変数を埋め込んだり、関数を呼び出すことで環境に応じて変更される部分を抽象化することができます  
この項目ではTemplateで使える様々な構文について説明します  

### 基本的な構文  
Templateにおいて何れの構文も `{{ }}` に囲まれて表記されます  
`{{ }}` 以外の部分はただのk8sマニフェストです、つまりただのyamlです  
(ちなみに [Go言語のTemplate機能](https://golang.org/pkg/text/template/) から派生しているようです)  
中括弧内側の前後にダッシュ `{{- -}}` をつけることができ、前後の改行を含めた空白文字をトリムします  


```yaml
hoge:
  {{- $piyo := "aaa" }}
  - "fuga"

# ↓↓展開されると↓↓

hoge:
  - "fuga"
```

### コメント
`/* */` で囲まれた部分はコメント構文になります  
通常のプログラミング言語のコメントと同様で展開時には無視されます  

```
{{- /* a comment */ -}}
```

### 変数
変数には二種類あり、ルートオブジェクトからpathを辿って呼び出されるものとローカル変数の参照です  
「ルートオブジェクト」「ローカル変数」については勝手に名前をつけたものなので公式ではありません  
変数はそのまま参照先の値が展開されます  

#### ルートオブジェクト
Templateで呼び出せる様々な事前定義オブジェクトやvalues.yamlに定義された値を取り出すための `Values` オブジェクトへの参照を持つrootです  
後述する `_helpers.tpl` 内で定義できる名前付きtemplateや、range構文では呼び出し時に渡した値がルートオブジェクトとなります  
`{{ . }}` がルートオブジェクトそのものを表す記法となり、ここからpathを記載していきます  

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.myapp.serviceName }}
---
# values.yaml
myapp:
  serviceName: "myservice"
```
Valuesの他にもルートオブジェクトには様々な事前定義オブジェクトがあり以下のドキュメントが一覧となります  
https://helm.sh/docs/chart_template_guide/#built-in-objects  

#### ローカル変数
各Templateファイル内で変数を定義することができます  

```yaml
  {{- $volumeName := "myvolume" -}}
    volumeMounts:
      - mountPath: /etc/{{ $volumeName }}/
        name: {{ $volumeName }}
    volumes:
      - name: {{ $volumeName }}
```
見ての通りだとは思いますが `{{- $volumeName := "myvolume" -}}` で変数に束縛し `{{ $volumeName }}` で展開できます  

### 関数
Template構文内では事前定義された様々な関数を使うことができます  
残念ながら定義の一覧のドキュメントは存在しないようです  
ただし、Helm側で定義されている関数の他、以下の関数の多くが実装されているそうです  
http://masterminds.github.io/sprig/  
また、事前定義オブジェクトの中にも定義されている関数があります( `.Files.get` など)  
関数の呼び出しは `関数名 引数1 [引数2 ...]` という形になります  

関数はパイプ(`|`) を使うことでチェインすることができます  

```
{{ .Files.get "files/main.cf" | nindent 4}}
```
以下よく使う関数を抜粋して紹介します  
紹介の表記は `関数名 <引数1の型> [<引数2の型> ...]` という形にします  
リファレンスがないので型といっても数値か文字列かオブジェクトとか、といった程度になってしまいますが…  

- `nindent <numer> <string>`  
第2引数に渡した文字列の各行の先頭に第1引数の数値分だけ半角スペース(つまりインデント)を埋め込みます  
yaml等の文字列を展開する場合にこの関数でインデントを調整することができます  

- `toYaml`  
オブジェクトをyaml形式で展開してくれます  
values.yamlに記載されているyamlを直接マニフェストファイルに展開したい場合などにこの関数を使います  

```yaml
# values.yaml
test:
  hoge:
    fuga: piyo

# deployment.yaml
expand:
  {{- nindent 2 (toYaml .Values.test) }}

# ↓↓展開されると↓↓
expand:
  hoge:
    fuga: piyo
```

- `include <string> <object>`  
`_helpers.tpl` に定義した名前付きtemplateを展開します  
第2引数のオブジェクトは名前付きtemplate側でのルートオブジェクトとして扱われ、基本的にはルートオブジェクトをそのまま渡すパターン `{{ inculude "fugafuga" . }}` が多いです  
以下のサンプルではあえてルートオブジェクト以外のものを渡してみます  

```yaml
# values.yaml
test:
  hoge:
    fuga: piyo

# _helpers.tpl
{{- define "fugafuga" -}}
- {{ toYaml . -}}
- {{ toYaml . -}}
{{- end }}

# deployment.yaml
expand:
  {{- include "fugafuga" .Values.test.hoge | nindent 2 }}

# ↓↓展開されると↓↓
expand:
  - fuga: piyo
  - fuga: piyo
```

- `tpl <string> <object>`  
第1引数の文字列をTemplateとして解釈し、第2引数をルートオブジェクトとして展開します  
`_helpers.tpl` に定義しているものは `include` で展開すればTemplateとして解釈されるので  
こちらは外部ファイルを取ってきたあとに適用する場合が多いです  
Template構文を展開するだけなのでyaml以外の文字列でも適用は可能です  

```yaml
# values.yaml
nginxValues:
  nginxUser: "nginx"

# files/nginx.conf
user {{ .nginxUser }};
http{
    # いろいろ
}

# configmap.yaml
data:
  nginx.conf: |
{{ tpl (.Files.Get "files/nginx.conf") .Values.nginxValues | nindent 4 }}

# ↓↓展開されると↓↓
data:
  nginx.conf: |
    user nginx;
    http{
        # いろいろ
    }
```

- `.Files.Get <string>`  
外部からファイルを文字列として取り込みます  
第1引数はファイルパスでChartのルートディレクトリ( `Chart.yaml` の置いてあるディレクトリ)からの相対パスを記載します  

## アクション
関数のほか、Templateを制御するためいくつかのキーワードが用意されています  
一般的なプログラミング言語でいう `if` や `for` などにあたります、これをHelmではアクションと呼びます  

### if/else
特に語ることのない一般的なif文です  
値が以下の場合にはすべて `false` として判定されます  
https://helm.sh/docs/chart_template_guide/#if-else  

```
A pipeline is evaluated as false if the value is:

a boolean false
a numeric zero
an empty string
a nil (empty or null)
an empty collection (map, slice, tuple, dict, array)
```

注意点としてダッシュ付きの中括弧を使用することを推奨するくらいです  

```
spec:
  replicas: {{ if .Values.safety -}}
2
{{- else -}}
1
{{- end -}}
```

### with
引数に渡したオブジェクトをルートオブジェクトとしたスコープを展開できます  
引数に渡したオブジェクトが `if/else` アクションで紹介したfalseにあたる値の場合はwithブロック自体が展開されません、つまり簡易的な `if` としても使用できます  
withブロック内で `.Files` オブジェクトなどの事前定義オブジェクトを参照する場合は外側で変数に束縛しましょう、このテクニックは `range` アクションでも使用できます  

```yaml
# values.yaml
configPaths:
  nginxConfPath: "files/nginx.conf"

# configmap.yaml
{{- $files = .Files -}}

data:
{{- with .Values.configPaths -}}
  nginx.conf: |
{{ $files.get .nginxConfPath | nindent 4 }}
{{- end -}}
```

### range
引数に渡した配列でループを回します  
ループ内のスコープでは配列の値自体がルートオブジェクトとなります  

```yaml
# values.yaml
configPaths:
  - key: nginx.conf
    path: "files/nginx.conf"
  - key: postfix.main.cf
    path: "files/main.cf"

# comfigmap.yaml
{{- $files = .Files -}}

data:
{{- range .Values.configPaths -}}
  {{ .key }}: |
{{ $files.get .path | nindent 4 }}
{{- end -}}

# ↓↓展開されると↓↓
data:
  nginx.conf: |
    # nginx.confの内容
  main.cf: |
    # main.cfの内容
```

### template
`_helpers.tpl` に定義してある名前付きtemplateを展開します  
ただし、include関数と違いルートオブジェクトを渡すことができません  

## _helpers.tpl
Chart内で共通で使うことのできるテンプレートスニペットを定義します、これは「名前付きテンプレート」と呼ばれます  
`values.yaml` が差し替え可能な変数、ローカル変数が定義したTemplateファイル内でのみ使える変数、_helpers.tplはチャート内で自由に使える変数、という形に捉えることができます  

`define` アクションを使うことで名前付きテンプレートを定義することができます  
もちろん、テンプレートなので定義内で変数を展開したり、関数やアクションを使用することもできます  
(ただし、呼び出し側で想定通りのオブジェクトを渡す必要があります、templateアクションではなくincluede関数を使う必要もあります)  

```
{{- define "myapp.namespace" -}}
{{ .Chart.Name }}-space
{{- end }}
```

---

Helm Template構文における基本的な構文は以上となります  
ひとまずこれだけ覚えていれば問題なくHelm Templateを読み書きできると思います  
少し特殊なものが多いのでGo Templateを触れていない(自分もそうだったのですが)人間から見るとなかなかつらいもの、特に関数周りは公式ドキュメントに記載や一覧がないのでとっつきにくいと感じたので今回の記事を書かせて頂きました  
この記事が誰かの役に立つことを願います  
