---
title: 玩饥荒联机版日志
date: 2019-03-22
---

# 玩饥荒联机版日志

## 搭建独立服务器

1. 安装 `SteamCMD`。首先创建 `steam` 用户，然后下载 `SteamCMD`：<https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz> 解压安装，安装位置任意，详细步骤及依赖关系推荐参阅此处。
2. 运行 `steamcmd.sh`
3. 登陆并安装服务器：

   ```shell
   login anonymous
   force_install_dir /home/steam/dstserver
   app_update 343050
   validate
   quit
   ```

4. 配置服务器。可在 `Windows` 上使用图形界面启动具有洞穴的服务器，然后清除该存档记录并将其复制到 `/home/steam/.klei/DoNotStarveTogether` 目录中即可。也可以先将存档复制到上述目录，然后使用下述脚本清除存档：

   ```shell
   # 其中 /home/steam/.klei/DoNotStarveTogether/GlxinWorld 为世界存档目录
   rm -rf /home/steam/.klei/DoNotStarveTogether/GlxinWorld/{Master,Caves}/save
   ```

5. 启动服务器。

   ```shell
   # 必须进入该目录，否则可能会无法加载
   # 其中 Cluster_1 改为对应的存档名称
   cd /home/steam/dstserver/bin
   ./dontstarve_dedicated_server_nullrenderer -console -cluster Cluster_1 -shard Caves
   ./dontstarve_dedicated_server_nullrenderer -console -cluster Cluster_1 -shard Master
   ```

Tip：有些系统（如：`CentOS 7`）不提供 `libcurl-gnutls.so.4` 那么将其链接到 `libcurl.so.4` 即可。

多世界并存：若要在一台服务器上运行多个饥荒服务器，则需要修改存档的 `Master` 及 `Caves` 目录中 `server.ini` 的 `server_port` 使其各不相同，且在 `10998-11018` 范围内。且每个存档的 `cluster.ini` 中 `master_port` 应唯一，该端口用于存档的地面与洞穴之间通信。饥荒服务器在启动时同样会监听 `127.0.0.1:10888/UDP` 端口和一个非固定 `TCP` 端口，可能这些也需要修改。初步猜测该端口可能用于存档的地面与洞穴之间通信，多个存档应修改该端口。

跨服务器世界：即将洞穴服务器与地面服务器分离，将存档的 `cluster.ini` 中 `master_ip` 与 `master_port` 配置为地面服务器的 IP 地址和端口即可。

## 一键启动脚本

其中变量按需修改，若不希望每次启动都检查更新，将脚本中检查更新的行注释即可。另外，最新版不购买可能导致搜素不到世界的问题，故可以按照下一节介绍安装 `247691` 版本。

```shell
#!/bin/bash

steamcmd_dir="$HOME/Steam"
install_dir="$HOME/dstserver"
cluster_name="GlxinWorld"
dontstarve_dir="$HOME/.klei/DoNotStarveTogether"

function fail()
{
        echo Error: "$@" >&2
        exit 1
}

function check_for_file(){
    if [ ! -e "$1" ]; then
            fail "Missing file: $1"
    fi
}

cd "$steamcmd_dir" || fail "Missing $steamcmd_dir directory!"
check_for_file "$steamcmd_dir/steamcmd.sh"
check_for_file "$dontstarve_dir/$cluster_name/cluster.ini"
check_for_file "$dontstarve_dir/$cluster_name/cluster_token.txt"
check_for_file "$dontstarve_dir/$cluster_name/Master/server.ini"
check_for_file "$dontstarve_dir/$cluster_name/Caves/server.ini"

# 检查更新，若不需要每次启动都检查更新，将其注释即可
~/Steam/steamcmd.sh +force_install_dir "$install_dir" +login anonymous +app_update 343050 validate +quit

check_for_file "$install_dir/bin"

cd "$install_dir/bin" || fail
run_shared=(./dontstarve_dedicated_server_nullrenderer)
run_shared+=(-console)
run_shared+=(-cluster "$cluster_name")
run_shared+=(-monitor_parent_process $$)

"${run_shared[@]}" -shard Caves  | sed 's/^/Caves:  /' &
"${run_shared[@]}" -shard Master | sed 's/^/Master: /'
```

## 下载指定版本

