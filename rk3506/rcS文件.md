这是你嵌入式 Linux**开机最核心的总启动脚本 /etc/init.d/rcS**，它的唯一任务就是：**按数字顺序，自动执行 /etc/init.d/ 下所有 S 开头的开机脚本**，并且对 `.sh` 后缀和无后缀脚本做了优化处理。

# 一、文件基本信息

- 文件名：**rcS**
- 路径：`/etc/init.d/rcS`
- 身份：**BusyBox 嵌入式 Linux 开机总入口**
- 作用：统一调度所有开机自启服务
- 语言：**POSIX Shell (/bin/sh)**

---

# 二、逐行语法 + 大白话翻译

sh

```
#!/bin/sh
```

→ 用 Shell 执行，脚本固定开头

sh

```
# Start all init scripts in /etc/init.d
# executing them in numerical order.
```

→ 注释：启动 /etc/init.d 下的脚本，**按数字顺序执行**

sh

```
for i in /etc/init.d/S??* ;do
```

→ 循环遍历：只找 `/etc/init.d/` 下**S 开头、后面至少跟 2 个字符**的文件（就是你的 S00、S36、S99 脚本）

sh

```
[ ! -f "$i" ] && continue
```

→ 忽略无效文件 / 坏软链接，不是正规文件就跳过

sh

```
case "$i" in
```

→ 判断文件名，分两种情况处理

sh

```
*.sh)
    (
        trap - INT QUIT TSTP
        set start
        . $i
    )
    ;;
```

→ **如果后缀是 .sh**（比如 S36wifibt-init.sh）

- 关掉中断信号，避免被打断
- 用 `. $i` **直接加载执行**（速度更快，不开新进程）

sh

```
*)
    $i start
    ;;
```

→ **如果不是 .sh 后缀**（比如 S99input-event-daemon）

- 直接运行 `脚本名 start` 启动

sh

```
esac
done
```

→ 结束判断→结束循环

---

# 三、核心工作流程（开机必走）

1. 开机 → 内核启动 → 运行 **rcS**
2. rcS 找到 `/etc/init.d/` 所有 **S 开头** 脚本
3. **按文件名数字从小到大执行**（S00→S36→S99）
4. 后缀是 `.sh` → 快速加载执行
5. 无后缀 → 直接调用 start 启动
6. 所有 S 脚本执行完，系统开机完成

---

# 四、关键规则（解答你之前的疑问）

1. **S 开头才会开机自启**，非 S 开头不执行
2. **执行顺序只看文件名数字**，和后缀无关
3. **.sh/ 无后缀 都能正常启动**，只是执行方式不同
4. 这就是你设备**没有 rcS.d/rc3.d**的原因 —— 用 rcS 统一管理

---

# 五、和你的脚本关系

- `S36wifibt-init.sh`（.sh 后缀）→ 快速加载执行
- `S99input-event-daemon`（无后缀）→ 直接 `start` 启动
- 两个**都会开机自动运行**

