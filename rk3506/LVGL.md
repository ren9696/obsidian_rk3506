[[DRM]]

LVGL 官方的 `lvgl.mk` 里面**默认会重新定义 CFLAGS**，将原有的覆盖掉

为什么必须加 `#define LV_CONF_INCLUDE_SIMPLE 1`？这个宏是用来告诉 LVGL：不要用复杂的相对路径去找配置文件，直接用系统头文件路径找 `lv_conf.h`！

[[DRM 头文件找不到]]

[[LVGL的自定义时钟]]

📚 LVGL 触摸功能移植 — 全流程知识点总结
一、LVGL 的整体架构

Apply
┌─────────────────────────────────┐
│         你的应用程序             │
│     (main.c, screen.c)          │
├─────────────────────────────────┤
│         LVGL 核心库             │
│   (lvgl/src/core, widgets ...)  │
├────────────────┬────────────────┤
│  显示驱动(DISP)  │  输入驱动(INDEV) │
│  (drm.c)        │  (evdev.c)      │
├────────────────┴────────────────┤
│          Linux 内核              │
│   DRM/KMS 驱动  │   evdev 输入层  │
│  /dev/dri/card0  │  /dev/input/event0 │
└────────────────┴────────────────┘
关键概念： LVGL 是一个分层架构：

上层：你的 UI 代码（创建按钮、标签等）
中层：LVGL 核心库（处理事件分发、渲染等）
底层：显示驱动 + 输入驱动（连接硬件）
最底层：Linux 内核驱动（DRM 显示、evdev 输入）
二、你遇到的具体知识点
1️⃣ LVGL 初始化流程
C

Apply
void screen_init(void)
{
    lv_init();              // 1. 初始化 LVGL 核心库
    lv_port_disp_init();    // 2. 初始化显示驱动（DRM）
    lv_port_indev_init();   // 3. 初始化输入驱动（触摸）
    create_rtu_ui();        // 4. 创建 UI 界面
}
知识点： 这四步顺序固定，必须先初始化库，再初始化硬件驱动，最后创建 UI。

2️⃣ 显示驱动（DISP）
C

Apply
static void lv_port_disp_init(void)
{
    drm_init();  // 初始化 DRM 显示

    static lv_disp_draw_buf_t draw_buf;    // 绘制缓冲区
    lv_disp_draw_buf_init(&draw_buf, buf, NULL, 720 * 720);

    static lv_disp_drv_t disp_drv;          // 显示驱动
    lv_disp_drv_init(&disp_drv);
    disp_drv.flush_cb = drm_flush;          // 刷新回调函数
    disp_drv.hor_res  = 720;                // 水平分辨率
    disp_drv.ver_res  = 720;                // 垂直分辨率
    lv_disp_drv_register(&disp_drv);
}
知识点： LVGL 通过回调函数与硬件交互。显示驱动需要：

一个缓冲区 draw_buf（存储像素数据）
一个刷新回调 flush_cb（把缓冲区内容送到屏幕）
3️⃣ 输入驱动（INDEV）- 触摸
C

Apply
static void lv_port_indev_init(void)
{
    evdev_init();  // 打开 Linux evdev 设备

    static lv_indev_drv_t indev_drv;          // 输入驱动
    lv_indev_drv_init(&indev_drv);
    indev_drv.type     = LV_INDEV_TYPE_POINTER;  // 类型：触摸
    indev_drv.read_cb  = evdev_read;            // 读取回调
    lv_indev_drv_register(&indev_drv);
}
知识点：

LV_INDEV_TYPE_POINTER — 触摸屏/鼠标（绝对坐标）
LV_INDEV_TYPE_KEYPAD — 键盘
LV_INDEV_TYPE_ENCODER — 旋转编码器
输入驱动同样通过回调函数 read_cb 工作
4️⃣ evdev 的工作原理
什么是 evdev？

Linux 内核的输入事件框架
所有输入设备（触摸屏、键盘、鼠标）都通过 /dev/input/eventX 提供事件
每个事件是 struct input_event 结构体：
C

Apply
struct input_event {
    struct timeval time;  // 时间戳
    __u16 type;           // 事件类型（EV_ABS, EV_KEY, EV_SYN）
    __u16 code;           // 事件编码（ABS_MT_POSITION_X, BTN_TOUCH）
    __s32 value;          // 事件值（坐标值、按键状态）
};
event0 的 hexdump 解析：


Apply
0003 0035 0176 0000
│    │    │
│    │    └─ value = 0x0176 (374)
│    └────── code  = 0x0035 (ABS_MT_POSITION_X)
└─────────── type  = 0x0003 (EV_ABS)
5️⃣！最重要的知识点：LVGL 的 Tick 时钟 ⏱️
这个问题困扰了你最久！

C

