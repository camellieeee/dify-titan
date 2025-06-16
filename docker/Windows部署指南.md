# Dify Windows Server 2019 部署指南

## 问题分析

您遇到的 `no matching manifest for windows` 错误是因为 Dify 的 Docker 镜像只支持 Linux 平台（linux/amd64 和 linux/arm64），不支持 Windows 平台。

## 解决方案

### 方案一：使用 WSL2（强烈推荐）

#### 1. 启用 WSL2 功能

以管理员身份运行 PowerShell，执行以下命令：

```powershell
# 启用 WSL 功能
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# 启用虚拟机平台功能
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

#### 2. 重启系统

重启 Windows Server 2019 系统。

#### 3. 下载并安装 WSL2 Linux 内核更新包

从微软官网下载：https://aka.ms/wsl2kernel

#### 4. 设置 WSL2 为默认版本

```powershell
wsl --set-default-version 2
```

#### 5. 安装 Linux 发行版

```powershell
# 安装 Ubuntu 20.04
wsl --install -d Ubuntu-20.04
```

#### 6. 在 WSL2 中安装 Docker

进入 WSL2 Ubuntu 环境：

```bash
# 更新包管理器
sudo apt update

# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 将当前用户添加到 docker 组
sudo usermod -aG docker $USER

# 重新登录或执行
newgrp docker
```

#### 7. 安装 Docker Compose

```bash
# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose --version
```

#### 8. 部署 Dify

```bash
# 克隆项目（如果还没有）
git clone https://github.com/langgenius/dify.git
cd dify/docker

# 启动服务
docker-compose up -d
```

### 方案二：使用修改后的配置文件（备选方案）

如果无法使用 WSL2，可以尝试使用我们提供的 Windows 专用配置：

#### 1. 确保 Docker 运行在 Linux 容器模式

在 Docker Desktop 中：
- 右键点击系统托盘中的 Docker 图标
- 选择 "Switch to Linux containers"（如果当前是 Windows 容器模式）

#### 2. 配置 Docker 代理（如果需要）

创建或编辑 `%USERPROFILE%\.docker\config.json`：

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://your-proxy:port",
      "httpsProxy": "http://your-proxy:port",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
```

#### 3. 使用 Windows 专用配置文件

我们已经为您创建了 `docker-compose.windows.yaml` 文件，其中包含了所有必要的 `platform: linux/amd64` 配置。

#### 4. 运行启动脚本

双击运行 `start-windows.bat` 脚本，或在命令提示符中执行：

```cmd
cd /d D:\code\dify-titan\docker
start-windows.bat
```

#### 5. 手动启动（如果脚本失败）

```cmd
# 创建必要目录
mkdir volumes\app\storage
mkdir volumes\db\data
mkdir volumes\redis\data
mkdir volumes\weaviate
mkdir volumes\sandbox\dependencies
mkdir volumes\sandbox\conf
mkdir volumes\plugin_daemon

# 启动服务
docker-compose -f docker-compose.windows.yaml --profile weaviate up -d
```

## 网络和代理配置

### 1. Docker 守护进程代理配置

如果需要通过代理访问互联网，创建或编辑 `%PROGRAMDATA%\docker\config\daemon.json`：

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
  "proxies": {
    "http-proxy": "http://your-proxy:port",
    "https-proxy": "http://your-proxy:port",
    "no-proxy": "localhost,127.0.0.1,*.local"
  }
}
```

### 2. 环境变量配置

创建 `.env` 文件来自定义配置：

```env
# 代理设置
HTTP_PROXY=http://your-proxy:port
HTTPS_PROXY=http://your-proxy:port

# 数据库配置
POSTGRES_PASSWORD=your_secure_password
REDIS_PASSWORD=your_secure_password

# 其他配置
EXPOSE_NGINX_PORT=80
EXPOSE_NGINX_SSL_PORT=443
```

## 故障排除

### 1. 镜像拉取失败

```cmd
# 手动拉取镜像
docker pull --platform linux/amd64 langgenius/dify-api:1.4.2
docker pull --platform linux/amd64 langgenius/dify-web:1.4.2
docker pull --platform linux/amd64 postgres:15-alpine
docker pull --platform linux/amd64 redis:6-alpine
```

### 2. 端口冲突

检查端口占用：

```cmd
netstat -ano | findstr :80
netstat -ano | findstr :443
```

### 3. 权限问题

确保 Docker 有足够的权限访问文件系统：

```cmd
# 以管理员身份运行命令提示符
# 或者修改 Docker Desktop 的设置，允许访问相应的驱动器
```

### 4. 内存不足

确保系统有足够的内存（建议至少 8GB）：

```cmd
# 检查系统内存
wmic computersystem get TotalPhysicalMemory
```

## 验证部署

### 1. 检查服务状态

```cmd
docker-compose -f docker-compose.windows.yaml ps
```

### 2. 查看日志

```cmd
# 查看所有服务日志
docker-compose -f docker-compose.windows.yaml logs

# 查看特定服务日志
docker-compose -f docker-compose.windows.yaml logs api
docker-compose -f docker-compose.windows.yaml logs web
```

### 3. 访问服务

- Web 界面：http://localhost
- API 文档：http://localhost/console/api/docs

## 性能优化建议

1. **增加 Docker 内存限制**：在 Docker Desktop 设置中增加内存分配
2. **使用 SSD 存储**：将 Docker 数据目录放在 SSD 上
3. **配置镜像加速器**：使用国内的 Docker 镜像加速器
4. **定期清理**：定期清理未使用的镜像和容器

```cmd
# 清理未使用的资源
docker system prune -a
```

## 总结

推荐使用 **方案一（WSL2）**，因为它提供了更好的兼容性和性能。如果无法使用 WSL2，则可以尝试方案二，但可能会遇到一些兼容性问题。

无论使用哪种方案，关键是要确保 Docker 运行在 Linux 容器模式下，并且正确配置了网络代理（如果需要的话）。 