下面按你的 UOS20 专业版环境来讲。你的机器是海光 C86，属于 **x86_64 / amd64** 架构，所以 Docker 安装包应选 `amd64`。

总体建议：

```text
Docker：安装 Docker Engine + Compose Plugin，不建议优先装 Docker Desktop。
Node.js：建议使用 nvm 或 fnm 管理多版本 Node.js。
Docker 数据目录：放 ext4 盘，不要放 NTFS 盘。
```

Docker 官方 Debian 安装文档目前推荐通过 apt 仓库安装 `docker-ce`、`docker-ce-cli`、`containerd.io`、`docker-buildx-plugin`、`docker-compose-plugin`；同时官方也提醒要先卸载可能冲突的旧包，例如 `docker.io`、`docker-compose`、`podman-docker`、`containerd`、`runc`。([Docker Documentation][1])

---

# 一、安装前确认

## 1. 确认架构

```bash
uname -m
dpkg --print-architecture
```

你的机器正常应返回：

```text
x86_64
amd64
```

## 2. 确认 UOS 是否能使用 sudo

如果执行 `sudo` 提示权限不足，需要先开启 UOS 开发者模式。统信官方说明，开发者模式可获得 root 权限、执行 sudo 操作、安装未在应用商店上架的非签名应用；入口在“控制中心 → 开发者模式”，支持在线和离线两种方式。([faq.uniontech.com][2])

测试：

```bash
sudo -v
```

能正常输入密码并返回即可。

## 3. 确认 UOS 底层 Debian 版本

```bash
cat /etc/os-release
cat /etc/debian_version 2>/dev/null || true
lsb_release -a 2>/dev/null || true
```

UOS20 很多环境是 Debian 10 系衍生，Docker 官方当前文档主要列出 Debian 12 Bookworm 和 Debian 11 Bullseye 为支持版本；如果你的 UOS 是 Debian 10/Buster 系，可能需要使用旧版 Docker CE 仓库、UOS 源里的 `docker.io`，或离线安装 deb 包。([Docker Documentation][1])

---

# 二、推荐安装方式：Docker Engine 官方 apt 仓库

## 1. 卸载旧版本或冲突包

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
    sudo apt-get remove -y "$pkg" 2>/dev/null || true
done
```

这一步不会自动删除 `/var/lib/docker` 下已有镜像、容器、卷和网络；Docker 官方文档也说明这些数据不会随着卸载包自动删除。([Docker Documentation][1])

---

## 2. 安装依赖

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

---

## 3. 添加 Docker GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

---

## 4. 判断应该使用哪个 Debian codename

先执行：

```bash
cat /etc/debian_version 2>/dev/null || true
```

大致对应关系：

| `/etc/debian_version` | Docker 仓库 codename        |
| --------------------- | ------------------------- |
| `12.x`                | `bookworm`                |
| `11.x`                | `bullseye`                |
| `10.x`                | `buster`，属于旧环境，可能需要旧包或镜像站 |
| 其他 / 无法判断             | 先不要盲目添加源                  |

你可以这样自动判断：

```bash
DEBIAN_VER="$(cat /etc/debian_version 2>/dev/null | cut -d. -f1)"

case "$DEBIAN_VER" in
  12) DOCKER_CODENAME="bookworm" ;;
  11) DOCKER_CODENAME="bullseye" ;;
  10) DOCKER_CODENAME="buster" ;;
  *)  DOCKER_CODENAME="" ;;
esac

echo "$DOCKER_CODENAME"
```

如果输出为空，先不要继续，说明你的系统版本识别不标准。

---

## 5. 添加 Docker apt 源

如果上一步输出是 `bookworm` 或 `bullseye`，推荐用官方源：

```bash
DOCKER_CODENAME="bullseye"   # 按你机器实际结果改成 bookworm / bullseye / buster

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian ${DOCKER_CODENAME} stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

然后：