Apply
// lv_conf.h 中
#define LV_TICK_CUSTOM 1
#if LV_TICK_CUSTOM
    #define LV_TICK_CUSTOM_INCLUDE "time.h"
    #define LV_TICK_CUSTOM_SYS_TIME_EXPR (clock() / (CLOCKS_PER_SEC / 1000))
#endif
为什么 tick 如此重要？

LVGL 功能	依赖 tick？
触摸输入读取	✅ 每 LV_INDEV_DEF_READ_PERIOD（30ms）读取一次
屏幕定时刷新	✅ 每 LV_DISP_DEF_REFR_PERIOD（30ms）刷新一次
定时器回调	✅ 比如你的 data_update_cb 每秒执行
动画	✅ LVGL 内置动画
UI 渲染	❌ 不直接依赖，但刷新周期依赖
没有 tick 时： LVGL 内部时间永远为 0，所有周期性任务（包括读取触摸！）都不会执行。

tick 的两种实现方式：

方式	配置	说明
手动调用	LV_TICK_CUSTOM = 0	在 main loop 中调用 lv_tick_inc(1)
自动获取	LV_TICK_CUSTOM = 1	提供一个获取当前毫秒数的函数，如 clock()
6️⃣ LVGL 的 主循环（Main Loop）
C

Apply
while (1) {
    lv_task_handler();  // LVGL 核心处理：读取输入、刷新显示、触发定时器
    usleep(100);        // 稍微等待，减少 CPU 占用
}
知识点： lv_task_handler() 是 LVGL 的心脏：

检查是否有定时器到期 → 执行回调
检查输入设备是否需要读取 → 调用 read_cb
检查是否有区域需要重绘 → 调用 flush_cb
7️⃣ 条件编译和配置系统

Apply
lv_conf.h       — LVGL 核心功能配置
lv_drv_conf.h   — 驱动配置（evdev、DRM、触摸类型...）
配置触摸时你接触到的宏：
宏	作用
USE_EVDEV 1	启用 evdev 输入驱动
EVDEV_NAME "/dev/input/event0"	指定触摸设备节点
EVDEV_SWAP_AXES	是否交换 X/Y 坐标
EVDEV_CALIBRATE	是否校准坐标范围
8️⃣ Makefile 构建系统
Makefile

Apply
include $(LVGL_DIR)/lvgl.mk          # 包含所有 LVGL 源文件
include $(LV_DRIVERS_DIR)/lv_drivers.mk  # 包含所有驱动源文件
LVGL 使用 .mk 文件自动管理源文件列表，你只需要 include 即可。

三、这次调试过程中用到的 Linux 调试命令
Bash
Run
# 查看所有输入设备
cat /proc/bus/input/devices

# 读取触摸设备原始事件（硬件级测试）
hexdump /dev/input/event0

# 查看设备权限
ls -l /dev/input/event0

# 查看运行中的进程（发现 input-event-daemon）
ps
四、学习 LVGL 的建议路线

Apply
第1步：理解架构
├── 初始化流程 (lv_init → 驱动 → UI)
├── main loop (lv_task_handler)
└── tick 时钟 (这次踩的坑！)

第2步：学习控件
├── lv_label（标签）
├── lv_btn（按钮）
├── lv_tabview（标签页）
├── lv_table（表格）
└── ... 更多控件

第3步：学习样式
├── lv_style_t（样式对象）
├── 状态（LV_STATE_DEFAULT, LV_STATE_PRESSED...）
├── 盒子模型（padding, border, margin）
└── Flex/Grid 布局

第4步：学习事件
├── lv_event_t（点击、长按、滑动...）
├── 事件冒泡
└── 自定义事件

第5步：学习高级主题
├── lv_timer（定时器）
├── lv_anim（动画）
├── 字体管理（你之前修复过 my_cn_font.c）
└── 内存管理
推荐学习资源：

📖 LVGL 官方文档：https://docs.lvgl.io/
💻 官方示例：https://docs.lvgl.io/master/examples.html
🎯 小项目练习：先做一个"点击按钮改变标签文字"的 demo
五、你这次攻克的关键问题一览
#	问题	原因	修复
1	static_bitmap 编译错误	LVGL v8 移除了该字段	加条件编译 #if LVGL_VERSION_MAJOR < 8
2	#include 缺少文件名	编辑出错	恢复正确头文件
3	触摸无反应	没有 tick 时钟源	开启 LV_TICK_CUSTOM = 1
4	ABS_MT_TRACKING_ID 判断	只处理 ==0 不处理 >0	改为 >= 0
你现在已经不是 LVGL 小白了！🎊 你已经：

✅ 理解了 LVGL 的分层架构
✅ 成功移植了 DRM 显示驱动
✅ 成功移植了 evdev 触摸驱动
✅ 解决了 tick 时钟这个经典坑
✅ 看懂了 evdev 的原始事件数据
✅ 修复了字体兼容性问题
如果还有什么疑问或者想深入学习某个部分，随时问我！加油 💪