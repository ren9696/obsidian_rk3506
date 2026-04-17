[[DRM]]
LVGL 官方的 `lvgl.mk` 里面**默认会重新定义 CFLAGS**，将原有的覆盖掉

为什么必须加 `#define LV_CONF_INCLUDE_SIMPLE 1`？这个宏是用来告诉 LVGL：不要用复杂的相对路径去找配置文件，直接用系统头文件路径找 `lv_conf.h`！

[[DRM 头文件找不到]]