```bash
sudo apt-get update
```

如果你在国内网络下访问 Docker 源慢，可以把上面的地址替换为清华 TUNA Docker CE 镜像地址。清华镜像站说明 Docker CE 镜像可以按官方文档使用，只需把 `download.docker.com` 替换为 `mirrors.tuna.tsinghua.edu.cn/docker-ce`。([mirrors.tuna.tsinghua.edu.cn][3])

例如：

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian ${DOCKER_CODENAME} stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

---

## 6. 安装 Docker Engine

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

启动并设为开机自启：

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

验证：

```bash
docker --version
docker compose version
sudo docker run --rm hello-world
```

成功时会看到类似：

```text
Hello from Docker!
```

---

# 三、建议把 Docker 数据目录放到 ext4 盘

你前面规划过：

```text
/data/uos/40_Containers_VM/docker
```

非常适合作为 Docker 数据目录。不要把 Docker 数据目录放 NTFS。Docker 镜像、容器、卷、overlay2、容器日志都需要 Linux 文件系统语义，NTFS 容易出权限、软链接、性能和一致性问题。

## 1. 停止 Docker

```bash
sudo systemctl stop docker
sudo systemctl stop containerd
```

## 2. 创建目录

```bash
sudo mkdir -p /data/uos/40_Containers_VM/docker
```

## 3. 配置 `/etc/docker/daemon.json`

如果文件不存在就新建：

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

说明：

```text
data-root：Docker 镜像、容器、卷的数据目录
log-driver/log-opts：限制容器日志无限膨胀
```

## 4. 重启 Docker

```bash
sudo systemctl daemon-reload
sudo systemctl start containerd
sudo systemctl start docker
```

验证：

```bash
docker info | grep -i "Docker Root Dir"
```

应看到：

```text
Docker Root Dir: /data/uos/40_Containers_VM/docker
```

---

# 四、让普通用户免 sudo 使用 Docker

默认情况下，`docker` 命令需要 root 权限。可以把当前用户加入 `docker` 组：

```bash
sudo usermod -aG docker "$USER"
```

然后注销重新登录，或执行：

```bash
newgrp docker
```

验证：

```bash
docker run --rm hello-world
```

注意：加入 `docker` 组后，当前用户基本等同于具备较高系统控制能力，因为 Docker 可以挂载宿主机目录、运行特权容器。只给可信账号加这个组。

---

# 五、配置 Docker 走 v2rayA 代理

你之前 v2rayA 的本地端口是：

```text
HTTP：127.0.0.1:20171
SOCKS5：127.0.0.1:20170
```

Docker 拉镜像时，不是你的 shell 进程直接访问网络，而是 `dockerd` 守护进程访问网络。因此你在终端里写：

```bash
export http_proxy=http://127.0.0.1:20171
```

通常不会影响 Docker pull。Docker 官方文档专门说明了 Docker daemon 的代理配置，守护进程需要单独配置代理。([Docker Documentation][4])

## 方法 A：systemd 方式，兼容性好

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

创建代理配置：

```bash
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:20171"
Environment="HTTPS_PROXY=http://127.0.0.1:20171"
Environment="NO_PROXY=localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
EOF
```

使配置生效：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查：

```bash
systemctl show --property=Environment docker
```

然后测试：

```bash
docker pull hello-world
```

## 方法 B：daemon.json 方式

Docker 官方文档也提供了在 `daemon.json` 中配置代理的方式，但要注意它适用于 Docker Engine，不适用于 Docker Desktop；Docker Desktop 会忽略 `daemon.json` 中的代理配置。([Docker Documentation][4])

如果你已经有 `daemon.json`，要合并配置，不要覆盖。示例：

```json
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "proxies": {
    "http-proxy": "http://127.0.0.1:20171",
    "https-proxy": "http://127.0.0.1:20171",
    "no-proxy": "localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
  }
}
```

重启：

```bash
sudo systemctl restart docker
```

