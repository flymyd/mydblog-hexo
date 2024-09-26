---
title: 黑豹X2（Panther-x2）刷机并驱动NPU/VPU、Jellyfin转码
toc: true
date: 2024-09-09 17:05:26
tags: [刷机, RockChip]
categories: [单片机/开发板]
---
# 零、前言
黑豹X2是一款非常不错的ARM设备，配置为RK3566 + 4G RAM + 32G EMMC，带千兆网口和TF卡插槽，有无线和蓝牙。以下介绍如何安装Armbian系统以及Docker和Jellyfin的配置。需要注意的是，很多Armbian系统由于内核问题，不支持NPU和VPU。请注意选择系统版本。
# 一、刷机

## 1、下载所需文件
* [RockChip驱动](https://download.t-firefly.com/product/Board/RK356X/Tool/Window/DriverAssitant/DriverAssitant_v5.1.1.zip)，解压并安装。
* [RockChip量产工具](https://download.t-firefly.com/product/Board/RK356X/Tool/Window/AndroidTool/RKDevTool_Release_v2.84.zip)，解压后修改``config.ini``：
```bash
[Language]
Kinds=2
Selected=1
LangPath=Language\
```

* [量产工具Config](https://116-142-255-131.pd1.cjjd19.com:30443/download-cdn.cjjd19.com/123-573/dd11fee5/1765511-0/dd11fee50c7cec04a7b49f7c0075d23d/c-m2?v=5&t=1725935625&s=1725935625e4615494c210ff330b4163d4278a59c1&r=GZ6LGK&bzc=2&bzs=313736353531313a34393136323331393a313234393a30&filename=config.cfg&x-mf-biz-cid=24adcf0e-0352-4fc9-9982-e9310db5b63d-6eaa77&auto_redirect=0&cache_type=1&xmfcid=4bbff9bc-6d69-42c8-b8c0-db98dd74fb8e-0-abf611255)，下载后替换``RKDevTool_Release_v2.84``的``config.cfg``
* [RK3566 MiniLoader](https://download.t-firefly.com/product/Board/RK356X/Firmware/MiniLoaderAll/RK3566/MiniLoaderAll.bin)
* [Armbian镜像](https://github.com/HelloTheAsia/amlogic-s9xxx-armbian/releases/download/Armbian_bullseye_save_2024.07/Armbian_24.8.0_rockchip_panther-x2_bullseye_6.1.57_server_2024.07.09.img.gz)  ，解压得到``Armbian_24.8.0_rockchip_panther-x2_bullseye_6.1.57_server_2024.07.09.img``

以上资源均已上传至网盘留档，[网盘链接](https://pan.baidu.com/s/16xSVD4kl_ut-QJpdK2NO2w?pwd=1234)，提取码``1234``。

## 2、开始刷机
* 启动``RKDevTool``
* 插好双头USB-A线，连接电脑
* 按住复位键的同时插电源
* 量产工具提示发现一个LOADER设备
* 点击``高级功能``——进入Maskrom——回到``下载镜像``——等待下方显示``发现一个MASKROM设备``
* 点击``下载镜像``——找到路径右侧的三个点，此为文件选择列
* 点击``Boot``分区列——选择``MiniLoaderAll.bin``
* 点击``System``分区列——选择``Armbian_24.8.0_rockchip_panther-x2_bullseye_6.1.57_server_2024.07.09.img``

* 点击``执行``，等待刷写完毕，机器会自动重启。

# 二、连接SSH并初始化
## 1、连接SSH
插上网线，登录路由器后台，找到黑豹X2的IP地址，系统默认为DHCP获取IP。如果不方便登录路由器后台，可以使用nmap来尝试扫描。例：局域网网段为``192.168.1.X``，则使用命令：

```bash
PS C:\Users\MYD-Work> nmap -p 22 --open 192.168.1.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2024-09-09 15:04 中国标准时间
Nmap scan report for 192.168.1.64
Host is up (0.00013s latency).

PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: XX:27:XX:XX:A7:XX (Unknown)

Nmap done: 256 IP addresses (32 hosts up) scanned in 8.63 seconds
```
可知IP为``192.168.1.64``。
在终端中以root登录SSH，然后按引导进行初始化配置：

```bash
ssh root@192.168.1.64
```

## 2、初始化
### 换源
```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
rm /etc/apt/sources.list
nano /etc/apt/sources.list

复制如下源：
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free

不会用nano的可以按如下方法保存退出：
Ctrl-O
回车
Ctrl-X

sed -i.bak 's#http://apt.armbian.com#https://mirrors.tuna.tsinghua.edu.cn/armbian#g' /etc/apt/sources.list.d/armbian.list
apt update
```
### 安装蓝牙模块驱动

```bash
armbian-config
```
选择Network-BT install 即可。

### 安装一些包
``apt install vim cmake lm-sensors python3 pip -y``

### 检查驱动

```bash
root@armbian:~# ls /dev/dri
by-path  card0  card1  renderD128  renderD129
root@armbian:~# ls /dev/mpp_service
/dev/mpp_service
root@armbian:~# cat /sys/class/devfreq/fde40000.npu/available_frequencies
200000000 297000000 400000000 600000000 700000000 800000000 900000000
root@armbian:~# sensors
soc_thermal-virtual-0
Adapter: Virtual device
temp1:        +37.2°C  (crit = +115.0°C)

gpu_thermal-virtual-0
Adapter: Virtual device
temp1:        +38.9°C  (crit = +115.0°C)
```
# 三、安装Docker和Jellyfin
## 1、安装Docker

```bash
apt install docker.io -y
```
### 安装Docker Compose
* 下载二进制文件：``https://github.com/docker/compose/releases/latest/download/docker-compose-Linux-aarch64``
* 改名为``docker-compose``并复制到``/usr/bin``下
* 赋可执行权限：``chmod +x /usr/bin/docker-compose``
## 2、安装Jellyfin

```bash
mkdir -p /root/jellyfin/config
mkdir -p /root/jellyfin/media
vi docker-compose.yml
```
以下是``docker-compose.yml``的内容：

```yaml
version: '2.4'

services:
  jellyfinny:
    image: nyanmisaka/jellyfin:latest-rockchip
    container_name: jellyfinny
    privileged: true
    ports:
      - "8096:8096"
    volumes:
      - /root/jellyfin/config:/config
      - /tmp:/cache
      - /root/jellyfin/media:/media
    devices:
      - /dev/dri:/dev/dri
      - /dev/dma_heap:/dev/dma_heap
      - /dev/mali0:/dev/mali0
      - /dev/rga:/dev/rga
      - /dev/mpp_service:/dev/mpp_service
      - /dev/iep:/dev/iep
      - /dev/mpp-service:/dev/mpp-service
      - /dev/vpu_service:/dev/vpu_service
      - /dev/vpu-service:/dev/vpu-service
      - /dev/hevc_service:/dev/hevc_service
      - /dev/hevc-service:/dev/hevc-service
      - /dev/rkvdec:/dev/rkvdec
      - /dev/rkvenc:/dev/rkvenc
      - /dev/vepu:/dev/vepu
      - /dev/h265e:/dev/h265e
    restart: unless-stopped
```
执行``docker-compose up -d``以启动容器，访问``http://ip:8096``以初始化Jellyfin。

## 3、配置转码
进入Jellyfin后，选择设置——控制台——播放，在``转码``中选择硬件加速：Rockchip MPP（RKMPP），将``启用硬件解码``中的所有选项勾选，滑动到最下方保存即可。

# 四、NPU的驱动（可选）
* 下载[RK_NPU_SDK](https://pan.baidu.com/s/1ZVKGD0nrlTFC-jDSTAjqbg) ，提取码``1234``
* 解压``RK_NPU_SDK_1.5.0/rknpu2_1.5.0.7z``，提取出``rknpu2/runtime/RK356X/Linux/librknn_api/aarch64``，将里面的两个``.so``文件复制到``armbian``的``/usr/lib``下即可。