1. 进入 `SteamDB` 查找游戏，记录 `AppID`。
2. 进入 `Depots` 找到对应平台，记录 `DepotID`。
3. 点击 `DepotID` => `Manifests` 找到需要的版本，记录 `ManifestID`。
4. `Windows` 下运行 `steam://nav/console` 进入控制台/`Linux` 下执行 `steamcmd.sh`。
5. 若是 `Linux` 用户，需要执行 `login anonymous` 登陆；若不是 `Linux` 用户则跳过。
6. 执行 `download_depot <AppID> <DepotID> <ManifestID>` 进行下载。
7. 执行 `quit` 退出。

若下载失败，可退出从第 4 步重新开始。实际测试发现，下载成功率非常低，故建议使用外网服务器，且具有不低的带宽。

上述过程得到 DST 的信息如下：

- AppID: 343050
- Linux DepotID: 343052
- Latest Version ManifestID: 3637127330667398786
- Version 247691 ManifestID: 6994825278996354537
- SteamDB URL: <https://steamdb.info/app/343050/>

进入控制台（`Win + R` 运行）:

```shell
steam://nav/console
```

下载 `247691` 版本游戏：

```shell
download_depot 343050 343052 6994825278996354537
```

`Linux` 可在控制台直接执行：

```shell
steamcmd.sh +login anonymous +download_depot 343050 343052 6994825278996354537 +quit
```

`SteamCMD` 根据游戏 `depot id` 进行存储，故一个游戏可下载多个 `depot` 对应的版本，但每个 `depot` 仅能下载一个 `manifest` 对应的版本。

## 参考资料

