# VPS安装Docker：从系统准备到跑起第一个容器的完整流程，KVM 机型和内存怎么选一次说清

在一台全新 VPS 上装 Docker，走官方软件源的完整流程大概十分钟，命令都能直接复制粘贴。真正让人翻车的反而是两件安装之外的事：机器是不是 KVM 架构，以及有没有顺手装了 Ubuntu 自带的旧版 docker.io 包。这篇文章从选机器讲到跑起第一个真实容器，中间把内存大小、日志上限、swap 这些小内存机器才会遇到的问题一并处理掉。

> **赶时间的版本**：机型选 KVM、系统用 Ubuntu 22.04/24.04、软件源走 Docker 官方源，三件事做对就成功了一大半。VPS 还没着落的话，RackNerd 的年付 KVM 套餐长期十几美元起，1GB 内存款够跑几个轻量容器——👉 [查看 RackNerd 当前促销套餐](https://bit.ly/my-rn)。

## 动手之前，先确认 VPS 满足三个条件

能不能顺利装 Docker，下单前看三处就够：虚拟化架构、操作系统版本、内存和硬盘。任何一处不达标，装是能装，用起来一定别扭。

### 架构必须是 KVM

Docker 依赖 namespace、cgroup、overlayfs 这些内核特性，需要在虚拟机里对内核有完整控制权。KVM 是完整虚拟化，每台机器有自己独立的内核，装 Docker 没有任何障碍；OpenVZ、LXC 这类和宿主机共享内核的“伪 VPS”，Docker 轻则装不上，重则装上了但运行行为诡异。已经有机器的话，一条命令就能查：


systemd-detect-virt


输出 kvm 就放心往下走；输出 openvz 或 lxc，直接换机器，别浪费时间排查。买之前拿不准，就认准明确标注 KVM 的商家——RackNerd 全线 VPS 都是 KVM，这也是它出现在这篇文章里的主要原因。👉 [开通一台 KVM 架构的 RackNerd VPS](https://bit.ly/my-rn)。

### 系统选 Ubuntu 22.04 或 24.04 LTS

Docker 官方源对这两个 LTS 版本支持最顺，下文所有命令在两个版本上完全通用。Debian 12 也能装，只是软件源地址要换成 debian 目录；不想逐行敲命令的话，官方一键脚本会自动识别发行版并装好全部组件：


curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh


本文主线还是按手动流程走，每一步看得见，出问题时知道去哪查。

### 内存 1GB 是底线，硬盘 20GB 起步

Docker 引擎本身只吃两三百 MB 内存，开销大头在容器上。跑一两个轻量服务（博客、反向代理、监控探针、机器人），1GB 够用；要同时带数据库和应用容器，2GB 以上才从容。硬盘方面，Ubuntu 系统本身占 3–4GB，剩下全部留给镜像和构建缓存，Docker 的旧镜像积累速度比多数人预想的快，20GB 是比较实际的起点。

## RackNerd 套餐参考与选择规则

入门自托管用途，多数人选 1GB 或 2.5GB 两档就够了。RackNerd 是一家预算向的美国 VPS 商家，机房覆盖洛杉矶、圣何塞、达拉斯、芝加哥、纽约等美国城市，另有阿姆斯特丹节点，常年挂着一批年付促销款 KVM 套餐，下面是常见档位的参考配置：

| 套餐 | 配置（vCPU / 内存 / SSD） | 月流量 | 参考年付价 | 购买链接 |
| --- | --- | --- | --- | --- |
| 1GB KVM | 1 核 / 1GB / 20GB | 1.5TB | 约 $11 | [ 查看实时价格](https://bit.ly/my-rn) |
| 2.5GB KVM | 2 核 / 2.5GB / 40GB | 4TB | 约 $22 | [ 查看实时价格](https://bit.ly/my-rn) |
| 3GB KVM | 2 核 / 3GB / 45GB | 4.5TB | 约 $26 | [ 查看实时价格](https://bit.ly/my-rn) |
| 3.5GB KVM | 2 核 / 3.5GB / 50GB | 5TB | 约 $30 | [ 查看实时价格](https://bit.ly/my-rn) |
| 5GB KVM | 3 核 / 5GB / 65GB | 7TB | 约 $40 | [ 查看实时价格](https://bit.ly/my-rn) |

两点提醒：同一套餐在不同机房价格会差几美元，洛杉矶通常是最便宜的选择之一；黑五、新年这类节点还会有限时活动价，表中数字以订购页实时显示为准。端口速率 1Gbps，流量按月计。除了这些常驻促销款，活动页偶尔还有更低配或更高配的限量套餐。

按用途选的规则很简单：只跑两三个轻量容器，1GB 够；带数据库的建站（WordPress 加 MySQL 这类）从 2.5GB 起步；打算长期叠好几个 Compose 项目，直接看 3.5GB 或 5GB。档位之间差价一年也就十几美元，省下的是以后升级重装的麻烦。👉 [查看 RackNerd 各机房与套餐的当前价格](https://bit.ly/my-rn)。

## 第一步：更新系统，清理旧包

拿到新机器，SSH 登进去先把系统补丁打满，顺便确认没有残留的旧版 Docker 包：


sudo apt update && sudo apt upgrade -y


如果内核有更新，`sudo reboot` 重启一次再继续。RackNerd 的 Ubuntu 模板是干净的，下面的清理命令通常扫不出东西，但跑一遍无害：


for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove -y $pkg; done


## 第二步：添加官方源，一条命令装齐

Ubuntu 自带的 docker.io 包版本落后官方仓库几个月，而且不带 Compose v2 插件，装它后面大概率要返工。正确顺序是先加 Docker 官方源，再装五个包。

先装基础依赖并导入官方 GPG 密钥：


sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc


写入软件源，系统版本代号会自动获取，22.04 和 24.04 都适用：


echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update


验证源是否生效：


apt-cache policy docker-ce


Candidate 显示 5:27.x 或更新的版本号就对了。如果列出来的是 Ubuntu 归档里的旧版本，说明源没加成功，回头检查上一步。

接着安装，一条命令五个包全到位：


sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


五个包分别是守护进程、命令行客户端、底层容器运行时、新版镜像构建器和 Compose v2 插件。注意 Compose v2 的写法是 `docker compose`（分开两个词），老教程里带连字符的 `docker-compose` 是已经过时的独立工具，看到可以绕开。

设为开机自启并立即启动：


sudo systemctl enable --now docker
systemctl status docker


看到 active (running) 就成了。

## 第三步：免 sudo 使用，跑一次验证

把当前用户加进 docker 组，以后每条命令就不用都带 sudo：


sudo usermod -aG docker $USER


退出 SSH 重新登录让组生效，然后跑一次官方验证：


docker run --rm hello-world
docker --version
docker compose version


看到 "Hello from Docker!" 就说明整套安装是通的，`--rm` 会在容器退出后自动删掉它，不留垃圾。

> 安全提醒：docker 组的权限等于 root。组内用户可以把宿主机文件系统挂进容器随意修改，只给真正拥有服务器管理权的人加组。

顺带说一个和机房位置有关的差异：RackNerd 这类海外节点拉取 Docker Hub 镜像是直连的，hello-world 几秒钟就能拉完，不需要配置任何镜像加速器。国内机器装 Docker 时最头疼的“拉不动镜像”，在海外 VPS 上天然不存在，这也是不少人专门买一台海外机器跑容器的原因。

## 装完马上做的三个收尾配置

新装好的 Docker 开箱能用，但有三处配置值得趁部署业务前处理，每处五分钟以内。

### 给容器日志设上限

默认设置下，每个容器的输出会无限写进一个 JSON 文件，一个话痨应用几天就能把小硬盘写满。编辑 /etc/docker/daemon.json：

json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}


单个日志文件 10MB、最多保留 3 个，一个容器的日志上限 30MB。改完 `sudo systemctl restart docker`，趁没有业务容器时操作代价最小。

### 1GB 内存的机器加一块 swap

1GB 内存跑容器没问题，但 docker build 阶段的编译操作容易把内存顶满、导致构建中途失败。加 2G swap 能明显提高成功率：


sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab


最后一行让它重启后继续生效。2.5GB 以上内存的机型可以跳过。

### 放行业务端口

很多“容器明明跑起来了浏览器却打不开”的情况，最后都查到 ufw 头上——新装系统后通常只放行了 22 端口。跑容器前先放行计划用的端口：


sudo ufw allow 8080/tcp


没启用 ufw 就不用管，直接下一步。

## 跑起第一个真实容器

hello-world 只能验证安装，第一个有用的容器用 Nginx：


docker run -d \
  --name test-nginx \
  -p 8080:80 \
  --restart unless-stopped \
  nginx:alpine


参数拆开看：`-d` 后台运行；`--name` 起个好记的名字；`-p 8080:80` 把宿主机 8080 端口映射到容器的 80；`--restart unless-stopped` 让容器在宿主机重启或进程崩溃后自动拉起，VPS 上跑服务这个参数基本必加；nginx:alpine 是体积最小的官方 Nginx 镜像。

浏览器访问 http://你的VPS的IP:8080，看到 Nginx 欢迎页，说明安装、端口映射、防火墙整条链路都通了。测试完清理掉：


docker stop test-nginx && docker rm test-nginx


之后正式部署服务，建议直接用 Compose 把配置写进 compose.yaml 文件，比一长串 docker run 参数好维护得多。日常高频命令就这几条：

- `docker ps`：看正在运行的容器
- `docker ps -a`：连已停止的容器一起看
- `docker compose logs -f`：跟踪当前目录所有服务的日志
- `docker compose pull && docker compose up -d`：更新镜像并应用
- `docker system prune -a`：清理无用镜像和构建缓存，小硬盘机器定期跑一次

## 常见报错速查

安装和运行阶段的高频报错基本就这四种：

- **permission denied while trying to connect to the Docker daemon socket**：加组后没有重新登录，断开 SSH 重连即可。
- **port is already allocated**：端口被占用，多半是系统自带的 Nginx 或 Apache。用 `sudo ss -tulpn | grep :8080` 找到占用进程，停掉它或者给容器换个端口。
- **容器启动后立刻退出**：容器内的应用崩了，`docker logs 容器名` 看具体原因，常见的是缺环境变量或者配置文件没挂载到镜像期望的路径。
- **apt-cache policy docker-ce 显示的还是 Ubuntu 归档版本**：官方源没加对，检查 /etc/apt/sources.list.d/docker.list 里的版本代号和系统是否一致。

## 几个常见问题

**1GB 内存跑 Docker 到底够不够？**
轻量负载够用。一两个博客、反代、探针、机器人没有压力；MySQL 和 WordPress 同时跑就勉强了，这种情况直接上 2.5GB。

**OpenVZ 架构的 VPS 为什么不行？**
OpenVZ 和宿主机共享内核，你拿不到独立内核，Docker 依赖的 cgroup、namespace 特性受宿主机限制。买之前用 systemd-detect-virt 查一下，或者直接认准 KVM。

**在 RackNerd 的机器上装 Docker 要额外交钱吗？**
不用。Docker CE 社区版免费，官方源安装不产生任何额外费用，你的成本只有 VPS 本身的费用。

**20GB 硬盘会不会很快被镜像吃满？**
取决于更新频率。日志设了上限、再配合定期 `docker system prune -a`，轻量使用撑一年问题不大；镜像玩得多的，建议一开始就选 40GB 以上的档位。

## 按用途决定入手哪一档

- 只跑一两个轻量服务、预算压到最低：1GB KVM 年付款够用，年付十几美元的量级
- 要带数据库建站、同时跑几个服务：2.5GB 或 3GB，多花的十几美元换来不折腾
- 长期叠多个 Compose 项目或者想留余量：3.5GB 或 5GB

下单前最后确认两件事：机房位置——同一套餐不同机房价差几美元，在延迟和价格之间挑个平衡；系统模板选 Ubuntu 22.04 或 24.04，装 Docker 最顺。就算选错了也不用慌，控制面板里可以随时重装系统。系统装好后，把第二步的命令原样贴进终端，十分钟后你的第一个容器就能跑起来。👉 [用年付价格开通 RackNerd KVM VPS](https://bit.ly/my-rn)。
