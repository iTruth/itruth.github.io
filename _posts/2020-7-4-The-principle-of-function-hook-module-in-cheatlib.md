---
title: cheatlib中函数钩子模块的原理
date: 2020-7-4 12:40:00 +0800
categories: [Tutorials, Function Hook]
tags: [tutorials, function hook, cheatlib]
---

#### [点此查看cheatlib全部源代码](https://github.com/iTruth/cheatlib)
## 函数钩子的原理
函数钩子本质上劫持函数调用让一个函数执行前先去执行我们的函数然后在我们的函数里决定是否要执行源函数  
本质上就是在函数头写一个jmp指令直接跳到我们的函数.因为参数已经压栈所以我们的函数定义要保证和被Hook函数的定义保持一致  
在某些外挂的应用下一般而言会写在dll里然后注入到目标程序里去替换对应函数为自己dll中的函数,下面我将介绍这种方法的原理  

## 实现原理
### FuncHook
```c

/* 说明:  将pOrigAddr处的函数直接替换为pHookAddr处的函数执行
* 注意:  pOrigAddr和pHookAddr处的函数定义必须一致
*        此函数一般写在dll中,注入到程序中将程序中的函数替换为dll中的
* 参数:  pOrigAddr - 源函数地址
*        pHookAddr - hook函数地址
* 返回值:PFuncHookInfo */
PFuncHookInfo FuncHook(LPVOID pOrigAddr, LPVOID pHookAddr)
{
        DWORD oldProtect;
        VirtualProtect(pOrigAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect);
        PFuncHookInfo ptInfo = (PFuncHookInfo)malloc(sizeof(FuncHookInfo));
        if(ptInfo == NULL) return NULL;
        ptInfo->pOrigFuncAddr = pOrigAddr;
        ptInfo->pHookFuncAddr = pHookAddr;
        ptInfo->last_return_value = 0;
        ptInfo->pbOpCode = (BYTE*)malloc(sizeof(BYTE)*5);
        if(ptInfo->pbOpCode != NULL) memcpy(ptInfo->pbOpCode, pOrigAddr, 5);
        JmpBuilder((BYTE*)pOrigAddr, (DWORD)pHookAddr, (DWORD)pOrigAddr);
        VirtualProtect(pOrigAddr, 5, PAGE_EXECUTE, &oldProtect);
        return ptInfo;
}
```
这里首先通过VirtualProtect函数改变页属性,使其变得可读可写可执行
> VirtualProtect(pOrigAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect);

然后定义PFuncHookInfo来保存必要的信息,其中PFuncHookInfo结构体定义如下:
```c
typedef struct _FuncHookInfo{
        LPVOID pOrigFuncAddr;       // 代码源地址
        LPVOID pHookFuncAddr;       // Hook代码源地址
        BYTE *pbOpCode;             // 机器码用于恢复现场
        int last_return_value;      // CallOrigFunc源函数返回值(eax)
        int last_return_2nd_value;  // 在返回值是有两个整型值的结构体时这里保存第二个元素(edx)
} FuncHookInfo, *PFuncHookInfo;
```
为了能够恢复现场,我们将函数的前5字节保存下来
> if(ptInfo->pbOpCode != NULL) memcpy(ptInfo->pbOpCode, pOrigAddr, 5);

然后直接在函数开头构建jmp指令来跳到我们的函数中
> JmpBuilder((BYTE*)pOrigAddr, (DWORD)pHookAddr, (DWORD)pOrigAddr);

其中函数JmpBuilder的实现如下
```c
void IntToByte(int i, BYTE *bytes)
{
        assert(bytes != NULL);
        bytes[0] = (byte) (0xff & i);
        bytes[1] = (byte) ((0xff00 & i) >> 8);
        bytes[2] = (byte) ((0xff0000 & i) >> 16);
        bytes[3] = (byte) ((0xff000000 & i) >> 24);
}

void JmpBuilder(BYTE *pCmdOutput, DWORD dwTargetAddr, DWORD dwCurrentAddr)
{
        assert(pCmdOutput != NULL);
        pCmdOutput[0] = 0xE9;
        DWORD jmpOffset = dwTargetAddr - dwCurrentAddr - 5;
        IntToByte(jmpOffset, pCmdOutput+1);
}
```
此函数将在给定地址上构建jmp指令

