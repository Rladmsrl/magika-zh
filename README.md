# Magika 网站

这是一个简单的网站，用于解释如何使用 Magika。您可以通过[点击此链接在浏览器中加载它](https://google.github.io/magika/)。

## 本地启动

您需要在本地安装 Node.js 和 Yarn。然后，执行：

```bash
yarn install
yarn run dev
```

该网站将在 `http://localhost:5173/` 上可用。

## 使用 Docker 启动

您也可以使用 Docker 在容器中运行该网站。

### 使用 Docker Compose

这是最简单的方法。

```bash
docker-compose up -d
```

该网站将在 `http://localhost:8080/` 上可用。

要停止服务，请运行：
```bash
docker-compose down
```

### 使用 Dockerfile

您也可以手动构建 Docker 镜像并运行容器。

1.  **构建镜像:**
    ```bash
    docker build -t magika-website .
    ```

2.  **运行容器:**
    ```bash
    docker run -d -p 8080:80 --name magika-website-container magika-website
    ```

该网站将在 `http://localhost:8080/` 上可用。

要停止并删除容器，请运行：
```bash
docker stop magika-website-container
docker rm magika-website-container
```