你的环境里，我更建议先用 **systemd 方式**，排障更直观。

---

# 六、UOS20 / Debian 10 系统安装失败时怎么办

如果你执行：

```bash
sudo apt-get update
```

出现类似：

```text
The repository does not have a Release file
```

或者安装 `docker-ce` 找不到包，说明当前 Docker 官方源不完全适配你的 UOS 版本。

可以按这个顺序处理：

## 方案 1：使用 UOS / Deepin 自带仓库里的 Docker

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose
```

然后：

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo docker run --rm hello-world
```

优点：

```text
兼容 UOS 概率高
安装简单
```

缺点：

```text
版本可能较旧
Compose 可能还是老的 docker-compose v1
```

## 方案 2：使用 Docker CE 旧版 buster 仓库

如果确认 UOS20 是 Debian 10/Buster 系，可尝试：

```bash
DOCKER_CODENAME="buster"
```

然后按前面的 Docker 源方式安装。

但这属于旧系统兼容路线，后期升级要谨慎。

## 方案 3：离线下载 deb 包安装

Docker 官方文档也说明，如果无法使用 apt 仓库，可以下载对应发行版的 deb 包手工安装，但升级也需要手动管理。([Docker Documentation][1])

适合：

```text
内网环境
apt 源不可用
企业交付现场
需要固定 Docker 版本
```

---

# 七、安装后建议装几个常用工具

```bash
sudo apt-get install -y git curl wget jq tree htop btop ncdu
```

容器开发常用：

```bash
sudo apt-get install -y make gcc g++ python3 python3-pip python3-venv
```

如果你后面要跑数据库、Nginx、Redis、PostgreSQL、MySQL、MinIO、Nacos、OpenWebUI 之类服务，Docker 会非常适合你的售前 PoC / 演示环境。

---

# 八、Node.js 多版本管理工具

有，而且非常建议你使用。你现在已经安装了 Node.js，但后面用 Windsurf、前端工具、AI 工具、文档工具时，经常会遇到：

```text
项目 A 要 Node 18
项目 B 要 Node 20
项目 C 要 Node 22
某些老项目只能用 Node 16
```

所以不要长期只依赖系统 apt 安装的 Node.js。

主流选择有三个：

| 工具      | 推荐程度 | 特点                              |
| ------- | ---: | ------------------------------- |
| `nvm`   |    高 | 最经典、兼容性最好、资料最多                  |
| `fnm`   |    高 | 速度快，Rust 编写，体验轻量                |
| `Volta` |   中高 | 更适合团队项目固定 Node/npm/yarn/pnpm 版本 |

nvm 官方说明它是 Node.js 版本管理器，按用户安装、按 shell 调用，支持 POSIX shell；使用方式就是 `nvm install`、`nvm use` 来安装和切换版本。([GitHub][5])
fnm 官方项目说明它是快速、简单的 Node.js 版本管理器，Linux 下可用安装脚本安装。([GitHub][6])
Volta 官方定位是 JavaScript 工具链管理器，可以固定项目的 Node、npm、yarn 等版本，适合确保项目成员使用一致工具链。([Volta Documentation][7])

---

# 九、我建议你选哪个 Node 管理工具

你的情况：

```text
售前咨询为主
偶尔写 Python / Java / C/C++
希望 Windsurf 作为主要工作工具
可能使用前端工具、AI 工具、文档工具
不是专职前端团队开发
```

我的建议：

```text
首选 nvm：稳定、资料多、排障容易。
如果你追求启动速度和简洁体验，再选 fnm。
暂时不建议一开始用 Volta，除非你有很多 package.json 项目需要锁定团队 Node 版本。
```

---

# 十、安装 nvm

## 1. 如果你已经通过 apt 安装了 Node.js

先看当前 Node 来源：

```bash
which node
node -v
npm -v
dpkg -l | grep -E 'nodejs|npm'
```

