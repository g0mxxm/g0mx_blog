---
title: "3CX Supply Chain Attack Analysis"
description: "This blog will tell something about 3CX Supply Chain Attack"
date: 2023-04-05T15:00:00+08:00
Tags: ["Malware Analysis","3CX","Supply Chain AttacK"]
draft: false
weight: 1020
cover: 
    image: '/cover/malware.jpg'
---

# 3CX 供应链攻击  
## 概述  
近日，企业电话管理系统供应商3CX遭到供应链攻击的事情，在安全圈中吵得沸沸扬扬。由其提供的VOIP应用程序3CXDesktopApp，遭到了攻击者的恶意篡改，其中包括了Windows和MacOS两个平台上所使用的客户端程序。
## 整体流程  
![3CX-2023-05-09-10-15-53](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-15-53.png)  
## 详细分析  
### MSI_Installer  
该恶意样本的外壳是一个带有有效数字签名的msi安装程序。  
![3CX-2023-05-09-10-17-13](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-17-13.png)  
安装后，整体目录结构如下图所示(%Programs% -> %Programs%/app)，图中黄框圈中的文件为本文重点关注的文件。  
![3CX-2023-05-09-10-17-24](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-17-24.png)  
### Mal_3CX_Launcher
在安装完成后，其会通过"%Programs%"下的"3CXDesktopApp.exe"来启动"%Programs%/app"下的"3CXDesktopApp.exe"。  
![3CX-2023-05-09-10-17-40](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-17-40.png)  
![3CX-2023-05-09-10-17-55](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-17-55.png)  
### Mal_3CX  
"%Programs%/app"下的"3CXDesktopApp.exe"同样是一个带有数字签名的程序，但在其导入表中，我们发现其引入了一个没有数字签名和任何版权信息的DLL文件(ffmpeg.dll)。  
![3CX-2023-05-09-10-19-05](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-19-05.png)  
![3CX-2023-05-09-10-19-12](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-19-12.png)  
### ffmpeg.dll  
在"ffmpeg.dll"中其利用"CreateFileW"、"GetFileSize"、"ReadFile"将同级目录(%Programs%/app)下的"d3dcompiler_47.dll"读取到内存中，可以看到"d3dcompiler_47.dll"是一个带有数字签名的文件。  
![3CX-2023-05-09-10-20-28](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-20-28.png)  
![3CX-2023-05-09-10-20-35](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-20-35.png)  
随后其判断指定内存地址是否是以"4D5A"开头，以判断是否成功将"d3dcompiler_47.dll"读取到内存中；再通过循环遍历的方式在内存中搜索加密块的开始标识"FEEDFACE"。  
![3CX-2023-05-09-10-20-58](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-20-58.png)  
若成功找到标识头，则循环读取硬编码密钥"3jB(2bsG#@c7"到内存中，根据流程可以确定这是一个RC4密钥。  
![3CX-2023-05-09-10-21-13](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-21-13.png)  
![3CX-2023-05-09-10-21-20](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-21-20.png)  
利用刚刚获取到的RC4密钥，对先前通过标识"FEEDFACE"定位到的加密块进行解密，得到一段shellcode。  
![3CX-2023-05-09-10-21-45](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-21-45.png)  
快速浏览一下shellocde，可以看到和文件操作相关的函数，以及"GetProcAddr"和"LoadLibray"这个获取API的经典组合。且在上图中我们可以看到，在执行shellcode的CALL后还有一个"VirtualProtect"，那么这个shellcode大概率还是一个过程体。  
![3CX-2023-05-09-10-23-05](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-23-05.png)  
### samcli.dll  
不出所料，在[shellcode+0x65D]的位置找到了一个PE文件头，在shellcode执行结束后，我们也在内存中发现了解密后的DLL文件。  
![3CX-2023-05-09-10-23-23](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-23-23.png)  
![3CX-2023-05-09-10-23-31](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-23-31.png)  
![3CX-2023-05-09-10-24-28](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-24-28.png)  
可以看到shellcode最后通过CALL调用了解密后DLL中的导出函数"DllGetClassObject"，并且其将"1200 2400 \"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) 3CXDesktopApp/18.11.1197 Chrome/102.0.5005.167 Electron/19.1.9 Safari/537.36\""以参数的形式传入"DllGetClassObject"中。  
![3CX-2023-05-09-10-24-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-24-48.png)  
![3CX-2023-05-09-10-25-08](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-25-08.png)  
在DllGetClassObject中其调用了"CreateThread"来创建新的线程，并调用第二个"VirtualProtect"修改区段权限，以便后续执行。  
![3CX-2023-05-09-10-25-24](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-25-24.png)  
跟进到新创建的线程中，可以看到上文中提及的参数"1200 2400 \"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) 3CXDesktopApp/18.11.1197 Chrome/102.0.5005.167 Electron/19.1.9 Safari/537.36\""以命令行参数的形式传给了新线程。  
![3CX-2023-05-09-10-25-36](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-25-36.png)  
![3CX-2023-05-09-10-25-44](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-25-44.png)  
![3CX-2023-05-09-10-25-49](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-25-49.png)  
随后，其从同级目录下的"manifest"文件中读取随机值。  
![3CX-2023-05-09-10-26-17](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-26-17.png)  
![3CX-2023-05-09-10-26-39](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-26-39.png)  
此段恶意代码并不会立即执行，而是利用"Sleep"函数进行休眠，若未到达指定时间到则会直接退出。  
![3CX-2023-05-09-10-27-01](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-27-01.png)  
休眠结束后，其从"HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography"下读取"MachineGuid"作为后续和C2服务器通信的标识。  
![3CX-2023-05-09-10-27-17](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-27-17.png)  
![3CX-2023-05-09-10-27-36](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-27-36.png)  
随后，先通过和"0xF"取余的方式，获取"icon%d"中"%d"的对应值；完成拼接后，再访问对应的"hxxps://raw.githubusercontent.com/IconStorages/images/main/icon%d.ico"获取15个包含加密后C2地址的ICO文件。  
![3CX-2023-05-09-10-27-57](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-27-57.png)  
![3CX-2023-05-09-10-28-14](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-28-14.png)  
得到ICO文件后，将其读入内存，并寻找标识"$"，这是加密数据的开始位置，找到位置后，调用解密函数对加密数据进行解密。  
![3CX-2023-05-09-10-28-28](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-28-28.png)  
![3CX-2023-05-09-10-28-53](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-28-53.png)  
跟进后可以发现，其通过base64和AES-GCM对加密数据进行解密，解密后得到C2地址。 
![3CX-2023-05-09-10-29-21](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-29-21.png)  
随后，其会将之前取得"MachineGuid"与"__tutma"进行拼接(__tutma = {MachineGuid})作为标识符，放入与C2服务器通信的请求包中。  
![3CX-2023-05-09-10-29-37](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/3CX-2023-05-09-10-29-37.png)  
由于，攻击者警惕性较高，已将C2服务器下线，无法获取后续的payload，因而本文也就搁笔于此。
## 总结  
由于无法拿到最后的payload，目前还无法确定攻击者的真实意图，但从攻击的整体流程和手法上来看，幕后黑手的身份必定非同凡响！此外，在2023年3月中旬左右，便有少量3CX用户发出了告警，但都被当作了误报进行处理，直到2023年3月29日，此次供应链攻击事件才被正式确认。要知道，3CX在全球190多个国家，拥有超过1200万用户，可见这是一起非常严重的供应链攻击事件，涉及到安全的各个层面。值得我们持续关注和深思。
