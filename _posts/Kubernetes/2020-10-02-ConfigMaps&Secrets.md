---
layout:     post
title:      Chapter 10. ConfigMaps&Secrets
subtitle:   configuring applications
date:       2020-10-02
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

这一章我们学习当在 Kubernetes 中运行应用时，如何给它传递配置数据。

### 配置容器化的应用

在正式介绍如何向 Kubernetes 中运行的应用传递配置数据之前，我们先看看容器化的应用是如何配置的。

除了直接把配置硬编码到代码中，一般是通过 *命令行参数* 来配置应用。随着配置选项逐渐增多，可以把参数写到一个 *配置文件* 中。

还有一种传递配置选项的方式是通过 *环境变量*。应用不读取配置文件和命令行参数，而是查询一个特定的环境变量的值。

环境变量的方式比命令行参数的方式更受欢迎。一是因为在 Docker 容器中使用配置文件有些麻烦，要么是通过把配置文件写到容器镜像中（类似于硬编码，每次修改都要 rebuild image），要么是通过挂载包含配置文件的 Volume（容器启动之前要先写到Volume）。二是因为每个访问镜像的人都可以看到配置，一个保密的内容可能会暴露。

配置容器化应用一般有这三种方式：

+ 传递命令行参数；
+ 通过 Volume 挂载配置文件到容器中；
+ 设置环境变量；

接下来我们先介绍这几种方式。

### 向容器传递命令行参数

到目前为止，我们创建的容器都是运行镜像中定义的默认命令。但是 Kubernetes 也允许我们在 Pod 的容器定义时覆盖命令。接下来看看如何实现。

#### Docker中定义命令和参数

首先我们解释一下，在容器中执行的命令是有两部分组成：*command* 和 *arguments*。在一个 Dockfile 中，两个指令定义这两部分：

+ *ENTRYPOINT*：定义了容器启动时执行的命令；
+ *CMD*：制定了传递给 *ENTRYPOINT* 的参数；

即便你也可以使用 *CMD* 来指定容器启动时执行的命令，但还是建议通过 *ENTRYPOINT* 来指定命令，用 *CMD* 指定默认的参数。因为这样的话就可以直接不指定参数运行镜像：

```bash
$ docker run <image>
```

或者指定额外的参数，来覆盖在 Dockerfile 中 *CMD* 设置的参数：

```bash
$ docker run <image> <arguments>
```

另外，*ENTRYPOINT* 执行的指令支持两种格式：

+ shell 格式：例如，ENTRYPOINT node app.js
+ exec 格式：例如，ENTRYPOINT ["node", "app.js"]

两者的区别是指定的命令是否在 shell 中被调用执行。在 Chapter 3 的 Dockfile 中我们使用了 *exec* 格式，这样直接运行进程，不是在 shell 中。

```
ENTRYPOINT ["java", "-jar","app.jar"]
```

我们来修改一个脚本和镜像，让它循环的时间间隔可以配置。我们添加一个 *INTERVAL* 变量并用第一个命令行参数初始它，如下所示。

```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1   # INTERVAL用第一个参数初始化
echo Configured to generate new fortune every $INTERVAL seconds    # echo打印信息
while :
do
    echo $(date) Hit
    sleep $INTERVAL
done
```

我们已经修改了这个脚本。现在我们要修改 Dockfile 使得它用 *exec* 格式的指令，并使用 *CMD* 把 *INTERVAL* 初始化为 10。如下所示。

```bash
FROM ubuntu
COPY ./loop.sh bin/loop.sh 
ENTRYPOINT ["bin/loop.sh"]  # exec格式
CMD ["10"]  # 默认参数为10
```

现在我们构建镜像，并上传到 DockerHub。

```
$ docker build -t zhaodockerhub/loop .
$ docker tag zhaodockerhub/loop zhaodockerhub/loop:args
$ docker push zhaodockerhub/loop:args
```

现在我们运行测试。

