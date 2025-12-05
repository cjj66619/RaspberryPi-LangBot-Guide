# 📖 树莓派 QQ 机器人部署手册

**目标**：在树莓派 (4B/5) 上部署一套稳定、防风控、可接入 AI 的 QQ 机器人。
**核心组件**：

1.  **NapCat (Docker 版)**：负责登录 QQ，收发消息。
2.  **LangBot (Python 源码版)**：机器人的大脑，负责处理逻辑和对接 AI。

-----

## 🟢 第一阶段：基础环境准备

这一步是为了让树莓派能飞快下载软件，不再龟速。

### 1\. 换阿里源

默认的国外源在国内非常慢，必须换成阿里云（经作者亲测比清华源好用）。请**一行一行**复制进终端执行：

```bash
# 替换主软件源为阿里云
sudo sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list

# 替换安全更新源为阿里云
sudo sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list

# 替换树莓派官方源为阿里云
sudo sed -i 's/archive.raspberrypi.com/mirrors.aliyun.com\/raspberrypi/g' /etc/apt/sources.list.d/raspi.list

# 刷新生效 (这一步必须看到很多 aliyun 的网址在跑)
sudo apt update
```

### 2\. 安装 Docker 

Docker 作为容器是运行 NapCat 的必备工具。

```bash
# 一键安装 Docker 及其插件
sudo apt install docker.io docker-compose -y

# 设置开机自启 (防止重启后服务挂掉)
sudo systemctl enable docker
sudo systemctl start docker

# 给当前用户授权 (这样以后你就不用每次输 sudo 了)
# 注意：$USER 会自动替换成你的用户名
sudo usermod -aG docker $USER

# 刷新用户组权限 (立刻生效，不用重启)
newgrp docker
```

-----

## 🟡 第二阶段：部署 NapCat 

### 1\. 创建专属目录

我们需要一个地方存放机器人的配置文件。

```bash
# 创建文件夹结构 (-p 表示连同父目录一起创建)
mkdir -p ~/my-bot/napcat/config

# 进入这个目录
cd ~/my-bot

# 创建编排文件
nano docker-compose.yml
```

### 2\. 编写 `docker-compose.yml`

在编辑器中粘贴以下内容（这是容器的说明书）：

```yaml
version: '3.8'
services:
  napcat:
    image: mlikiowa/napcat-docker:latest  # 使用官方最新镜像
    container_name: napcat
    network_mode: bridge
    ports:
      - "6099:6099"   # 网页管理端口
      - "3001:3001"   # HTTP API 端口
    volumes:
      - ./napcat/config:/app/napcat/config  # 把配置挂载出来，方便修改
      - ./napcat/logs:/app/napcat/logs      # 把日志挂载出来，方便排查
    environment:
      - TZ=Asia/Shanghai  # 设置时区为中国，防止日志时间不对
    mac_address: 02:42:AC:11:00:02  # 固定网卡地址，防止腾讯认为你换设备了
    restart: always  # 崩溃或重启后自动复活
```

*保存方法：按 `Ctrl+O` -\> 回车 -\> `Ctrl+X`。*

### 3\. 启动容器

```bash
# -d 表示在后台静默运行
docker compose up -d
```

### 4\. 配置防风控 (关键步骤！)

为了不被腾讯踢下线，必须强制使用手表协议。

1.  **创建配置文件**：

    ```bash
    nano ~/my-bot/napcat/config/webui.json
    ```

2.  **写入以下内容** (直接粘贴)：

    ```json
    {
      "host": "0.0.0.0",
      "port": 6099,
      "token": "662023",           
      "loginRate": 3,
      "lang": "zh_CN",
      "protocol": "Watch",         
      "autoLoginAccount": "1283133303" 
    }
    ```

      * `token`: 设置为你自己的管理密码。
      * `protocol`: **必须是 "Watch"** (手表协议风控最松)。
      * `autoLoginAccount`: 填入你的机器人 QQ 号。

3.  **重启生效**：

    ```bash
    docker restart napcat
    ```

### 5\. 执行登录 (网页操作)

1.  电脑浏览器打开：`http://<树莓派IP>:6099/webui`
2.  输入 Token：`662023`
3.  点击 **“快速登录”** 。
4.  点击机器人头像（第一次进入需要扫码）。
5.  **验证**：看到头像亮起，显示“在线”，即成功。
6.  <img width="1962" height="1319" alt="image" src="https://github.com/user-attachments/assets/fa71bd63-8a94-4f8d-b0c5-8e8d87c1e2db" />


-----

## 🔵 第三阶段：部署 LangBot (机器人大脑)

*Docker 版 LangBot 暂不支持树莓派架构，所以我们要用 Python 源码运行。*

### 1\. 下载源码与环境

```bash
# 回到主目录
cd ~

# 使用国内镜像克隆源码 (解决 GitHub 连不上的问题)
git clone https://gitcode.com/RockChinQ/LangBot.git my-bot-brain

# 进入目录
cd my-bot-brain

# 安装 Python 虚拟环境工具
sudo apt install python3-venv -y

# 创建一个独立的 Python 环境 (防止搞乱系统)
python3 -m venv venv

# 激活环境 (激活后命令行前面会出现 (venv))
source venv/bin/activate

# 安装依赖 (使用阿里云镜像加速，飞快)
pip install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple/
pip install . -i https://mirrors.aliyun.com/pypi/simple/
```

