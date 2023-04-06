---
title: "Fake-net Cookbook"
description: "This is a cookbook for Fake-net!"
date: 2023-04-06T14:58:00+08:00
Tags: ["Fake-net","Malware Analysis"]
draft: false
weight: 1030
cover: 
    image: '![20230406150202](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/20230406150202.png)'
---

# Fake-net Cookbook
在样本分析的过程中，若样本在访问的C2服务器或URL时，发现其已经下线或失效，那么恶意样本将会直接结束运行，那我们的分析将很难再进一步。Fake-net可以很好的解决这一问题，而不再需要我们花费时间自行搭建一个服务端来进行模拟通信。  
## 模拟https通信  
### 配置前  
以访问"https://raw.githubusercontent.com"为例。在未进行相关配置之前，会因缺少证书而导致无法通过https协议访问指定域名。  
![2023-04-06-11-29-09](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-11-29-09.png)  
### 配置  
根据上图中的错误信息，可以明确得知是因为缺少有效的证书，而导致了错误的产生，那么创建相应的https证书，并将其写入Fake-net的配置文件中，即可通过fake-net实现模拟https请求。  
第一步，通过powershell在"Cert:\LocalMachine\My\"下，创建出指定域名的证书。  
```ps1
New-SelfSignedCertificate -DnsName "raw.githubusercontent.com" -CertStoreLocation "Cert:\LocalMachine\My\"
```  
第二步，打开MMC，添加证书管理单元。  
![2023-04-06-13-20-43](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-20-43.png)  
在"个人"下面找到刚刚创建的证书，并将其加入到"受信任的根证书颁发机构"中。  
![2023-04-06-13-25-04](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-25-04.png)  
随后导出该证书，且在导出时请选择"是，导出私钥"。  
![2023-04-06-13-29-24](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-29-24.png)  
第三步，由于导出的证书文件是".pfx"文件，而fake-net配置文件(在"Linstner/ssl_utils"下)中需要的是".pem"格式的文件。因此，我们需要转换文件格式，并将公钥写入"server.pem"中，私钥写入"privkey.pem"中。  
![2023-04-06-13-31-40](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-31-40.png)  
写入私钥命令:
```ps1
openssl.exe pkcs12 -in $(The path of certificate) -nocerts -out pkey.pem
openssl.exe rsa -in pkey.pem -out $(The path of peivate.pem)
```  
写入公钥命令：
```ps1
openssl.exe pkcs12 -in $(The path of certificate) -vlcerts -nokeys -out $(The path of server.pem)
```  
Fake-net不仅可以模拟简单的连通性请求，其还可以模拟文件传输，其会根据你所请求的文件类型，返回对应类型的默认文件。  
![2023-04-06-13-53-42](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-53-42.png)  
若需要返回指定文件，我们还可以通过配置"defaultFiles"文件夹下的文件，实现模拟下载指定文件的需求。例如，我们想通过"hxxps://raw.githubusercontent.com/IconStorages/images/main/icon14.ico"下载"icon14.ico"这个文件，只需在"defaultFiles"下根据URL构造对应的路径，并将"icon14.ico"这个文件放入其中即可。  
![2023-04-06-13-58-49](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-58-49.png)  
### 配置后  
完成前三步配置，即可解决通过https访问指定域名无证书的问题。  
![2023-04-06-13-45-45](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-13-45-45.png)  
配置"defaultFiles"后，可以完成指定文件下载。  
![2023-04-06-14-04-50](https://raw.githubusercontent.com/g0mxxm/blog_pictures/main/blog/Fake-net%20Cookbook/2023-04-06-14-04-50.png)  