```bash
$ docker run -it zhaodockerhub/loop:args  # 默认参数
Configured to generate new fortune every 10 seconds
Wed Mar 17 11:45:23 UTC 2021 Hit
Wed Mar 17 11:45:33 UTC 2021 Hit
Wed Mar 17 11:45:43 UTC 2021 Hit

(press Ctrl+C)

$ docker run -it zhaodockerhub/loop:args 5  # 覆盖CMD参数
Configured to generate new fortune every 5 seconds
Wed Mar 17 11:46:02 UTC 2021 Hit
Wed Mar 17 11:46:07 UTC 2021 Hit
Wed Mar 17 11:46:12 UTC 2021 Hit
```

#### Kubernetes中覆盖命令和参数

在 Kubernetes 中，当我们描述一个 container 时，我们可以选择覆盖 *ENTRYPOINT* 和 *CMD*。如下所示，我们通过设置描述文件中 container 的 *command* 和 *args* 属性。

```
kind: Pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

绝大多数情况下我们都只是自定义参数，一般不轻易修改命令。

为了运行有自定义时间间隔的 Pod，我们创建以下 YAML 文件。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortuneloop
spec:
  containers:
  - image: zhaodockerhub/loop:args
    args: ["3"]  # 添加了 args 数组自定义参数
    name: fortune 
```

如果我们有多个自定义参数，则可以按照以下格式。其中字符串无需用双引号，但数字需要用双引号。

```
args:
- foo
- bar
- "15"
```

用 *kubectl create* 命令创建上述 Pod 后，等 Pod 为运行状态时，可查看它的 log。可以看到按照 Pod 中设置的每隔 3 秒打印。

```
$kubectl logs fortuneloop
Configured to generate new fortune every 3 seconds
Wed Mar 17 11:48:04 UTC 2021 Hit
Wed Mar 17 11:48:07 UTC 2021 Hit
Wed Mar 17 11:48:10 UTC 2021 Hit
Wed Mar 17 11:48:13 UTC 2021 Hit
Wed Mar 17 11:48:16 UTC 2021 Hit
Wed Mar 17 11:48:19 UTC 2021 Hit
```

### 为容器设置环境变量

Kubernetes 也允许我们为一个 Pod 的每个容器指定一个环境变量的列表。如下图所示。

![img](/img/post/post_env.png)

和容器的命令和参数一样，一旦容器被创建，环境变量就不能修改了。

我们看看如何修改上一节的 loop.sh 文件让它可以从环境变量中配置。

```bash
#!/bin/bash
trap "exit" SIGINT   # 把INTERVAL变量初始化一行删掉
echo Configured to generate new fortune every $INTERVAL seconds    # echo打印信息
while :
do
    echo $(date) Hit
    sleep $INTERVAL
done
```

我们只需要把变量 INTERVAL 初始化的一行删除。因为这是 bash 脚本，我们不需要做其他的。如果是 Java 代码，我们要使用 *System.getenv("INTERVAL")*。

#### 在容器的定义中指定环境变量

在构建上面的镜像并 push 到 DockerHub 后，我们创建一个新 Pod 来运行它。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortuneloop
spec:
  containers:
  - image: zhaodockerhub/loop:env
    env:
    - name: INTERVAL  # 添加环境变量
      value: "3"
    name: fortune 
```

#### 一个环境变量引用其他环境变量的值

我们也可以通过 *$(var)* 语法来引用一个已经存在的环境变量的值。如下所示。

```yml
env:
  - name: FIRST_VAR
    value: "foo"
  - name: SECOND_VAR
    value: "$(FIRST_VAR)bar"   # 引用已经存在的环境变量的值
