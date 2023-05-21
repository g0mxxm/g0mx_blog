---
title: "Linux SRC Cookbook"
description: "This blog will record some commonly used commands in Linux src"
date: 2023-02-23T15:00:00+08:00
Tags: ["SRC","Cookbook","Linux"]
draft: false
weight: 1017
cover: 
    image: ''
---

# Linux应急响应常用命令
## 进程相关
### 总览  
- 查看内存和CPU占用情况
```sh
top
```
- 查看所有进程的详细信息
```sh
ps -auxwf (包括PID、PPID、CPU、内存、启动时间、命令行参数等)
lsof (所用打开的文件和进程的详细信息)
```
- 查看进程的工作目录
```sh
ls -alR /proc/*/cwd
ls -alR /proc/*/cwd 2>/dev/null | grep tmp
ls -alR /proc/*/cwd 2>/dev/null | grep dev
```
- 查看被删除了但仍在运行的进程
```sh
ls -alR /proc/*/cwd 2>/dev/null | grep deleted
```
- 查看网络行为
```sh
netstat -plan (所有网络连接的详细信息)
netstat -plant (所有TCP连接的详细信息)
ss -a -e -i (所有套接字的详细信息)
```
### 某一进程
- 查看命令行参数信息
```sh
cat /proc/<PID>/comm
cat /proc/<PID>/cmdline
```
- 查看某进程所在路径
```sh
ls -al /proc/<PID>/exe
```
- 查看某进程环境变量
```sh
cat /proc/<PID>/environ
```
## 目录和文件相关  
**需要关注的目录**  
- /tmp
- /var/tmp
- /dev/shm
- /var/run
- /var/spool
- user home  

**持久化**
- /etc/rc.local(包含了系统启动时需要执行的命令), 
- /etc/initd(含了系统服务的启动脚本), 
- /etc/rc*.d(包含了系统启动时需要执行的脚本), 
- /etc/modules(包含了系统启动时需要加载的内核模块), 
- /etc/cron*(包含了系统定时任务的配置信息), 
- /var/spool/cron/*(每个用户的定时任务的配置信息)  

**命令**
- 文件夹详细信息
```bash
ls -lap (显示当前文件夹下所有文件和目录的详细信息)
```
- 查看当前文件夹下所有文件类型
```bash
file * -p
```
- 查看隐藏的文件夹
```bash
find / -type d -name ".*"
```
- 查看属性为"不可变"的文件和目录
```bash
lsattr / -R 2> /dev/null | grep "\----i"
```
- 查找设置了SUID/SGID的文件
```bash
find / -type f \( -perm -04000 -o -perm 
-02000 \) -exec ls -lg {} \; 
```
- 查找没有对应用户和组的文件
```bash
find / \( -nouser -o -nogroup \) -exec 
ls -lg {} \;
```
- 查找系统中所有ELF文件
```bash
find / -type f -exec file -p '{}' \; | 
grep ELF
```
- 查找所有命名管道文件
```bash
find / -type p
```
- 查找一天内被修改的文件
```bash 
find / -mtime -1
```
- MD5校验不通过的文件
```bash
rpm -Va | grep ^..5.
```
- Debian软件包完整性校验
```bash
debsums -c
```
## 用户相关  
- 查找所有"authorized_keys"文件
```bash
find / -name authorized_keys
```
- 查找".history"结尾的文件
```bash
find / -name .*history (所有)
ls -alR / 2> /dev/null | grep .*history 
| grep null (不包含"null")
```
- 查找所有UID\GID为0的用户
```bash
grep ":0:" /etc/passwd (即root用户)
```
- 查看所有sudo权限的用户和组
```bash
cat /etc/sudoers
```
- 查看所有用户和组
```bash
cat /etc/group
```
- 查看计划任务(即持久化)
```bash
crontab -l (当前用户任务计划列表)
atq (列出所有at任务)
systemctl list-timers --all (列出所有定时器)
```
## 日志
- 列出/var/log下所有日志详情
```bash
ls -al /var/log/*
```
- 登录日志
```bash
utmpdump /var/log/wtmp (显示/var/log/wtmp文件中的登录和注销记录)
utmpdump /var/run/utmp 
utmpdump /var/log/btmp
last (显示所有用户最近的登录记录)
lastb (显示所有用户最近的登录失败记录)
```
- 查找包含"[[:cntrl:]]"的日志
```bash
grep [[:cntrl:]] /var/log/*.log
```


