# Buildroot 自动构建（工业标准，推荐正式开发用）

如果你用 **Buildroot** 构建系统（嵌入式 90% 都用它），可以让工具**自动编译 APP + 自动拷贝脚本 + 自动打包镜像**，一劳永逸。

## 步骤 1：在 Buildroot 中创建自定义软件包

plaintext

```
buildroot/package/
└── myapp/          # 新建文件夹
    ├── Config.in    # 配置菜单
    ├── myapp.mk    # 编译/拷贝规则
    └── myapp_daemon # 你的守护脚本
```

## 步骤 2：写 Config.in

config

```
config BR2_PACKAGE_MYAPP
    bool "myapp"
    help
      My embedded application
```

## 步骤 3：写 myapp.mk（核心！自动拷贝文件）

makefile

```
MYAPP_VERSION = 1.0
MYAPP_SITE = /path/to/your/app/source # 你的APP源码路径
MYAPP_SITE_METHOD = local

# 编译APP（交叉编译）
define MYAPP_BUILD_CMDS
    $(TARGET_CC) -o $(@D)/myapp $(@D)/main.c
endef

# 拷贝APP到镜像/usr/bin/
define MYAPP_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/myapp $(TARGET_DIR)/usr/bin/myapp
    $(INSTALL) -D -m 0755 $(BR2_EXTERNAL_MYAPP_PATH)/myapp_daemon $(TARGET_DIR)/etc/init.d/myapp_daemon
    # 创建开机自启软链接
    ln -sf ../init.d/myapp_daemon $(TARGET_DIR)/etc/rcS.d/S99myapp
endef

$(eval $(generic-package))
```

## 步骤 4：make menuconfig 开启你的包

bash

运行

```
make menuconfig
# 找到 -> User-provided apps -> [*] myapp
```

## 步骤 5：一键编译全镜像

bash

运行

```
make
```

编译完成后，直接生成**包含你的 APP + 脚本**的完整镜像！

---

# 最关键的 3 个坑（新手必看）

1. **APP 架构错误**
    
    必须用 **交叉编译器**（arm-linux-gcc）编译，不能用电脑 gcc！
    
2. **文件没有执行权限**
    
    必须加 `chmod +x`，否则镜像里跑不起来。
    
3. **软链接没创建**
    
    没有 `/etc/rcS.d/S99myapp`，开机不会自启。
    

---

# 最终效果

你烧录镜像到板子 → 开机 → 系统自动执行：

1. `/etc/init.d/myapp_daemon start`
2. APP 后台运行
3. 监控程序自动启动
4. 崩溃 → 1 秒自动重启

完全实现**量产级嵌入式产品**的标准运行模式！