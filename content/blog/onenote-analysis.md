---
title: "Onenote Malware Analysis"
description: "This blog will discuss something about Onenote malware"
date: 2023-03-05T15:00:00+08:00
Tags: ["Malware Analysis","Onenote","Maldoc","Windows"]
draft: false
weight: 1019
cover: 
    image: '/cover/malware.jpg'
---

# **"后宏时代" —— OneNote恶意样本的新衣**
## 概述  
随着网络安全不断地发展，检测技术不断地革新和完善，恶意样本的"旧外衣"大都难逃安全设备的法眼，因而便衍生出了"后宏时代"一词。辞旧迎新，从22走到23，恶意样本也添置了属于自己的"新外衣" —— OneNote。  
## 整体流程  
![onenote-analysis-2023-05-15-20-24-23](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-24-23.png)  
## OneNote_peDropper  
可以看到该样本的后缀是".one"，即Microsoft的OneNote文件。看一下它的文件头，发现其并不像office其它套件那样，是有含义的可识别字符，如：PK、OLE、rtf、PDF等等。  
![onenote-analysis-2023-05-15-20-25-37](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-25-37.png)  
Microsoft文档中解释到，OneNote文件的前16个字节是一个GUID，用于判断其具体文件类型。  
![onenote-analysis-2023-05-15-20-25-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-25-48.png)  
而该文档下的推荐阅读也引起了我的兴趣，"FileDataStoreObject"一看就像个能藏东西的地方。  
![onenote-analysis-2023-05-15-20-26-04](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-26-04.png)  
其结构如下图所示，也就是说从guidHeader开始算起，向后36个字节就是数据存储的起始之处，而被存储数据的长度紧跟在guidHeader之后，占8个字节。通过这些条件，就能获取到我们所需的内容。  
![onenote-analysis-2023-05-15-20-26-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-26-15.png)  
在转换GUID，进行查找之前，可以先尝试搜索已知的magic头，来看看有没有我们想要的信息。非常幸运，在offset为0x2aa4的地方找到了一个PE文件。  
![onenote-analysis-2023-05-15-20-26-25](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-26-25.png)  
从'MZ'向前推，可以看到标识："FileDataStoreObject"的GUID(红框)和我们所需要的数据长度(黄框)。根据文件头和长度将PE文件dump下来。  
![onenote-analysis-2023-05-15-20-26-35](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-26-35.png)  
## PE_vbsDropper  
从OneNote中dump下来的PE文件也非常有趣，从它的描述就可知这是个Dropper。  
![onenote-analysis-2023-05-15-20-27-36](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-27-36.png)  
由于功能明确，不用再动手分析，通过兰眼沙箱可以看到，其释放了"new.vbs"文件。  
![onenote-analysis-2023-05-15-20-27-43](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-27-43.png)  
## VBS_ps1Downloader  
由于该vbs文件被填充了大量的":"进行占位，因而其所占空间如此之大。  
![onenote-analysis-2023-05-15-20-28-01](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-28-01.png)  
然关键之处，不过寥寥数行耳。其从目标地址下载并执行了一个名为"rr.ps1"的powershell脚本。  
![onenote-analysis-2023-05-15-20-28-29](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-28-29.png)    
兰眼沙箱也完整的还原了整体攻击流程:  
![onenote-analysis-2023-05-15-20-28-41](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-28-41.png)  
## Powershell_peDropper  
通过文件大小，基本可以确定其中包含了下一阶段的payload。  
![onenote-analysis-2023-05-15-20-28-54](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-28-54.png)  
该ps1脚本的功能是，对"payload"中的值进行base64解密，将解密后的值dump下来得到下一阶段的样本。  
![onenote-analysis-2023-05-15-20-29-02](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-02.png)  
(都是base64相关的内容，暂且略去不表)  
![onenote-analysis-2023-05-15-20-29-09](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-09.png)  
## .NET_peDropper  
ps1脚本中的payload是个PE文件(.NET程序)  
![onenote-analysis-2023-05-15-20-29-17](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-17.png)  
![onenote-analysis-2023-05-15-20-29-22](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-22.png)  
在这个.NET的"main"函数中存在一些无意义的变量名称，但关键信息(红框)是可识别的，主要就是进行base64解码，并将解码后的payload注入到"RegAsm.exe"中,功能明确，不再进行过多分析。  
![onenote-analysis-2023-05-15-20-29-32](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-32.png)  
## QuasarRAT
将payload从内存中dump下来，根据红框中信息和几个class中的方法，很快就确定了这个最后的payload是QuasarRAT，功能也十分明了，没啥新的东西，此处便不再过度赘述。  
![onenote-analysis-2023-05-15-20-29-39](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/onenote-analysis-2023-05-15-20-29-39.png)  
## 总结  
正可谓你方唱罢我登场，各类免杀手段在如今是更迭不断，层出不穷。本文以一在野OneNote恶意样本为例，对OneNote文件格式和该样本的整体攻击流程进行了介绍和还原。希望本文可以对各位读者带来一些帮助。