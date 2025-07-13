# Dockerfile创建Docker镜像

Dockerfile是一个创建镜像所有命令的文本文件， 包含了一条条指令和说明，每条指令构建一层，通过 docker build命令，根据Dockerfile的内容构建镜像。

Dockerfile里的每一条指令的内容, 就是描述该层如何构建，有了 Dockefile，就可以制定自己的docker镜像规则，只需要在Dockerfile上添加或者修改指令， 就可生成不同的 docker 镜像。

## Dockerfile的优势

Dockerfile 包含了镜像制作的完整操作流程，其他开发者可以通过 Dockerfile 了解并复现制作过程。

Dockerfile 中的每一条指令都会创建新的镜像层，这些镜像可以被 Docker Daemon 缓存。再次制作镜像时，Docker 会尽量复用缓存的镜像层（using cache），而不是重新逐层构建，这样可以节省时间和 磁盘空间。

Dockerfile 的操作流程可以通过docker image history [镜像名称] 查询，方便开发者查看变更记录。

## Docker build 构建流程

docker build命令会读取Dockerfile的内容，并将Dockerfile的内容发送给 Docker 引擎，最终 Docker  引擎会解析Dockerfile中的每一条指令，构建出需要的镜像。 镜像的构建流程如下：

1. docker build会将 context 中的文件打包传给 Docker daemon。如果 context 中 有.dockerignore文件，则会从上传列表中删除满足.dockerignore规则的文件。（注意：如果上下文中有 相当多的文件，可以明显感受到整个文件发送过程）
2. docker build命令向 Docker server 发送 HTTP 请求，请求 Docker server 构建镜像，请求中 包含了需要的 context 信息。
3. Docker server 接收到构建请求之后，开始构建镜像。

在Docker server 构建镜像时的步骤如下：

1. 创建一个临时目录，并将 context 中的文件解压到该目录下。
2.  读取并解析 Dockerfile，遍历其中的指令，根据命令类型分发到不同的模块去执行。
3. Docker 构建引擎为每一条指令创建一个临时容器，在临时容器中执行指令，然后 commit 容器， 生成一个新的镜像层。
4. 最后，将所有指令构建出的镜像层合并，形成 build 的最后结果。最后一次 commit 生成的镜像 ID  就是最终的镜像 ID。

为了提高构建效率，docker build默认会缓存已有的镜像层。如果构建镜像时发现某个镜像层已经被缓存，就会直接使用该缓存镜像，而不用重新构建。如果不希望使用缓存的镜像，可以在执行docker  build命令时，指定--no-cache=true参数。

Docker 匹配缓存镜像的规则为：

- 遍历缓存中的基础镜像及其子镜像，检查这些镜像的构建指令是否和当前指令完全一致，如果不一样，则说明缓存不匹配。
- 对于ADD、COPY指令，还会根据文件的校验和 （checksum）来判断添加到镜像中的文件是否相同，如果不相同，则说明缓存匹配。

需要注意的时，缓存匹配检查不会检查容器中的文件。比如，当使用RUN apt-get -y update命令更新了容 器中的文件时，缓存策略并不会检查这些文件，来判断缓存是否匹配。

可以通过docker history命令来查看镜像的构建历史。

## Dockerfile的关键词

```dockerfile
FROM # 设置镜像使用的基础镜像
MAINTAINER # 设置镜像的作者
RUN # 编译镜像时运行的脚步
CMD # 设置容器的启动命令
LABEL # 设置镜像标签
EXPOSE # 设置镜像暴露的端口
ENV # 设置容器的环境变量
ADD # 编译镜像时复制上下文中文件到镜像中
COPY # 编译镜像时复制上下文中文件到镜像中
ENTRYPOINT # 设置容器的入口程序
VOLUME # 设置容器的挂载卷
USER 设置运行 # RUN CMD ENTRYPOINT的用户名
WORKDIR 设置 # RUN CMD ENTRYPOINT COPY ADD 指令的工作目录
ARG # 设置编译镜像时加入的参数
ONBUILD # 设置镜像的ONBUILD 指令
STOPSIGNAL # 设置容器的退出信号量
```

## Dockerfile 实践

使用golang开发了应用程序，编写Dockerfile 进行镜像构建：

```dockerfile
FROM golang:1.18
ENV env1=env1value
ENV env2=env2value
MAINTAINER yangshuangxin
 # 仅指定镜像元数据内容
LABEL hello 1.0.0
RUN git clone https://gitee.com/nickdemo/helloworld.git
WORKDIR helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
# 编译go程序
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
EXPOSE 80
# 之前容器执行的程序和参数
CMD ["./app","--param1=p1","--param2=p2"]
```

然后进行容器构建：

```shell
docker build  -t hello:1.0.0 -f Dockerfile .
```

可以把上下文的内容拷贝到其他目录进行编译打包：

```dockerfile
FROM golang:1.18
MAINTAINER yangshuangxin
LABEL hello 1.0.0
# 拷贝上下文的内容到工作路径
COPY ./helloworld /go/src/helloworld
WORKDIR /go/src/helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
EXPOSE 80
CMD ["./app","--param1=p1","--param2=p2"]
```

