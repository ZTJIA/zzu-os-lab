# 环境配置
作者采用 Ubuntu 18.04 进行操作。其他版本理论上也可以，只是作者在尝试更新版本时会遇到一些莫名其妙的错误。

Linux 0.11 环境部署按照下列链接：[哈尔滨工业大学《操作系统》课程实验指导手册、实验环境（64位支持）及源码](https://github.com/DeathKing/hit-oslab?tab=readme-ov-file)

出现如下界面说明部署成功：

<img src="https://github.com/ZTJIA/zzu-os-lab/blob/main/success.png" width="400">

如果出现黑屏，则说明进入的是调试模式。这时在系统终端中输入 `c` 并按下回车键，就可以正常显示。

<img src="https://github.com/ZTJIA/zzu-os-lab/blob/main/success.png" width="400">

# 操作步骤

## 1. 使用下列命令进行系统挂载：
```bash
cd ~/oslab
./mount-hdc
```

## 2. 添加系统调用程序
在`~/oslab/hdc/usr/root`下建立`whoami.c`，`iam.c`两个文件，并实现功能。一个简单的示例如下：
`iam.c`
```bash
#define __LIBRARY__
#include <errno.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

_syscall1(int,iam,char*,name);

int main(int argc,char* argv[]){
	if(argc<=1){
		printf("No input!\n");
		return -1;	
	}
	else{
		if(iam(argv[1])<0){
			printf("sys_call wrong!\n");
			return -1;	
		}
	}
	return 0;
}
```
`whoami.c`
```bash
#define __LIBRARY__
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <errno.h>

_syscall2(int, whoami, char*,name,unsigned int,size);

int main(){
	int num;
	char a[128] = {0};

	num = whoami(a,80);
	if(num >= 0){
		printf("%s\n",a);
	}
	else{
		printf("sys_call exception");
		return -1;
	}
	return 0;
}
```

## 3 添加系统调用号
按照下列路径找到`unistd.h`：
`~/oslab/hdc/usr/include/unistd.h`
打开该文件。找到形如`#define __NR_setregid  71`，在其后添加下列语句：
```bash
#define __NR_whoami   72
#define __NR_iam      73
```
修改后保存。

## 4 取消挂载
```bash
cd ~/oslab
sudo umount hdc
```

## 5 添加函数声明
在`linux-0.11/include/linux`下找到`sys.h`文件，打开进行编辑：
添加下列语句：
```bash
extern int sys_whoami();
extern int sys_iam();
```

并在函数表`fn_ptr sys_call_table[]`的末尾中增加两个函数引用：
```bash
sys_whoami, sys_iam
```
保存后关闭文件

## 6 实现内核函数
在`~/oslab/linux-0.11/kernel`下新建文件，命名为`who.c`，内容如下：
```bash
//who.c
#define __LIBRARY__
#include <asm/segment.h>
#include <unistd.h>
#include <errno.h>

char a[80] = {0};
int sys_iam(const char* name){
	int i = 0;
	while( (get_fs_byte(name + i)) != `\0`){
		i++;
	}
	if( i >= 24){
		return -EINVAL;
	}
	printk("len(a):%d\n",i);
	i = 0;
	while( (get_fs_byte(name + i)) != `\0`){
		a[i] = get_fs_byte(name + i);
		i++;
	}
	return i;
}

int sys_whoami(char* name, unsigned int size){
	int i = 0;
	while( (a[i]) != `\0`){
		i++;
	}
	if( i > size){
		return -1;
	}
	i = 0;
	while( (a[i]) != `\0`){
		put_fs_byte(a[i],name + i);		
		i++;
	}
	return i;
}
```

## 7 修改`Makefile`文件，使刚才添加的`who.c`可以和其它`Linux`代码编译链接到一起。修改内容如下：
我们要修改的是`~/oslab/linux-0.11/kernel/Makefile`。需要修改两处：
第一处:
```bash
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o
```
改为：
```bash
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o who.o
```
添加了 who.o。

第二处:
```bash
### Dependencies:
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
```
改为：
```bash
### Dependencies:
who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
```
添加了who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h。
修改完成后，在kernel目录下打开终端，使用 make 编译即可。
## 8 回到oslab目录，使用`./run`打开Bochs，在Bochs中编译`iam.c、whoami.c`：
```bash
gcc -o iam iam.c
gcc -o whoami whoami.c
```

## 9 测试结果：
在Bochs中依次输入：
```bash
iam XXX
whoami
```
如果正常显示XXX，则说明实验成功。
