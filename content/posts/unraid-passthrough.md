---
title: "Unraid Passthrough"
date: 2022-11-27T12:18:54+08:00
draft: false
---

总结记录一下在 unRAID(v6.11) 中如何直通核显 gpu，usb 设备等。
```
CPU: AMD R7-5700G
Motherboard: Asus TUF Gaming B550 Plus WIFI II
Memory: 2 x 32G DDR3200
```
# 主机配置
1. 设置 unRAID 启动方式为 Legacy。在主机的 BIOS 设置打开 CSM / Legacy Boot 等选项。
2. 设置内置显卡为主要启动项，打开 multiple display 选项，并分配最大的共享内存。
3. 打开 SVM（虚拟化支持）选项。

# 设置 unRAID
1. 进入 unRAID 中配置 System Devices 分组，并选定 Bind Selected to IOMMU 中分得越开越利于后续步骤，具体操作步骤可以参考[1]。需要注意的是，通过反复拔插确定好需要直通的 usb port 的位置以及对应的 usb controller，后面整个直通给虚拟机，这样子虚拟机就可以轻松使用 usb 设备了。

2. 在 unRAID 中禁用 amdgpu 内核模块(因为后面需要直通它，主机就不使用了)
```
# /boot/config/modprobe.d/amdgpu.conf

blacklist amdgpu
```

3. 新建虚拟 Windows 10 主机。
可以使用下面的配置
```
Machine: Q35-6.2
BIOS: SeaBIOS // 必选
Hyper-V: yes
```
其他 I/O 设备，如网卡、硬盘尽量使用 virtio 类型连接。光驱选择 SATA 连接。

4. 先在 VNC 作为主要显示输出的情况下安装 Windows 10，安装过程中需要依赖 virtio.iso，具体过程参考[2]。
5. 安装完 Windows 以后打开远程桌面，确保能够通过别的机器连接上，后续显卡驱动安装的步骤会通过远程连接进行。
6. Dump 显卡 vbios，具体方法可以参考[3][4]，把文件改名为 .rom 放在 `/mnt/user/system/graph-bios/` 目录。
7. 修改虚拟机的 XML 配置，主要参考[5][6][7]。
```xml
   <hyperv>
   	  <!-- 省略若干行 -->
      <vendor_id state='on' value='AMD5700GOFDL'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    <!-- 省略若干行 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x0'/> <!-- -->
      </source>
      <rom file='/mnt/user/system/graph-bios/vbios_1638.rom'/> <!-- 此处表明是显卡 -->
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0' multifunction='on'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x1'/> <!-- 此处 domain, bus, slot 值与显卡相同，表示是显卡上的声音输出设备 -->
      </source>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x1'/> <!-- 修改此处的 bus值与显卡对应的相同，并将 function 改为 0x1 -->
    </hostdev>
  <!-- 省略若干行 -->
  <qemu:commandline>
    <qemu:arg value='-set'/>
    <qemu:arg value='device.hostdev0.x-vga=on'/>
  </qemu:commandline>
</domain>
  <!-- 如果上面的 commandline 在启动时候报关于 hostdev0 not defined 错误，用下面的配置替代
  <qemu:override>
    <qemu:device alias='hostdev0'>
      <qemu:frontend>
        <qemu:property name='x-vga' type='bool' value='true'/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
  -->
```
8. 启动 Windows 虚拟机，远程连接上以后安装 AMD 驱动程序，然后以管理员权限执行 `RadeonResetBugFixService install`。这个补丁必须安装，否则会出现重启或关闭虚拟机后主机 CPU 飙升的 bug。

## 参考
1. [Quick and Easy PCIe Device Passthrough for VMs](https://www.youtube.com/watch?v=xsuRFeyqbt4)
1. 如何[安装使用 virtio](https://youtu.be/Tr912VpLVIk?list=PL6MCtOroZNDDIzeTPVxCqKtwdaFTF15AR&t=339)
1. [使用 UBU 从主板 BIOS 中抽出 vBios](https://forums.unraid.net/topic/112649-amd-apu-ryzen-5700g-igpu-passthrough-on-692/page/6/#comment-1134762)
1. [How to Easily Dump the vBios from any GPU for Passthrough](https://www.youtube.com/watch?v=FWn6OCWl63o)
1. [Advanced GPU passthrough techniques on Unraid](https://www.youtube.com/watch?v=QlTVANDndpM)
1. [Unraid Forum 上的讨论](https://forums.unraid.net/topic/112649-amd-apu-ryzen-5700g-igpu-passthrough-on-692/?do=findComment&comment=1161884)
1. [关于 hostdev0 not defined 错误的讨论](https://bbs.archlinux.org/viewtopic.php?id=276409)
