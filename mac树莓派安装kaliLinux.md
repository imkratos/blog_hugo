title: mac树莓派安装kaliLinux
date: 2015-12-04 21:13:51
tags: [树莓派,kali,linux,mac]
categories: [树莓派]
---
树莓派到手大概有一周了，想折腾下kali linux，记录下安装步骤。

准备工作，必须品：
1. 树莓派板子一个
2. 5V电源一个
3. 8G SD卡一张
4. kali linux img [下载地址](https://www.offensive-security.com/kali-linux-vmware-arm-image-download/) 选择 "RaspberryPi 2" 版本即可
5. 无线网卡或者网线
6. 显示屏

### windows安装
怎样在Windows下将Kali安装到SD卡上
- 下载普通版的树莓派专用Kali Linux，并解压img。
- 下载名为Win32DiskImager的压缩包并解压其中exe后缀的软件。
将SD卡插入你的PC中（记得用读卡器）。
- 双击打开Win32DiskImager.exe，如果你用的是WIN7或WIN8，则需要点击鼠标右键并选择“以管理员身份运行”。
- 如果该软件无法自动侦测到你的SD卡，你就要在右上角的下拉菜单中找到SD卡并手动选择它。
- 在软件的图像文件部分点击小文件夹的图标，找到你刚刚下载的Raspbian.img文件。
- 点击“写入 or write”按键，Win32DiskImager就会帮你完成其它步骤。安装过程结束后，你就可以拔出SD卡然后将其插入树莓派了。

<!--more-->

### mac or linux 安装
怎样在OSX下将Kali安装到SD卡上
- 下载普通版的树莓派专用Kali Linux，并解压img。
- SD卡插入电脑
- 使用`df -h` 命令查看你插入的SD卡，一般类似 `/dev/disk0s4` 每个人的机器数字是不一样的
- 使用`diskutil list`查看SD卡的名字，一般类似  `/dev/disk4`
- 然后使用`diskutil unmount /dev/disk0s4` 命令，卸载已经挂载的SD卡
- 使用`diskutil list`查看是否已经卸载，并使用`sudo dd bs=4m if=kali-linux.img(这里是你解压img的名字) of=/dev/rdisk2(这里是你SD卡的位置)`
- 写入完成就可以了，然后执行`diskutil unmountDisk /dev/disk2`推出SD卡。！！！这一步一定要执行，否则可能不会正常开机。你就可以拔出SD卡然后将其插入树莓派了。

### 树莓派开机
- 插入SD卡后可正常开机，kali linux 默认用户名密码 root toor 一定要自行修改，否则很可能会被人猜出来的。

### 说下坑
- 我查了网上各种资料，都没有给出IMG的下载地址，官网的armhf不能用好么，查了查文档也没找到在哪写的armhf和armel的区别。最终使用了上面我给出的下载地址，下完之后才可以用。
- 还有，下载之后要验证一下和官网的验证值是否一致，否则有可能自己就成了老黑客的肉鸡，验证方式下载网站已经给出。
