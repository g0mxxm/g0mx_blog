---
title: "Anti Debugger"
description: "The methods about anti-debugger on Windows & how to detect and bypass."
date: 2023-02-23T15:02:06+08:00
Tags: ["Anti Tech","Frida"]
draft: yes
weight: 1030
cover: 
    image: 
---

## IsDebuggerPresent()
IsDebuggerPresent是由Microsoft提供的用来检测程序是否处于调试状态的API，若返回值为1，则代表程序处于被调试状态。  
![2023-02-20-09-42-19](https://raw.githubusercontent.com/g0mxxm/Picture/main/images/2023-02-20-09-42-19.png)
返回值保存在eax中，只需将其修改为0，即可绕过。  
![](2023-02-20-09-47-42.png)  
## Check_PEB_BeingDebugged
通过检查PEB中的BeingDebugged字段，来判断程序是否处于调试状态。  
```C++
typedef struct _PEB {
  BYTE                          Reserved1[2];
  BYTE                          BeingDebugged;   //detect debugger
  BYTE                          Reserved2[1];
  PVOID                         Reserved3[2];
  PPEB_LDR_DATA                 Ldr;
  PRTL_USER_PROCESS_PARAMETERS  ProcessParameters;
  PVOID                         Reserved4[3];
  PVOID                         AtlThunkSListPtr;
  PVOID                         Reserved5;
  ULONG                         Reserved6;
  PVOID                         Reserved7;
  ULONG                         Reserved8;
  ULONG                         AtlThunkSListPtr32;
  PVOID                         Reserved9[45];
  BYTE                          Reserved10[96];
  PPS_POST_PROCESS_INIT_ROUTINE PostProcessInitRoutine;
  BYTE                          Reserved11[128];
  PVOID                         Reserved12[1];
  ULONG                         SessionId;
} PEB, *PPEB;
```  
在32位程序中，可以通过访问fs:[30]来获取PEB基址，其中BeingDebugged标志位和基址存在两个字节的offset，若获取的值为1，则只需将其改为0，即可绕过。
![](2023-02-20-09-59-48.png)  
在64位程序中，可以通过访问gs:[60]来获取PEB的基址，并且BeingDebugged的offset大小依旧为2，绕过方法和32位相同。
![](2023-02-20-10-09-11.png)  
## Check_PEB_NtGlobalFlag  
通过检查PEB中的NtGlobalFlag字段(offset为0x68)，来判断程序是否处于调试状态。    
![](2023-02-20-10-35-46.png)  
在32位程序中依旧通过访问fs:[30]来获取PEB基址，并通过偏移得到NtGlobalFlag。若进程未处于调试状态，则其值应为0；若进程处于被调试状态，则其中标志位: FLG_HEAP_ENABLE_TAIL_CHECK(0x10) & FLG_HEAP_ENABLE_FREE_CHECK(0x20) & FLG_HEAP_VALIDATE_PARAMETERS(0x40)将被设置，即NtGlobalFlag的值为0x70。只需将目标值修改为0，即可完成绕过。64位程序同理，此处不再进行过多赘述。  
![](2023-02-20-10-44-26.png)  
## CheckRemoteDebuggerPresent()
CheckRemoteDebuggerPresent也是由Microsoft提供的用来检查进程是否被调试的API，若返回值非0，则代表进程处于被调试状态。  
![](2023-02-20-11-06-14.png)  
由上图可知，结果存储在第二个参数的所在地址中，因此，若当返回值非0时，将其修改为0，即可完成绕过。  
![](2023-02-20-11-12-20.png)  
![](2023-02-20-11-13-13.png)  
## NtQueryInformationProcess()  
Microsoft提供的NtQueryInformationProcess API可以用来查询多种目标进程的相关信息，恶意样本构造者也利用这一功能，对程序是否处于调试状态进行检测。
![](2023-02-20-11-40-22.png)  
主要有以下几种方式:  
* ProcessBasicInformation(0)
* ProcessDebugPort(7)
* ProcessDebugObjectHandle(30)
* ProcessDebugFlags(31)  

Microsoft官方文档中对，对前两种检测方法进行了说明，后面两种由于涉及为公开的结构体，因而并未给出相应说明。  
![](2023-02-20-13-49-40.png)  
下面我们给出从搜索引擎中获取到的_PROCESSINFOCLASS结构体，及我们所需字段的offset。
```C++
typedef enum _PROCESSINFOCLASS
{
    ProcessBasicInformation, 					// 0x0
    ...
    ProcessDebugPort,							// 0x7
    ...
    ProcessDebugObjectHandle,					// 0x1E
    ProcessDebugFlags, 							// 0x1F
    ...
} PROCESSINFOCLASS;
```
### ProcessBasicInformation
其中通过ProcessBasicInformation对进程是否处于被调试状态的检测，其原理与检查PEB中的BeingDebugged标志的原理是相同的，只是不再通过访问fs:[30]来获取PEB基址了。下面我们来看一下ProcessBasicInformation的结构:  
```C++
typedef struct _PROCESS_BASIC_INFORMATION {
    NTSTATUS ExitStatus;
    PPEB PebBaseAddress;
    ULONG_PTR AffinityMask;
    KPRIORITY BasePriority;
    ULONG_PTR UniqueProcessId;
    ULONG_PTR InheritedFromUniqueProcessId;
} PROCESS_BASIC_INFORMATION;
```  
可以看到其中结构体中第二个元素是PEB基址，如下图所示:
![](2023-02-20-14-06-08.png)  
绕过方式也和Check_PEB_BeingDebugged所用绕过方法相同，只需将BeingDebugged标志位置0，即可完成绕过。  
![](2023-02-20-14-09-12.png)  
### ProcessDebugPort
通过ProcessDebugPort也可以对程序是否处于被调试状态进行检测，若返回值不为0，则说明程序处于被调试状态，另外需注意，当获取信息失败时，返回值为STATUS_INFO_LENGTH_MISMATCH (0xC0000004)，这时需要对测试代码进行相应调整，在编译成64位程序进行测试时，也可能会出现这种错误。  
同样我们需要关注第三个参数，其保存着返回结果，显而易见我们只需要将0xFFFFFFFF(-1)修改为0，即可完成绕过。  
![](2023-02-20-15-05-53.png)  
### ProcessDebugObjectHandle  
通过ProcessDebugObjectHandle也可以对程序是否处于被调试状态进行检测，若返回值不为0，则说明程序处于被调试状态。  
同样我们需要关注第三个参数，其保存着返回结果，显而易见我们只需要将红框中的数值修改为0，即可完成绕过。
![](2023-02-20-16-59-28.png)  
### ProcessDebugFlags  
通过ProcessDebugObjectHandle也可以对程序是否处于被调试状态进行检测，与其它几种不同的是其返回值为0，则说明程序处于被调试状态。
那我们将其第三个参数指向地址中的值改为非0值，即可完成绕过。  
![](2023-02-20-17-21-51.png)  
## FindWindowW  
可以通过Microsoft提供的FindWindowA来对最上层窗口的ClassName以及WindowName进行检查，若存在"OLLYDBG"、"ID"、"X32DBG"等和调试器相关的字段，则判定程序正在被调试。
![](2023-02-20-17-39-50.png)  
若返回值不为0，那将返回值置0，即可完成绕过。  
![](2023-02-20-17-54-05.png)  
## CheckProcessFileName  
恶意软件制造者通常还会检查进程的文件名，来判断其是否处于被调试状态，一般来说会用到的API有:
* CreateToolhelp32Snapshot，其会为进程创建快照，快照中的内容包括进程所使用的堆、模块以及线程，其返回值是访问所创建快照的句柄。
* Process32FirstW，检索快照中所遇到的第一个进程的信息，其返回值是一个指向PROCESSENTRY32W结构体所在位置的指针，而判断程序是否被调试的文件名就存在于该结构体的szExeFile[MAX_PATH]字段中。
```C++
typedef struct tagPROCESSENTRY32W {
  DWORD     dwSize;
  DWORD     cntUsage;
  DWORD     th32ProcessID;
  ULONG_PTR th32DefaultHeapID;
  DWORD     th32ModuleID;
  DWORD     cntThreads;
  DWORD     th32ParentProcessID;
  LONG      pcPriClassBase;
  DWORD     dwFlags;
  WCHAR     szExeFile[MAX_PATH];
} PROCESSENTRY32W;
```
* Process32NextW，随后利用该API继续查询快照中的下一个进程的相关信息。  

可以看到，基本流程就是先在内存中写入与调试器，逆向工具相关的字符串，再去快照中进行对比，我们可以直接将这些字符串patch掉，或者对字符串比较结果的返回值进行修改，即可完成绕过。  
![](2023-02-20-19-01-03.png)  
## NtSetInformation()
NtSetInformation是由Microsoft提供的用来设置线程状态的API，其在反调试上有两种应用方法。其一，隐藏一个带有反调试功能的子线程，以绕过反反调试手段；七二，将主线程隐藏，若进程处于调试状态，则会造成程序崩溃，不再继续运行。下面会以隐藏主线程为例，来看一下如何进行绕过。  
![](2023-02-21-19-36-25.png)  
第一种方式，我们可以将0x11修改为其它值(想了解更为详细的参数作用，可查阅与THREADINFOCLASS结构体相关的内容)，即可完成绕过，虽然会返回错误码，但不会影响程序的正常运行。
![](2023-02-21-19-43-25.png)  
第二种方式，可以在NtSetInformation处下断，将给eax的值修改为不是0xD的即可，建议为0，具体为啥我也不太明白，但确实可以完成绕过。
![](2023-02-21-19-48-08.png)

