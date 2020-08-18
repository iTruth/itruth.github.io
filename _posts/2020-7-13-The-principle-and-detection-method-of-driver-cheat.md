---
title: 驱动外挂的原理及检测手段
date: 2020-7-13 12:40:00 +0800
categories: [Tutorials, Driver Cheat]
tags: [tutorials, driver cheat, ring0]
---

因为PatchGuard技术的存在导致游戏在驱动层的保护不能像以前那样通过SSDT Hook或者IDT Hook来做了,游戏厂家不能擅自关闭PatchGuard来强行Hook.这样留给驱动挂的空间就很大了  
我将以一个 __自瞄挂的原理__ 为例子展示驱动外挂的几种实现方式及检测手段

## 相同的驱动获取及控制手段
要想实现 __自瞄驱动挂__ 基本上都是读取游戏数据然后直接操作鼠标,不同之处就是操纵鼠标的方式
下面是所有驱动挂的几个相同之处
### 相同点1 - 获取鼠标的驱动对象
一个自瞄驱动挂的实现首先必然需要获得鼠标的驱动对象,在任何驱动挂里应该都是一样的
可以通过ObReferenceObjectByName来获取鼠标驱动对象  
其中ObReferenceObjectByName是未公开的函数,声明一下就能用.其声明如下
```c
NTSTATUS ObReferenceObjectByName(
        PUNICODE_STRING ObjectName,
        ULONG Attributes,
        PACCESS_STATE AccessState,
        ACCESS_MASK DesiredAccess,
        POBJECT_TYPE ObjectType,
        KPROCESSOR_MODE AccessMode,
        PVOID ParseContest,
        PVOID* Object
);

extern POBJECT_TYPE* IoDriverObjectType; // 这是ObjectType参数的实参
```
其中鼠标驱动的名称为\\\\Driver\\\\MouClass  
那么具体获取鼠标驱动对象过程如下:
```c
  NTSTATUS status = STATUS_SUCCESS;
  UNICODE_STRING mouseName;
  RtlInitUnicodeString(&mouseName, L"\\Driver\\MouClass");
  // 获取到鼠标驱动对象将保存至此
  PDRIVER_OBJECT mouseDriver;

  status = ObReferenceObjectByName(&mouseName, OBJ_CASE_INSENSITIVE, NULL, 0, *IoDriverObjectType, KernelMode, NULL, &mouseDriver);
  if (!NT_SUCCESS(status)) {
    return status;
  } else {
    // 获取失败了需要解引用
    ObDereferenceObject(mouseDriver);
  }
```
这样一个鼠标驱动对象就在mouseDriver变量中了