## Dockerfile 多阶段构建 

Docker 17.05版本以后，新增了Dockerfile多阶段构建。所谓多阶段构建，实际上是允许一个Dockerfile  中出现多个 FROM 指令。这样做有什么意义呢？ 

多个 FROM 指令并不是为了生成多根的层关系，最后生成的镜像，仍以最后一条 FROM 为准，之前的  FROM 会被抛弃。那么之前的FROM 又有什么意义呢？

每一条 FROM 指令都是一个构建阶段，多条 FROM 就是多阶段构建，虽然最后生成的镜像只能是最后 一个阶段的结果，但是，能够将前置阶段中的文件拷贝到后边的阶段中，这就是多阶段构建的最大意义。

最大的使用场景是将编译环境和运行环境分离，比如需要构建一个Go语言程序，那么就需要 用到go命令等编译环境，当时最后只需要运行环境的镜像，不需要开发环境的进行，减少容器的体积。

```dockerfile
FROM golang:1.18
ADD ./helloworld /go/src/helloworld/
WORKDIR /go/src/helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
FROM alpine:latest
MAINTAINER yangshuangxin
LABEL hello 1.0.0
WORKDIR /app/
COPY --from=0 /go/src/helloworld/app ./
EXPOSE 80
CMD ["./app","--param1=p1","--param2=p2"]
```

--from=0就是第一阶段编译环境，也通过as关键词，为构建阶段指定别名，可以提高可读性：

```shell
FROM golang:1.18  as stage0
ADD ./helloworld /go/src/helloworld/
WORKDIR /go/src/helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
FROM alpine:latest
MAINTAINER yangshuangxin
LABEL hello 1.0.0
WORKDIR /app/
COPY --from=stage0 /go/src/helloworld/app ./
EXPOSE 80
CMD ["./app","--param1=p1","--param2=p2"]
```

## Dockerfile 的 ADD 与 COPY

ADD 与 COPY 只能拷贝上下文传入的文件。

```dockerfile
COPY <src> <dest>  # 将上下文中源文件，拷贝到目标文件
ADD <src> <dest>
COPY prefix* /destDir/  # 将所有prefix 开头的文件拷贝到 destDir 目录下
COPY prefix?.log /destDir/  #支持单个占位符，例如: prefix1.log、prefix2.log 等

# 对于目录而言，COPY 和 ADD 命令具有相同的特点：只复制目录中的内容而不包含目录自身
COPY srcDir /destDir/  # 只会将源文件夹srcDir下的文件拷贝到 destDir 目录下
```

COPY 区别于ADD在于Dockerfile中使用multi-stage。例如上面的例子把第一个阶段的编译结果拷贝到第二个镜像构建阶段。

ADD 命令除了不能用在 multi-stage 的场景下，ADD 命令可以完成 COPY 命令的所有功能，并且还可以 完成两类的功能：

- 解压压缩文件并把它们添加到镜像中，对于宿主机本地压缩文件，ADD命令会自动解压并添加到镜像。
- 从 url 拷贝文件到镜像中，需要注意：url 所在文件如果是压缩包，ADD 命令不会自动解压缩。

目标文件位置要注意路径后面是否带 "/" ，带斜杠表示目录，不带斜杠表示文件名 。文件名里带有空格，需要再 ADD（或COPY）指令里用双引号的形式标明，例如：

```dockerfile
ADD "space file.txt" "/tmp/space file.txt"
```

## Dockerfile 的 CMD 与 ENTRYPOINT

### CMD

CMD 指令提供容器运行时的默认值，这些默认值可以是一条指令，也可以是一些参数。一个dockerfile中可以有多条CMD指令，但只有最后一条CMD指令有效。

CMD指令在构建镜像时并不执行任何命令，而是在容器启动时默认将CMD指令作为第一条执行的命令。如果在命令行界面运行docker run 命令时指定命令参数，则会覆盖CMD指令中的命令。

CMD参数格式是在**CMD指令与ENTRYPOINT指令配合**时使用，CMD指令中的参数会添加到 ENTRYPOINT指令中。

使用shell 和exec 格式时，命令在容器中的运行方式与RUN 指令相同。不同在于，RUN指令在构 建镜像时执行命令，并生成新的镜像。

```dockerfile
# CMD 指令有三种格式
# shell 格式
CMD <command>
 # exec格式，推荐格式
CMD ["executable","param1","param2"]
 # 为ENTRYPOINT 指令提供参数
CMD ["param1","param2"]
```

### ENTRYPOINT

ENTRYPOINT指令和CMD指令类似，都可以让容器每次启动时执行相同的命令，但它们之间又有不 同。一个Dockerfile中可以有多条ENTRYPOINT指令，但只有最后一条ENTRYPOINT指令有效。

当使用shell格式时，ENTRYPOINT指令会忽略任何CMD指令和docker run 命令的参数，并且会运行在bin/sh -c中。

推荐使用exec格式，使用此格式，docker run 传入的命令参数将会覆盖CMD指令的内容并且附加到ENTRYPOINT指令的参数中。

