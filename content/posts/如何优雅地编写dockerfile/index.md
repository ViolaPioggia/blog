---
title: "如何优雅地编写Dockerfile"
date: "2023-09-15"
---

## 导语

docker 自上手以来一直成为了我的一个非常好用的工具，在部署第三方组件的时候非常方便，但是当我想把自己的项目放到docker里运行的时候常常会出现各种各样的问题

```
FROM alpine:latest

WORKDIR /app

COPY  ./user .
COPY  ./config.yaml ./server/service/user/config.yaml

# 暴露端口
EXPOSE 10001

CMD ["./user"]
```

这是我最近一个项目编写的dockerfile，可以看到结构非常的简单。但是有几个问题，

1，我的电脑是mac ，arm64芯片架构。编写的镜像无法在常见的amd64芯片架构的机器上运行

2，因为直接 `COPY` 根目录下的文件再编译会每次都下载一遍依赖，导致浪费的时间非常多，所以我出此下策直接`COPY`目录下的二进制文件进行运行

于是为此我专门去官网上阅读了教程，总结出了此篇文章

## Quick Start

### 编写Dockerfile

```
# syntax=docker/dockerfile:1
FROM golang:1.20-alpine
WORKDIR /src
COPY . .
RUN go mod download
RUN go build -o /bin/client ./cmd/client
RUN go build -o /bin/server ./cmd/server
ENTRYPOINT [ "/bin/server" ]
```

这是一个简单的示例，让我们看看能做什么

1. `# syntax=docker/dockerfile:1` 这决定了我们使用的 Dockerfile syntax 版本，他确保了有权限获取最新的 Docker build 特征。平时可以省略这行代码，会有一个推荐的默认值

3. `FROM golang:1.20-alpine` 顾名思义，`FORM`根据一个基础镜像进行新镜像的构建

5. `WORKDIR /src` 在 Dockerfile 中，`WORKDIR` 指令用于设置容器内部的工作目录（working directory）。它会影响后续命令在容器内执行时的默认路径。 具体来说，`WORKDIR` 指令可以有以下作用：

7. 设置默认工作目录：通过指定 `WORKDIR`，你可以将容器内部的默认路径设置为指定的目录。这意味着后续的命令在容器内执行时，默认情况下会在这个目录下进行操作，而不需要在每个命令中显式指定完整路径。

9. 简化命令：通过设置工作目录，你可以在 Dockerfile 中使用相对路径来引用文件和目录，而无需使用绝对路径。这样可以简化命令，使其更易读和维护。

11. 提高可移植性：通过使用 `WORKDIR`，你可以确保在不同的环境中运行容器时，文件和目录的引用路径保持一致。这提高了容器的可移植性，使其在不同的主机上更容易使用和部署。

13. `COPY . .` 把宿主机的文件（右）复制到镜像`WORKDIR` 指定的工作目录下（左）

15. `RUN go mod download` 拉取 go mod 依赖

17. `RUN go build -o /bin/client ./cmd/client` go build 编译文件到 /bin/client

19. `RUN go build -o /bin/server ./cmd/server` go build 编译文件到 /bin/server

21. `ENTRYPOINT [ "/bin/server" ]` 使用`ENTRYPOINT`命令运行 /bin/server 下的二进制文件

### 构建镜像

使用 `--tag`指定镜像的名称

```
$ docker build --tag=buildme .
```

### 运行容器

`--name`指定容器名称 `--rm`代表停止容器后自动删除 `--detach`含义与`-d`相同，代表在后台运行

```
$ docker run --name=buildme --rm --detach buildme
```

这样就运行起了一个名为 buildme 的镜像

使用 `exec -it xxx`进入容器内部

```
$ docker exec -it buildme /bin/client
```

使用下列命令停止容器运行

```
$ docker stop buildme
```

## 层

以 go 来讲，go build 之前往往会拉取编译所需的依赖，如果拉取依赖不够优雅，那么所编写的Dockerfile绝对谈不上优雅