```

在上面描述中，第二个环境变量的值就变为 "foobar"。

以上介绍的 *命令行参数* 和 *环境变量* 方式，都把配置数据直接合并在 Pod 的描述文件中。这种硬编码的方式不适合配置数据变化的情况。接下来，我们学习如何将这两者解耦。

### Decoupling configuration with a ConfigMap

App 配置的核心要点就是，配置数据要和源代码分离，以适应运行环境频繁变化的场景。在 Kubernetes 中，我们将 Pod 的描述作为源代码的话，那很显然配置数据就要移到 Pod 的描述文件之外。

#### Introducing ConfigMaps

Kubernetes 提供一个 ConfigMap 对象来独立配置数据。ConfigMap 包含一系列表示配置数据的 key/value 对。应用并不是直接读取 ConfigMap，而是 ConfigMap 的内容通过环境变量的方式或者 Volume 文件的方式传递到容器中。如下图所示。

![img](/img/post/post_configmap1.png)

不管容器以哪种方式消费 ConfigMap，ConfigMap 单独保存配置数据可以使我们一相同的名字配置多个 ConfigMap，每一个对应于一种环境（test, dev, prod）。因为 Pod 是一名字引用 ConfigMap，所以你可以在每种环境中使用不同的配置，但可以使用相同的 Pod 文件。

![img](/img/post/post_configmap2.png)

#### Creating ConfigMap

我们先创建一个 ConfigMap，可以直接使用 *kubectl create configmap* 命令创建。并且，可以直接从字段中创建，也可以从文件中创建，甚至从整个目录创建。

```bash
$ kubectl create configmap fortune-config --from-literal=interval=25   
# 从字段创建名为fortune-config的ConfigMap，包含字段interval:25 

$ kubectl create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two
# 从多个字段创建名外myconfigmap的ConfigMap，包含字段bar:baz和one:two
```

注意：ConfigMap 的 key 只能包含字母、数字、连接线、下划线、点。

当然，你也可以写如下的 YAML 文件，然后用 *kubectl create -f* 命令创建。

```yml
apiVersion: v1
kind: ConfigMap 
metadata:
  name: fortune-config
data:
  sleep-interval: "25" 
```

```bash
$ kubectl create configmap my-config --from-file=config-file.conf
# 从文件创建名为my-config的ConfigMap，key为文件名，value是文件内容

$ kubectl create configmap my-config --from-file=customkey=config-file.conf
#从文件创建名为my-config的ConfigMap，key为指定的customekey，value是文件内容

$ kubectl create configmap my-config --from-file=/path/to/dir
# 从目录创建名为my-config的ConfigMap，key为每个文件名，value为对应文件的内容
```

当创建 ConfigMap 时，也可以使用以上命令的组合形式。如下所示。

```bash
$ kubectl create configmap my-config 
\ --from-file=foo.json          # 从文件创建
\ --from-file=bar=foobar.conf   # 从指定key的文件创建
\ --from-file=config-opts/      # 从目录创建
\ --from-literal=some=thing     # 从字段创建
```

创建的 ConfigMap 如下图所示。

![img](/img/post/post_createconfigmap.png)

#### Pass a ConfigMap Entry to container as Env

ConfigMap 创建好之后，现在学习如何把它传递给 Pod 的 container。我们先通过最简单的方式传递一个 ConfigMap Entry。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL   # 正在设置这个环境变量
      valueFrom:
        configMapKeyRef:
          name: fortune-config  # 从这个ConfigMap
          key: sleep-interval   # 取这个key的值设置
```

定义了一个名为 INTERVAL 的环境变量，用名为 fortune-config 的 ConfigMap 中的 sleep-interval key 对应的 value 初始化。实现的效果如下图所示。

![img](/img/post/post_pass_a_configmapentry_as_env.png)

#### Pass all ConfigMap Entries to container as Env

如果一个 ConfigMap 包含多个 entry，那我们要按照上面的例子一条一条加吗？其实不然。Kubernetes 提供了 *envForm* 属性，可以一次性暴露所有的 entry 给 container。

假设我们有一个名为 my-config-map 的 ConfigMap，其中包含 FOO，BAR，FOO_BAR 三个key。那我们可通过如下方式传递给 container。

