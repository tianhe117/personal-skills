---
name: python-web-docker
description: 为我自己维护的 Python Web 工具新增或调整 Docker Compose 部署。用于 Python 3.12 slim 镜像、宿主源码挂载、git pull 更新和本地 Compose 覆盖配置。
---

# Python Web 工具的 Docker 部署

先检查项目的依赖文件、实际启动命令、监听地址、端口和数据目录。在应用目录放置 `Dockerfile`、`compose.yaml` 和 `compose.override.example.yaml`。沿用项目已有的启动方式和数据位置。

- 基础镜像固定为 `python:3.12-slim`。用 `WORKDIR /app`。构建时只复制安装依赖所需的文件，例如 `COPY requirements.txt .`，再执行 `RUN pip install --no-cache-dir -r requirements.txt`。禁止 `COPY . .`，也避免无关文件进入构建上下文。用 `.dockerignore` 排除 Git 元数据、虚拟环境、本地配置、密钥和运行数据。
- 用 Compose 绑定挂载 `.:/app`，应用代码和资源由宿主机工作目录提供。把依赖安装在挂载目录之外。实际启动命令使用项目已有入口，并让应用在容器内监听 `0.0.0.0`。持久化数据写到挂载目录内的指定路径或现有数据卷。
- 显式设置 `image` 和 `container_name`。使用符合项目实际用途的小写短横线名称，镜像加显式版本标签，容器名用同一个名称但不加标签。例如参考项目使用 `wecom-checkin:1.0` 和 `wecom-checkin`。已有部署改名时，检查项目身份和宿主机端口是否会与旧容器冲突。
- 主配置放共享默认值。通过环境变量 `TZ: Asia/Shanghai` 将容器时区设为北京时间。默认对宿主机所有网卡开放，端口映射使用 `宿主机端口:容器端口`，不指定宿主机 IP。把端口、环境变量等可定制项写入受 Git 跟踪的 `compose.override.example.yaml`，指导用户复制成 `compose.override.yaml`，并把真实覆盖文件加入 `.gitignore`。示例只放占位值，不放密钥。

主配置示例。将 `my-tool` 和端口替换成实际值。

```yaml
services:
  app:
    build: .
    image: my-tool:1.0
    container_name: my-tool
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    ports:
      - "8000:8000"
    volumes:
      - .:/app
```

覆盖文件示例。服务键须与主文件一致。覆盖端口列表使用 `!override`，以免默认端口合并进来。此语法需要 Docker Compose 2.24.4 或更高版本。

```yaml
services:
  app:
    ports: !override
      - "8080:8000"
```

在项目文档中说明以下操作，命令从 Compose 文件所在目录运行，并使用实际服务键。

- 首次部署或修改依赖、基础镜像、Dockerfile 后，执行 `docker compose up -d --build app`。
- 仅更新代码时，`git pull` 后文件会立即映射进容器。默认执行 `docker compose restart app` 让 Python 进程加载新代码，无需重建镜像。只有项目本身启用了自动重载，才可以省略重启。
- 只改端口或其他 Compose 配置时，执行 `docker compose up -d app` 应用配置。
- 有 Docker Compose 时，分别用 `docker compose -f compose.yaml config --quiet` 和 `docker compose -f compose.yaml -f compose.override.example.yaml config --quiet` 校验。没有 Docker 时说明运行验证尚未完成。