当运行构建时，构建器会尝试重用早期构建中的层。 如果图像的某个图层未更改，则构建器会从构建缓存中选取它。 如果自上次构建以来某个层发生了更改，则必须重新构建该层以及后续的所有层。

上一节中的 Dockerfile 将所有项目文件复制到容器中 (COPY . .)，然后在以下步骤中下载应用程序依赖项 (RUN go mod download)。 如果您要更改任何项目文件，那么 COPY 层的缓存就会失效。 它还会使后续所有层的缓存失效。

![Layer cache is bust](images/cache-bust.png)

由于 Dockerfile 指令的当前顺序，构建器必须再次下载 Go 模块，尽管自上次以来没有任何包发生更改。

> **如何优化这一点呢？**

我们可以先把 go.mod && go.sum 拷贝进来，然后下载依赖。因为 go.mod && go.sum 一般不会变化，即使项目代码有所改变也不需要重新下载依赖，大大节省了构建的时间

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine
  WORKDIR /src
- COPY . .
+ COPY go.mod go.sum .
  RUN go mod download
+ COPY . .
  RUN go build -o /bin/client ./cmd/client
  RUN go build -o /bin/server ./cmd/server
  ENTRYPOINT [ "/bin/server" ]
```

![Reordered](images/reordered-layers.png)

## 多阶段构建

这个功能主要是为了解决一次性`COPY`整个项目代码导致镜像大小特别大的问题，如果不使用多阶段构建大型项目动辄就会1G+

### 增加阶段

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine
  WORKDIR /src
  COPY go.mod go.sum .
  RUN go mod download
  COPY . .
  RUN go build -o /bin/client ./cmd/client
  RUN go build -o /bin/server ./cmd/server
+
+ FROM scratch
+ COPY --from=0 /bin/client /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

### 异步构建

```
  # syntax=docker/dockerfile:1
- FROM golang:1.20-alpine
+ FROM golang:1.20-alpine AS base
  WORKDIR /src
  COPY go.mod go.sum .
  RUN go mod download
  COPY . .
+
+ FROM base AS build-client
  RUN go build -o /bin/client ./cmd/client
+
+ FROM base AS build-server
  RUN go build -o /bin/server ./cmd/server

  FROM scratch
- COPY --from=0 /bin/client /bin/server /bin/
+ COPY --from=build-client /bin/client /bin/
+ COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

`FROM golang:1.20-alpine AS base`其中 `AS base`意味着将此镜像“作为”了 base

在后续 `--from=xxx`中可以作为二次构建的基础镜像

![Stages executing in parallel](images/parallelism.gif)

可以看到两个阶段是同时构建的

### 目标构建

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine AS base
  WORKDIR /src
  COPY go.mod go.sum .
  RUN go mod download
  COPY . .

  FROM base AS build-client
  RUN go build -o /bin/client ./cmd/client

  FROM base AS build-server
  RUN go build -o /bin/server ./cmd/server

- FROM scratch
- COPY --from=build-client /bin/client /bin/
- COPY --from=build-server /bin/server /bin/
- ENTRYPOINT [ "/bin/server" ]

+ FROM scratch AS client
+ COPY --from=build-client /bin/client /bin/
+ ENTRYPOINT [ "/bin/client" ]