```yml
spec:
  containers:
  - image: some-image
    envFrom:  # 使用envForm属性，而非env属性
    - prefix: CONFIG_  # 所有环境变量以这个为前缀（非必须）
      configMapRef:
        name: my-config-map  # 引用这个ConfigMap
```

由于可以指定前缀，所以容器中会出现两个环境变量：CONFIG_FOO，CONFIG_BAR。如果我们不设置前缀，那么容器中出现的环境变量就和 ConfigMap 中的 key 一样。

但是 FOO_BAR 并没有出现在容器中。这是因为 CONFIG_FOO-BAR 不是一个合法的环境变量名字（包含了连接线），所以 Kubernetes 不会转换这个 key。

#### Pass a ConfigMap Entry to container as Args

这里我们学习如何把 ConfigMap 的值暴露为一个参数，传递给容器的主进程。我们不能直接在 *pod.spec.containers.args* 引用 ConfigMap 的 entry。但是可以先把 ConfigMap entry 初始化为一个环境变量，然后我们在参数中引用这个环境变量。如下图所示。

![img](/img/post/post_pass_a_configmapentry_as_args.png)

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap 
spec:
  containers:
  - image: luksa/fortune:args
    env:   # 这是之前学的pass ConfigMap entry as Env Variable
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]  # 在参数中引用这个环境变量
```

#### Pass all ConfigMap Entries to container using ConfigMap Volume

前面介绍的通过环境变量或命令行参数传递配置信息的方式只适合一些短的配置信息。我们的 ConfigMap 也可以包含整个配置文件。当我们想把包含配置文件的 ConfigMap 暴露给 container 时，我们应该使用一个特殊的 volume 类型，称为 configMap Volume。

一个 configMap volume 将以文件的形式暴露 ConfigMap 的每个 entry。容器的进程以读文件内容的方式获取 entry 的值。

我们实践一个例子。我们将使用一个配置文件，来配置运行在 web-server 容器中的 Nginx web server，这个容器就是 Volume 时的 web server Pod。假设我们想要 Nginx server 压缩返回给 client 的响应报文。那么配置文件应该如下。

```bash
server {
  listen          80;
  server_name     www.kubia-example.com;
  
  gzip on;   # 压缩文本和XML文件
  gzip_types text/plain application/xml;
  
  location / {
    root /usr/share/nginx/html; 
    index index.html index.htm;
  }
}
```

创建一个新目录 *configmap-files* 来存储上面的配置文件，配置文件命名为 *my-nginx-config.conf*。为了让 ConfigMap 也包含一个 *sleep-interval* entry，我们在该目录添加一个文本文件，如下图所示。

![img](/img/post/post_configmap-files.png)

第一步，从该目录创建 ConfigMap。

```
$ kubectl create configmap fortune-config --from-file=configmap-files
configmap "fortune-config" created
```

你可以查看创建的 ConfigMap。可以看到 ConfigMap 包含两个 entry，key 为对应的文件名。

```
$ kubectl get configmap fortune-config -o yaml
apiVersion: v1
data:
  my-nginx-config.conf: | 
    server {
      listen          80;
      server_name     www.kubia-example.com;

      gzip on;   
      gzip_types text/plain application/xml;

      location / {
        root /usr/share/nginx/html; 
        index index.html index.htm;
      }
    }
  sleep-interval: | 
    25
kind: ConfigMap
....
```

创建好 ConfigMap 后，第二步，我们在 Volume 中使用 ConfigMap 的entries。首先我们创建一个 Volume，它通过名字引用 ConfigMap；然后我们将这个 Volume 挂载到容器上。

Nginx 默认从 */etc/nginx/nginx.conf* 读取配置文件。配置文件自动包括了 */etc/nginx/conf.d/* 目录下的所有 *.conf* 文件。所以我们要把 Volume 挂载在这个路径下。

![img](/img/post/post_pass_configmapentry_as_volume.png)

下面是 Pod 的描述文件示例。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume 
spec:
  containers:
  - image: nginx:alpine
    name: web-server 
    volumeMounts: 
    ...
    - name: config
      mountPath: /etc/nginx/conf.d  # 挂载configMap volume在这个路径下
      readOnly: true 
    ...
  volumes:
  ...
  - name: config
    configMap:  # volume引用这个ConfigMap
      name: fortune-config
  ...
```

