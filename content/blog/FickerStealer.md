---
title: "Ficker Stealer Analysis"
description: "This blog will discuss something about FickerStealer and its packer how to work"
date: 2023-05-29T11:05:00+08:00
Tags: ["Malware Analysis","Stealer","Windows","packer"]
draft: false
weight: 1015
cover: 
    image: '/cover/malware.jpg'
---

# FickerStealer - rust窃密木马
## 概述
本文将主要介绍该窃密木马的执行全流程，且该流程也被当前大多数Windows恶意样本所采用，通过该执行流程，可以很好的隐藏自身，以逃避安全设备和安全软件的检测，可以说是目前常见逃逸思路中较为主流的一种思路。  
整体流程如下:  
![](2023-05-24-17-43-10.png)  
注:   
* [pe_load_shellcode](#第一阶段pe_load_shellcode)  
解密第二阶段shellcode并加载  
* [shellcode_load_shellcode](#第二阶段shellcode_load_shellcode)  
解密 & 解压第三阶段shellcode并加载  
* [shellcode_load_FickerStealer](#第三阶段shellcode_load_fickerstealer)  
解密 & 以"Process Hollowing"的手法加载FickerStealer
* [FickerStealer](#第四阶段fickerstealer)  
Rust窃密木马  

## 细节解析
### 第一阶段(PE_Load_Shellcode)
初始化阶段，该样本和绝大多数恶意样本相同，都采用了LoadLibrary和GetProcAddress来动态获取API。  
![FickerStealer-2023-05-29-10-51-50](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-51-50.png)  
与其他恶意样本有所区别的是，其所使用的并不是比较经典的VirtualAlloc，而是LocalAlloc来进行内存的分配。  
![FickerStealer-2023-05-29-10-52-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-52-15.png)  
随后其通过循环调用FUN_410F60，将下一阶段所需的数据填入刚刚分配的内存之中。  
![FickerStealer-2023-05-29-10-52-29](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-52-29.png)  
FUN_410F60中填充内存的部分如下:  
![FickerStealer-2023-05-29-10-52-45](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-52-45.png)  
观察一下填充到内存中的值，可以看到并没有明显的shellcode特征出现，由此推断后面一定存在解密函数。  
![FickerStealer-2023-05-29-10-53-03](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-53-03.png)  
再接着看下去，我们看到了一个有意思的"VirtualProtect"，大概率是出于躲避静态检测的考量，其将"VirtualProtect"做了混淆处理 -> "VertualBritect"，再通过修改内存的方式进行动态去混淆，随后动态获取"VirtualProtect"，从而修改上图中的内存权限。  
![FickerStealer-2023-05-29-10-53-26](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-53-26.png)  
继续跟进，看到了一个深度为2的循环调用，经过分析其功能是: 解密shellcode。其用到的解密算法为TEA，且其在解密函数中插入了大量垃圾指令(下图右侧红框中黄色高亮部分)，以达到提高逆向分析难度的目的。  
![FickerStealer-2023-05-29-10-53-41](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-53-41.png)  
去混淆后，解密算法还原如下:  
```C
int *__stdcall Decrypt_shellcode_411040(int *Value)
{
  unsigned int y; // esi
  unsigned int z; // edi
  unsigned int sum; // ebx
  int v4; // ebp
  int *result; // eax
  int v6; // ecx
  int v7; // [esp+10h] [ebp-6E4h]
  int v8; // [esp+10h] [ebp-6E4h]
  int v9; // [esp+14h] [ebp-6E0h]
  int v10; // [esp+14h] [ebp-6E0h]
  unsigned int v12; // [esp+2ACh] [ebp-448h] BYREF
  int key[4]; // [esp+2B0h] [ebp-444h]

  y = *Value;
  z = Value[1];
  v12 = *Value;
  sum = 0xC6EF3720;
  key[0] = dword_440150;
  key[1] = dword_440154;
  key[3] = dword_440158;
  key[2] = dword_44015C;
  v4 = 32;
  while ( 1 )
  {
    v7 = key[2] + (y >> 5);
    v9 = (sum + y) ^ (key[3] + 16 * y);
    v8 = v9 ^ v7;
    z -= v8;
    (_TEA_Y_Operation)(&v12, (sum + z) ^ (key[1] + (z >> 5)) ^ v10);
    sum += 0x61C88647;
    if ( !--v4 )
      break;
    y = v12;
  }
  result = Value;
  v6 = v12;
  Value[1] = z;
  *Value = v6;
  return result;
}
```  
完成shellcode解密后，修改EIP到shellcode起始位置，开始执行shellcode。  
![FickerStealer-2023-05-29-10-54-05](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-54-05.png)  
### 第二阶段(Shellcode_Load_Shellcode)
本阶段的shellcode，主要由两部分组成: 通过"LoadLibrary"和"GetProcAddress"来动态获取API & 加载，解密，解包并执行下一阶段shellcode。  
![FickerStealer-2023-05-29-10-54-15](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-54-15.png)  
在动态获取API的过程中，其先通过硬编码和自实现的checksum方法，通过查找PE文件的导出表，获取到"LoadLibrary"和"GetProcAddress"的内存地址。  
![FickerStealer-2023-05-29-10-54-26](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-54-26.png)  
随后，其通过拼接所需DLL和API的字符串，再调用"LoadLibrary"和"GetProcAddress"来获取所需的API。  
![FickerStealer-2023-05-29-10-54-40](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-54-40.png)  
接下来，我们关注shellcode处理部分，其中主要包含解密和解包两部分(下图黄框)。  
![FickerStealer-2023-05-29-10-54-54](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-54-54.png)  
解密部分非常简单，直接采用了循环异或。  
![FickerStealer-2023-05-29-10-55-07](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-55-07.png)  
解密后的内存如下:  
![FickerStealer-2023-05-29-10-55-26](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-55-26.png)  
随后调用"VirtualAlloc"分配内存，之后进入解包环节，解包部分不再对细节进行过多赘述，这种CFG是不是和带压缩壳的文件的CFG高度相似，都是循环套循环，尤其程序出口的切片会被多次访问。经过调试也验证了我们的想法。  
![FickerStealer-2023-05-29-10-55-43](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-55-43.png)  
解包后的内存如下:(是不是感觉不要太亲切)  
![FickerStealer-2023-05-29-10-56-01](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-56-01.png)  
随后一个JMP，到达下一阶段shellcode的入口。  
![FickerStealer-2023-05-29-10-56-34](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-56-34.png)  
### 第三阶段(Shellcode_Load_FickerStealer)  
本阶段的shellcode，由两部分构成: 动态获取API & 进程镂空启动FickerStealer。其中动态获取API的部分和第二阶段基本相同，便不再过多赘述。直接看进程镂空部分。  
通过动态调试，我们获取到了该阶段shellcode中进程镂空部分的所有API以及整体流程，如下图所示:  
![FickerStealer-2023-05-29-10-57-23](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-57-23.png)  
进程镂空(Process Hollowing)的整体流程，大致为如下四步:  
* 创建一个挂起状态的进程(CreateProcess(handle, flag = CREATE_SUSPENDED))  
![FickerStealer-2023-05-29-10-57-40](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-57-40.png)  
* 取消目标进程的内存映射(ZwUnmapViewOfSection | NtUnmapViewOfSection)  
* 为指定进程分配内存 & 写入数据(VirtualAllocEx + WriteProcessMemory)  
* 启动payload(SetThreadContext + ResumeThread)  
![FickerStealer-2023-05-29-10-57-51](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-57-51.png)  
![FickerStealer-2023-05-29-10-58-18](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-58-18.png)  

### 第四阶段(FickerStealer)
通过下面的主函数CFG，可以发现该样本的整体流程非常复杂，且其几乎不具有可识别的字符串。  
![FickerStealer-2023-05-29-10-58-36](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-58-36.png)  
通过对程序进行整体分析，发现了大量类似的结构，且跟进FUN_0042590c后，可以很明确的判断出这是一个解密函数，且发现其被多次调用，可以将其当作本次分析的入手点。  
![FickerStealer-2023-05-29-10-58-53](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-58-53.png)  
![FickerStealer-2023-05-29-10-59-14](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-59-14.png)  
将解密函数参数和解密算法进行还原，结果如下:  
![FickerStealer-2023-05-29-10-59-32](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-59-32.png)  
解密后我们得到了很多有意思的字符串:  
* 获取受害者IP地址  
![FickerStealer-2023-05-29-10-59-48](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-10-59-48.png)  
* 从远端下载文件  
![FickerStealer-2023-05-29-11-00-02](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-11-00-02.png)  
* 与浏览器相关的文件  
![FickerStealer-2023-05-29-11-00-22](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-11-00-22.png)  
* 与虚拟货币相关的文件  
![FickerStealer-2023-05-29-11-00-39](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-11-00-39.png)  
* 和Discord、Pidgin、Steam相关的文件  
![FickerStealer-2023-05-29-11-00-51](https://g0mx-picbed.oss-cn-beijing.aliyuncs.com/blogs/pictures/FickerStealer-2023-05-29-11-00-51.png)  

通过这些字符串，我们基本确定这是一个窃密木马，窃取了哪些数据也一目了然了，我们便不再进行过多的分析和赘述。  
## 总结  
本文对该窃密木马的整体执行流程进行了详细的分析，旨在通过此样本来介绍目前较为主流的一种样本逃逸思路。  