最后恢复页属性为只可执行
> VirtualProtect(pOrigAddr, 5, PAGE_EXECUTE, &oldProtect);

### FuncUnhook
```c
/* 说明:    撤销函数钩子
* 参数:    ptInfo  - FuncHook函数返回值
* 返回值:  void */
void FuncUnhook(PFuncHookInfo ptInfo)
{
        assert(ptInfo != NULL && ptInfo->pbOpCode != NULL);
        DWORD oldProtect;
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect);
        memcpy(ptInfo->pOrigFuncAddr, ptInfo->pbOpCode, 5);
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE, &oldProtect);
        free(ptInfo->pbOpCode);
        free(ptInfo);
}
```
此函数先修改页属性为可读可写可执行
> VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect);

然后恢复函数开头的代码
> memcpy(ptInfo->pOrigFuncAddr, ptInfo->pbOpCode, 5);

最后恢复页属性并释放资源
> VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE, &oldProtect);
>        free(ptInfo->pbOpCode);
>        free(ptInfo);

好了,到现在为止一个函数Hook的基本功能就算是完成了.现在到了最重要的部分,如何在我们自己的函数中去执行源函数
### 函数的返回值问题
如果源函数有返回值那么我们先要考虑函数的返回值如何保存,一般而言函数返回一个值都是保存至eax里,那么如果返回的是一个结构体呢?
#### 结构体的两种返回方式
##### 特殊方式
让我们先写一段代码看看这种比较特殊的返回方式
```c
#include <stdio.h>

typedef struct _st{
        int a;
        int b;
} st, *pst;

st test()
{
        return (st){1, 2};
}

int main()
{
        printf("%d\n", test().a);
        return 0;
}
```
我们定义了一个名为st的结构体,其中包含了两个int类型的变量
>typedef struct _st{
>    int a;
>    int b;
> } st, *pst;

我们在test函数中直接返回这个结构体
> return (st){1, 2};

最后在main函数中打印test函数返回的结构体中的第一个元素
> printf("%d\n", test().a);

现在我们看看test函数的汇编是什么样的
```asm
00401510 | B8 01000000           | mov eax,0x1                                         |
00401515 | BA 02000000           | mov edx,0x2                                         | edx:&"ALLUSERSPROFILE=C:\\ProgramData"
0040151A | C3                    | ret                                                 |
```
可以看到,它仅仅只是将1和2保存到eax和edx里,所以如果函数返回的结构体里只包含了两个整型值的话那么其值将会被保存到eax和edx里  
注意: 只有在结构体里面有两个或两个以下的元素并且元素都是整型值时才会采取这种返回方式
##### 一般方式
我们将代码改一改,将st结构体改成有三个int类型元素的结构体来看看有什么不同
```c
#include <stdio.h>

typedef struct _st{
        int a;
        int b;
        int c;
} st, *pst;

st test()
{
        return (st){1, 2, 3};
}

int main()
{
        printf("%d\n", test().a);
        return 0;
}
```
仅仅只是多了个元素而已,现在让我们看看test函数的汇编
```asm
00401510 | 55                    | push ebp                                            |
00401511 | 89E5                  | mov ebp,esp                                         |
00401513 | 8B45 08               | mov eax,dword ptr ss:[ebp+0x8]                      |
00401516 | C700 01000000         | mov dword ptr ds:[eax],0x1                          |
0040151C | 8B45 08               | mov eax,dword ptr ss:[ebp+0x8]                      |
0040151F | C740 04 02000000      | mov dword ptr ds:[eax+0x4],0x2                      | puts
00401526 | 8B45 08               | mov eax,dword ptr ss:[ebp+0x8]                      |
00401529 | C740 08 03000000      | mov dword ptr ds:[eax+0x8],0x3                      |
00401530 | 8B45 08               | mov eax,dword ptr ss:[ebp+0x8]                      |
00401533 | 5D                    | pop ebp                                             |
00401534 | C3                    | ret                                                 |
```
是不是一下子多了好多?我们自己看看下面这一行汇编
> 00401513 | 8B45 08               | mov eax,dword ptr ss:[ebp+0x8]                      |