第三步，我们验证 Nginx 是否正在使用配置文件。

```bash
$ kubectl port-forward web-server 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
$ curl -H "Accept-Encoding: gzip" -I localhost:8080 
HTTP/1.1 200 OK
Server: nginx/1.11.1
Date: Thu, 18 Aug 2016 11:52:57 GMT
Content-Type: text/html
Last-Modified: Thu, 18 Aug 2016 11:52:55 GMT Connection: keep-alive
ETag: W/"57b5a197-37"
Content-Encoding: gzip  # 响应已经被压缩了
```

我们可以查看 */etc/nginx/conf.d/* 目录。可以看到，ConfigMap 的 entry 以文件形式添加到该目录下。即便 sleep-interval 没有被使用，也被添加进来了。

```
$ kubectl exec fortune-configmap-volume -c web-server ls /etc/nginx/conf.d
my-nginx-config.conf
sleep-interval
```

*sleep-interval* 其实是在另一个 html-generator container 中使用的。如果不想在这个容器中添加无关的 *sleep-interval*，我们可以使用如下的设置。

```yml
volumes:
- name: config
  configMap:
    name: fortune-config  # 挂载这个ConfigMap
    items:  # 选择包含哪一个entry
    - key: my-nginx-config.conf  # 想包含这个key的entry
      path: gzip.conf  # entry的值被保存在这个文件中
```

当指定某个 entry 时，我们需要给它设置一个文件名。再查看 */etc/nginx/conf.d/* 目录，可以看到只有一个 *gzip.conf* 文件。

有一点需要讨论，当我们把 volume 挂载为目录时，我们就覆盖了这个目录原来存在的文件。在上面的例子中没什么影响，但是如果把 volume 挂载到 */etc* 目录，因为所有原始文件都保存在这，就会因为访问不到而出现问题。

为了把单个 ConfigMap entry 挂载为文件，同时不隐藏这个目录的文件。我们可以使用 *subPath* 属性。

```yml
spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/someconfig.conf  # 挂载一个文件而非目录
      subPath: myconfig.conf  # 只挂载这个entry
```

可实现这样的效果。

![img](/img/post/post_mount_single_file.png)

另外，默认情况下，一个 configMap Volume 的文件访问权限是 *644(-rw-r—r--)*。我们可以在 volume 的描述中设置。

```
volumes:
- name: config
  configMap:
    name: fortune-config 
    defaultMode: "6600"  # -rw-rw------
```

#### Updating app’s config without restarting the app

前面说过，使用环境变量和命令行参数作为配置数据源的缺陷就是不能热更新。所以使用 ConfigMap 并暴露为 volume 可以带来无需重启即可更新配置数据的能力。

对于之前创建的 Nginx web server，我们可以通过 *kubectl edit* 命令修改它的 ConfigMap，将 *gzip on* 修改为 *gzip off*。然后我们查看容器内部挂载到对应目录的文件。

```bash
kubectl exec web-server -c web-server cat /etc/nginx/conf.d/my-nginx-config.conf
```

等一会儿，可以看到配置文件发生了变化。这个过程可能会持续一段时间，不会立刻生效。

注意，如果我们在容器中挂载的是单个文件而不是整个 volume，也即在 volume 的描述中使用了 *subPath*，文件不会被更新。

但是 Nginx 仍然会压缩 response，直到你显式地 reload 配置文件。

```
$ kubectl exec web-server -c web-server -- nginx -s reload
```

执行完上面的 reload 命令后，再次测试 Nginx，会发现已经关闭了压缩。