### 相同点2 - 控制驱动挂的手段
用户要想控制驱动首先要通过CreateFile来打开驱动设备  
也就是说在驱动里必然先要创建好一个设备供用户打开,具体步骤如下
```c
// 设备名和符号名的定义
#define ITRUTH_DEVICE_NAME L"\\Device\\iTruth_Device_20d04fe0"
#define ITRUTH_SYMB_NAME L"\\DosDevice\\iTruth_Device"

        // 函数里设备创建过程
        UNICODE_STRING dev_name;
        RtlInitUnicodeString(&dev_name, ITRUTH_DEVICE_NAME);
        UNICODE_STRING sddl;
        // SDDL语法请参考https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/sddl-for-device-objects
        RtlInitUnicodeString(&sddl, L"D:P(A;;GA;;;BU)");
        // 这个请自己定,每个驱动都不能一样
        GUID dev_guid = { 0x3cff2c3aL, 0x320f, 0xf5aa, "iTruth" };

        // 创建设备
        status = IoCreateDeviceSecure(
                DriverObject,
                0,
                &dev_name,
                FILE_DEVICE_UNKNOWN,
                FILE_DEVICE_SECURE_OPEN,
                FALSE,
                &sddl,
                &dev_guid,
                &iTruth_Device
        );

        if (NT_SUCCESS(status)) {
                UNICODE_STRING dos_dev_name;
                RtlInitUnicodeString(&dos_dev_name, ITRUTH_SYMB_NAME);
                // 为设备绑定符号链接,用户只能通过这个符号链接打开设备
                IoCreateSymbolicLink(&dos_dev_name, &dev_name);

                // 绑定处理函数
                DriverObject->MajorFunction[IRP_MJ_CREATE] = iTruth_DriverDispatch;
                DriverObject->MajorFunction[IRP_MJ_CLOSE] = iTruth_DriverDispatch;
                DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = iTruth_DriverDispatch;
        }
```
在用户层控制驱动基本上就是DeviceIoControl函数了,此函数向指定驱动发送IO控制码(CTL_CODE),其中控制码可以由一个名为CTL_CODE的宏来定义,这个宏的声明如下
```c
#define CTL_CODE(DeviceType, Function, Method, Access) (
  ((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method)
)
```
第一个参数是驱动的类型,驱动挂一般就填FILE_DEVICE_UNKNOWN即可.  
第二个是操作码,这个可以自已定.  
第三个是传递缓冲区的方式,METHOD_BUFFERED是比较简单的方式  
第四个是完成此动作需要的权限,不知道就填FILE_ANY_ACCESS  
下面是IOCTL的两个例子:  
```c
#define IOCTL_ID_DBGPRINT CTL_CODE(FILE_DEVICE_UNKNOWN, 0x700, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_ID_FUNCHOOK CTL_CODE(FILE_DEVICE_UNKNOWN, 0x701, METHOD_BUFFERED, FILE_WRITE_DATA)
```
用户层的CTL_CODE和内核里的应该是一样的,所以这个控制码的定义一般单独放在一个 __头文件__ 内  
驱动的入口函数是 __DriverEntry__ ,其中第一个参数是PDRIVER_OBJECT类型的代表了当前驱动  
这个参数里有一个名为 __MajorFunction__ 的数组,里面包含了各种驱动处理函数  
这个数组第 __IRP_MJ_DEVICE_CONTROL__ (0x0e)个元素是处理IO的回调函数,这个值是自己指定的  
一个典型的处理IRP_MJ_DEVICE_CONTROL的函数如下  
```c
NTSTATUS DriverDispatch(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
        NTSTATUS status = STATUS_SUCCESS;
        PIO_STACK_LOCATION irp_loc = IoGetCurrentIrpStackLocation(Irp);

        if (DeviceObject == myDevice) {
                if (irp_loc->MajorFunction == IRP_MJ_CREATE || irp_loc->MajorFunction == IRP_MJ_CLOSE) {
                        IoCompleteRequest(Irp, IO_NO_INCREMENT);
                        return STATUS_SUCCESS;
                }

                for (PLIST_ENTRY pl = dev_list_head.Flink; pl != (PLIST_ENTRY)&dev_list_head.Flink; pl = pl->Flink) {
                        PITRUTH_DEV_ENTRY dev_entry = CONTAINING_RECORD(pl, ITRUTH_DEV_ENTRY, dev_list);
                        if (dev_entry->flt_dev_obj == DeviceObject) {
                                if (irp_loc->MajorFunction == IRP_MJ_DEVICE_CONTROL) {
                                        switch (irp_loc->Parameters.DeviceIoControl.IoControlCode) {
                                        //这里可以看到我们判断当前需要的功能部分
                                        case IOCTL_ID_DBGPRINT:
                                                KdPrint((Irp->AssociatedIrp.SystemBuffer));
                                                Irp->IoStatus.Status = STATUS_SUCCESS;
                                                break;
                                        case IOCTL_ID_FUNCHOOK:
                                                KdPrint(("Kernel: Hook\n"));
                                                Irp->IoStatus.Status = STATUS_SUCCESS;
                                                break;

                                        default:
                                                IoSkipCurrentIrpStackLocation(Irp);
                                                return IoCallDriver(dev_entry->next_dev_obj, Irp);
                                        }
                                }

                                IoSkipCurrentIrpStackLocation(Irp);
                                return IoCallDriver(dev_entry->next_dev_obj, Irp);
                        }
                }
        }
}
```
可以看到里面有根据我们的CTL_CODE来执行功能的switch语句,这个就是具体的控制原理