+ FROM scratch AS server
+ COPY --from=build-server /bin/server /bin/
+ ENTRYPOINT [ "/bin/server" ]
```

现在我们可以在指令的末尾加上`--target=xxx`,指定要构建阶段的名称，这样镜像会进一步缩小，可以避免编译多个二进制文件

```
$ docker build --tag=buildme-client --target=client .
$ docker build --tag=buildme-server --target=server .
$ docker images buildme 
REPOSITORY       TAG       IMAGE ID       CREATED          SIZE
buildme-client   latest    659105f8e6d7   20 seconds ago   4.25MB
buildme-server   latest    666d492d9f13   5 seconds ago    4.2MB
```

## 挂载

挂载顾名思义，就是使容器与主机之间的文件共享，能够确保读写高性能以及数据持久化

### cache mount

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine AS base
  WORKDIR /src
  COPY go.mod go.sum .
- RUN go mod download
+ RUN --mount=type=cache,target=/go/pkg/mod/ \
+     go mod download -x
  COPY . .

  FROM base AS build-client
- RUN go build -o /bin/client ./cmd/client
+ RUN --mount=type=cache,target=/go/pkg/mod/ \
+     go build -o /bin/client ./cmd/client

  FROM base AS build-server
- RUN go build -o /bin/server ./cmd/server
+ RUN --mount=type=cache,target=/go/pkg/mod/ \
+     go build -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

这里我们直接指定依赖为主机上的 /go/pkg/mod/ ，下载依赖也会下载到这个目录下，这同样能够解决重复下载依赖的问题

### bind mount

效果与上面类似

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine AS base
  WORKDIR /src
- COPY go.mod go.sum .
  RUN --mount=type=cache,target=/go/pkg/mod/ \
+     --mount=type=bind,source=go.sum,target=go.sum \
+     --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x
  COPY . .

  FROM base AS build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      go build -o /bin/client ./cmd/client

  FROM base AS build-server
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      go build -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

也可以像这样直接挂载根目录然后直接编译目标文件

```
  # syntax=docker/dockerfile:1
  FROM golang:1.20-alpine AS base
  WORKDIR /src
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,source=go.sum,target=go.sum \
      --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x
- COPY . .

  FROM base AS build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
+     --mount=type=bind,target=. \
      go build -o /bin/client ./cmd/client

  FROM base AS build-server
  RUN --mount=type=cache,target=/go/pkg/mod/ \
+     --mount=type=bind,target=. \
      go build -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

还有一种 `--mount=type=`volume 就不细说了

### 区别

`--mount=type=bind` 和 `--mount=type=cache` 是 Docker 命令行选项中不同的挂载类型，它们在挂载方式和行为上有所区别。

1. `--mount=type=bind`：

- 挂载方式：使用绑定挂载，将主机上的文件或目录与容器中的指定路径进行绑定。

- 行为特点：容器与主机共享相同的文件系统，对挂载目录的读取和写入操作会直接反映在主机文件系统中。

- 适用场景：适用于需要在容器和主机之间共享文件的场景，如应用程序开发和调试、日志文件的读取等。

1. `--mount=type=cache`：

- 挂载方式：使用缓存挂载，将主机上的数据目录缓存挂载到容器中。

- 行为特点：容器可以从缓存中读取数据，提高读取性能；写入操作直接写入主机上的数据目录，不写入缓存。

- 适用场景：适用于对大量数据进行频繁读取的场景，可提高读取性能，但不适用于需要频繁写入数据的场景。

要选择使用哪种挂载类型，需要根据具体的需求和场景进行考虑：

- 如果需要在容器和主机之间实现文件共享和交互，并且需要对挂载目录进行频繁的读取和写入操作，那么可以选择 `--mount=type=bind`。

- 如果主要是在容器中进行大量的数据读取操作，并且对读取性能有较高的要求，而写入操作相对较少，那么可以考虑使用 `--mount=type=cache`，以提高读取性能。

需要注意的是，`--mount=type=cache` 挂载类型在 Docker 的某些版本中可能不可用，具体取决于你使用的 Docker 版本和配置。如果 `--mount=type=cache` 不可用，你可以尝试使用其他类型的挂载，如 `--mount=type=bind` 或 `--mount=type=volume`，根据你的需求选择适当的挂载类型。

## 构建命令参数

```
  # syntax=docker/dockerfile:1
- FROM golang:1.20-alpine AS base
+ ARG GO_VERSION=1.20
+ FROM golang:${GO_VERSION}-alpine AS base
  WORKDIR /src
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,source=go.sum,target=go.sum \
      --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x

  FROM base AS build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -o /bin/client ./cmd/client

  FROM base AS build-server
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

在这个示例中，`GO_VERSION`被默认设置为了1.20

但是如果在构建命令时输入

```
$ docker build --build-arg="GO_VERSION=1.19" .
```

那么`GO_VERSION`就会被设置为1.19

### 注入值

这个功能我感觉不是很常用，就先摆在这里了

```
// cmd/server/main.go
var version string