如果是 apt 安装的，可以保留，也可以卸载。为了避免混乱，我建议卸载系统 Node.js，然后交给 nvm 管理：

```bash
sudo apt-get remove -y nodejs npm
sudo apt-get autoremove -y
```

如果你担心影响现有工具，可以先不卸载，nvm 生效后一般会通过 PATH 优先级覆盖系统 Node。

---

## 2. 安装 nvm

官方安装方式：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

如果你用 bash，执行：

```bash
source ~/.bashrc
```

验证：

```bash
command -v nvm
```

应该返回：

```text
nvm
```

---

## 3. 安装 Node LTS

```bash
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'
```

验证：

```bash
node -v
npm -v
which node
```

`which node` 应该类似：

```text
/home/你的用户名/.nvm/versions/node/vxx.x.x/bin/node
```

---

## 4. 安装多个 Node 版本

```bash
nvm install 18
nvm install 20
nvm install 22
```

查看：

```bash
nvm ls
```

切换：

```bash
nvm use 20
node -v
```

设置默认：

```bash
nvm alias default 20
```

---

## 5. 每个项目固定 Node 版本

在项目目录下创建：

```bash
echo "20" > .nvmrc
```

进入项目后执行：

```bash
nvm use
```

如果该版本没装：

```bash
nvm install
```

建议你的 Windsurf 项目这样管理：

```text
/data/uos/20_Windsurf_Workspace/presale/xxx_project/.nvmrc
/data/uos/20_Windsurf_Workspace/python/xxx_project/.nvmrc
```

---

# 十一、安装 fnm，作为 nvm 的替代方案

如果你更想用轻量快速的工具，可以用 fnm。

安装：

```bash
curl -fsSL https://fnm.vercel.app/install | bash
```

然后重新打开终端，或：

```bash
source ~/.bashrc
```

安装 Node LTS：

```bash
fnm install --lts
fnm use --lts
fnm default lts-latest
```

查看：

```bash
fnm list
node -v
npm -v
```

项目里可以使用 `.node-version`：

```bash
echo "20" > .node-version
fnm use
```

`fnm` 和 `nvm` 不建议同时长期混用。二选一即可。

---

# 十二、npm / pnpm / yarn 建议

Node 18+ 通常内置 `corepack`，建议启用：

```bash
corepack enable
```

安装 pnpm：

```bash
corepack prepare pnpm@latest --activate
pnpm -v
```

以后项目尽量用：

```text
npm：通用
pnpm：速度快、节省空间，推荐
yarn：已有项目要求时再用
```

如果你做 Windsurf + AI 工具 + 前端文档工具，`pnpm` 很实用。

---

# 十三、你的最终推荐配置

## Docker

```text
安装方式：Docker Engine + Compose Plugin
数据目录：/data/uos/40_Containers_VM/docker
代理方式：systemd 给 dockerd 配置 127.0.0.1:20171
验证命令：docker run --rm hello-world
```

## Node.js

```text
版本管理：nvm
默认版本：Node LTS
项目固定：.nvmrc
包管理：npm + pnpm
Windsurf 项目目录：/data/uos/20_Windsurf_Workspace
```

## 不建议

```text
不要把 Docker data-root 放 NTFS。
不要长期混用 apt nodejs、nvm、fnm、volta。
不要把 node_modules 放 NTFS。
不要把 Docker Desktop 当作 UOS 首选方案。
不要用 root 用户日常开发 Node 项目。
```

最适合你的路线是：

```text
Docker 负责 PoC / 演示 / 服务环境
nvm 负责多版本 Node.js
Windsurf 负责代码和自动化
ext4 盘负责开发、缓存、容器和编译
NTFS 盘只负责资料、模板、归档和跨 Windows/UOS 共享
```

