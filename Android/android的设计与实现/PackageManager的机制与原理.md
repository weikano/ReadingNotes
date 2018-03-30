### PM的功能主要包含以下部分
> 1. 权限处理：包括对系统和应用定义的Permission和Permission Group的增删改查
> 2. 包处xu理：包括扫描并安装和卸载apk，查询包的UID、GID、包名、系统默认程序信息等
> 3. 比较两个包的signature信息
> 4. 查询Activity、Provider、Service、Receiver信息
> 5. 查询Application、Package、Resource、SharedLibrary、Feature信息
> 6. Intent匹配

### 主要代码位于
> android.content.pm.PackageManager
>
> android.server.pm.PackageManagerService
>
> android.server.pm.Installer
>
> android.server.pm.Settings
>
> android.content.pm.PackageParser
>
> android.os.FileObserver
>
> android.app.ApplicationPackageManager
>
> androd.defcontainer.DefaultContainerService
>
> frameworks/base/cmds/installd/install.c
>
> frameworks/base/cmds/installd/commands.c
>
> frameworks/base/cmds/installd/utils.c

### PackageManager的启动过程
> SystemServer.startBootstrapServices会创建PackageManagerService对象并调用main方法
>
> main方法中会调用构造方法，构造方法中会对Settings进行初始化并添加某些信息
>
> installPackageAsUser和scanPackageLPw是重要的方法
>
>INstaller仅仅是创建data/data/packagename/目录