这行汇编似乎在取函数的第一个参数,但奇怪的是我们的函数明明是是无参的.
然后看下一行汇编
> 00401516 | C700 01000000         | mov dword ptr ds:[eax],0x1                          |

你会发现这第一个参数还是一个地址,这句汇编把0x1也就是我们结构体的第一个元素的值写了进去.
最后我们回到main函数来看看test函数的调用过程
```asm
00401543 | 8D4424 14             | lea eax,dword ptr ss:[esp+0x14]                     | [esp+14]:sub_401570
00401547 | 890424                | mov dword ptr ss:[esp],eax                          | Arg1 = [esp]:sub_401535+1A
0040154A | E8 C1FFFFFF           | call <st.test>                                      | test
```
然后你会发现,这个第一个参数的地址来自于main函数的空间,test函数将直接把结构体写入到地址的指定空间内  
这时你就明白了在一般情况下返回结构体的函数会隐式接受一个用于保存结构体的空间地址作为其第一个参数,然后将构建的结构体直接写进去.这就相当于返回了一个结构体  
现在你已经知道了一个函数是如何返回值的,下面就要考虑如何在我们自己的函数中调用源函数了  

## 调用源函数
因为函数的返回方式不唯一,所以调用源函数需要分那个源函数是否是返回结构体的,我们先看一般情况,也就是返回值是不是一个结构体的情况
### CallOrigFunc宏
```c
/* 说明:  在Hook函数里调用源函数
* 注意:  函数参数必须一致,否则会出现栈损
*        不支持返回结构体的函数,否则可能会覆盖栈内的合法数据
* 参数:  PFuncHookInfo ptInfo  - FuncHook函数的返回值
*        ...                   - 函数参数 */
#define CallOrigFunc(ptInfo, ...) do{ \
        DWORD oldProtect; \
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect); \
        memcpy(ptInfo->pOrigFuncAddr, ptInfo->pbOpCode, 5); \
        cheatlib_func_caller(ptInfo->pOrigFuncAddr, __VA_ARGS__); \
        __asm__ __volatile__( \
                        "movl %%eax, %0;" \
                        "movl %%edx, %1;":: \
                        "m"(ptInfo->last_return_value), \
                        "m"(ptInfo->last_return_2nd_value): \
                        "eax", "edx"); \
        JmpBuilder((BYTE*)ptInfo->pOrigFuncAddr, (DWORD)ptInfo->pHookFuncAddr, (DWORD)ptInfo->pOrigFuncAddr); \
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE, &oldProtect); \
} while(0)

void __attribute__((naked)) cheatlib_func_caller(LPVOID pOrigFuncAddr, ...)
{
        __asm__ __volatile__(
                        "popl %%eax;"
                        "popl %%ebx;"
                        "pushl %%eax;"
                        "jmp *%%ebx;"
                        :);
}
```
在函数的开头依然是先修改页属性
> VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect); \

因为我们要执行源函数所以必须先恢复我们改掉的函数头
> memcpy(ptInfo->pOrigFuncAddr, ptInfo->pbOpCode, 5); \

然后调用源函数
> cheatlib_func_caller(ptInfo->pOrigFuncAddr, \__VA_ARGS__); \

下面我们看看cheatlib_func_caller具体干了什么
> void \__attribute__((naked)) cheatlib_func_caller(LPVOID pOrigFuncAddr, ...)

  
首先可以看到\__attribute__((naked)),这就是说这个函数是一个裸函数.这也就意味着编译器不会对此函数做任何处理.里面嵌入的汇编是什么样的最后就是什么样的  
  
因为参数已经压栈了,所以当执行到这个函数开头时堆栈应该是下面这样的:  
  
返回地址  
参数1 - 源函数地址(pOrigFuncAddr)  
参数2  
...  
参数n  
  
下面看看函数里的前3句汇编
> "popl %%eax;"
>        "popl %%ebx;"
>        "pushl %%eax;"