func main() {
    if version != "" {
        log.Printf("Version: %s", version)
    }
```

```
  # syntax=docker/dockerfile:1
  ARG GO_VERSION=1.20
  FROM golang:${GO_VERSION}-alpine AS base
  WORKDIR /src
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,source=go.sum,target=go.sum \
      --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x

  FROM base AS build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -o /bin/client ./cmd/client

  FROM base AS build-server
+ ARG APP_VERSION="v0.0.0+unknown"
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
- go build -o /bin/server ./cmd/server
+     go build -ldflags "-X main.version=$APP_VERSION" -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
```

现在只需要使用下列的命令即可更改 main 文件中的 `version`变量

```
$ docker build --target=server --build-arg="APP_VERSION=v0.0.1" --tag=buildme-server .
$ docker run buildme-server
2023/04/06 08:54:27 Version: v0.0.1
2023/04/06 08:54:27 Starting server...
2023/04/06 08:54:27 Listening on HTTP port 3000
```

## 导出二进制文件

有些时候我们不想把文件打包成镜像，我们只想导出一个二进制文件，我们可以使用`local`来达成这个目的

```
$ docker build --output=. --target=server .
```

像这样，将编译生成的二进制文件导出到当前目录下

如果想获得多个二进制文件，可以更改成如下所示

```
  # syntax=docker/dockerfile:1
  ARG GO_VERSION=1.20
  FROM golang:${GO_VERSION}-alpine AS base
  WORKDIR /src
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,source=go.sum,target=go.sum \
      --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x

  FROM base as build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -o /bin/client ./cmd/client

  FROM base as build-server
  ARG APP_VERSION="0.0.0+unknown"
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -ldflags "-X main.version=$APP_VERSION" -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]
+
+ FROM scratch AS binaries
+ COPY --from=build-client /bin/client /
+ COPY --from=build-server /bin/server /
```

## 测试

```
$ docker run -v $PWD:/test -w /test \
  golangci/golangci-lint golangci-lint run
```

会发现有如下报错

```
cmd/server/main.go:23:10: Error return value of `w.Write` is not checked (errcheck)
        w.Write([]byte(translated))
              ^
```

让我们修改以下代码

```
  # syntax=docker/dockerfile:1
  ARG GO_VERSION=1.20
+ ARG GOLANGCI_LINT_VERSION=v1.52
  FROM golang:${GO_VERSION}-alpine AS base
  WORKDIR /src
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,source=go.sum,target=go.sum \
      --mount=type=bind,source=go.mod,target=go.mod \
      go mod download -x

  FROM base AS build-client
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -o /bin/client ./cmd/client

  FROM base AS build-server
  ARG APP_VERSION="0.0.0+unknown"
  RUN --mount=type=cache,target=/go/pkg/mod/ \
      --mount=type=bind,target=. \
      go build -ldflags "-X main.version=$APP_VERSION" -o /bin/server ./cmd/server

  FROM scratch AS client
  COPY --from=build-client /bin/client /bin/
  ENTRYPOINT [ "/bin/client" ]

  FROM scratch AS server
  COPY --from=build-server /bin/server /bin/
  ENTRYPOINT [ "/bin/server" ]

  FROM scratch AS binaries
  COPY --from=build-client /bin/client /
  COPY --from=build-server /bin/server /