[1]: https://docs.docker.com/engine/install/debian/?utm_source=chatgpt.com "Install Docker Engine on Debian | Docker Docs"
[2]: https://faq.uniontech.com/search?keywords=%E5%BC%80%E5%8F%91%E8%80%85%E6%A8%A1%E5%BC%8F&utm_source=chatgpt.com "搜索 | 统信软件-知识分享平台"
[3]: https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/?utm_source=chatgpt.com "Docker Ce | 镜像站使用帮助 | 清华大学开源软件镜像站 ..."
[4]: https://docs.docker.com/engine/daemon/proxy/?utm_source=chatgpt.com "Daemon proxy configuration | Docker Docs"
[5]: https://github.com/nvm-sh/nvm?utm_source=chatgpt.com "GitHub - nvm-sh/nvm: Node Version Manager - POSIX-compliant bash script ..."



----
你这个结果说明：**Docker 已经安装成功，问题出在 Docker daemon 拉取镜像时无法访问镜像仓库。**

错误点是：

```text id="z0h4fp"
Unable to find image 'hello-world:latest' locally
docker: error response from daemon: Get "https://xxxx:443": i/o timeout
```

含义是：

```text id="0hhl2a"
本地没有 hello-world 镜像
Docker daemon 尝试访问远端 registry
访问 https://xxxx:443 超时
```

这不是 `buster` 本身的问题，也不是 Docker 没装好，而是 **dockerd 的网络没有通**。你终端能翻墙，不代表 Docker 能翻墙，因为 `docker pull` 是由后台的 `dockerd` 守护进程访问网络，Docker 官方文档也明确说明 Docker daemon 需要单独配置代理。([Docker Documentation][1])

---

# 一、先判断是 Docker Hub 直连问题，还是代理没给 dockerd 生效

先执行：

```bash
curl -I https://registry-1.docker.io/v2/
```

如果网络正常，常见返回应类似：

```text
HTTP/1.1 401 Unauthorized
```

`401` 是正常的，说明能连到 Docker Hub，只是未认证。

如果直接超时，再测试走 v2rayA：

```bash
curl -I -x http://127.0.0.1:20171 https://registry-1.docker.io/v2/
```

再测：

```bash
curl -I -x http://127.0.0.1:20172 https://registry-1.docker.io/v2/
```

判断：

| 测试结果                            | 说明                         |
| ------------------------------- | -------------------------- |
| 不带代理超时，带 `20171/20172` 返回 `401` | Docker daemon 需要配置代理       |
| 带 `20171/20172` 也超时             | v2rayA 的 HTTP 代理端口或节点本身有问题 |
| `20171` 失败，`20172` 成功           | dockerd 代理用 `20172`        |
| 两个都成功                           | 推荐 dockerd 用 `20171`，规则更简单 |

你之前 `20171` 访问 Google HTTPS 出现过 OpenSSL 错误，所以这里必须实际测试 Docker Hub。不要直接假设 `20171` 一定可用。

---

# 二、给 Docker daemon 配置 v2rayA 代理

假设你测试后发现这个能通：

```bash
curl -I -x http://127.0.0.1:20171 https://registry-1.docker.io/v2/
```

那就给 Docker daemon 配置 `20171`。

如果 `20172` 才能通，把下面的 `20171` 改成 `20172`。

执行：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

创建代理配置：

```bash
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:20171"
Environment="HTTPS_PROXY=http://127.0.0.1:20171"
Environment="NO_PROXY=localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
EOF
```

重新加载并重启 Docker：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

确认代理是否已经注入到 dockerd：

```bash
systemctl show --property=Environment docker
```

应看到类似：

```text
Environment=HTTP_PROXY=http://127.0.0.1:20171 HTTPS_PROXY=http://127.0.0.1:20171 ...
```

然后再测：

```bash
docker pull hello-world
docker run --rm hello-world
```

Docker 官方文档说明，Docker daemon 的代理可通过 systemd 环境变量或 daemon 配置文件设置；在 Linux Engine 环境下，这正是处理 `docker pull` 访问外部 registry 的正确位置。([Docker Documentation][1])

