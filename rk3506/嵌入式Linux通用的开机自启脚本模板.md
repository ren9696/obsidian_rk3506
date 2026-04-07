一个**嵌入式 Linux 通用的开机自启脚本模板**，可以直接复制到你的开发板上用，适配绝大多数嵌入式 SysV init 系统。

### 嵌入式自启脚本模板（以自定义 APP 为例）

这个脚本会实现：启动 / 停止 / 重启你的应用程序，并且能被 init 系统识别和自动执行。

bash

运行

```
#!/bin/sh
# /etc/init.d/myapp
# 嵌入式 Linux 自定义应用自启脚本
# 作者：自定义
# 适配：所有使用 SysV init 的嵌入式 Linux (ARM/MIPS 等)

# 配置区（改这里就行）
APP_NAME="myapp"                  # 应用名称
APP_PATH="/usr/bin/myapp"         # 你的应用程序绝对路径
APP_ARGS=""                       # 应用启动参数（没有就留空）
PID_FILE="/var/run/${APP_NAME}.pid" # 进程PID文件

# 函数定义
start() {
    # 检查程序是否已运行
    if [ -f $PID_FILE ] && kill -0 $(cat $PID_FILE) 2>/dev/null; then
        echo "${APP_NAME} 已在运行..."
        return 0
    fi

    # 启动应用（后台运行）
    echo "启动 ${APP_NAME}..."
    $APP_PATH $APP_ARGS &
    echo $! > $PID_FILE

    # 检查是否启动成功
    sleep 1
    if [ -f $PID_FILE ] && kill -0 $(cat $PID_FILE) 2>/dev/null; then
        echo "${APP_NAME} 启动成功！"
    else
        echo "${APP_NAME} 启动失败！"
        rm -f $PID_FILE
        return 1
    fi
}

stop() {
    # 检查程序是否运行
    if [ ! -f $PID_FILE ] || ! kill -0 $(cat $PID_FILE) 2>/dev/null; then
        echo "${APP_NAME} 未运行..."
        return 0
    fi

    # 停止应用
    echo "停止 ${APP_NAME}..."
    kill $(cat $PID_FILE) 2>/dev/null
    sleep 2

    # 强制杀死（如果没停掉）
    if kill -0 $(cat $PID_FILE) 2>/dev/null; then
        echo "强制杀死 ${APP_NAME}..."
        kill -9 $(cat $PID_FILE) 2>/dev/null
    fi

    rm -f $PID_FILE
    echo "${APP_NAME} 已停止！"
}

restart() {
    stop
    start
}

status() {
    if [ -f $PID_FILE ] && kill -0 $(cat $PID_FILE) 2>/dev/null; then
        echo "${APP_NAME} 正在运行 (PID: $(cat $PID_FILE))"
    else
        echo "${APP_NAME} 未运行"
    fi
}

# 主逻辑：处理命令参数
case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    status)
        status
        ;;
    *)
        echo "用法: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac

exit 0
```

---

### 第一步：修改脚本适配你的应用

只需要改 **配置区** 的 3 行内容：

1. `APP_NAME="myapp"` → 改成你的应用名（比如 `led_control`）
2. `APP_PATH="/usr/bin/myapp"` → 改成你的程序实际路径（比如 `/home/root/app/led_demo`）
3. `APP_ARGS=""` → 如果程序需要参数（比如 `-c /etc/config.cfg`），就填在这里

---

### 第二步：部署脚本到嵌入式板子

1. 把脚本传到开发板的 `/etc/init.d/` 目录：
    
    bash
    
    运行
    
    ```
    # 假设脚本文件名叫 myapp，传到板子上
    scp myapp root@你的板子IP:/etc/init.d/
    ```
    
2. 给脚本添加可执行权限（必须！）：
    
    bash
    
    运行
    
    ```
    chmod +x /etc/init.d/myapp
    ```
    
3. 设置开机自启（创建软链接到 rcS.d 目录）：
    
    bash
    
    运行
    
    ```
    # S99 表示启动优先级（数字越大越晚启动），myapp 是脚本名
    ln -s /etc/init.d/myapp /etc/rcS.d/S99myapp
    ```
    

---

### 第三步：测试脚本

bash

运行

```
# 启动应用
/etc/init.d/myapp start

# 查看状态
/etc/init.d/myapp status

# 重启应用
/etc/init.d/myapp restart

# 停止应用
/etc/init.d/myapp stop
```