为了能够让 Pod 自动做滚动更新，我们可以引入一种很简单的方法，通过修改 pod annotations 的方式强制触发滚动更新。如下例所示，每次通过修改 version/config 来触发滚动更新。

```
$ kubectl patch deployment my-nginx --patch '{"spec": {"template": {"metadata": {"annotations": {"version/config": "20180411" }}}}}'
```

### Using Secrets to pass sensitive data to containers

截止到目前，ConfigMap 传递的都是不加密的数据。但是配置文件也可能包含证书、密钥等加密信息。这时就用到 Secret。

#### Introducing Secrets

Secrets 和 ConfigMap 非常相似，都保存 key-value 对，都可以通过相同的方式来使用。

+ Pass Secrets to Container as Env
+ Export Secret entries in a volume

Kubernetes 通过让 Secret 只保存在要访问该 Secret 的 Pod 所在 Node 上，并且只保存在内存中。在 Master Node 上，etcd以加密形式保存以确保安全性。

在 Service 篇章中，当介绍 Ingress 资源时，我们已经用 Secret 来保存 TLS 证书。现在我们详细学习。

#### Introducing the default token Secret

当我们用 *kubectl describe* 命令查看 Pod 的详细信息时，我们可以发现 Pod 的 Volume 位置挂载了 default-token-Secret。每个 Pod 都会挂载一个这样的 Secret。我们可以通过 *kubectl get secret* 来找到对应的 Secret。

当我们查看 default-token-Secret 的详细信息时，会发现包含三个 entries：ca.crt，namespace，token。它们被加密显示。根据 Pod 中的描述，它们被存储在 */var/run/secrets/kubernetes.io/serviceaccount* 目录。具体如下图所示。

![img](/img/post/post_default_token_secret.png)

#### Creating a Secret

我们现在创建一个 Secret，来使得 Nginx web server 可以 HTTPS 访问。首先，我们生成证书和私钥文件。

```
$ openssl genrsa -out https.key 2048
$ openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.kubia-example.com
```

然后为了更好学习 Secret，我们创建一个冗余的文件 *foo*，内容是 bar。

```
$ echo bar > foo
```

现在，我们像创建 ConfigMap 一样创建 Secret。

```
kubectl create secret generic fortune-https \--from-file=https.key  
\--from-file=https.cert 
\--from-file=foo
```

#### Comparing ConfigMaps and Secrets

我们查看创建的 Secret，可以看到内容都是 Base64-encoded 字符串。因为 Secret 中可能包含二进制，Base64 encoding 可以允许在 JSON 和 YAML 中包含二进制。

```
apiVersion: v1
data:
  foo: YmFyCg==
  https.cert: LS0tLS ....
  https.key: LS0tLS1 ....
kind: Secret
....
```

由于并不是所有的敏感数据都是二进制的。所以允许我们通过 *stringData* 来设置 Secret 的某些值。

```
apiVersion: v1
stringData:
  foo: plain text
data:
  foo: YmFyCg==
  https.cert: LS0tLS ....
  https.key: LS0tLS1 ....
kind: Secret
....
```

*stringData* 域是 write-only。当 *kubectl get -o yaml* 时*stringData* 域不会显示，而是和 data 域的内容一起显示。

#### Using Secret in Pod

现在我们要在 Pod 中使用 Secret。首先，我们 * kubectl edit configmap* 修改 *my-nginx-config.conf* 为如下内容。

```
server {
  listen          80;
  listen          443 ssl;
  server_name     www.kubia-example.com;
  
  ssl_certificate certs/https.cert; 
  ssl_certificate_key certs/https.key; 
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
  ssl_ciphers HIGH:!aNULL:!MD5;
  
  location / {
    root   /usr/share/nginx/html; 
    index  index.html index.htm;
  }
}
```

这样配置了 server 去读 */etc/nginx/certs* 目录下的证书和私钥。所以我们要把 Secret 挂载在这个目录。