意思是将"返回地址"和"参数1 - 源函数地址(pOrigFuncAddr)"出栈并保存至eax和ebx里并重新将"返回地址"压栈,执行完这些堆栈会变成下面这样:  
  
返回地址  
参数2  
...  
参数n  
  
最后直接jmp到源函数中,这样就正常执行源函数了
> "jmp *%%ebx;"

回到CallOrigFunc中,在调用完源函数我们需要保存返回值
```c
        __asm__ __volatile__( \
                        "movl %%eax, %0;" \
                        "movl %%edx, %1;":: \
                        "m"(ptInfo->last_return_value), \
                        "m"(ptInfo->last_return_2nd_value): \
                        "eax", "edx"); \
```
只是简单的将eax和edx保存一下

最后重新将源函数头改回来并恢复页属性
> JmpBuilder((BYTE*)ptInfo->pOrigFuncAddr, (DWORD)ptInfo->pHookFuncAddr, (DWORD)ptInfo->pOrigFuncAddr); \
>        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE, &oldProtect); \

这样就实现了调用源函数的过程

### CallOrigFunc_RetStruct宏
```c
/* 说明:  在Hook函数里调用源函数
* 注意:  函数参数必须一致,否则会出现栈损
*        只支持返回结构体的函数,否则会出现栈损
*        如果结构体内的元素都是整型且数量小于或等于二的话
*        那么元素将分别保存在eax和edx里
*        这个情况下不适合使用此宏,而是使用CallOrigFunc宏
* 参数:  PFuncHookInfo ptInfo  - FuncHook函数的返回值
*        void *pSaveStructAddr - 函数返回的结构体保存位置
*        ...                 - 函数参数 */
#define CallOrigFunc_RetStruct(ptInfo, pSaveStructAddr, ...) do{ \
        DWORD oldProtect; \
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE_READWRITE, &oldProtect); \
        memcpy(ptInfo->pOrigFuncAddr, ptInfo->pbOpCode, 5); \
        cheatlib_ret_struct_func_caller(pSaveStructAddr, ptInfo->pOrigFuncAddr, __VA_ARGS__); \
        JmpBuilder((BYTE*)ptInfo->pOrigFuncAddr, (DWORD)ptInfo->pHookFuncAddr, (DWORD)ptInfo->pOrigFuncAddr); \
        VirtualProtect(ptInfo->pOrigFuncAddr, 5, PAGE_EXECUTE, &oldProtect); \
} while(0)

void __attribute__((naked)) cheatlib_ret_struct_func_caller(LPVOID pStructAddr, LPVOID pOrigFuncAddr, ...)
{
        __asm__ __volatile__(
                        "popl %%eax;"
                        "popl %%ebx;"
                        "popl %%ecx;"
                        "pushl %%ebx;"
                        "pushl %%eax;"
                        "jmp *%%ecx;"
                        :);
}
```
和CallOrigFunc区别是这个宏只用于处理一般情况下的返回结构体函数,其实现和CallOrigFunc差不多,大家可自行理解
## 应用实例
```c
#include "cheatlib_funchook.h"
#include <stdio.h>
#include <windows.h>

PFuncHookInfo ptInfo;

int WINAPI hmsgbox(HWND hWnd,LPCTSTR lpText,LPCTSTR lpCaption,UINT uType)
{
        CallOrigFunc(ptInfo, hWnd, "Your MessageBoxA has been hooked!", lpCaption, uType);
        return 0;
}

int hprintf(const char* str, ...){
        MessageBox(NULL, "hooked printf", str, MB_OK);
        return 0;
}

int main()
{
        ptInfo = FuncHook((LPVOID)&MessageBoxA, (LPVOID)&hmsgbox);
        FuncHook((LPVOID)&printf, (LPVOID)&hprintf);
        printf("main: printf()");
        MessageBoxA(NULL, "main: MessageBoxA()", "Info", MB_OK);
        FuncUnhook(ptInfo);
        MessageBoxA(NULL, "main: MessageBoxA()", "Info", MB_OK);
        return 0;
}
```
