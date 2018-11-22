# 使用jprobe获取sys_execve参数
## 测试环境：

```bash
[root@localhost test2]# lsb_release -a
LSB Version:	:core-4.1-amd64:core-4.1-noarch:cxx-4.1-amd64:cxx-4.1-noarch:desktop-4.1-amd64:desktop-4.1-noarch:languages-4.1-amd64:languages-4.1-noarch:printing-4.1-amd64:printing-4.1-noarch
Distributor ID:	CentOS
Description:	CentOS Linux release 7.5.1804 (Core) 
Release:	7.5.1804
Codename:	Core
[root@localhost test2]# uname -r
3.10.0-862.el7.x86_64

```

## 起因：
在上述测试环境下，编译yulong的驱动并加载发现会导致系统死机：

![](https://github.com/lovewinxp/jprobes_hook/blob/master/jpg/1.png)

怀疑是不是因为使用偏移方法获取系统调用表存在问题，在3.10.0-514版本下获取到的地址是一致的，如下图：

![](https://github.com/lovewinxp/jprobes_hook/blob/master/jpg/2.png)

而在3.10.0-862版本下获取到的地址是不一致的：

![](https://github.com/lovewinxp/jprobes_hook/blob/master/jpg/3.png)

接着把正确的地址硬编码到代码里来确认是否还是存在问题，结果使用硬编码的方法依旧会导致存在死机。后来想到hook其他函数是否也会存在问题？尝试hook sys_open发现可以hook成功，看来是sys_execve的问题，最后定位到应该是汇编代码里的堆栈平衡在这个版本下有点问题，但不熟汇编所以只能查查是否有其他版本解决。

## jprobe结构体：
查找资料后发现jprobe也可以hook，先看下jprobe的结构。
jprobe结构体定义如下（/usr/src/kernels/3.10.0-862.el7.x86_64/include/linux/kprobes.h）：
```c

/*
 * Special probe type that uses setjmp-longjmp type tricks to resume
 * execution at a specified entry with a matching prototype corresponding
 * to the probed function - a trick to enable arguments to become
 * accessible seamlessly by probe handling logic.
 * Note:
 * Because of the way compilers allocate stack space for local variables
 * etc upfront, regardless of sub-scopes within a function, this mirroring
 * principle currently works only for probes placed on function entry points.
 */
struct jprobe {
	struct kprobe kp;
	void *entry;	/* probe handling code to jump to */
};
```
包含了一个kprobe结构和一个entry指针，它保存的是探测点执行回调函数的地址，当触发调用被探测函数时，保存到该指针的地址会作为目标地址跳转执行（probe handling code to jump to），也就是需要执行的自定义函数。

##jprobe_example.c
自带的示例代码很简单，先实例化一个my_jprobe，使用自定义函数jdo_fork替换掉系统的do_fork，然后在初始化函数中，调用register_jprobe函数向kprobe子系统注册my_jprobe，这样结束了，比修改系统调用表hook简单很多。
```c
static struct jprobe my_jprobe = {
	.entry			= jdo_fork,
	.kp = {
		.symbol_name	= "do_fork",
	},
};

static int __init jprobe_init(void)
{
	int ret;

	ret = register_jprobe(&my_jprobe);
	if (ret < 0) {
		printk(KERN_INFO "register_jprobe failed, returned %d\n", ret);
		return -1;
	}
	printk(KERN_INFO "Planted jprobe at %p, handler addr %p\n",
	       my_jprobe.kp.addr, my_jprobe.entry);
	return 0;
}
```

对照着jprobe_example.c改了下代码，手动加载驱动进行测试：
```bash
insmod syshook_execve.ko
```
## 改造后运行效果：
对照着jprobe_example.c改了下代码测试成功。

![](https://github.com/lovewinxp/jprobes_hook/blob/master/jpg/4.png)

## 参考：
[https://my.oschina.net/macwe/blog/603583](https://my.oschina.net/macwe/blog/603583 "https://my.oschina.net/macwe/blog/603583")

[https://blog.csdn.net/ddk3001/article/details/51485135](https://blog.csdn.net/ddk3001/article/details/51485135 "https://blog.csdn.net/ddk3001/article/details/51485135")

[http://www.cnblogs.com/LittleHann/p/3854977.html?utm_source=tuicool&utm_medium=referral](http://www.cnblogs.com/LittleHann/p/3854977.html?utm_source=tuicool&utm_medium=referral "http://www.cnblogs.com/LittleHann/p/3854977.html?utm_source=tuicool&utm_medium=referral")
