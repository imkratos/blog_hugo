title: 'archLinux安装遇到的问题'
date: 2015-12-24 20:25:32
tags: [linux,arch ]
categories: [linux]
---

 - pacman -Syy 更新之后才能用
 - 启动ssh服务需要安装openssh,`pacman -S openssh`
 - 启动sshd服务`systemctl start sshd `
 - 设置为开机启动服务`systemctl enable sshd.service`
 - 安装vim之后，显示libncursesw.so.6不能找到，运行`pacman -Syu`升级即可
<!--more-->
