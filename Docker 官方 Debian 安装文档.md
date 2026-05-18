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
[6]: https://github.com/Schniz/fnm?utm_source=chatgpt.com "GitHub - Schniz/fnm: Fast and simple Node.js version manager, built ..."
[7]: https://www.voltajs.com/zh/guide/getting-started.html?utm_source=chatgpt.com "Volta 快速安装配置和入门指南 | Volta"