### 2\. 设置开机自启 (Systemd 服务)

让 LangBot 像系统服务一样在后台运行，而不是关掉窗口就停了。

1.  **新建服务文件**：
    ```bash
    sudo nano /etc/systemd/system/langbot.service
    ```
2.  **写入内容**：
    ```ini
    [Unit]
    Description=LangBot Service
    After=network.target docker.service  # 只有联网且 Docker 启动后才启动它

    [Service]
    Type=simple
    User=cjj  # 这里必须是你树莓派的用户名
    WorkingDirectory=/home/cjj/my-bot-brain  # 工作目录
    # 指定用虚拟环境里的 Python 启动
    ExecStart=/home/cjj/my-bot-brain/venv/bin/python3 main.py
    Restart=always  # 挂了自动重启
    RestartSec=10   # 等10秒再重启

    [Install]
    WantedBy=multi-user.target
    ```
3.  **激活服务**：
    ```bash
    sudo systemctl daemon-reload  # 刷新配置
    sudo systemctl enable langbot # 设置开机自启
    sudo systemctl start langbot  # 立刻启动
    ```

-----

## 🟣 第四阶段：连接与 AI 配置

### 1\. 对接 (让大脑连接身体)

  * **LangBot 端** (`http://<树莓派IP>:5300`)：
      * 菜单 -\> **机器人** -\> **创建**。
      * 协议：**OneBot V11**。
      * 连接模式：**反向 WebSocket**。
      * 主机：`0.0.0.0`
      * 端口：`2280` (这是 LangBot 专门开给 QQ 连的门)。
      * 访问令牌不需要管
      * 点击保存。
      * <img width="2430" height="1323" alt="image" src="https://github.com/user-attachments/assets/6de82377-0292-4a8d-995a-bae875273764" />

  * **NapCat 端** (`http://<树莓派IP>:6099/webui`)：
      * 菜单 -\> **网络配置** -\> **WebSocket (反向)** -\> **添加**。
      * URL：`ws://127.0.0.1:2280/ws` (因为都在树莓派里，填本机 IP 即可)。
      * 启用并保存。
      * <img width="2449" height="1170" alt="image" src="https://github.com/user-attachments/assets/f717f5e8-d057-4a09-ac23-129e3f4325e0" />

      * **验证**：回到 LangBot 页面,刷新。

### 2\. 接入 AI 模型 (注入灵魂)

  * **LangBot 端** -\> **模型配置**：
      * 供应商：**SiliconFlow** (推荐) 或 OpenAI。
      * API Key：`sk-xxxx` (你的密钥)。
      * Base URL：`https://api.siliconflow.cn/v1` (如果是 SiliconFlow)。
      * 模型名称：`deepseek-ai/DeepSeek-V3`。
      * 点击“检查可用性” -\> 保存。
  * **LangBot 端** -\> **流水线**：
      * 找到 `default` 流水线。
      * **对话模型**：选择刚才添加的 AI 模型。
      * 保存。
      * <img width="921" height="366" alt="image" src="https://github.com/user-attachments/assets/82c304b4-e56e-401a-8b4d-07c1bc28afc4" />


-----

## 🛠️ 终极维护手册 (遇到问题查这里)

| 故障现象 | 诊断思路 | 修复命令 |
| :--- | :--- | :--- |
| **机器人不回消息** | 可能是程序卡死或掉线 | `sudo systemctl restart langbot` (重启大脑)<br>`sudo docker restart napcat` (重启QQ) |
| **网页打不开** | 检查服务是否活着 | `sudo systemctl status langbot` (看大脑状态)<br>`sudo docker ps` (看QQ状态) |
| **QQ 发不出图/被冻结** | 风控了，需要清除缓存重登 | 1. `rm ~/my-bot/napcat/config/qq.json`<br>2. `sudo docker restart napcat`<br>3. 去网页点“快速登录” |
| **无法修改配置文件** | 权限被锁定了 | `sudo chown -R $USER:$USER ~/my-bot` |
| **想看机器人日志** | 实时监控 | `sudo journalctl -u langbot -f` (按 Ctrl+C 退出) |
| **想看 QQ 日志** | 查登录报错 | `sudo docker logs napcat --tail 100 -f` |

-----
---

## ❤️ 致谢 (Acknowledgements)

本教程的实现离不开以下优秀的开源项目，向原作者致以崇高的敬意：

* 🧠 **大脑 (Logic)**: **[LangBot](https://github.com/RockChinQ/LangBot)** by [@RockChinQ](https://github.com/RockChinQ)
    * 一个强大的 LLM 原生即时通信机器人平台，支持多种模型和插件生态。
* 🐱 **身体 (Client)**: **[NapCatQQ](https://github.com/NapNeko/NapCatQQ)** by [@NapNeko](https://github.com/NapNeko)
    * 基于 NTQQ 的现代化无头 Bot 框架，稳定且高效。
* 🐳 **容器 (Docker)**: **[NapCat-Docker](https://github.com/NapNeko/NapCat-Docker)** by [@mlikiowa](https://github.com/mlikiowa)
    * 提供了开箱即用的 Docker 镜像支持。

**饮水思源**：如果你觉得本教程对你有帮助，请务必去给以上原作者的仓库点一个 ⭐️ **Star**！
