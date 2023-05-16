---
title: "Linux Minner Analysis"
description: "This blog will discuss something about Linux minner"
date: 2023-05-05T15:00:00+08:00
Tags: ["Malware Analysis","Minner","Linux"]
draft: false
weight: 1018
cover: 
    image: '/cover/malware.jpg'
---

# 概述
## 相关样本
经调查分析，本次攻击中的样本主要包括:  
* shell类: 
  * [adxintrin_b & cronman](#adxintrinb--cronman)  
  这两个shell文件功能高度相似，且作用于攻击的整个生命周期。其主要行为包括：获取恶意文件、运行挖矿病毒、持久化、清楚记录、横向移动。
  * [xbash](#xbash)  
  该文件是一个Makeself文件，其主要功能是释放"cronman"并执行。
* ELF类
  * [libgcc_a & xfitaarch.sh](#libgcca--xfitaarchsh)  
  这两个文件为本次攻击的最终payload，其为XMrig挖矿病毒的变种。  
  * [spirit & sshpass](#spirit--sshpass)  
  该文件为linux下ssh暴力破解组件，其会对内网中的其他主机进行ssh暴破，以实现横向移动。  
## 整体流程
![Linux_minner-2023-05-16-09-08-56](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-08-56.png)  
注：  
* [获取](#获取文件)  
  * 通过wget & curl，从C2服务器(5.133.65.53 & 45.142.212.30)拉取恶意样本及相关组件。  
  * 通过scp，从已失陷主机中获取所需恶意样本及相关组件。  
* [传播](#传播)
  * 利用spirit & sshpass，通过ssh暴破，在内网进行横向移动  
  * 利用xinetd，实现端口转发，重定向至恶意C2服务器  
* [持久化](#持久化)
  * 设置计划任务
  * 修改sshd配置，并配置ssh公钥  
* [清理痕迹](#清理痕迹)
  * 删除日志 & 命令行历史记录
  * 禁用登录日志 & 授权日志
* [环境配置](#环境配置与检测)
  * 配置"ld.so.preload"
  * 自更新
  * 检查防火墙 & 杀软
  * 修改DNS
  * 创建服务"crtend_b"

# 细节分析  
## shell文件  
### adxintrinb & cronman  
由于两文件功能完全一致，我们放在一起进行分析。同时还存在一个名为"crtres_c"的文件，其功能主要为利用"crontab"定时启动"adxintrin_b"。  
#### 获取文件  
* 通过wget & curl，从C2服务器拉取相关文件  
![Linux_minner-2023-05-16-09-13-19](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-13-19.png)  
* 通过ssh，从已失陷主机中获取相关文件  
其先利用"tar -zcf"命令，将攻击所需的文件进行打包。  
![Linux_minner-2023-05-16-09-13-31](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-13-31.png)  
随后，其通过"scp"命令将刚刚打好的压缩包传入目标服务器，随后通过ssh发送命令，将压缩包进行解压，并执行压缩包中"/etc/cron.daily/xbash"这个脚本。
![Linux_minner-2023-05-16-09-13-42](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-13-42.png)  
#### 环境配置与检测  
* 修改"/etc/ld.so.preload"文件  
其会根据系统架构，将对应的".so"文件写入"/etc/ld.so.preload"，使得后续的ELF文件得以正常运行。  
![Linux_minner-2023-05-16-09-14-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-14-15.png)  
* 自更新  
其会判断xbash文件的修改时间距现在是否超过100天，若超过100天，则重新获取xbash文件。  
![Linux_minner-2023-05-16-09-14-28](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-14-28.png)  
* 检查防火墙  
![Linux_minner-2023-05-16-09-14-42](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-14-42.png)  
* 检查并关闭杀软  
![Linux_minner-2023-05-16-09-14-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-14-48.png)  
* 修改DNS  
![Linux_minner-2023-05-16-09-15-31](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-15-31.png)  
* 创建服务"crtend_b"  
SERVICE_NAME为"crtend_b"，且会将其添加至开机自启中。  
![Linux_minner-2023-05-16-09-15-44](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-15-44.png)  
#### 持久化  
* 修改sshd配置  
图中相关参数如下：		
        file="/etc/ssh/sshd_config"
		param[1]="PermitRootLogin "
		param[2]="PubkeyAuthentication"
		param[3]="AuthorizedKeysFile"
		param[4]="PasswordAuthentication"
		param[5]="HostKeyAlgorithms"
		param[6]="PubkeyAcceptedKeyTypes"  
![Linux_minner-2023-05-16-09-16-29](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-16-29.png)  
* 写入ssh公钥  
![Linux_minner-2023-05-16-09-16-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-16-48.png)  
* 配置计划任务  
  * 在小时计划任务中加入文件"0anacron"，该文件功能为运行xbash。  
![Linux_minner-2023-05-16-09-17-00](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-17-00.png)  
  * 在日计划任务中加入文件xbash。  
![Linux_minner-2023-05-16-09-17-11](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-17-11.png)  
  * 通过"crontab"，为xbash设置定时任务。  
![Linux_minner-2023-05-16-09-17-34](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-17-34.png)  
  * 通过"crontab"，定时启动"crtres_c"。  
图中"$FILE_RES"的值为"crtres_c"  
![Linux_minner-2023-05-16-09-20-37](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-20-37.png)  
"crtres_c"的主要功能为，在"crontab"为"adxintrin_b"添加一个60分钟执行一次的任务计划。  
![Linux_minner-2023-05-16-09-20-54](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-20-54.png)  
* 设置开机自启  
![Linux_minner-2023-05-16-09-21-09](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-21-09.png)  
![Linux_minner-2023-05-16-09-22-17](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-22-17.png)  
#### 传播  
* 利用spirit & sshpass进行ssh暴破  
spirit为开源的渗透工具，其会根据同级目录下的IP地址簿(即"ip.txt"文件)进行存活性探测，以方便后续传播。其同级目录下的sshpass文件为暴力破解组件，会根据同级目录下的密码本(即"alllib"文件)对可以访问到的服务器进行ssh暴破。下图为目录结构：  
![Linux_minner-2023-05-16-09-22-34](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-22-34.png)  
同时也利用"crontab"为spirit设置定时计划，定期进行ssh暴破。  
![Linux_minner-2023-05-16-09-22-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-22-48.png)  
* 利用利用xinetd，实现端口转发 & 重定向  
其会向下载xinetd，随后其向etc/xinetd.d/文件夹写入timesync、http_stream、https_stream、smtp_forward几个配置文件，实现将80、443、708、757、8080端口的访问转发到远程恶意主机上，通过这样的配置，当有客户端对本机的相应端口访问时，就会重定向至恶意主机实现代理。下面是其中一例：  
![Linux_minner-2023-05-16-09-23-03](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-23-03.png)  
#### 清理痕迹  
* 删除日志 & 命令行记录  
删除的日志包括：
![Linux_minner-2023-05-16-09-23-24](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-23-24.png)  
相关功能实现如下：  
![Linux_minner-2023-05-16-09-23-38](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-23-38.png)  
![Linux_minner-2023-05-16-09-23-49](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-23-49.png)  
* 禁用登录日志 & 授权日志 & 命令行记录  
相关功能实现如下：  
![Linux_minner-2023-05-16-09-25-04](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-25-04.png)  
![Linux_minner-2023-05-16-09-25-24](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-25-24.png)  
![Linux_minner-2023-05-16-09-25-41](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-25-41.png)  

### xbash
该脚本是一个"Makeself"文件，其可以自解压并执行上文中提到的[cronman](#adxintrin_b--cronman)文件，是持久化中的重要一环。  
![Linux_minner-2023-05-16-09-25-59](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-25-59.png)  

## ELF文件  
### libgcca & xfitaarch.sh  
libgcc_a & xfitaarch.sh是基于XMrig的基础，进行二次开发得到的挖矿病毒。XMRig是一个可以使用CPU/GPU配合RandomX、KawPow、CryptoNight和AstroBWT等算法进行虚拟货币挖矿的开源、跨平台程序。其二进制文件可以分别用于在Windows、Linux、macOS和FreeBSD平台上进行挖矿。此次攻击的样本在原基础上增加了hwloc模块，hwloc模块主要是解决不同架构中查询硬件资源的问题。  
![Linux_minner-2023-05-16-09-26-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-26-15.png)  
### spirit & sshpass  
spirit为开源的渗透工具，其同级目录下的sshpass文件为暴力破解组件，目录结构请见["传播"](#传播)部分。  
sshpass参数如下：  
![Linux_minner-2023-05-16-09-26-31](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Linux_minner-2023-05-16-09-26-31.png)  

# 排查  
可以通过排查以下信息，判断服务器是否失陷。
* 典型目录：
  * /root/gcclib 其中包含挖矿病毒
  * /etc/lib 其中包含adxintrin_b
  * /usr/lib/local 其中包含crtres_c
  * /etc/xbash 其中包含cronman* & xbash  
  * /usr/spirit 其中包含spirit & sshpass
  * /etc/xinetd.d 其中是否包含重定向至恶意C2的配置信息
* 计划任务  
  * /etc/cron.daily 检查其中是否包含xbash
  * /etc/cron.hourly 检查其中是否包含0anacron
  * /etc/crontab 检查其中是否包含恶意样本相关文件
* 开机自启  
  * 检查/etc/init.d 检查其中是否包含xinetd & crtend_b  
* 服务 & 进程  
  * 查看是否有占用资源过高的进程  
  * 查看是否存在服务"crtend_b"






