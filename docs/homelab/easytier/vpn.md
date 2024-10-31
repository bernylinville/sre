---
order: 1
category: 折腾
tag:
  - VPN
  - HomeLab
---

# EasyTier 组网

## 背景

公司华为防火墙，导致我无法使用 WireGuard 组网（被自动阻断）。

## 解决方案

使用 [EasyTier](https://github.com/EasyTier/EasyTier) 组网。

## 组网过程

### OpenWrt 安装 EasyTier 并配置端口转发

也可以使用 EasyTier 共享公网节点 `tcp://public.easytier.top:11010`。

```bash
opkg update
opkg install unzip
wget -O /tmp/easytier-linux-x86_64-v2.0.3.zip https://github.com/EasyTier/EasyTier/releases/download/v2.0.3/easytier-linux-x86_64-v2.0.3.zip
cd /tmp
unzip easytier-linux-x86_64-v2.0.3.zip
cp easytier-linux-x86_64/* /usr/bin/
nohup easytier-core 2>&1 &
```

### 服务器配置

家里服务器和公司开发机器都是 debian 系统，所以配置过程基本一致。

```bash
wget -O /tmp/easytier-linux-x86_64-v2.0.3.zip https://github.com/EasyTier/EasyTier/releases/download/v2.0.3/easytier-linux-x86_64-v2.0.3.zip
cd /tmp
unzip easytier-linux-x86_64-v2.0.3.zip
cp easytier-linux-x86_64/* /usr/local/bin/
```

```bash
# 家里服务器 service 配置
sudo cat <<EOF > /etc/systemd/system/easytier.service
[Unit]
Description=EasyTier Service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/easytier-core -i 10.188.188.1 -n 192.168.66.0/24 -p tcp://${HOME_DDNS_DOMAIN}:11010 --network-name easytier_network

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable easytier --now
sudo systemctl status easytier
```

```bash
# 公司开发机器 service 配置
sudo cat <<EOF > /etc/systemd/system/easytier.service
[Unit]
Description=EasyTier Service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/easytier-core -i 10.188.188.2 -n 10.0.66.0/24 -p tcp://${HOME_DDNS_DOMAIN}:11010 --network-name easytier_network

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable easytier --now
sudo systemctl status easytier
```

```bash
# mac 配置
# 安装 easytier
wget -O ~/Downloads/easytier-macos-aarch64-v2.0.3.zip https://github.com/EasyTier/EasyTier/releases/download/v2.0.3/easytier-macos-aarch64-v2.0.3.zip
cd ~/Downloads
unzip easytier-macos-aarch64-v2.0.3.zip
cp easytier-macos-aarch64/* /usr/local/bin/

# 安装 serviceman
curl -sS https://webi.sh/serviceman | sh

cat <<EOF > ~/.config/easytier.toml
# 实例名称，用于在同一台机器上标识此 VPN 节点
instance_name = ""
# 主机名，用于标识此设备的主机名
hostname = ""
# 实例 ID，一般为 UUID，在同一个 VPN 网络中唯一
# instance_id = ""
# 此 VPN 节点的 IPv4 地址，如果为空，则此节点将仅转发数据包，不会创建 TUN 设备
ipv4 = "10.188.188.3"
# 由 Easytier 自动确定并设置IP地址，默认从10.0.0.1开始。警告：在使用 DHCP 时，如果网络中出现 IP 冲突，IP 将自动更改
dhcp = false

# 监听器列表，用于接受连接
listeners = [
"tcp://0.0.0.0:11010",
"udp://0.0.0.0:11010",
"wg://0.0.0.0:11011",
"ws://0.0.0.0:11011/",
"wss://0.0.0.0:11012/",
]

# 退出节点列表
exit_nodes = [
]

# 用于管理的 RPC 门户地址
rpc_portal = "127.0.0.1:15888"

[network_identity]
# 网络名称，用于标识 VPN 网络
network_name = "easytier_network"
# 网络密钥，用于验证此节点属于 VPN 网络
network_secret = ""

# 这里是对等连接节点配置，可以多段配置
[[peer]]
uri = "tcp://${HOME_DDNS_DOMAIN}:11010"

# [[peer]]
# uri = ""

# 这里是子网代理节点配置，可以有多段配置
[[proxy_network]]
cidr = "10.0.1.0/24"

[[proxy_network]]
cidr = "10.0.2.0/24"

#wg配置信息
[vpn_portal_config]
#VPN客户端所在的网段，下面为示例
client_cidr = "10.14.14.0/24"
#wg所监听的端口(请勿和listeners的wg冲突)
wireguard_listen = "0.0.0.0:11012"

[flags]
# 连接到对等节点使用的默认协议
default_protocol = "tcp"
# TUN 设备名称，如果为空，则使用默认名称
dev_name = ""
# 是否启用加密
enable_encryption = true
# 是否启用 IPv6 支持
enable_ipv6 = true
# TUN 设备的 MTU
mtu = 1380
# 延迟优先模式，将尝试使用最低延迟路径转发流量，默认使用最短路径
latency_first = false
# 将本节点配置为退出节点
enable_exit_node = false
# 禁用 TUN 设备
no_tun = false
# 为子网代理启用 smoltcp 堆栈
use_smoltcp = false
# 仅转发白名单网络的流量，支持通配符字符串。多个网络名称间可以使用英文空格间隔。如果该参数为空，则禁用转发。默认允许所有网络。例如：'*'（所有网络），'def*'（以def为前缀的网络），'net1 net2'（只允许net1和net2）
foreign_network_whitelist = "*"
EOF

sudo serviceman add -name easytier -system \
--workdir /var/log/easytier \
-groupname wheel -username root \
-cap-net-bind \
-- easytier-core -c ~/.config/easytier.toml
```

## 测试

```bash
% ansible -i hosts.yml easytier -m ping -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
[WARNING]: Platform linux on host nas is using the discovered Python interpreter at /usr/bin/python3.11, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.17/reference_appendices/interpreter_discovery.html for more information.
nas | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.11"
    },
    "changed": false,
    "ping": "pong"
}
[WARNING]: Platform linux on host y9000p is using the discovered Python interpreter at /usr/bin/python3.11, but future installation of another Python interpreter could change the meaning of that path. See https://docs.ansible.com/ansible-
core/2.17/reference_appendices/interpreter_discovery.html for more information.
y9000p | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.11"
    },
    "changed": false,
    "ping": "pong"
}

```
