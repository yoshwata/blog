---
title: "Kustomize Patch"
date: 2021-11-23T01:16:54Z
draft: false
ShowToc: true
TocOpen: true
---

### 概要

kustomizeを利用するとベースのyamlからパッチをして、「ほとんどの部分は同じだが一箇所だけリソースが異なる」といったようなリソースを作成することができます。差分だけ記述すれば済むため、yamlの定義ははシンプルで管理しやすくなります。環境ごとに多数のリソースを扱うといった場合は特に有効です。

### 1 基本

kustomizeで`namespace`や`name`のプリフィックス、ポストフィックス、`label`、`annotation`をつけることは簡単です。以下のファイルを用意します。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

`kustomization.yaml`
```yaml
resources:
- deployment.yaml
namespace: myns
namePrefix: pre-
nameSuffix: -suf
commonLabels:
  label1: lab1
  label2: lab2
commonAnnotations:
  annotation1: ann1
  annotation2: ann2
```

kustomizeを実行します。

```yaml
$ kubectl kustomize ./
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    annotation1: ann1
    annotation2: ann2
  labels:
    label1: lab1
    label2: lab2
  name: pre-my-nginx-suf
  namespace: myns
spec:
  replicas: 2
  selector:
    matchLabels:
      label1: lab1
      label2: lab2
      run: my-nginx
  template:
    metadata:
      annotations:
        annotation1: ann1
        annotation2: ann2
      labels:
        label1: lab1
        label2: lab2
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
```

`kustomization.yaml`の指定通りに`name`や`namespace`が挿入されていることが分かります。`commonAnnotation`によってDeploymentの`metadata`はもちろんですが`podTemplate`のmetadataにも挿入されます。`commonLabels`についても同様ですが、`selector`にも挿入される点が`commonAnnotation`とはことなります。

### 2 configMapGenerator

#### 2-1 基本

`configMapGenerator`を利用すると`files`に指定したファイルから簡単に`configMap`を生成することができます。

`foo.propaerties`
```sh
FOO=foo
```

`hoge.propaerties`
```sh
HOGE=hoge
```

`kustomization.yaml`
```yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - hoge.properties
  - foo.properties
```

この構成でコマンドを実行します。

```yaml
$ kubectl kustomize ./
apiVersion: v1
data:
  foo.properties: |
    FOO=foo
  hoge.properties: |
    HOGE=hoge
kind: ConfigMap
metadata:
  name: example-configmap-1-gfc789ggc8
```

各`properties`ファイルに記載された内容が`configMap`の`data`として登録されていることが分かります。`name`には独特なhash値がサフィックスとして付与されていますが、この値はdataの値によって変わり、ランダム値というわけではありません。

#### 2-2 Deploymentへの適用

`configMapGenerator`の`name`でdeploymentから参照することで、deploymentからcofigmapを使用させることができます。

以下の`deployment.yaml`を用意します。`kustomization.yaml`と`*.property`ファイルは2-1と同様のため省略。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: example-configmap-1
```

この状態でkustomizeします。

```yaml
$ kubectl kustomize ./
apiVersion: v1
data:
  foo.properties: |
    FOO=foo
  hoge.properties: |
    HOGE=hoge
kind: ConfigMap
metadata:
  name: example-configmap-1-gfc789ggc8
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: my-app
        name: app
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: example-configmap-1-gfc789ggc8
        name: config
```

configMapが生成されており、それがdeploymentの方で参照されているのがわかります。

### 3 secretGenerator

`configMapGenerator`に似ていますが、secretsも`secretGenerator`を利用することで生成することができます。

password.txt
```
username=admin
password=secret
```

kustomization.yaml
```yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
```

この状態でkustomizeします。

```yaml
$ kubectl kustomize ./
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  name: example-secret-1-2kdd8ckcc7
type: Opaque
```

`configMapGenerator`同様`data`の値によって`name`にサフィックスが付きます。secretの場合は`data`の要素がbase64エンコードされています。

deploymentから参照可能な点も`configMapGenerator`同様です。

### 4 generatorOptions

`generatorOptions`を利用すると`configMapGenerator`や`secretGenerator`で生成するリソースに共通の設定をまとめて定義することが可能です。

`kustomization.yaml`
```yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
- name: example-configmap-4
  literals:
  - FOO=Bar
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
```

password.txtは`secretGenerator`で定義したものと同様のため省略します。この状態でkustomizeします。

```yaml
$ kubectl kustomize ./
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-3
---
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-4
---
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-secret-1
type: Opaque
```

`disableNameSuffixHash`によって各リソースの`name`のサフィックスが無効化されています。またconfigmap/secretに共通の`labels`/`annotations`定義が付与されています。

### 5 patchesStrategicMerge

以下のようなyamlを準備します。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

`increase_replicas.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
```

`set_memory.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
          limits:
            memory: 512Mi
