---
description: "从源码构建——在本地构建并运行 Conductor 服务器和 ui-next，用于开发和测试。"
---
# 从源码构建

从源码在本地构建并运行 Conductor 服务器和 `ui-next`。默认配置使用内存持久化且无索引——服务器停止时所有数据都会丢失。此配置仅供开发和测试使用。

如需持久化后端，请使用 [Docker Compose](deploy.md) 或配置一个数据库后端。


## 前置条件

- Java (JDK) 21+
- （可选）[Docker](https://www.docker.com/get-started/) 用于运行测试


## 构建并运行服务器

1. 克隆仓库：

    ```shell
    git clone https://github.com/conductor-oss/conductor.git
    cd conductor
    ```

2. 用 Gradle 运行：

    ```shell
    cd server
    ../gradlew bootRun
    ```

    要使用自定义配置文件：

    ```shell
    CONFIG_PROP=config.properties ../gradlew bootRun
    ```

3. 服务器现在开始运行：

    | URL | 说明 |
    |:----|:---|
    | `http://localhost:8080/swagger-ui/index.html` | REST API 文档 |
    | `http://localhost:8080/api/` | API 基础 URL |


## 从预编译 JAR 运行

作为从源码构建的替代方案，可以下载并运行预编译的 JAR：

```shell
export CONDUCTOR_VER=3.21.10
export REPO_URL=https://repo1.maven.org/maven2/org/conductoross/conductor-server
curl $REPO_URL/$CONDUCTOR_VER/conductor-core-$CONDUCTOR_VER-boot.jar \
  --output conductor-core-$CONDUCTOR_VER-boot.jar
java -jar conductor-core-$CONDUCTOR_VER-boot.jar
```


## 从源码运行 ui-next

### 前置条件

- 运行在 8080 端口的 Conductor 服务器
- Node.js 18+
- pnpm 10.x（用 `corepack enable` 激活 `ui-next/package.json` 中固定的版本）

### 步骤

```shell
cd ui-next
corepack enable
pnpm install
```

在 `.env` 中配置后端 URL（入库的默认值指向本地服务器）：

```shell
VITE_WF_SERVER=http://localhost:8080
```

启动开发服务器：

```shell
pnpm dev
```

UI 可在 [http://localhost:1234](http://localhost:1234) 访问。如需运行时功能开关（feature flags）和认证配置，请将 `public/context.js.example` 复制为 `public/context.js` 并编辑该副本。

要构建用于生产托管的编译产物：

```shell
pnpm build
```

生产构建的产物会写入 `ui-next/dist/`。
