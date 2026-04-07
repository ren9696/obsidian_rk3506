将您的RTU主应用程序放在 **/usr/sbin/**​ 目录下最为合适。

这是Linux系统中存放系统管理员使用的基本系统二进制文件（即可执行程序）的标准位置。您的RTU应用作为核心后台服务/守护进程，属于系统管理范畴，放在这里符合规范，也便于通过systemd等系统服务进行管理。

**后续步骤建议：**

1. 在您的Buildroot构建脚本中，将编译好的可执行文件安装到 `$(TARGET_DIR)/usr/sbin/`。
    
2. 创建一个systemd服务单元文件（如 `rtu-app.service`），放在 `/etc/systemd/system/`中，以便系统启动时自动运行您的应用。
    
3. 相关的配置文件（如您提到的JSON配置）建议放在 `/etc/`目录下（例如 `/etc/rtu-config.json`）。
---

好的，我们来详细解释这两点，这是将您的应用程序集成到嵌入式Linux系统并实现开机自启动的关键步骤。

### 第一点详解：在Buildroot中安装应用程序

这指的是在**构建系统镜像的阶段**，如何将您编译好的RTU应用程序自动放置到目标文件系统的 `/usr/sbin/`目录下。

**核心概念**：Buildroot 通过 `*.mk`文件（通常是 `your_app.mk`）来定义如何下载、编译和安装一个软件包。您需要为您的RTU应用程序创建这样一个包定义。

**具体步骤和解释**：

1. **创建应用程序包定义**：
    
    在Buildroot的 `package/`目录下，为您RTU应用创建一个子目录，例如 `package/rtu-app/`。在该目录下创建两个关键文件：
    
    - `Config.in`：用于在 `make menuconfig`时显示配置选项。
        
    - `rtu-app.mk`：核心的构建规则文件。
        
    
2. **编写 `rtu-app.mk`的安装规则**：
    
    在这个文件中，最关键的部分是定义 `RTU_APP_INSTALL_TARGET_CMDS`。这个变量下的命令会在Buildroot构建的**最后阶段——安装阶段**执行，将文件拷贝到目标文件系统镜像中。
    
    ```
    # rtu-app.mk 示例片段
    define RTU_APP_INSTALL_TARGET_CMDS
        # 1. 创建目标目录（通常已存在，但确保无误）
        $(INSTALL) -d -m 0755 $(TARGET_DIR)/usr/sbin
        # 2. 将主机编译好的可执行文件，安装到目标文件系统的指定位置
        $(INSTALL) -D -m 0755 $(@D)/rtu_app_binary $(TARGET_DIR)/usr/sbin/rtu-app
        # 3. （可选）安装配置文件到 /etc
        $(INSTALL) -D -m 0644 $(RTU_APP_PKGDIR)/config/rtu-default.json $(TARGET_DIR)/etc/rtu-config.json
    endef
    ```
    
    **关键变量解释**：
    
    - `$(TARGET_DIR)`：这是Buildroot内部变量，指向正在构建的**目标根文件系统**的根目录。所有操作都是针对这个虚拟目录进行的，最终它会被打包成 `rootfs.tar`或镜像文件。
        
    - `$(@D)`：指向您的应用程序源码解压和编译的构建目录。
        
    - `$(INSTALL)`：一个安全的安装命令工具，比直接使用 `cp`更规范。
        
    - `-m 0755`：设置文件权限，`0755`表示所有者可读、写、执行，组和其他用户可读、执行。
        
    

**这样做的好处**：每次使用Buildroot执行 `make`命令全量或重新构建您的应用包时，这个过程都会自动完成。您无需在设备启动后手动复制文件，实现了构建的自动化。

---


系统使用 **sysVinit**，那么服务管理方式与 systemd 完全不同。
核心是将应用配置为一个 **sysVinit 服务脚本**。

以下是针对您使用 sysVinit 系统的详细解释和步骤调整：

### 第一点：应用程序安装目录

**结论**：您的 RTU 主应用程序依然建议安装在 **`/usr/sbin/`**。

**原因**：这个目录存放的是系统管理级别的二进制文件，与使用哪种初始化系统（init）无关。它只是可执行文件的存放位置。

在您的 `rtu-app.mk`中，安装命令保持不变：

```
define RTU_APP_INSTALL_TARGET_CMDS
    $(INSTALL) -d -m 0755 $(TARGET_DIR)/usr/sbin
    $(INSTALL) -D -m 0755 $(@D)/rtu_app_binary $(TARGET_DIR)/usr/sbin/rtu-app
endef
```

---

### 第二点详解：创建 sysVinit 启动脚本（替代 systemd 服务）

sysVinit 通过运行 `/etc/init.d/`目录下的 Shell 脚本来管理服务，并通过 `/etc/rcN.d/`（N 为运行级别）目录中的符号链接来控制服务的启动（Start）与停止（Stop）顺序。

**具体步骤：**

1. **创建 init 脚本**
    
    在您的 Buildroot 应用包目录下，创建一个标准的 init 脚本，例如 `S99rtu-app`。脚本需要接受 `start`、`stop`、`restart`等参数。
    
    ```
    #!/bin/sh
    ### BEGIN INIT INFO
    # Provides:          rtu-app
    # Required-Start:    $network $local_fs $syslog
    # Required-Stop:     $network $local_fs $syslog
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: RTU Main Application Daemon
    ### END INIT INFO
    
    DAEMON=/usr/sbin/rtu-app
    NAME="rtu-app"
    PIDFILE=/var/run/$NAME.pid
    DESC="RTU Main Application"
    
    case "$1" in
      start)
        echo -n "Starting $DESC: "
        start-stop-daemon -S -b -m -p $PIDFILE -x $DAEMON
        echo "$NAME."
        ;;
      stop)
        echo -n "Stopping $DESC: "
        start-stop-daemon -K -p $PIDFILE
        echo "$NAME."
        ;;
      restart|force-reload)
        $0 stop
        sleep 1
        $0 start
        ;;
      *)
        echo "Usage: /etc/init.d/$NAME {start|stop|restart}" >&2
        exit 1
        ;;
    esac
    
    exit 0
    ```
    
    **脚本关键点解释**：
    
    - `### BEGIN INIT INFO`：这是 **LSB 头信息**，被 `update-rc.d`等工具用来解析依赖和设置默认运行级别。
        
    - `Required-Start`：声明本服务需要 `$network`（网络）、`$local_fs`（本地文件系统）等先启动。这定义了**依赖关系**。
        
    - `Default-Start: 2 3 4 5`：表示在运行级别 2、3、4、5 下默认启动此服务（这些是多用户模式）。
        
    - `start-stop-daemon`：一个用于管理守护进程的实用工具，可以启动进程、创建 PID 文件，并确保不会重复启动。
        
    - `PIDFILE`：用于记录守护进程的主进程 ID，便于可靠地停止服务。
        
    
2. **在 Buildroot 中安装 init 脚本并启用服务**
    
    修改您的 `rtu-app.mk`，安装脚本并创建启用服务的链接。
    
    ```
    define RTU_APP_INSTALL_TARGET_CMDS
        # 1. 安装应用程序二进制文件
        $(INSTALL) -d -m 0755 $(TARGET_DIR)/usr/sbin
        $(INSTALL) -D -m 0755 $(@D)/rtu_app_binary $(TARGET_DIR)/usr/sbin/rtu-app
    
        # 2. 安装 init 脚本到 /etc/init.d/
        $(INSTALL) -D -m 0755 $(RTU_APP_PKGDIR)/S99rtu-app $(TARGET_DIR)/etc/init.d/S99rtu-app
    
        # 3. （关键）使用 Buildroot 工具在构建时创建运行级别链接
        # 这行命令会在目标文件系统的 /etc/ 下创建 rc?.d/ 链接，指向刚刚安装的脚本。
        $(HOST_DIR)/sbin/update-rc.d -f S99rtu-app defaults
    endef
    ```
    
    **关键解释**：
    
    - `update-rc.d`是一个工具，它会根据脚本中的 LSB 头信息（特别是 `Default-Start`），在 `/etc/rc2.d/`、`/etc/rc3.d/`等目录下创建以 `S`（Start）和 `K`（Kill）开头的符号链接，并加上序号（如 `S99rtu-app`）来决定启动和停止的顺序。
        
    - `defaults`参数表示使用脚本中定义的默认运行级别。
        
    - 这样，当系统以多用户级别启动时，就会自动执行 `/etc/rcN.d/S99rtu-app start`，从而启动您的服务。
        
    
3. **在设备上的操作（构建时已自动化，无需手动）**
    
    理论上，通过以上 Buildroot 配置，服务已经集成到镜像中。设备启动后，您仍然可以手动管理服务：
    
    - **启动服务**：`/etc/init.d/S99rtu-app start`
        
    - **停止服务**：`/etc/init.d/S99rtu-app stop`
        
    - **重启服务**：`/etc/init.d/S99rtu-app restart`
        
    - **查看状态**：通常通过检查进程 `ps aux | grep rtu-app`或 PID 文件 `cat /var/run/rtu-app.pid`来判断。
        
    

### 总结对比与您的操作清单

|任务|使用 systemd 时|使用 **sysVinit**​ 时（您当前情况）|
|---|---|---|
|**应用安装位置**​|`/usr/sbin/rtu-app`|**`/usr/sbin/rtu-app`（不变）**​|
|**服务定义文件**​|`rtu-app.service`单元文件|**`S99rtu-app`init 脚本（Shell脚本）**​|
|**服务文件安装位置**​|`/usr/lib/systemd/system/`|**`/etc/init.d/`**​|
|**启用开机自启**​|`systemctl enable rtu-app`|**在 Buildroot 中用 `update-rc.d ... defaults`**​|
|**构建时集成**​|安装 `.service`文件|**安装 init 脚本并运行 `update-rc.d`**​|

**您的下一步操作：**

1. 在 `package/rtu-app/`目录下创建符合 LSB 标准的 init 脚本 `S99rtu-app`。
    
2. 更新 `rtu-app.mk`文件，添加安装该脚本和调用 `update-rc.d`的命令。
    
3. 在 Buildroot 配置中确保您的 `rtu-app`包被选中，然后重新编译系统镜像。
    
4. 将新镜像烧录到设备，您的 RTU 应用就会在系统启动时自动运行了。