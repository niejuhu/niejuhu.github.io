Android Init Process
====================

Android boot.img中的内核完成初始化之后，启动init进程。init进程完成以下工作：

1. 解析init.rc脚本。此脚本配置了Android启动中要执行的任务
2. 根据init.rc配置，完成初始化工作，包括：
    * 在启动和运行的各个阶段，根据条件执行对应的action（动作）
    * 启动系统服务，监控并控制系统服务的执行
    * 提供系统property服务

init.rc
-------

init.rc使用Android init language编写，语法规则参考
android-src/system/core/init/readme.txt。

init进程解析init.rc脚本，生成如下数据结构。之后根据条件执行这些动作和服务。

                            action                action
    +-----------+         +----------+         +----------+
    |action_list| ------> | name     | ------> | name     | -----> ...
    +-----------+         | cmds     |         | cmds     |
                          | ...      |         | ...      |
                          +----------+         +----------+

                            service              service
    +------------+        +----------+         +----------+
    |service_list| -----> | name     | ------> | name     | -----> ...
    +------------+        | opts     |         | opts     |
                          | ...      |         | ...      |
                          +----------+         +----------+

执行初始化动作
-----------

解析完init.rc文件之后，init程序将要执行的action按照一定顺序放入`action_queue`链表中：

    // 将name为early-init的action放入链表
    action_for_each_trigger("early-init", action_add_queue_tail);
    ...
    // 将name为init的action放入链表
    action_for_each_trigger("init", action_add_queue_tail);
    ...

除了init.rc中定义的action，init程序还支持代码中内建action:

    queue_builtin_action(property_service_init_action, "property_service_init");
    queue_builtin_action(signal_init_action, "signal_init");

数据结构如图：

                            action                action
    +------------+        +----------+         +----------+
    |action_queue| -----> | name     | ------> | name     | -----> ...
    +------------+        | cmds     |         | cmds     |
                          | ...      |         | ...      |
                          +----------+         +----------+

最后init程序遍历链表，依次执行其中的action：

    for (;;) {
        // 每次执行一个action的一个command
        execute_one_command();
        // 每执行一个command都可能导致property改变、引发trigger、service退出等事件，
        // 后面的代码处理这些情况
        ...
    }

对于每个action，init程序依次执行其中的命令（cmds）。有些命令会将新的action放入
`action_queue`末尾并最终被执行。

系统服务
------

在action执行过程中，通过`start xxx`或者`class_start xxx`启动某个或者某一类service。
例如启动console和adbd服务：

    on property:ro.debuggable=1
        start console
    on property:ro.kernel.qemu=1
        start adbd

例如启动所有core类型的服务：

    on boot
        ...
        class_start core


signal_init_action内建动作定义了一个接受SIG_CHLD信号的处理程序，并创建一个socket。
当某个service退出之后，init会受到SIG_CHLD信号，SIG_CHLD信号的处理程序将此事件写入socket，
init循环读取socket，获取service退出事件，根据service属性作对应的处理（重启服务等）。

Android支持通过组合键启动service。init进程通过读取`/dev/keychord`文件监听组合键，
启动对应的service。

property服务
-----------

init进程的另一个功能是初始化property服务，并提供property的写入功能。同时监控property的变化，
根据property变化，执行特定的action。
