---
title: "Using Lima as Docker Destkop Alternative"
date: 2021-11-29T13:05:34+08:00
draft: false
---

随着 Docker Desktop 越来越笨重，我现在已经用 Lima 完全来替换掉它。

## 介绍
[Lima](https://github.com/lima-vm/lima) 是一个在 macOS 上运行虚拟化 Linux 系统的工具。
官方称可以把 Lima 理解为 "macOS subsystem for Linux" 或者 "containerd for Mac"。

## 安装

1. 在 macOS 主机上最简单的方式是通过 Homebrew 安装
```bash
brew install lima
```
> 我比较喜欢自己构建，会自己手动构建 qemu 以后放在 $HOME/.local 下

2. 启动虚拟机

创建一个 `default.yaml`，内容如下（根据自己主机的情况调整 memory 和 disk 大小）
```yaml
cpus: 2
memory: "10GiB"
disk: "100GiB"
images:
  - location: "https://cloud-images.ubuntu.com/impish/current/impish-server-cloudimg-amd64.img"
    arch: "x86_64"
mounts:
  - location: "~"
    writable: false
  - location: "/tmp/lima"
    writable: true
ssh:
  localPort: 60006
containerd:
  system: false
  user: false
portForwards:
  - guestSocket: "/run/user/{{.UID}}/docker.sock"
    hostSocket: "{{.Home}}/opt/docker/docker.sock"
```

3. 执行 `limactl start ./default.yaml` 启动虚拟机

可以通过 `ps -ef` 看到
```
501  1589  1577   0 Sun07PM ttys001  1847:13.31 /usr/local/bin/qemu-system-x86_64 -cpu host -machine q35,accel=hvf -smp 2,sockets=1,cores=2,threads=1 -m 10240 -drive if=pflash,format=raw,readonly=on,file=/usr/local/share/qemu/edk2-x86_64-code.fd -boot order=c,splash-time=0,menu=on -drive file=/Users/nevill/.lima/default/diffdisk,if=virtio -cdrom /Users/nevill/.lima/default/cidata.iso -netdev user,id=net0,net=192.168.5.0/24,dhcpstart=192.168.5.15,hostfwd=tcp:127.0.0.1:60006-:22 -device virtio-net-pci,netdev=net0,mac=52:55:55:63:06:20 -device virtio-rng-pci -display none -device virtio-vga -device virtio-keyboard-pci -device virtio-mouse-pci -parallel none -chardev socket,id=char-serial,path=/Users/nevill/.lima/default/serial.sock,server=on,wait=off,logfile=/Users/nevill/.lima/default/serial.log -serial chardev:char-serial -chardev socket,id=char-qmp,path=/Users/nevill/.lima/default/qmp.sock,server=on,wait=off -qmp chardev:char-qmp -name lima-default -pidfile /Users/nevill/.lima/default/qemu.pid

501  1643     1   0 Sun07PM ??         0:00.89 ssh: /Users/nevill/.lima/default/ssh.sock [mux]
501  1649  1577   0 Sun07PM ttys001    0:00.01 ssh -F /dev/null -o IdentityFile="/Users/nevill/.lima/_config/user" -o IdentityFile="/Users/nevill/.ssh/google_compute_engine" -o IdentityFile="/Users/nevill/.ssh/id_rsa" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o NoHostAuthenticationForLocalhost=yes -o GSSAPIAuthentication=no -o PreferredAuthentications=publickey -o Compression=no -o BatchMode=yes -o IdentitiesOnly=yes -o Ciphers="^aes128-gcm@openssh.com,aes256-gcm@openssh.com" -o User=nevill -o ControlMaster=auto -o ControlPath="/Users/nevill/.lima/default/ssh.sock" -o ControlPersist=5m -p 60006 127.0.0.1 -- sshfs :/Users/nevill /Users/nevill -o slave -o ro -o allow_other
501  1652  1577   0 Sun07PM ttys001    0:00.01 ssh -F /dev/null -o IdentityFile="/Users/nevill/.lima/_config/user" -o IdentityFile="/Users/nevill/.ssh/google_compute_engine" -o IdentityFile="/Users/nevill/.ssh/id_rsa" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o NoHostAuthenticationForLocalhost=yes -o GSSAPIAuthentication=no -o PreferredAuthentications=publickey -o Compression=no -o BatchMode=yes -o IdentitiesOnly=yes -o Ciphers="^aes128-gcm@openssh.com,aes256-gcm@openssh.com" -o User=nevill -o ControlMaster=auto -o ControlPath="/Users/nevill/.lima/default/ssh.sock" -o ControlPersist=5m -p 60006 127.0.0.1 -- sshfs :/tmp/lima /tmp/lima -o slave -o allow_other
501  1662  1660   0 Sun07PM ttys001    0:00.01 /usr/bin/ssh -F /dev/null -o IdentityFile="/Users/nevill/.lima/_config/user" -o IdentityFile="/Users/nevill/.ssh/google_compute_engine" -o IdentityFile="/Users/nevill/.ssh/id_rsa" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o NoHostAuthenticationForLocalhost=yes -o GSSAPIAuthentication=no -o PreferredAuthentications=publickey -o Compression=no -o BatchMode=yes -o IdentitiesOnly=yes -o Ciphers="^aes128-gcm@openssh.com,aes256-gcm@openssh.com" -o User=nevill -o ControlMaster=auto -o ControlPath="/Users/nevill/.lima/default/ssh.sock" -o ControlPersist=5m -t -q -p 60006 127.0.0.1 -- cd "/Users/nevill" || cd "/Users/nevill" ; exec bash --login
```
注意到 qemu 进程的存在，还有一些 ssh 进程用来转发端口。

运行 `lima` 即可进入虚拟机访问。


## 安装并使用 Docker

1. 在虚拟机里面安装 rootless docker [2]

```bash
$ lima
nevill@lima-default:/Users/nevill$ curl -fsSL https://get.docker.com/rootless | sh

# 启动 docker 服务
nevill@lima-default:/Users/nevill$ systemctl --user daemon-reload
nevill@lima-default:/Users/nevill$ systemctl --user restart docker

# 虚拟机启动的时候启动容器服务
nevill@lima-default:/Users/nevill$ sudo loginctl enable-linger $(whoami)

# 确定 docker 正常运行
nevill@lima-default:/Users/nevill$ docker info
```

2. 在主机环境中使用 docker

安装 [docker cli](https://download.docker.com/mac/static/stable/x86_64/docker-20.10.9.tgz)

> nerdctl 当前还不能完全取代 docker-cli [1]。 至少需要等 kind 能够通过 nerdctl 正常启动我才会替换，目前依然使用 docker-cli 来进行容器、镜像的相关操作。

运行 `DOCKER_HOST=unix:///Users/nevill/opt/docker/docker.sock docker info` 看是否能成功返回结果。

如果成功可以执行 `docker context create lima-default --docker "host=unix:///Users/nevill/opt/docker/docker.sock"` 来保存。

## 同 Docker 的兼容性

Lima 会将主机的 $HOME 目录完全映射到虚拟机中，因此在主机的 $HOME 下执行类似 `docker run -v $PWD:/somedir` 的命令是完全可以正常工作的。

## Reference
1. https://github.com/containerd/nerdctl/issues/349
2. https://rootlesscontaine.rs/getting-started/docker/
