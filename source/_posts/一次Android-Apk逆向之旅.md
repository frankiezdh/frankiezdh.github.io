---
title: 一次Android Apk逆向之旅
date: 2018-05-17 15:57:19
tags: ["逆向", "Android"]
---

# 缘起
一直在找微信自动应答相关的Xposed插件，但是没找到很趁手的。最近闲逛酷安时，发现一款自动应答的Xposed插件。试用了一下，功能还是比较牛的。于是想看看是怎么实现的。

<!-- more -->

# Round 1
从酷安上下载apk，然后用[jadx](https://github.com/skylot/jadx)反编译。反编译完，Ctrl-Shift-F查找“findAndHook”，但只发现一处。看了一下代码，显然和微信消息的自动回复没有什么直接关系。

![1.png](https://i.loli.net/2018/05/17/5afd38631ffec.png)

# Round 2
既然找不到，肯定是隐藏起来了。首先怀疑是加壳了。网上找了一圈，发现和市面上“爱加密”等加壳软件加壳后的特征对不上。看来不是常规加壳，还得继续看代码。果不其然，再看代码发现有个DexClassLoader，应该就是通过它来加载真是的dex。

```java
 private void loadDexClass(ClassLoader classLoader) {
        IOException iOException;
        DexClassLoader dexClassLoader;
        Constructor constructor;
        NameNotFoundException nameNotFoundException;
        Exception exception;
        Context context = null;
        d.a("fx----------fx1");
        File dir = this.appContext.getDir("cache", 0);
        String str = dir.getAbsolutePath() + File.separator + "classes4.jar";
        File file = new File(str);
        try {
            Context createPackageContext = this.context.createPackageContext("cn.fxnn.wcautoreply", 2);
            try {
                d.a("fx----------fx2");
                if (file.exists()) {
                    file.delete(); // 删除
                    b.a(createPackageContext, this.appContext, "bb", "cc", "nn", file.getAbsolutePath());
                } else {
                    file.createNewFile();
                    b.a(createPackageContext, this.appContext, "bb", "cc", "nn", file.getAbsolutePath());
                }
                context = createPackageContext;
        ……

            d.a("fx----------fx3");
            dexClassLoader = new DexClassLoader(str, dir.getAbsolutePath(), null, classLoader);
            file.delete(); // 删除解密后的文件
            d.a("fx----------fx4");
            // ！！没错就是这里！！
            this.libClazz = dexClassLoader.loadClass(context.getPackageName() + ".hook.WechatHook");
            constructor = this.libClazz.getConstructor(new Class[]{String.class, String.class});
            d.a("fx----------fx5");
            this.hook = (WCHinterface) constructor.newInstance(new Object[]{this.pName, this.vName});
            this.hook.test();

```

# Round 3

找到了DexClassLoader，文件就找到了。从代码来看，真正实现业务的jar包名字是classes4.jar。但这个文件是通过asserts下的aa、bb和nn加密的，并不能静态直接获取。而且程序在解密后把解密后的文件给删除了。只能使用smali修改的方式来去除file.delete()了。先用apktool解压zip包，在找到HookerHelper.smali，查找“delete”关键字，一共两处（164和263行）都删除。再通过apktool重新打包并通过apksigner.jar重新签名得到Apk。将Apk安装，激活模块启动手机，可以拿到classes4.jar（在/data/data/com.tecent.mm/app_cache目录下）。
```smali
    invoke-virtual {v8}, Ljava/io/File;->delete()Z
```

# Round 4
拿到classes4.jar后，再用jadx反编译。反编译完成后，Ctrl-Shift-F查找“findAndHook”，终于发现真正的业务hook。
![2.png](https://i.loli.net/2018/05/17/5afd3d80dfa8c.png)


## 最后
解开真正的classes4.jar包时，发现代码中有很多日志，这些日志对于理解代码是如何做到自动回复有帮助，于是想能够获取这部分日志。但是日志开关默认都是关的，还是用smali修改。

```smali
#hook\utils\b.smali

# direct methods
.method public static a(Ljava/lang/String;Ljava/lang/String;)Z
    .locals 6

    const/4 v0, 0x0 # 这里改成const/4 v0, 0x1

    if-eqz p0, :cond_0

    const-string v1, ""

    invoke-virtual {v1, p0}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v1
```

同时那个鸡肋的加密算法，也会阻止classes4.jar的使用，修改之。
![3.png](https://i.loli.net/2018/05/17/5afd3f2486e3e.png)


修改完成后，修改后的classes4.jar改名成nn，并覆盖asserts下的nn。再重新打包、签名和安装运行，就可以看整个软件运行过程。