接下来，我们修改 Pod 的描述文件。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https 
spec:
  containers:
  - image: luksa/fortune:env
    name: html-generator 
    env:
    - name: INTERVAL    # 传递环境变量
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval 
    volumeMounts:  # 挂载共享目录
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine 
    name: web-server 
    volumeMounts:
    - name: html   # 挂载共享目录
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config  # 挂载configMap
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443 
  volumes:
  - name: html  # 共享目录
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:  # 选择entry
      - key: my-nginx-config.conf  # 只暴露这个entry
        path: https.conf  #指定这个文件名
  - name: certs
    secret:
      secretName: fortune-https
```

上面的 Pod 可达到如下的效果。

![img](/img/post/post_fortune-https.png)

创建完 Pod 后，我们测试 Nginx。

```
$ kubectl port-forward fortune-https 8443:443 
Forwarding from 127.0.0.1:8443 -> 443 Forwarding from [::1]:8443 -> 443
$ curl https://localhost:8443 -k -v
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8443 (#0)
....
* Server certificate:
*  subject: CN=www.kubia-example.com
*  start date: Mar 18 07:56:52 2021 GMT
*  expire date: Mar 16 07:56:52 2031 GMT
*  issuer: CN=www.kubia-example.com
*  SSL certificate verify result: self signed certificate 
....
Don't let your mind wander -- it's too little to be let out alone.
```

可以看到我们得到了响应，并且可以核查 server 的证书是否和我们生成的匹配。

我们也可以通过以下命令看到 Secret 使用 in-memory 文件系统。

```bash
$ kubectl exec fortune-https -c web-server -- mount | grep certs
tmpfs on /etc/nginx/certs type tmpfs (ro,relatime)
```

最后，我们学习把 Secret 的 entry 暴露为环境变量。方式和 ConfigMap 一样，假设我们要把 foo 暴露为环境变量 FOO_SECRET，可以在容器的描述中添加如下。

```yml
env:
- name: FOO_SECRET
  valueFrom:
    secretKeyRef:  # 变量从secret中设置
      name: fortune-https  # 从这个Secret中
      key: foo  # 暴露这个key
```

即便可以通过环境变量暴露 Secret。但也不建议这么做。因为应用通常会在 log 中打印环境变量，会造成 secret 信息的泄露。

#### Understanding image pull Secrets

我们已经学习了如何把 Secret 传递给应用。但有时 Kubernetes 要求我们向他传递证书。例如，当我们想使用 private image registry 时，我们就要通过 Secret。

为了运行一个使用私人仓库镜像的 Pod，我们要先创建一个包含 Docker 登录信息的 Secret，然后在 Pod 描述文件中的 *imagePullSecrets* 域引用它。

首先，我们创建 Secret。这里我们没有创建 *generic* Secret，而是创建了 *docker-registry* Secret，包含 DockerHub username, password, and email。

```
kubectl create secret docker-registry mydockerhubsecret 
\ --docker-username=<username> 
\ --docker-password=<password> 
\ --docker-email=<email>
```

然后，我们在 Pod 描述文件中引用。

```
apiVersion: v1
kind: Pod
metadata:
  name: private-pod 
spec:
  imagePullSecrets:
  - name: zhaodockerhubsecret 
  containers:
  - image: zhaodockerhub/privateimage:tag
    name: main
```

### 总结

本篇章我们介绍了如何给容器传递配置数据。

+ 在 Pod 的描述文件中覆盖容器景象定义的默认命令；
+ 传递命令行参数给容器进程；
+ 为容器进程设置环境变量；
+ ConfigMap存储配置数据，以解耦配置数据和 Pod 描述文件；\
  短配置数据可以用命令行参数和环境变量形式暴露；长配置数据可以用文件挂载形式暴露；
+ Secret 存储敏感数据，以文件挂载形式安全地传递给容器；
+ 创建 *docker-registry* Secret 来 pull private image；

参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