+
+ FROM golangci/golangci-lint:${GOLANGCI_LINT_VERSION} as lint
+ WORKDIR /test
+ RUN --mount=type=bind,target=. \
+     golangci-lint run
```

想运行`lint`的阶段，必须运行以下指令，正常编译Dockerfile是不会执行这一阶段的，因为这个阶段时独立的

```
$ docker build --target=lint .
```

## 多架构构建

让构建出来的镜像在本机运行十分简单，但是如果想在不同架构的各种各样的机器上运行就要稍微多花一番工夫

这是最简单的方式，不需要修改Dockerfile

```
$ docker build --target=server --platform=linux/arm/v7 .
```

但是这样依然不能同时构建出多个架构上运行的镜像，为此我们需要用到 buildx 工具

### 构建Buildx

首先确保 Buildx client 已安装

```
$ docker buildx version
github.com/docker/buildx v0.10.3 79e156beb11f697f06ac67fa1fb958e4762c0fab
```

然后创建一个 builder

```
$ docker buildx create --driver=docker-container --name=container
```

可以使用 `docker buildx ls`查看

```
$ docker buildx ls
NAME/NODE           DRIVER/ENDPOINT               STATUS
container           docker-container
  container_0       unix:///var/run/docker.sock   inactive
default *           docker
  default           default                       running
desktop-linux       docker
  desktop-linux     desktop-linux                 running
```

完成✅

### Build using emulation

- \`--builder=container 选择一个新的 builder

- `--platform=linux/amd64,linux/arm/v7,linux/arm64/v8` 一次性构建多个架构

```
$ docker buildx build \
    --target=binaries \
    --output=bin \
    --builder=container \
    --platform=linux/amd64,linux/arm64,linux/arm/v7 .
```

![Build pipelines using emulation](images/emulation.png)

### Platform build arguments

这种方法远没有上一种简单，省流就是说通过更改参数来进行多阶段构建，就不详细介绍了

## 杂项

### 1，ADD与COPY的区别

在 Docker 中，`ADD` 和 `COPY` 是两个用于复制文件到容器内部的指令。它们在行为和使用情况上有一些区别。

1. `COPY` 指令：

- 用法：`COPY <源路径> <目标路径>`

- 行为特点：
    - `COPY` 指令将主机上的文件或目录复制到容器内部的目标路径。
    
    - 源路径可以是主机上的相对路径或绝对路径，而目标路径是容器内部的路径。
    
    - 如果源路径是一个目录，那么它将递归地复制目录及其内容到容器内部。
    
    - `COPY` 指令只能复制本地文件系统中的文件，不能从 URL 或其他网络位置复制文件。

- 适用场景：适用于从主机文件系统复制文件或目录到容器内部的场景，例如复制应用程序的源代码或静态资源文件等。

1. `ADD` 指令：

- 用法：`ADD <源路径> <目标路径>`

- 行为特点：
    - `ADD` 指令与 `COPY` 指令类似，也用于将文件或目录复制到容器内部的目标路径。
    
    - 与COPY指令不同的是，ADD指令还支持一些额外的特性：
    
    - 自动解压缩：如果源路径是一个压缩文件（例如.tar、.tar.gz、.tar.bz2、.tar.xz、.zip），`ADD` 指令会自动将其解压缩到目标路径。
    
    - 远程文件下载：如果源路径是一个 URL，`ADD` 指令会尝试下载该文件并复制到容器内部。

- 适用场景：适用于从主机文件系统复制文件或目录到容器内部，并且需要支持自动解压缩或远程文件下载的场景，例如复制应用程序的依赖包或远程配置文件等。

总结起来，`COPY` 指令用于将主机文件系统中的文件或目录复制到容器内部，而 `ADD` 指令除了具备 `COPY` 指令的功能外，还支持自动解压缩和远程文件下载的特性。在大多数情况下，`COPY` 指令是更常见且推荐的选择，除非有特定的需求需要使用 `ADD` 指令的额外特性。

### 2，因为基础镜像不具备Shanghai时区导致报错

在Dockerfile中使用例如`yum install xxx`下载缺失的组件，又碰到了再补充

## 引用

我总结出来的总归有些疏漏，详情还请见官方文档

https://docs.docker.com/develop/develop-images/dockerfile\_best-practices/

https://docs.docker.com/build/guide/
