---
title: init process
category: Android
---

Android boot.img中的内核完成初始化之后，启动init进程。init进程完成以下工作：

* 自身Log初始化
* 创建必需的目录结构, 挂载文件系统
* SELinux初始化
* Property系统初始化
* 解析init.rc脚本，执行相关任务

# Log初始化

Init使用与内核printk相同的缓冲区记录Log。通过`cat /proc/kmsg`或者`dmesg`命令可以查看到init输出的log。

Init定义三个级别的Log：

* ERROR
* NOTICE
* INFO

默认log级别为NOTICE，只有高于或等于默认级别的log才会输出。需要输出的log通过kmsg驱动输出到内核缓冲
区。

## 实现

Init使用libcutils实现log功能。

Init log初始化：

		// system/core/init/init.cpp
		int main()
		{
			...
			klog_init();
			klog_set_level(KLOG_NOTICE_LEVEL);
			...
		}

Init log定义：

		// system/core/init/log.h
		#define ERROR(x...)   init_klog_write(KLOG_ERROR_LEVEL, x)
		#define NOTICE(x...)  init_klog_write(KLOG_NOTICE_LEVEL, x)
		#define INFO(x...)    init_klog_write(KLOG_INFO_LEVEL, x)

klog相关函数定义在libctuls中：`system/core/libcutils/klog.c`

# 创建必需的目录结构，挂载文件系统

几个基础的文件系统需要先挂载(解析init.rc等需要使用)，主要是内核特殊文件系统：

		// system/core/init/init.cpp
		int main()
		{
			...
			mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
			mkdir("/dev/pts", 0755);
			mkdir("/dev/socket", 0755);
			mount("devpts", "/dev/pts", "devpts", 0, NULL);
			mount("proc", "/proc", "proc", 0, NULL);
			mount("sysfs", "/sys", "sysfs", 0, NULL);
			...
		}

其他文件系统通过解析并运行init.rc中的命令挂载，包括system、data等分区。

# SELinux初始化

Init是内核启动的第一个用户态进程，继承了内核的SELinux domain：kernel，此时SELinux策略尚未加载。

Init首先按照前文中设置自己的log的方法，设置SELinux的日志输出到内核缓冲区，后续可以通过`dmesg`等方法读取SELinux用户态日志。

检查SELinux的工作模式。Bootloader启动内核时通过`androidboot.selinux=[disabled,permissive,enforcing]`参数指定SELinux的工作模式。内核参数保存在/proc/cmdline文件中，Init解析此文件，获取工作模式，然后通过写入/sys/fs/selinux/enforce对应值来设置SELinux工作模式。

从/sepolicy文件加载SELinux策略。加载之后，SELinux就可以正常工作了。

由于根文件系统为ramfs，没有持久化保存文件的SELinux context，所以先通过`restorecon`设置/init文件的context，其中type为`init_exec`，然后重新执行自己。由于已经加载了策略，并且init有了正确的type，所以按照策略规定的转移规则（`domain_auto_trans(kernel, init_exec, init)`），新的init进程的SELinux domain成为init。

重新设置日志；加载策略文件`/file_contexts`和`property_contexts`，前者用于为文件设置type，后者用于为property属性设置type。实际上如果此时不加载这两个策略文件，后面使用的时候也会自动加载。

`/file_contexts`定义了/dev /sys等文件系统中文件的context，既然已经加载了此文件，此时调用restorecon来设置这些文件系统中文件的context。

## Implementation

		// system/core/init/init.cpp
		void selinux_initialize(bool in_kernel_domain)
		{
			// 设置日志
			selinux_callback cb;
			cb.func_log = selinux_klog_callback;

			if (in_kernel_domain) {
				// 加载策略/sepolicy
				selinux_android_load_policy();
				// 设置selinux工作模式
				security_setenforce(selinux_is_enforcing());
			} else {
				// 加载/file_contexts /property_contexts
				selinux_init_all_handles()
			}
		}
		int main()
		{
			...
			// 由kernel启动的init,is_first_stage=true
			// 由init启动的init, is_first_stage=false
			selinux_initialize(is_first_stage);

			// 设置/init context，重新执行init
			if (is_first_stage) {
				restorecon("/init");
				exec("/init", args);
			}
			...
			// 设置file context
			restorecon("/dev");
			...
		}

# Property系统初始化

## Property系统

Android property系统以字符串键值对的形式记录系统配置、运行状态等信息。例如设置中关于手机页面的版本号、adbd等服务是否运行等信息。

Android在运行时使用共享内存来保存所有的property，并通过/dev/__properties__文件节点来映射到每一个进程。其中init进程以读写方式映射，而其他进程以只读方式映射，所以只有init进程能直接写property，其他进程需要写时，通过socket请求init执行写操作。

所有的property有以下几个来源：

* 系统启动时从内核参数中获取

	* 内核命令行：/proc/cmdline中androidboot.XXX=YYY设置为属性ro.boot.XXX=YYY
	* dt参数：/proc/device-tree/firmware/android/ 目录下的每个文件XXX设置为ro.boot.XXX=<文件内容>
	* 部分ro.boot.XXX=YYY属性进一步设置为ro.XXX=YYY