CMD可以是参数，也可以是指令，ENTRYPOINT只能是命令；docker run 命令提供的运行命令参数可以覆盖CMD，但不能覆盖ENTRYPOINT。但是也可以通过docker run --entrypoint 替换容器的入口程序。

```dockerfile
# ENTRYPOINT指令有两种格式
# shell 格式
ENTRYPOINT <command>
 # exec 格式，推荐格式
ENTRYPOINT ["executable","param1","param2"]
```

例子如下：

```dockerfile
FROM golang:1.18 as stage0
ADD ./helloworld /go/src/helloworld/
WORKDIR /go/src/helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
FROM alpine:latest
MAINTAINER yangshuangxin
LABEL hello 1.0.0
WORKDIR /app/
COPY --from=stage0 /go/src/helloworld/app ./
EXPOSE 80
ENTRYPOINT ["./app","--param1=p1","--param2=p2"]
```

打包镜像和启动容器

```shell
docker build -t hello:1.0.0 .
# 指定启动命令和参数, 只有CMD时有用
docker run -d -p 80:80 hello:1.0.0 ./app --param4=5 --param6=7
# 指定启动命令为sh, 只有CMD时有用
docker run -dit -p 80:80 hello:1.0.0 sh
# 指定入口程序为sh, 存在ENTRYPOINT
docker run -dit -p 80:80 --entrypoint sh  hello:1.0.0
```

## Dockerfile 的参数配置

Dockerfile 使用arg来声明参数，在构建镜像时可以把自定义的参数传入到Dockfile中参与构建。Dockerfile 有预定义参数，可直 接在dockerfile中使用，无需使用arg来声明：HTTP_PROXY，http_proxy，HTTPS_PROXY，https_proxy， FTP_PROXY，ftp_proxy，NO_PROXY，no_proxy，ALL_PROXY，all_proxy。

```dockerfile
FROM golang:1.18 as s0
 ADD ./helloworld /go/src/helloworld/
 WORKDIR /go/src/helloworld
 # RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
 RUN go env -w GOPROXY=$http_proxy
 RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .

FROM alpine:latest as s1
ARG wd label tag # 申明自定义参数
RUN echo $wd,$label,$tag
MAINTAINER yangshuangxin
LABEL $label $tag
WORKDIR $wd
COPY --from=stage0 /go/src/helloworld/app ./
EXPOSE 80
ENTRYPOINT ["./app","--param1=p1","--param2=p2"]
```

构建时把参数传入：

```shell
 docker build -t hello:1.0.0 --build-arg "http_proxy=https://proxy.golang.com.cn,https://goproxy.cn,direct" --build-arg "wd=/home/app/" --build-arg "label=myhello" --build-arg "tag=1.0.0" --no-cache .
```

## Docker镜像构建缓存

对于多阶段构建，可以通过--target指定需要重新构建的阶段。--cache-from 可以指定一个镜像作为缓存 源，当构建过程中dockerfile指令与缓存镜像源指令匹配，则直接使用缓存镜像中的镜像层，从而加快构建进程。

可以将缓存镜像推送到远程注册中心，提供给不同的构建过程使用，在使用前需要先pull到本地。

```dockerfile
FROM golang:1.18 as s0
ADD ./helloworld /go/src/helloworld/
WORKDIR /go/src/helloworld
RUN go env -w GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
 
FROM alpine:latest as s1
MAINTAINER yangshuangxin
LABEL hello 1.0.0
WORKDIR /app
COPY --from=s0 /go/src/helloworld/app ./
EXPOSE 80
ENTRYPOINT ["./app","--param1=p1","--param2=p2"]
```



进行分阶段构建：

```shell
 docker build -t prehello:1.0.0 --target s0 .
 docker build -t hello:1.0.0 --cache-from prehello:1.0.0 --target s1 .
```

## Dockerfile的onbuild指令

onbuild指令将指令添加到镜像中，当镜像作为另一个构建的基础镜像时，将触发这些指令执行。触发指 令将在下游构建的上下文中执行。注意：onbuild指令不会影响当前构建，但会影响下游构建。

生成自定义基础镜像指令:

```dockerfile
FROM golang:1.18
ONBUILD ADD ./helloworld /go/src/helloworld/
ONBUILD WORKDIR /go/src/helloworld
ONBUILD RUN go env -w 
GOPROXY=https://proxy.golang.com.cn,https://goproxy.cn,direct
ONBUILD RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o app .
```

生成基础镜像：

```shell
docker build -t mygolang:1.0.0  .
```

通过自定义基础镜像构建新的镜像:

```shell
FROM mygolang:1.0.0 as s0
FROM alpine:latest as s1
MAINTAINER yangshuangxin
LABEL hello 1.0.0
WORKDIR /app
COPY --from=s0 /go/src/helloworld/app ./
EXPOSE 80
ENTRYPOINT ["./app","--param1=p1","--param2=p2"]
```

构建新的镜像时，触发基础进行的ONBUILD指令：

```shell
docker build -t hello:1.0.0 .
```