## 不同的鼠标控制手段
### 最简单的手段 - 通过过滤设备来控制鼠标
#### 原理
这种手段本质上是使用名为 __IoAttachDeviceToDeviceStack__ 的函数将自己创建的设备对象绑定到设备对象链中的最高层,然后使用自己Driver Dispatch回调函数来修改鼠标输入  
绑定过程如下:
```c
  // 在相同点那里已经展示了鼠标驱动设备的获取方式,所以这里省略
  // 获取鼠标驱动设备链中的第一个设备
        targetDevice = mouseDriver->DeviceObject;
        // 遍历设备链中的所有设备
        while (targetDevice)
        {
                // 创建一个过滤设备
                status = IoCreateDevice(pDriver, sizeof(DEV_EXTENSION), NULL, targetDevice->DeviceType, targetDevice->Characteristics, FALSE, &filterDevice);
                if (!NT_SUCCESS(status))
                {
                        filterDevice = targetDevice = NULL;
                        return status;
                }
                // 在这步绑定
                nextDevice = IoAttachDeviceToDeviceStack(filterDevice, targetDevice);
                if (!nextDevice)
                {
                        IoDeleteDevice(filterDevice);
                        filterDevice = NULL;
                        return status;
                }
                targetDevice = targetDevice->NextDevice;
        }
```
到了这步绑定好读取的分发函数那么鼠标的读取IRP就会发送到我们的驱动然后我们即可对其处理
> pDriver->MajorFunction[IRP_MJ_READ] = MouseIRPMJRead;

下面是鼠标读取IRP的处理
```c
NTSTATUS MouseIRPMJRead(PDEVICE_OBJECT pDevice, PIRP pIrp, PVOID Context)
{
        UNREFERENCED_PARAMETER(pDevice);
        UNREFERENCED_PARAMETER(Context);
        PIO_STACK_LOCATION stack;
        PMOUSE_INPUT_DATA myData;
        stack = IoGetCurrentIrpStackLocation(pIrp);
        if (NT_SUCCESS(pIrp->IoStatus.Status))
        {
                // 获取鼠标数据
                myData = pIrp->AssociatedIrp.SystemBuffer;
                // 这里即可开始读取游戏数据并更改鼠标的IRP
        }
        if (pIrp->PendingReturned)
        {
                IoMarkIrpPending(pIrp);
        }
        return pIrp->IoStatus.Status;
}
```
#### 检测手段
毕竟是在设备栈上添加自己的设备,那么只需要一个设备黑名单即可.  
通过遍历设备栈只要找到了外挂创建的设备即判定非法
判断的核心代码:
```c
// 设备对象的拓展结构
typedef struct _DEVICE_EXTENSION {
        PDEVICE_OBJECT pDevice;
        UNICODE_STRING ustrDeviceName;
        UNICODE_STRING ustrSymLinkName;
} DEVICE_EXTENSION, * PDEVICE_EXTENSION;

        // 检测代码
        targetDevice = mouseDriver->DeviceObject;
        // 遍历设备链中的所有设备
        while (targetDevice)
        {
    PDEVICE_EXTENSION exdev = (PDEVICE_EXTENSION)targetDevice;
    // 现在targetDevice->ustrDeviceName就是设备名了,下面即可自行判断这个设备名是否合法

                targetDevice = targetDevice->NextDevice;
        }
```