---

### 关键注意事项（嵌入式必看）

1. **路径问题**：嵌入式系统的根文件系统可能是只读的，若 `/var/run` 不存在，把 `PID_FILE` 改成 `/tmp/${APP_NAME}.pid`（tmp 目录一般可写）。
2. **依赖问题**：如果你的应用依赖网络 / 串口 / 挂载分区，把启动优先级数字改大（比如 S80myapp），确保依赖服务先启动。
3. **权限问题**：脚本必须加 `chmod +x`，否则 init 系统执行不了。
4. **后台运行**：脚本里用 `&` 让程序后台运行，避免阻塞启动流程。

---

### 总结

1. 嵌入式 Linux 中 `/etc/init.d/` 是存放所有服务启动脚本的核心目录，脚本需包含 start/stop/restart 等标准指令。
2. 自定义应用自启的核心步骤：编写标准脚本 → 放到 `/etc/init.d/` → 加执行权限 → 软链接到 `/etc/rcS.d/`。
3. 脚本中通过 PID 文件管理进程，是嵌入式中保证服务稳定运行的常用方法。




我用**嵌入式 Linux 最直白、最接地气**的话给你讲清楚，不搞虚的。

# 一、3 个核心概念

## 1. 应用程序（App / 程序）

就是你写的 **可执行文件**。

- 比如：`led_test`、`gpio_demo`、`main`、`myapp`
- 它是**二进制文件**，双击 / 运行就能跑
- 只做一件事：**干活**

特点：

- 前台运行时，你能看到打印
- 关掉终端，它可能就死了
- 不会自己开机启动

**一句话：应用程序 = 干活的工具。**

---

## 2. 服务（Service）

**一直在后台默默运行的程序**，就叫服务。

比如：

- 网络服务
- SSH 服务
- 你的业务程序（开机就跑、一直跑）

特点：

- 后台运行
- 开机自启
- 能被启动 / 停止 / 重启
- 不占终端

**一句话：服务 = 后台常驻、自动运行的程序。**

---

## 3. 启动脚本（/etc/init.d/xxx）

**就是一个 shell 脚本，用来管理服务。**

它能干什么：

- start：启动服务
- stop：停止服务
- restart：重启
- status：看是否在跑

**启动脚本 ≈ 服务的 “遥控器”**

---

## 4. 软连接（Symbolic Link）

你可以把它理解成：

**Windows 里的「快捷方式」**

- 不是文件本身
- 只是指向原文件的一个 “指针”
- 删除软连接，原文件不受影响
- 修改原文件，所有软连接都生效

在嵌入式里：

plaintext

```
/etc/rcS.d/S99myapp
```

就是指向

plaintext

```
/etc/init.d/myapp
```

的**快捷方式**。

作用：

告诉系统：**开机时要按顺序执行这个脚本**。

---

# 二、它们之间到底是什么关系？（超级清晰版）

我给你画个**极简关系图**：

1. **你写了一个应用程序**
    
    → `/usr/bin/myapp`（干活的）
    
2. **你想让它变成开机自启的后台服务**
    
    → 就需要一个**启动脚本**
    
    → `/etc/init.d/myapp`
    
3. **你告诉系统：开机要启动它**
    
    → 建一个**软连接**到 `/etc/rcS.d/S99myapp`
    

---

**最终关系总结：**

- **应用程序**：真正干活的二进制
- **服务**：在后台一直跑的程序
- **启动脚本**：管理服务怎么启动、停止
- **软连接**：告诉系统开机要执行这个脚本

---

# 三、用生活例子秒懂（最容易记住）

- **应用程序 = 灯泡**
- **服务 = 一直亮着的灯**
- **启动脚本 = 开关**
- **软连接 = 墙上的开关面板（快捷方式）**

你要灯一直亮：

1. 要有**灯泡**（应用程序）
2. 要有**开关**（启动脚本）
3. 把开关装墙上，通电就亮（**软连接 + 开机自启**）

---

# 四、回到嵌入式，一句话总结

- **应用程序**：你写的 `.out` 或二进制文件
- **服务**：后台运行、开机自启的程序
- **启动脚本**：控制服务 start/stop/restart
- **软连接**：让系统开机时找到并执行脚本

---

如果你愿意，我可以下一步带你：

**用你自己的程序，一步步做成服务 + 开机自启**