---

# 三、同时建议配置 Docker 镜像加速

即使代理可用，Docker Hub 在国内网络下也经常慢。Docker 官方文档说明，可以通过 `/etc/docker/daemon.json` 的 `registry-mirrors` 配置 registry mirror。([Docker Documentation][2])

你之前已经建议过 Docker 数据目录放：

```text
/data/uos/40_Containers_VM/docker
```

所以建议你的 `/etc/docker/daemon.json` 这样写：

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查配置：

```bash
docker info | grep -A 10 -E "Docker Root Dir|Registry Mirrors"
```

应看到：

```text
Docker Root Dir: /data/uos/40_Containers_VM/docker
Registry Mirrors:
 https://docker.m.daocloud.io/
```

再试：

```bash
docker pull hello-world
docker run --rm hello-world
```

注意：国内公共镜像源可用性会变化。如果你有阿里云账号，更稳的是去阿里云容器镜像服务里开通自己的专属 Docker Hub 镜像加速地址，然后替换 `registry-mirrors`。

---

# 四、如果还是超时，按这个顺序定位

## 1. 看 Docker 实际访问的是哪个域名

你报错里写的是：

```text
https://xxxx:443
```

请重点看 `xxxx` 是什么。

常见有：

```text
registry-1.docker.io
auth.docker.io
production.cloudflare.docker.com
docker.m.daocloud.io
某个你配置的 mirror 域名
```

如果是 `docker.m.daocloud.io` 超时，说明镜像源不可达，换 mirror 或走代理。

如果是 `registry-1.docker.io` / `auth.docker.io` 超时，说明 Docker Hub 直连失败，需要代理。

---

## 2. 用 curl 分别测试这些域名

```bash
curl -I https://registry-1.docker.io/v2/
curl -I https://auth.docker.io/
curl -I https://production.cloudflare.docker.com/
```

再走代理测试：

```bash
curl -I -x http://127.0.0.1:20171 https://registry-1.docker.io/v2/
curl -I -x http://127.0.0.1:20171 https://auth.docker.io/
curl -I -x http://127.0.0.1:20171 https://production.cloudflare.docker.com/
```

如果 `20171` 不行，换：

```bash
curl -I -x http://127.0.0.1:20172 https://registry-1.docker.io/v2/
```

哪个端口 curl 能通，Docker daemon 就用哪个端口。

---

## 3. 看 Docker 日志

```bash
sudo journalctl -u docker -e --no-pager
```

实时观察：

```bash
sudo journalctl -u docker -f
```

另一个终端执行：

```bash
docker pull hello-world
```

日志里如果看到：

```text
proxyconnect tcp
connection refused
i/o timeout
no such host
TLS handshake timeout
```

对应判断：

| 日志                                                     | 可能原因                      |
| ------------------------------------------------------ | ------------------------- |
| `proxyconnect tcp 127.0.0.1:20171: connection refused` | v2rayA 没启动或端口不对           |
| `i/o timeout`                                          | dockerd 未走代理，或代理后链路超时     |
| `no such host`                                         | DNS 问题                    |
| `TLS handshake timeout`                                | 代理链路慢、节点异常、Docker Hub 被阻断 |
| `403/429`                                              | 镜像源限流或 Docker Hub 限制      |

---

# 五、你的推荐最终配置

我建议你当前用这个组合：

```text
Docker daemon 代理：v2rayA HTTP 端口 20171 或 20172
Docker data-root：/data/uos/40_Containers_VM/docker
Docker registry mirror：先配一个可用镜像源
Docker pull 测试：hello-world、alpine、nginx
```

最终配置文件如下。

## `/etc/systemd/system/docker.service.d/http-proxy.conf`