```

`kustomization.yaml`
```yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
```

kustomizeを実行します。
```yaml
kubectl kustomize ./
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 512Mi
```

`memory`の`limits`が`set_memory.yaml`から、レプリカ数が`increase_replicas.yaml`から参照されていることがわかります。

### 6 patchesJson6902

`patchesStrategicMerge`はすべてのリソース、フィールドをサポートしているわけではありません。任意のフィールドの変更を行う場合には`patchesJson6902`を利用します。`replace`/`add`/`remove`/`move`/`copy`といった操作が可能です。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      moved: target
      copied: target
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
        env:
          - name: FOO
            value: replaced
          - name: BAR
            value: removed
```

`kustomization.yaml`
```yaml
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
```

`patch.yaml`
```yaml
- op: replace
  path: /spec/replicas
  value: 3
- op: replace
  path: /spec/template/spec/containers/0/env/0/value
  value: "var"
- op: add
  path: /spec/template/spec/containers/0/hoge
  value: "fuga"
- op: remove
  path: /spec/template/spec/containers/0/env/1
- op: move
  from: /spec/template/spec/moved
  path: /spec/template/spec/containers/0/moved
- op: copy
  from: /spec/template/spec/copied
  path: /spec/template/spec/containers/0/copied
```

上記yamlを準備しkustomizeします。
```yaml
$ kubectl kustomize ./
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - copied: target
        env:
        - name: FOO
          value: var
        hoge: fuga
        image: nginx
        moved: target
        name: my-nginx
        ports:
        - containerPort: 80
      copied: target
```

`patchesStrategicMerge`で行ったレプリカ数の変更(`replace`)が`patchesJson6902`の場合でも行えていることが分かります。

要素追加(`add`)を行いたい場合には`path`に任意のパスと`value`を設定します、出力結果に`hoge`要素が追加されていることから効果が分かります。

要素削除(`remove`)の場合には`path`のみ指定します。元の`deployment.yaml`から`env`の1番目の要素が消えていることが分かります。

要素移動(`move`)する場合には`from`と`path`要素により行います。もともと`spec`直下にあった`moved`要素が`coutainers[0]`へ移動していることが分かります。

要素複製(`copy`)も`move`と同様`from`と`path`を指定します。`copy`の場合には`spec`直下の値はそのまま残っている点が`move`と異なります。

参考までにJson6902に仕様は以下となります。
https://datatracker.ietf.org/doc/html/rfc6902

### 7 vars

varsは、yaml中の対象の値で指定の文字列を置き換えることができます。

以下のファイルを準備します。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "$(MY_SERVICE_NAME)"]
```

`kustomization.yaml`
```yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

vars:
- name: MY_SERVICE_NAME
  objref:
    kind: Service
    name: my-nginx
    apiVersion: v1
```

`service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

以上のファイルを用意しkustomizeします。

```yaml
$ kubectl kustomize ./
apiVersion: v1
kind: Service
metadata:
  labels:
    run: my-nginx
  name: dev-my-nginx-001
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001
        image: nginx
        name: my-nginx
```

`kustomization.yaml`で定義したとおり、ServiceとDeployment両方の`name`にプリフィックスとサフィックスが付与されています。また注目すべきはDeploymentの`command`です。kustomize前には`$(MY_SERVICE_NAME)`となっていた部分がServiceの`name`と同じ値で置き換えられています。

### 8 configurations

configurationsを利用すると任意のリソースに対して`kustomization.yaml`で定義した変換ルールを適用できます。以下のyamlを準備します。

`resources.yaml`
```yaml
apiVersion: animal/v1
kind: Gorilla
metadata:
  name: gg
---
apiVersion: animal/v1
kind: AnimalPark
metadata:
  name: ap
spec:
  gorillaRef:
    name: gg
    kind: Gorilla
    apiVersion: animal/v1
```

`nameReference.yaml`
```yaml
nameReference:
  - kind: Gorilla
    fieldSpecs:
      - kind: AnimalPark
        path: spec/gorillaRef/name
```

`kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - resources.yaml

namePrefix: sample-

configurations:
  - nameReference.yaml
```

この状態でkustomizeすると以下のようになります。

```yaml
$ kubectl kustomize ./
apiVersion: animal/v1
kind: AnimalPark
metadata:
  name: sample-ap
spec:
  gorillaRef:
    apiVersion: animal/v1
    kind: Gorilla
    name: sample-gg
---
apiVersion: animal/v1
kind: Gorilla
metadata:
  name: sample-gg
```

出力結果の`metadata.name`については`kustomization.yaml`の`namePrefix`で定義したとおりのプリフィックスが付与されています。しかし、`spec.gorillaRef.name`については`nameReference.yaml`で対象としているためにプリフィックスが付与されています。`kustomization.yaml`と同じ変換ルールを異なる要素にも適用したい場合に有用です。

### 9 images

対象のリソースのimageをパッチする場合には`images`を使用します。以下のyamlを準備します。

`deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

`kustomization.yaml`
```yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: alpine
  newTag: "3.6"
```

この状態でkustomizeすると以下のようになります。

```yaml
$ kubectl kustomize ./
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: alpine:3.6
        name: my-nginx
        ports:
        - containerPort: 80
```

`kustomization.yaml`の通りimageがパッチされています。`images`は配列で指定可能なため、複数のコンテナの`image`をパッチすることも可能です。`images`にはあくまで`image`の`name`を入れるようにしてください。リソースの`name`ではありません。
