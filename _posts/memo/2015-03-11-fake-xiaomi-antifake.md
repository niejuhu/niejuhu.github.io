山寨小米手机的PackageInstaller程序被修改：当用户安装小米鉴定、安兔兔等鉴定、评分应用时，系统直接使用山寨版的程序替换要安装的程序，山寨版的程序想用户显示假的结果。下面是看到过得一款山寨小米中安装的山寨版小米鉴定应用。

# 伪加密

山寨版小米鉴定经过了伪加密，即修改zip文件中的加密标志，并不执行真正的加密。可通过通过[unpack.py](https://www.github.com/niejuhu/mytools)工具解压，重新打包即可得到原始apk。

# 判断设备

山寨版小米鉴定启动时会检测运行环境，如果当前手机是山寨小米手机，就执行检测，并显示结果为小米手机；否则抛出异常并退出。

    .class public Lcom/xiaomi/antifake/WelcomeActivity;

    # virtual methods
    .method protected onCreate(Landroid/os/Bundle;)V
        .locals 1
        .param p1, "savedInstanceState"    # Landroid/os/Bundle;

        .prologue
        .line 31
        invoke-direct {p0}, Lcom/xiaomi/antifake/WelcomeActivity;->checkLauncher()Z

        move-result v0

        if-eqz v0, :cond_0

        .line 33
        invoke-super {p0, p1}, Lcom/xiaomi/antifake/BaseActivity;->onCreate(Landroid/os/Bundle;)V

        .line 34
        const v0, 0x7f030007

        invoke-virtual {p0, v0}, Lcom/xiaomi/antifake/WelcomeActivity;->setContentView(I)V

        .line 36
        :cond_0
        return-void
    .end method

    .method private checkLauncher()Z
    .locals 4

    .prologue
    .line 121
    new-instance v0, Ljava/io/File;

    const-string v2, "/system/bin/chkcon"

    invoke-direct {v0, v2}, Ljava/io/File;-><init>(Ljava/lang/String;)V

    .line 122
    .local v0, "file":Ljava/io/File;
    const-string v2, "ro.build.version.creator"

    invoke-direct {p0, v2}, Lcom/xiaomi/antifake/WelcomeActivity;->getSystemProp(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v1

    .line 123
    .local v1, "version":Ljava/lang/String;
    if-eqz v1, :cond_0

    invoke-virtual {v1}, Ljava/lang/String;->length()I

    move-result v2

    const/4 v3, 0x5

    if-le v2, v3, :cond_0

    invoke-virtual {v0}, Ljava/io/File;->exists()Z

    move-result v2

    if-eqz v2, :cond_0

    const/4 v2, 0x1

    :goto_0
    return v2

    :cond_0
    const/4 v2, 0x0

    goto :goto_0
.end method

# 修改显示结果

山寨小米鉴定的结果永远显示是小米手机

# 样本

AntiFake_XYX.apk
