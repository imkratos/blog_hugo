title: '树莓派ntp时间同步问题'
date: 2015-11-30 22:48:37
tags: [树莓派,ntp同步]
categories: [树莓派]
---
今天刚到手树莓派，看了下时间，于是上网搜了下时间同步的问题。
然而我按照网上修改的同步服务器并不能解决问题，于是自己试着改了一下。

前提：你要选择正确了时区，运行`sudo raspi-config` 选择第4项，回车，继续选择第2个，回车，然后选择Aisa，回车，再次选择重庆或者上海，这样时区就调整好了。
网上大部分都说在`/etc/ntpd.conf` 下

`server 0.debain.pool.ntp.org iburst`

`server 1.debain.pool.ntp.org iburst`

`server 2.debain.pool.ntp.org iburst`

`server 3.debain.pool.ntp.org iburst`


调整服务器为`server asia.pool.ntp.org iburst`，我自己试了并不可以，于是我参考了一下我的Mac，用了苹果的时间同步服务器，最终修改为`server time.apple.com iburst`然后重启ntp服务
`sudo service ntp restart`即可。