### 比较好的方式 - Hook或直接调用MouseServiceClassCallBack
MouseClassServiceCallback是Mouclass提供的类服务回调函数。 驱动程序在其ISR dispatch completion routine中调用类服务回调。 类服务回调将输入数据从设备的输入数据缓冲区传输到类数据队列。  
所以这种方式更底层,目前见过的所有模拟鼠标的驱动基本也都是用的这种方式  
关键在于找到这个函数,寻找这个函数可以遍历设备对象也可以搜特征码  
遍历设备对象的寻找目标函数的方式如下  
```c
// 全局定义MouseClassServiceCallback
typedef VOID
(*MouseClassServiceCallback) (
    PDEVICE_OBJECT  DeviceObject,
    PMOUSE_INPUT_DATA  InputDataStart,
    PMOUSE_INPUT_DATA  InputDataEnd,
    PULONG  InputDataConsumed
    );
// 保存原始函数
MouseClassServiceCallback orig_MouseClassServiceCallback = NULL;

// 我们已经获取过鼠标的设备驱动保存在了mouseDriver中,现在获取鼠标的端口驱动
UNICODE_STRING mouNtName;
PDRIVER_OBJECT mouhidDriverObj;
RtlInitUnicodeString(&mouNtName, L"\\Driver\\Mouhid");
status = ObReferenceObjectByName(
    &mouNtName,
    OBJ_CASE_INSENSITIVE,
    NULL,
    0,
    IoDriverObjectType,
    KernelMode,
    NULL,
    &mouhidDriverObj
    );
if (!NT_SUCCESS(status)) {
  return status;
}
// 遍历mouclass下所有设备
PDRIVER_OBJECT targetDriverObj = mouseDriver->DeviceObject;
ULONG mouDriverStart = (ULONG)GetModlueBaseAdress("mouclass.sys", 0);
ULONG mouDriverSize = 0x2000;
MouseClassServiceCallback* MouSrvAddr = NULL;
while(targetDriverObj)
{
  // 遍历我们先找到的端口驱动的设备扩展下的每个指针
  for(PBYTE exdev = (PBYTE)mouhidDriverObj; i<4096; ++i; exdev+=sizeof(PBYTE))
  {
    if (!MmIsAddressValid(exdev)) {
      break;
    }

    //如果在设备扩展中找到一个地址位于mouClass模块中，就认为这是我们要的回调函数地址
    PVOID tmp = *(PVOID*)exdev;

    if ((tmp > mouDriverStart)&&(tmp < (PBYTE)mouDriverStart+mouDriverSize)) {
      orig_MouseClassServiceCallback  = (MouseClassServiceCallback)tmp;
      MouSrvAddr = (PVOID*)exdev;
      goto Done;
    }
  }
  targetDriverObj = targetDriverObj->NextDevice;
}
Done:
// 这里就获取到了哪个函数的地址并保存到了orig_MouseClassServiceCallback中
```
现在我介绍那两种方式以及检测手段
##### Hook方式
###### 原理
直接改函数地址即可
> *MouSrvAddr = myHookFuncAddr;

这种方式比较方便的地方是不用自己构建MOUSE_INPUT_DATA
###### 检测手段
这种方式直接检测函数地址即可

##### 直接调用方式
###### 原理
我们刚刚获取了函数指针那么直接调用就能使鼠标移动,麻烦的是要自己构造MOUSE_INPUT_DATA
###### 检测手段
这种方式目前 __没有__ 很好的检测手段,__如果各位大佬有办法请务必让本菜见识下__
## 杂谈
当然也是有通过Hook IDT的鼠标中断来实现的,这种方式麻烦的地方在于要为CPU里每个核心都做一遍Hook操作.而且也能简单的通过特征码的方式检测出来  
最重要的一点其实还在于如果做IDT Hook那么还不如直接修改空闲中断的DPL和中断程序地址来做中断提权,让我们的外挂程序有Ring0权限.我感觉这样才是更好的办法