* 系统启动时由init从文件读取，并写入共享内存。这些文件包括：

	* /default.prop
	* /system/build.prop
	* /vender/build.prop
	* /factory/factory.prop
	* /data/local.prop  // 仅在eng,userdebug版本且ro.debuggable=1时加载
	* /data/property/*  // 文件名为键，文件内容为值，这些都是应用程序运行时设置的属性

* 应用程序在运行时设置的属性。需要持久化的属性会保存在/data/property/目录下，系统下次启动时由init加载；其他临时属性只在内存中，重启后失效。

根据名称，有些property在写时被特殊处理：

* "ctl."开头的属性为init的控制属性，控制init服务的运行状态。对这些属性的写操作会被init拦截处理掉，不会真的写入共享内存，所以读这些属性的结果总是空。
* "selinux.reload_policy"和"selinux.restorecon_recursive"是SELinux的控制属性，同样会被init拦截处理掉
* "ro."开头的property为只读，不能修改
* "net."开头的property为DNS属性，写net.xxx=yyy会同时触发写net.change=xxx
* "persist."开头的property为需要持久化，写persist.xxx=yyy会同时写入文件（等同于 `echo yyy > /data/property/xxx`）

由于init负责所有property的写入，所以自然可以监听所有property的变化。init会根据特定property的变化做相应的任务，参考后面init.rc相关章节。

## 接口

1. libc接口，Android系统源码以及NDK程序调用

		#include <sys/system_properties.h>

		int __system_property_get(const char *name, char *value);
		int __system_property_set(const char *key, const char *value);

2. libcutils接口，Android系统源码程序调用

		#include <cutils/properties.h>

		int property_get(const char *key, char *value, const char *default_value);
		int property_set(const char *key, const char *value);

3. Java接口，Android系统源码调用

		import android.os.SystemProperties;

		public static String get(String key);
		public static void set(String key, String val);

## 实现

1. Init程序初始化property，包括：1）以读写方式创建、映射property共享内存；2）从内核参数、文件导入部分系统预置的property；3）启动property写服务；4）导入其他系统预置属性和应用设置的属性

	![](/images/init_property_init.png)

2. 其他程序初始化property

		// bionic/libc/bionic/libc_init_dynamic.cpp
		// 对于动态链接程序，__libc_preinit被编译到libc.so的.init_array section. Android linker
		// 加载libc.so的时候会首先执行这里的代码。
		_attribute__((constructor)) static void __libc_preinit()
		|-- __libc_init_common()
			|-- __system_properties_init()
				|-- // 只读映射/dev/__properties__共享内存
					int fd = open(property_filename, O_CLOEXEC | O_NOFOLLOW | O_RDONLY);
					// 兼容早期版本通过环境变量传递fd
					if (fd < 0) fd = get_fd_from_env();

					// 只读映射
					void* const map_result = mmap(NULL, pa_size, PROT_READ, MAP_SHARED, fd, 0);

# 解析init.rc并执行其中的命令和服务

## 解析init.rc

init.rc由Android Init Language编写，关于Android Init Language，参考Android源码：system/core/init/readme.txt。

init.rc解析得到三个链表：

* `service_list` 所有init.rc中service的链表
* `action_list` 所有init.rc中action的链表
* `action_queue` 所有等待被执行的action的链表，即被触发的action列表

数据结构大致如下：

![](/images/initrc.png)

除了在init.rc中声明action，init还可以在程序中通过`queue_builtin_action(cmd_func, name)`声明内建action。内建action包含一个trigger和一个command，name为trigger名称，cmd_func为command命令。内建action自动被添加到`action_list`和`action_queue`。

## 执行action，调度service

解析完init.rc之后，init将一些需要立即执行的action以及内置action放入`action_queue`，于是`action_queue`中就有了被触发的、等待执行的action。

		// system/core/init/init.cpp
		main()
		|-- init_parse_config_file("/init.rc");
		|-- // 执行early-init action
		|-- action_for_each_trigger("early-init", action_add_queue_tail);
		|-- // 内建action。基于当前的property执行property trigger的action
		|-- queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");

init从`action_queue`中依次执行每个action中的每个command，然后检查`service_list`中是否有service状态被更新，根据service状态进行调度（启动、停止、重启等）。

每个command执行过程中可能会触发新的trigger，新触发的trigger对应的action被放入`action_queue`中，对应的service状态被更新。

init进程是所有service进程和系统中所有孤儿进程的父进程。init在开始执行action之前设置捕捉SIGCHLD信号。当捕捉到SIGCHLD信号时，通过socket向自身发送一个消息。init收到自身发送的消息之后，调用wait函数获取退出的子进程的信息。如果退出的子进程为普通孤儿进程，则不需要做额外的工作。如果是service进程，则根据service的状态信息进行调度：重启、停止或者复位等。

		// system/core/init/init.cpp
		main()
		|-- while (true) {
		|--	  execute_one_command(); // 执行action_queue中下一个action的下一个command
		|--	  restart_processes(); // 调度service
		|--	  // 如果action_queue中还有command需要执行，或者有service等待调度，则epoll_wait很快返回，否则等待新的事件：SIGCHLD事件、property写事件等
		|--	  int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
		|-- }