如果 `20171` 可用：

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:20171"
Environment="HTTPS_PROXY=http://127.0.0.1:20171"
Environment="NO_PROXY=localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
```

如果 `20172` 可用，就改成：

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:20172"
Environment="HTTPS_PROXY=http://127.0.0.1:20172"
Environment="NO_PROXY=localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
```

## `/etc/docker/daemon.json`

```json
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

应用：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证：

```bash
systemctl show --property=Environment docker
docker info | grep -A 10 -E "Docker Root Dir|Registry Mirrors"
docker pull hello-world
docker run --rm hello-world
```

---

# 六、一个很关键的提醒

你在终端里执行：

```bash
export http_proxy=http://127.0.0.1:20171
export https_proxy=http://127.0.0.1:20171
```

只影响当前 shell 里的 `curl`、`wget` 等程序，**不影响 Docker daemon**。

所以这个能成功：

```bash
curl -x http://127.0.0.1:20171 https://registry-1.docker.io/v2/
```

不代表这个一定成功：

```bash
docker pull hello-world
```

必须给 `dockerd` 配 systemd 代理。

---

你现在下一步直接做这三条最关键：

```bash
curl -I -x http://127.0.0.1:20171 https://registry-1.docker.io/v2/
curl -I -x http://127.0.0.1:20172 https://registry-1.docker.io/v2/
systemctl show --property=Environment docker
```

如果 `20171/20172` 其中一个返回 `401 Unauthorized`，就把 Docker daemon 的代理端口改成那个端口，再重启 Docker。

[1]: https://docs.docker.com/engine/daemon/proxy/?utm_source=chatgpt.com "Daemon proxy configuration | Docker Docs"
[2]: https://docs.docker.com/docker-hub/image-library/mirror/?utm_source=chatgpt.com "Mirror the Docker Hub library | Docker Docs - Docker Documentation"

[6]: https://github.com/Schniz/fnm?utm_source=chatgpt.com "GitHub - Schniz/fnm: Fast and simple Node.js version manager, built ..."
[7]: https://www.voltajs.com/zh/guide/getting-started.html?utm_source=chatgpt.com "Volta 快速安装配置和入门指南 | Volta"
44

--------------------
有，但要注意：**国内镜像不是把 `https://registry-1.docker.io/v2/` 直接替换成另一个 URL 去测 curl，而是配置 Docker 的 `registry-mirrors`，或者拉镜像时改写镜像名前缀。**

`registry-1.docker.io` 是 Docker Hub 的官方 Registry API 入口。Docker 官方推荐的镜像方式是在 `/etc/docker/daemon.json` 里配置：

```json
{
  "registry-mirrors": ["https://<my-docker-mirror-host>"]
}
```

然后重启 Docker daemon。([Docker Documentation][1])

# 1. 目前更建议你用哪个国内源？

结合现在的情况，我建议你优先用：

```text
https://docker.m.daocloud.io
```

DaoCloud 官方公开镜像说明中给出的 Docker 配置方式就是：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ]
}
```

并且它支持 Docker Hub，也支持通过前缀方式拉取，例如 `m.daocloud.io/docker.io/library/nginx`。([GitHub][2])

不建议优先用这些老地址：

```text
https://docker.mirrors.ustc.edu.cn
https://registry.docker-cn.com
http://hub-mirror.c.163.com
https://dockerhub.azk8s.cn
```

例如中科大镜像站官方页面已经写明“所有镜像缓存已暂停服务”，并说明 Docker Hub 镜像缓存服务已关闭。([USTC开源软件镜像][3])

阿里云也可以用，但它现在有重要限制：阿里云官方文档说明 ACR 镜像加速“目前已停止同步最新镜像”，如果遇到镜像无法拉取或 `latest` 不是最新，建议改用 ACR 订阅海外源镜像或全球加速等方案。([阿里云帮助中心][4])

# 2. 你的 UOS Docker 推荐配置

编辑：

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

建议写成：

```json
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

然后重启 Docker：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

检查是否生效：