[steam_commands](https://gist.github.com/xPaw/fe7d275d31da14d70481)
[SteamCMD – Valve Developer Community](https://developer.valvesoftware.com/wiki/SteamCMD)
[饥荒联机版独立服务器搭建踩坑记录 – Blessing Studio](https://blessing.studio/deploy-dont-starve-together-dedicated-server/)
[【社区指南翻译】如何下载旧版的游戏 – 平台研讨 – SteamCN 蒸汽动力 – 驱动正版游戏的引擎！](https://steamcn.com/t258082-1-1)
[通过 depot 下载得到旧版游戏及一个衍生应用 – 平台研讨 – SteamCN 蒸汽动力 – 驱动正版游戏的引擎！](https://steamcn.com/t258079-1-1)
[Guide: How to download older versions of a game on Steam: Steam](https://www.reddit.com/r/Steam/comments/611h5e/guide_how_to_download_older_versions_of_a_game_on/)
[饥荒联机独立服务器搭建教程（三）：配置篇 | 天天の記事簿](http://blog.ttionya.com/article-1235.html)
[Create UDP to TCP bridge with socat/netcat to relay control commands for vlc media-player – Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/267118/create-udp-to-tcp-bridge-with-socat-netcat-to-relay-control-commands-for-vlc-med)

## Playing outside the LAN

对游戏端口 `UDP 10998/10999` 进行转发，可实现在外网进入内网的游戏服务器。其简单 `python2` 脚本如下：

agent_v3.py:

```python
#!/usr/bin/env python2
# -*- coding: utf-8 -*-

"""
Usage:
    agent.py -h | --help
    agent.py client [-p BASE_PORT]
    agent.py server [-p BASE_PORT]

Options:
    -h --help                       show this
    -p BASE_PORT, --port BASE_PORT  specify base port

Examples:
    agent.py client

"""


from threading import Thread
from docopt import docopt
from time import sleep
from socket import socket, AF_INET, SOCK_DGRAM

LOCALHOST_IP = '127.0.0.1'
MASTER_IP_ADDRESS = '172.18.135.5'
MASTER_PORT = 10999
CAVES_IP_ADDRESS = '172.18.135.5'
CAVES_PORT = 10998

BASE_TRANSFER_PORT = 10001

BUFFER_SIZE = 10485760


def remote_to_local(sock_local, sock_remote, addr, buffsize):
    while True:
        data = sock_remote.recv(buffsize)
        if len(data):
            sock_local.sendto(data, )
    pass


def local_to_remote(local, remote, buffsize):
    conn_dict = {}
    sock_local = socket(AF_INET, SOCK_DGRAM)
    sock_local.bind(local)
    sock_remote = None
    while True:
        data, addr = sock_local.recvfrom(buffsize)
        if addr in conn_dict:
            sock_remote = conn_dict[addr]
        else:
            sock_remote = socket(AF_INET, SOCK_DGRAM)
            sock_remote.connect(remote)
            Thread(target=remote_to_local, args=(sock_local, sock_remote, addr, buffsize))
        sock_remote.sendall(data)


def build_connection(local, remote, buffsize):
    thread = Thread(target=local_to_remote, args=(local, remote, buffsize))
    thread.setDaemon(True)
    thread.start()


def check_connection(conns):
    for conn in conns:
        if not conn['t_l2r'].isAlive():
            return False
        if not conn['t_r2l'].isAlive():
            return False
    return True


def main(client_mode, base_port):
    master_listen, master_remote = None, None
    caves_listen, caves_remote = None, None
    if client_mode:
        master_listen = ('0.0.0.0', MASTER_PORT)
        master_remote = (MASTER_IP_ADDRESS, base_port)
        caves_listen = ('0.0.0.0', CAVES_PORT)
        caves_remote = (CAVES_IP_ADDRESS, base_port + 1)
    else:
        master_listen = ('0.0.0.0', base_port)
        master_remote = (LOCALHOST_IP, MASTER_PORT)
        caves_listen = ('0.0.0.0', base_port + 1)
        caves_remote = (LOCALHOST_IP, CAVES_PORT)
    conns = []
    conns.append(build_connection(master_listen, master_remote, BUFFER_SIZE))
    conns.append(build_connection(caves_listen, caves_remote, BUFFER_SIZE))

    try:
        while check_connection(conns):
            sleep(1)
    except KeyboardInterrupt as e:
        print 'User interrupted'

    return None


if __name__ == '__main__':
    arguments = docopt(__doc__)
    main(arguments['client'], int(arguments['--port'] or BASE_TRANSFER_PORT))
```

脚本依赖 docopt。

forward_v1.py:

```python
#!/usr/bin/python2
# coding: utf-8


import sys

from time import sleep
from Queue import Queue
from select import select
from socket import socket, AF_INET, SOCK_DGRAM, SOL_SOCKET, SO_REUSEADDR

# 512KB
BUFFER_SIZE = 524288

LEVEL_INFO = 0
LEVEL_ERROR = 1


def log(level, text):
    if level == LEVEL_ERROR:
        print text


def do_forwarding(options):
    sock_remote = socket(AF_INET, SOCK_DGRAM)
    sock_remote.setblocking(False)
    sock_remote.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
    sock_remote.bind(options["bind"])

    connections = {}

    inputs = [sock_remote]
    outputs = []
    msg_queues = []

    while True:
        try:
            readable, writable, exceptional = select(inputs, outputs, [], 0.1)
        except KeyboardInterrupt:
            break
        if not (readable or writable or exceptional):
            continue
        for sock in readable:
            try:
                data, address = sock.recvfrom(BUFFER_SIZE)
            except Exception, ex:
                log(LEVEL_ERROR, 'Error: ' + repr(ex))
            if len(data) == 0:
                continue
            log(LEVEL_INFO, 'Received from: ' + repr(address))
            sleep
            if sock is sock_remote:
                if address in connections:
                    sock_out = connections[address]
                    log(LEVEL_INFO, 'Host already exists: ' + repr(address))
                else:
                    sock_out = socket(AF_INET, SOCK_DGRAM)
                    sock_out.connect(options["host"])
                    connections[address] = sock_out
                    inputs.append(sock_out)
                    log(LEVEL_INFO, 'Added remote host: ' + repr(address))
                addr_out = options["host"]
            else:
                for src_addr, src_sock in connections.iteritems():
                    if src_sock is sock:
                        addr_out = src_addr
                sock_out = sock_remote
            msg_queues.append({ "to": addr_out, "sock": sock_out, "data": data })
            outputs.append(sock_out)
            log(LEVEL_INFO, 'Add sock to output list: ' + repr(sock_out.getsockname()))
        for sock in writable:
            log(LEVEL_INFO, 'Sock want to send data: ' + repr(sock.getsockname()))
            remains_msgs = []
            for msg in msg_queues:
                if msg["sock"] is sock:
                    log(LEVEL_INFO, 'Hint message')
                    try:
                        sock.sendto(msg["data"], msg["to"])
                        log(LEVEL_INFO, 'Sent to: ' + repr(msg["to"]) + ' via: ' + repr(sock.getsockname()))
                    except Exception, ex:
                        log(LEVEL_ERROR, 'Error: ' + repr(ex))
                else:
                    remains_msgs.append(msg)
            msg_queues = remains_msgs
        if writable:
            outputs = []

    for addr, sock in connections.iteritems():
        sock.close()
    sock_remote.close()


def parse_arguments(args):
    if len(args) != 2:
        return None
    params = args[1].split(':')
    options = { "agent_port": 12345 }
    if len(params) == 3:
        options["bind"] = "127.0.0.1", int(params[0])
        options["host"] = params[1], int(params[2])
    elif len(params) == 4:
        options["bind"] = params[0], int(params[1])
        options["host"] = params[2], int(params[3])
    else:
        return None
    return options


def usage():
    print "Usage: forward.py [bind_ip:]bind_port:host_ip:host_port"
    exit(1)


def main():
    options = parse_arguments(sys.argv)
    if options:
        do_forwarding(options)
    else:
        usage()


if __name__ == '__main__':
    main()
```

`Server` 端监听需要转发的端口，并接受多条连接，建立表维护连接信息，在接收到数据时连带连接信息发送给连接到 `Server` 端的 `Client` 端。

## 问题

在启动 `dontstarve_dedicated_server_nullrenderer` 进程时，可能遇到如下错误：

- `[S_API FAIL] SteamAPI_Init() failed; SteamAPI_IsSteamRunning() failed.` 忽略即可。
- `Segmentation fault (core dumped)` 可能是游戏中的 `steamclient.so` 库文件与 `SteamCMD` 中的 `steamclient.so` 文件版本不匹配导致。删除该文件，并将 `SteamCMD` 中的该文件链接到游戏中该文件。

  ```shell
  # 将游戏中的 steamclient.so 备份为 steamclient.so.bak 后将 SteamCMD 中的 steamclient.so 链接到游戏中对应的位置
  # 下面命令假定 SteamCMD 安装在 /home/steam/Steam，饥荒联机版安装在 /home/steam/dstserver
  cd /home/steam/dstserver/bin/lib32 mv steamclient.so{,.bak}
  ln -s /home/steam/SteamCMD/linux32/steamclient.so /home/steam/dstserver/bin/lib32/steamclient.so
  # 若无法解决问题，则可以通过如下命令恢复
  mv /home/steam/dstserver/bin/lib32/steamclient.so.bak /home/steam/dstserver/bin/lib32/steamclient.so
  ```

  若运行上述命令解决问题后，则应将世界启动脚本中的更新指令移除。若不移除，游戏的更新、校验都会将上述修改覆盖，从而导致游戏无法运行。

  ```shell
  # 在脚本中找到下面这一行，并将其注释
  #~/Steam/steamcmd.sh +force_install_dir "$install_dir" +login anonymous +app_update 343050 validate +quit
  ```

## Backup & Restore

### Backup

```shell
# 进入存档目录
cd /home/steam/.klei/DoNotStarveTogether/GlxinWorld

# 建立存档备份目录
mkdir ../backup

# 执行备份
zip -r ../backup.$(date +%Y%m%d%H%M%S).zip .
```

### Restore

```shell
# 进入存档目录
cd /home/steam/.klei/DoNotStarveTogether/GlxinWorld

# 列出存档列表，并找出需要恢复的存档
ls ../backup

# 恢复存档
find . -delete && unzip ../backup/backup.20180614173011.zip
```

### Automatically backup

在基于 Systemd 的 Linux 系统上，创建如下两个文件实现每 3h 对存档做一次备份。由于在游戏运行状态中，存档目录会包含许多临时文件，

`/usr/lib/systemd/system/dst-backup.service`：

```conf
[Unit]
Description=Don't Starve Together Backup

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'cd /home/steam/.klei/DoNotStarveTogether/GlxinWorld && zip -r ../backup/backup.$(date +%%Y%%m%%d%%H%%M%%S).zip .'
User=steam
```

`/usr/lib/systemd/system/dst-auto-backup.timer`：

```conf
[Unit]
Description=Don't Starve Together Auto Backup

[Timer]
OnUnitActiveSec=3h
Unit=dst-backup.service

[Install]
WantedBy=multi-user.target
```
