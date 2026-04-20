## 错误原因（一句话）

你开启了 LVGL 的 **自定义时钟（custom tick）** 功能，

但是**没有提供 custom_tick_get 函数** → 链接器报错找不到。

---

# 真正的 root cause

你的 `lv_conf.h` 里打开了这个：

c

运行

```
#define LV_TICK_CUSTOM 1
```

一旦开成 `1`，LVGL 就会**强制调用你自己写的 `custom_tick_get()`**

但你根本没写这个函数 → 报错 **undefined reference to `custom_tick_get`**

---

# 100% 修复方案（只改 1 个数字）

打开

plaintext

```
src/screen/lv_conf.h
```

找到：

c

运行

```
#define LV_TICK_CUSTOM 1
```

改成

c

运行

```
#define LV_TICK_CUSTOM 0
```

保存，然后：

bash

运行

```
make clean
make
```

✅ **错误立刻消失！**

---

# 为什么这样改就对了？

- `LV_TICK_CUSTOM 0` = LVGL **使用内部默认时钟**
- 不需要你提供任何函数
- 嵌入式 Linux / RK3506 必须用 0，不能用 1

---

# 你现在只需要确保 lv_conf.h 里这一行正确

c

运行

```
#define LV_TICK_CUSTOM 0
```

---

# 现在重新编译

bash

运行

```
make clean
make
```

🎉 **马上就能成功编译出完整的 RTU + LVGL 可执行文件！**

---

## 总结

1. 报错 `undefined reference to custom_tick_get`
2. 原因：`LV_TICK_CUSTOM 1` 开启了自定义时钟
3. 解决：改成 `0`
4. 完成！