```bash
docker info | grep -A 10 -E "Docker Root Dir|Registry Mirrors"
```

应该能看到类似：

```text
Docker Root Dir: /data/uos/40_Containers_VM/docker
Registry Mirrors:
 https://docker.m.daocloud.io/
```

然后测试：

```bash
docker pull hello-world
docker run --rm hello-world
```

# 3. 如果 `docker pull hello-world` 仍然超时

可以直接用 DaoCloud 的“前缀方式”拉取。DaoCloud 文档里推荐的方式是给原镜像名前面加 `m.daocloud.io/`，例如 `docker.io/library/busybox` 变成 `m.daocloud.io/docker.io/library/busybox`。([GitHub][2])

你可以试：

```bash
docker pull m.daocloud.io/docker.io/library/hello-world:latest
```

然后打标签成本地常规名字：

```bash
docker tag m.daocloud.io/docker.io/library/hello-world:latest hello-world:latest
```

再运行：

```bash
docker run --rm hello-world
```

也可以试 DaoCloud 的 Docker Hub 前缀替换方式：

```bash
docker pull docker.m.daocloud.io/library/hello-world:latest
```

如果这个成功，也可以：

```bash
docker tag docker.m.daocloud.io/library/hello-world:latest hello-world:latest
docker run --rm hello-world
```

# 4. 镜像源和 v2rayA 代理是否还需要同时配？

建议同时保留。

你的最佳组合是：

```text
Docker registry-mirrors：解决 Docker Hub 拉取慢/超时
Docker daemon 代理：兜底解决认证、跳转、冷门镜像、非 Docker Hub 源
```

也就是：

```text
daemon.json 配 registry-mirrors
systemd drop-in 配 HTTP_PROXY / HTTPS_PROXY
```

因为有些镜像源是懒加载缓存，首次请求某些冷门镜像时，镜像站自己需要回源；有些镜像标签也可能延迟。DaoCloud 公开说明其缓存采用懒加载机制，缓存内容保留 90 天，并建议使用明确版本号 tag，`latest` 这种标签可能存在更新延迟。([GitHub][2])

# 5. 给你一套最终命令

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "data-root": "/data/uos/40_Containers_VM/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker

docker info | grep -A 10 -E "Docker Root Dir|Registry Mirrors"

docker pull hello-world
docker run --rm hello-world
```

如果还失败，直接用：

```bash
docker pull m.daocloud.io/docker.io/library/hello-world:latest
docker tag m.daocloud.io/docker.io/library/hello-world:latest hello-world:latest
docker run --rm hello-world
```

# 6. 判断是否配置成功

成功配置后：

```bash
docker info
```

里应该出现：

```text
Registry Mirrors:
 https://docker.m.daocloud.io/
```

但这只说明配置加载成功，不代表每个镜像都一定能拉。真正验证还是：

```bash
docker pull hello-world
docker pull alpine:3.20
docker pull nginx:1.27
```

以后建议你尽量不要拉 `latest`，而是使用明确版本号，例如：

```bash
docker pull nginx:1.27
docker pull redis:7.4
docker pull postgres:16
docker pull python:3.12-slim
```

这比 `latest` 更稳定，也更适合你后面做售前 PoC、演示环境和 Windsurf 开发环境。

[1]: https://docs.docker.com/docker-hub/image-library/mirror/ "Mirror the Docker Hub library | Docker Docs"
[2]: https://github.com/DaoCloud/public-image-mirror "GitHub - DaoCloud/public-image-mirror: 很多镜像都在国外。比如 gcr 。国内下载很慢，需要加速。致力于提供连接全世界的稳定可靠安全的容器镜像服务。 · GitHub"
[3]: https://mirrors.ustc.edu.cn/help/dockerhub.html "Docker Hub - USTC Mirror Help"
[4]: https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images "
    配置官方镜像加速器加速拉取Docker Hub镜像-容器镜像服务-阿里云
  "



-----
