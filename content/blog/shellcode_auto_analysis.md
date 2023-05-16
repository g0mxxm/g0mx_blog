---
title: "The Shellcode In Maldoc Auto Analysis"
description: "This blog will discuss something about how to auto analysis shellcode in maldoc"
date: 2022-11-17T15:00:00+08:00
Tags: ["Malware Analysis","shellcode","Maldoc","Windows"]
draft: false
weight: 1021
cover: 
    image: '/cover/malware.jpg'
---

PS: 本文中所用的shellcode自动提取工具的具体实现可以访问本人[github项目仓库](https://github.com/g0mxxm/shellcode_extractor_for_maldoc)，原理讲解请访问本人首发在看雪的[文章](https://bbs.kanxue.com/thread-275559.htm)。

# 背景  
随着国家对网络安全重视程度的不断增强，以及网络安全建设工作的全面推进，传统的恶意样本想要突破边界已非一件易事，因而各种黑产组织在如何突破边界上做了许多功课，其中最常用的手段就是钓鱼。攻击者利用精心构造的邮件和文档，进行社工，来达到突破边界的目的。在这些精心构造的文档中通常包含两种攻击方式：利用宏指令，以及利用office漏洞。以宏指令作为攻击前戏的样本分析报告，可以查看本人在9月15日发布于兰云银河实验室公众号上的“HANCITOR样本分析报告”，其中对宏指令的所作所为进行了详细的阐述。那今天我们就来谈一谈第二种手法，以利用office漏洞为基础的前戏都进行了哪些操作。
# 自动化  
在背景中也阐述了，目前maldoc十分流行，大家也一定经常和它打交道，在分析maldoc时，若其中存在宏或者shellcode，那么如何把宏和shellcode从中提取出来就是我们不得不面对的一个问题。对于宏，本公司兰眼沙箱、以及微步的云沙箱已经可以进行宏命令全自动提取，方便大家对宏的具体操作进行进一步分析；但对于shellcode的提取，往往采用手动方式进行查找、定位、提取，这一系列操作往往需要花费较长的时间，让分析过程变得十分痛苦。基于以上原因，本人实现了一个自动化提取OLE中shellcode的小工具，该工具成型时间较短，还不是很完善，但可以解决该情况下绝大多数问题，极大的提高了分析效率。  
# 看个例子  
选取利用什么漏洞的样本为例好呢，通过查看兰云银河实验室恶意样本库，发现“cve-2017-11882”该漏洞深受攻击者喜爱，那就以利用该漏洞的样本为例，快速分析一下其shellcode的行为。  
![shellcode_auto_analysis-2023-05-16-10-21-36](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-21-36.png)  
样本基本信息：  
![shellcode_auto_analysis-2023-05-16-10-24-20](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-24-20.png)  
动用上文提到的shellcode自动化提取工具，对其中的shellcode进行提取，此工具会先判断文档是否被加密，若文档处于被加密状态则先对其进行解密，随后再进行shellcode提取操作，提取到的shellcode会被写入“final_shellcode_file”文件中。  
![shellcode_auto_analysis-2023-05-16-10-24-44](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-24-44.png)  
分析shellcode的方式有很多，例如替换内存或打包成exe，随后进行动调即可，但本文的关键是如何提高分析shellcode的效率，因为这往往是恶意样本的前戏，不应花费过多的时间，因而静态分析和模拟执行应该作为分析的首选方案。  
静态分析，我们可以用二进制查看工具对提取出的shellcode进行快速查看，看看有没有一些可利用的信息。因为在提取过程中本工具会对提取到的shellcode进行解码操作，若提取的shellcode可以成功解码，那么就可以像本例一样快速且直观的获取到关键信息。可以看到其中有LoadLibraryW和GetProcAddress，看到这俩函数可以确定，其在获取目标函数的地址，方便后续使用；又看到ExpandEnvironmentStringsW，可以确定其是将其中的“%APPDATA%”进行扩展；随后再调用URLDownloadToFileW来下载后续文件，写入指定路径中。如此一个流程下来，仅用两三分钟的时间就理清了恶意样本前戏的主要行为是什么。  
![shellcode_auto_analysis-2023-05-16-10-25-11](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-25-11.png)  
当然静态分析往往都不太完美，可以看到上图中，有一些信息并没有解码成功，面对这种情况，我们可以选择模拟执行来获取更加准确的信息，通过shellcodedbg执行后的结果，可以更加清晰地看到，其调用了哪些函数，参数是什么，整体流程与上文通过静态分析得到的结果基本一致，但模拟执行向我们展示了准确的URL，以及从远端下载了什么。  
![shellcode_auto_analysis-2023-05-16-10-25-22](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-25-22.png)  
该网站服务器目前依然存活，但其中页面访问失败，可能攻击者已对该网站的文件结构进行了更新。  
![shellcode_auto_analysis-2023-05-16-10-25-50](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-25-50.png)  
![shellcode_auto_analysis-2023-05-16-10-25-57](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-25-57.png)  
并且我们可以看到VT上已经对其进行了标记，到此我们已完成对恶意样本前戏的分析，若能获取到下载文件，那么我们便可以对该文件进行更加深入的分析，由于本文重点不在于此，便不再过多赘述。  
![shellcode_auto_analysis-2023-05-16-10-33-44](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/shellcode_auto_analysis-2023-05-16-10-33-44.png)

# 总结  
可以看到，在通过shellcode自动提取工具快速得到shellcode后，通过简单的静态分析以及模拟执行可以非常快速的对shellcode进行分析，相较于传统地动态调试分析，虽然会存在一些细节上的缺失，但不影响最终结果，且效率极高，因而不失为一条上策。  