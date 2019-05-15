# Trojan-relevant-content 

木马相关功能技术 

## 11.1  进程遍历  

创建进程快照 

```
CreateToolhelp32Snapshot 建立快照  

Process32First  检索系统快照里面遇到的第一个进程信息 

Process32Next   检索快照记录中的下一个信息 

类比此过程  

遍历线程  Thread32First  

遍历进程模块快照  Module32First  

```


## 11.2  文件遍历 

Win32 API函数   
需要指明路径  

使用到了WIN32_FIND_DATA结构体  
结构体中 dwFileAttributes 判断搜索到的文件属性  是目录还是文件 

需要对 "." ".." 进行过滤  


```
FindFirstFile 搜索与名称匹配的文件或子目录 

FindNextFile 继续搜索文件

```

注意:  

1. 保存路径的缓冲区足够大 
2. 搜索.  / .. 过滤掉 避免死循环 

## 11.3  桌面截屏

用户层 GDI  Graphics Device Interface 屏幕画面抓取  

GDI 目前只支持BMP图片类型   

### 1. 绘制桌面位图 

设备上下文 DC 

GetDesktopWindow 获取桌面窗口的句柄  

GetDC 获取窗口设备上下文句柄  

CreateCompatibleDC 创建内存设备上下文 绘制桌面位图 

GetSystemMetrics函数获取屏幕的宽和高 

CreateCompatibleBitmap 创建兼容位图句柄  

SelectObject 设置内存上下文  

BitBlt 绘制

### 2. 绘制鼠标光标

GetCursorInfo  获取光标信息  

GetIconInfo  获取光标图标信息  

对图标位掩码hbmMask用SRCAND的方式来绘制 

对图标颜色位图 hbmColor以SRCPAINT的方式来绘制 


### 3. 保存位图 

引入头文件  atlimage.h  

Attach 将位图句柄hBitmap附加到CImage对象上 

Save将位图存储为图片格式 
```

BOOL SaveBmp(HBITMAP hBmp)
{
	CImage image;

	// 附加位图句柄
	image.Attach(hBmp);

	// 保存成jpg格式图片
	image.Save("mybmp1.jpg");

	return TRUE;
}

```

## 11.4  按键记录 

按键记录的方式

1. 全局键盘钩子 
2. GetAsyncKeyState 判断按键的状态 
3. 原始输入模型  直接从输入设备上获取信息 


利用原始输入模型来实现键盘记录 

流程
```
首先使用 RegisterRawInputDevices 注册原始输入设备  

获取元素输入数据 
GetRawInputData 获取数据 

保存按键信息  
    虚拟键盘码映射
```

### 思考

调用RegisterRawInputDevices函数 HOOK API可以实现对注册输入设备的监控  

## 11.5  远程CMD

通过创建命名管道 能够获取CMD的执行结果  

用到的函数 CreatePipe 创建匿名管道  

管道实质是共享内存   一个进程读  一个进程写 

分类:
1. 匿名管道

    父子进程间通信  
    单向 一端读 另一端写
    
2. 命名管道

    任意进程间通信  双向 
    
    同一时间一端读 一端写 

流程

```
初始化匿名管道安全属性结构体 SA

CreatePipe 创建匿名管道 

CreateProcess创建新的进程 执行CMD命令 

WaitForSingleObject 等待命令执行完毕 

ReadFile 读取数据 

释放资源

```


## 11.6 U盘监控 

U盘或其他设备插入 产生 

WM_DEVICECHANGE 消息  

监控 设备已插入操作 和 设备已移除操作  




## 11.7 文件监控 

ReadDirectoryChangesW 实现对计算机上所有文件的监控 

流程

```
打开目录  获取文件句柄 

设置目录监控  

判断文件操作类型  

```

注意文件目录字符串 最后必须加上反斜杠  "C:\\Windows\\"


## 11.8 自删除操作 

删除自身的操作  

### 1 MoveFileEx 重启删除 

MOVEFILE_DELAY_UNTIL_REBOOT 标志只能由拥有管理员权限的程序或本地系统权限的程序使用   

删除文件的路径开头加上 \\?\的前缀   以管理员身份运行 
```
BOOL RebootDelete(char *pszFileName)
{
	// 重启删除文件
	char szTemp[MAX_PATH] = "\\\\?\\";
	::lstrcat(szTemp, pszFileName);
	BOOL bRet = ::MoveFileEx(szTemp, NULL, MOVEFILE_DELAY_UNTIL_REBOOT);
	return bRet;
}


```

### 2  批处理命令删除  

利用批处理的特殊命令 


del  %0  执行完后会删除自身 而且不放入回收站  

流程
1. 构造自删除批处理文件  choice或ping指令延迟一定的时间   然后执行自删除命令 
2. 程序中创建新进程调用批处理文件   进程创建后立即退出整个程序 结束进程

注意: 

对于低于Windows 2003 VISTA版本的系统 例如XP 不支持choice命令  可以使用ping等方式 
