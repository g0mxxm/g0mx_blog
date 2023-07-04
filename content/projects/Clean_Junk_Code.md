---
title: "Clean_Junk_Code"
description: "This blog will discuss something about how to clean the junk code in a basic block!"
date: 2023-06-27T11:31:36+08:00
Tags: ["Obfuscate","C++"]
draft: true
weight: 25
cover: 
    image: 
---

# 混淆指令清洗
## 概述
本文将主要介绍，清洗一个"Basic Block"中无用指令的原理及实现方案。该项目用到的主要技术包括Capstone和活跃变量分析，最后，以本人在工作时所遇到的一恶意样本中"带混淆加密算法"分析为例，展示了该项目的[使用方式和实际效果](#usage--example)。  
## 语言及环境  
- 语言: C++
- 环境: VS2022
- 项目地址: https://github.com/g0mxxm/Anti-Obfuscation
## 实现细节
整体流程:
- 第一阶段: 解析PE文件，根据RVA获取相应的机器码
- 第二阶段: 利用Capstone对机器码进行反汇编; 随后以"活跃变量分析"为理论基础，对混淆指令进行清洗

### 解析PE文件  
本文主要关注的是，如何通过虚拟地址(Virtual Address, VA)来获取文件中的对应数据，因而不再对PE结构进行详细解释，重点聚焦于文件偏移和虚拟地址之间的转换。  
#### 原理  
以《加密与解密中》中的示意图为例:  
![Clean_Junk_Code-2023-07-04-09-40-33](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-40-33.png)  
做如下定义:  
* 文件在磁盘中的".text"段与文件头的偏移大小为"add1"，该偏移被称作——文件偏移地址(File Offset)
* 文件在内存中的".text"段与文件头的偏移大小为"add2"，该偏移被称作——相对虚拟地址(Relative Virtual Address, RVA)
* 假设"add1"和"add2"之间的差值为"$\Delta$k"  

可得出如下表达式:  
File Offset = RVA - $\Delta$k  
File Offset = VA - ImageBase - $\Delta$k  

#### 实现  
*过程如下* 
* 读取PE文件的DOS头，并通过其中的"e_lfanew"字段定位到NT头
* 根据NT头中的"NumberOfSections"字段遍历PE文件中的各个节  
* 根据每个节头中的"VirtualAddress"和"Misc.VirtualSize"字段判断目标"RVA"是否在该节中
* 若目标"RVA"在该节中，则利用节头中的PointerToRawData字段，根据上文中提到的公式"File Offset = RVA - $\Delta$k"，计算文件偏移。(注：其中$\Delta$k = section_header.PointerToRawData - section_header.VirtualAddress，有人可能会疑惑，上面公式中说的不是RVA吗，这里为什么是"section_header.VirtualAddress"，其实"section_header.VirtualAddress"就是目标节的RVA)

![Clean_Junk_Code-2023-07-04-09-40-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-40-15.png)  

### 活跃变量分析
#### 原理  
以下部分引用自——[活跃变量分析，可用表达式分析](https://blog.csdn.net/weixin_42815609/article/details/108022060)，在此感谢大佬分享。  
* 活跃变量:  
对于变量x和程序点p，如果在流图中沿着从p开始的某条路径会引用变量x在p点的值，则称变量x在点p是活跃的，否则称变量x在点p是不活跃的。  
![Clean_Junk_Code-2023-07-04-09-39-58](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-39-58.png)  
* 活跃性计算:  
  * 活跃变量数据流方程:  
  注: IN[N]为在基本块N入口处的活跃变量集合 & OUT[N]为在基本块B的出口处的活跃变量集合
  ![Clean_Junk_Code-2023-07-04-09-39-45](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-39-45.png)  
  * 计算活跃变量的迭代算法:    
  ![Clean_Junk_Code-2023-07-04-09-39-30](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-39-30.png)  
  具体例子可在"[活跃变量分析，可用表达式分析](https://blog.csdn.net/weixin_42815609/article/details/108022060)"一文中查阅

#### 实现  
该实现思路的完善和理顺来自——[利用活跃变量分析来去掉vmp的大部分垃圾指令](https://bbs.kanxue.com/thread-265950.htm)，在此感谢大佬分享。  
*过程如下*  
* 通过capstone提供的"cs_regs_access"获取并记录每条指令的"寄存器读写"情况(注: 对寄存器的"读"即为"use"，对寄存器的"写"即为"def")  
  ![Clean_Junk_Code-2023-07-04-09-39-19](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-39-19.png)  
* 以"基本块的最后一条指令为起点"，从后向前，根据上文提到的"活跃变量数据流方程"，计算每条汇编指令的出口活跃信息(OUT[N])和入口活跃信息(IN[N])  
  ![Clean_Junk_Code-2023-07-04-09-39-08](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-39-08.png)  
* 随后根据每条指令的"写"信息(即"def"信息)来判断该指令是否为垃圾指令，若"写"信息(即"def"信息)中的所有寄存器都不是出口活跃的，那么这条指令就是无用的，我们则将其删除  
  ![Clean_Junk_Code-2023-07-04-09-38-54](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-38-54.png)  
  ![Clean_Junk_Code-2023-07-04-09-38-43](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-38-43.png)  
* 不断重复"第二步 & 第三步"，直到某一次计算后，没有新的无用指令出现，至此，清洗完成  
  ![Clean_Junk_Code-2023-07-04-09-38-24](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/Clean_Junk_Code-2023-07-04-09-38-24.png)  

## Usage & Example  
注: 本例来自一个恶意Loader中的解密算法部分(样本MD5为:B7D90C9D14D124A163F5B3476160E1CF)，若对该样本的整体流程感兴趣，可阅读本人所撰写的该样本分析[文章](https://g0mx.netlify.app/blog/fickerstealer/)，可能会有所收获!  
去混淆前:  
```x86asm
.text:00411190                               loc_411190:                   ; CODE XREF: decrypt_shellcode_411040+111↑j
.text:00411190 2B 7C 24 10                   sub     edi, [esp+6F4h+var_6E4]
.text:00411194 C7 84 24 78 01 00 00 C5 BC 71+mov     [esp+6F4h+var_57C], 4B71BCC5h
.text:00411194 4B
.text:0041119F C7 84 24 1C 02 00 00 53 8D 22+mov     [esp+6F4h+var_4D8], 14228D53h
.text:0041119F 14
.text:004111AA C7 84 24 A4 00 00 00 55 D3 1F+mov     [esp+6F4h+var_650], 4B1FD355h
.text:004111AA 4B
.text:004111B5 C7 84 24 EC 00 00 00 40 0D 8E+mov     [esp+6F4h+var_608], 458E0D40h
.text:004111B5 45
.text:004111C0 C7 84 24 D4 01 00 00 6E E9 24+mov     [esp+6F4h+var_520], 7524E96Eh
.text:004111C0 75
.text:004111CB C7 84 24 A0 01 00 00 FA 0B 77+mov     [esp+6F4h+var_554], 6B770BFAh
........
.text:00412994 81 84 24 F4 01 00 00 52 F7 57+add     [esp+6F4h+var_500], 5657F752h
.text:00412994 56
.text:0041299F 81 AC 24 08 02 00 00 5C EB 35+sub     [esp+6F4h+var_4EC], 1B35EB5Ch
.text:0041299F 1B
.text:004129AA 81 84 24 40 01 00 00 C4 CB 36+add     [esp+6F4h+var_5B4], 7336CBC4h
.text:004129AA 73
.text:004129B5 81 6C 24 28 8B 9F 9C 39       sub     [esp+6F4h+var_6CC], 399C9F8Bh
.text:004129BD 81 84 24 80 02 00 00 7E C6 AC+add     [esp+6F4h+var_474], 20ACC67Eh
.text:004129BD 20
.text:004129C8 8B C7                         mov     eax, edi
.text:004129CA C1 E0 04                      shl     eax, 4
.text:004129CD 89 44 24 14                   mov     [esp+6F4h+var_6E0], eax
.text:004129D1 8B 84 24 B0 02 00 00          mov     eax, [esp+6F4h+var_444]
.text:004129D8 01 44 24 14                   add     [esp+6F4h+var_6E0], eax
.text:004129DC 81 3D 74 44 83 00 96 01 00 00 cmp     uBytes, 196h
.text:004129E6 8D 34 3B                      lea     esi, [ebx+edi]
.text:004129E9 75 0E                         jnz     short loc_4129F9
```  

下面我们利用指令去混淆工具对该段指令进行清洗。  
启动指令如下: 
```ps1
(base) PS C:\Users\g0mx> D:\Code_Files\Deobfusecate\Release\Clean_BB.exe -f D:\Malware_Analysis\packer\packer\mal_packer -s "411190" -e "4129EB"
```

清洗后的指令如下
```x86asm
Insn List After Clean:
0x411190:       sub             edi, dword ptr [esp + 0x10]
0x411194:       mov             dword ptr [esp + 0x178], 0x4b71bcc5
0x41119f:       mov             dword ptr [esp + 0x21c], 0x14228d53
0x4111aa:       mov             dword ptr [esp + 0xa4], 0x4b1fd355
0x4111b5:       mov             dword ptr [esp + 0xec], 0x458e0d40
.........
0x411826:       mov             dword ptr [esp + 0x210], 0x3884b2e9
0x411831:       mov             dword ptr [esp + 0x234], 0x5a5beac9
0x41183c:       mov             dword ptr [esp + 0x22c], 0x71a13663
0x411847:       mov             dword ptr [esp + 0x218], 0x36e78a71
0x411852:       mov             dword ptr [esp + 0x224], 0x4b2fcf14
0x4129c8:       mov             eax, edi
0x4129ca:       shl             eax, 4
0x4129cd:       mov             dword ptr [esp + 0x14], eax
0x4129dc:       cmp             dword ptr [0x834474], 0x196
0x4129e9:       jne             0x4129f9
clean over!!!
```
## 总结  
希望大家读的开心，有所收获。目前，本人也在学习基本块合并的相关知识，使得清洗范围可以扩大到一个函数或整个程序，希望各位大佬可以给出更多相关资料和意见。对于本文有任何想问的以及意见也欢迎随时指出。

## Reference  
* 看雪大佬会飞的鱼油——[利用活跃变量分析来去掉vmp的大部分垃圾指令](https://bbs.kanxue.com/thread-265950.htm)  
* CSDN上——[活跃变量分析，可用表达式分析](https://blog.csdn.net/weixin_42815609/article/details/108022060)
* [capstone官方文档](https://www.capstone-engine.org/)
* 看雪上大量相关文章，此处不再一一例举，再次感谢各位大佬的分享
* 《加密与解密(第四版)》& 龙书
