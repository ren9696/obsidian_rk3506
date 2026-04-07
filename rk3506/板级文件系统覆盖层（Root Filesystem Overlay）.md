这是开发板厂商（或您自己）添加自定义文件（如您的 `S89ML37init`）最标准、最常用的方式。覆盖层是一个目录，其中的文件结构会在 Buildroot 构建过程的最后阶段，**原样复制**到目标根文件系统的对应位置。

当前Buildroot配置中确实设置了多个根文件系统覆盖目录：

```
BR2_ROOTFS_OVERLAY="board/rockchip/common/base/board/rockchip/rk3506/fs-overlay/ board/rockchip/rk3506/fast-display-overlay/"
```

**这里有3个覆盖目录**（看起来您复制时格式有点问题，我重新整理一下）：

1. `board/rockchip/common/base/`
    
2. `board/rockchip/rk3506/fs-overlay/`
    
3. `board/rockchip/rk3506/fast-display-overlay/`
    

## 一、如何理解和查找文件

### 1. **覆盖目录的叠加顺序**

在Buildroot中，**后面目录的文件会覆盖前面目录的同名文件**。顺序如下：

- 第1层：`board/rockchip/common/base/`(最底层)
    
- 第2层：`board/rockchip/rk3506/fs-overlay/`(覆盖第1层)
    
- 第3层：`board/rockchip/rk3506/fast-display-overlay/`(覆盖前2层)
    

### 2. **查找脚本的具体步骤**

因为多个目录可能有相同的文件，所以要按照**从高到低的优先级**查找：

```
# 1. 首先在Buildroot根目录下查找所有可能的S89ML37init文件
cd /path/to/buildroot

# 2. 按照优先级顺序搜索（从高到低）
find board/rockchip/rk3506/fast-display-overlay -name "S89ML37init" 2>/dev/null
find board/rockchip/rk3506/fs-overlay -name "S89ML37init" 2>/dev/null
find board/rockchip/common/base -name "S89ML37init" 2>/dev/null

# 3. 或者使用单个命令查找所有位置
find board/rockchip -name "S89ML37init" 2>/dev/null
find board/rockchip/common -name "S89ML37init" 2>/dev/null
```

### 3. **快速定位脚本的方法**

```
# 直接查看所有init.d脚本的位置
find board/rockchip -path "*/etc/init.d/*" -type f 2>/dev/null | sort

# 如果知道脚本名字，直接搜索
grep -r "S89ML37init" board/rockchip/ 2>/dev/null

# 查看最终生成的init.d目录内容（构建后）
ls -la output/target/etc/init.d/
```

## 二、不同优先级目录的作用

### 1. **各目录的用途分析**

- **`board/rockchip/common/base/`**: Rockchip平台的通用基础配置，所有RK芯片共用
    
- **`board/rockchip/rk3506/fs-overlay/`**: RK3506芯片的特定文件系统覆盖
    
- **`board/rockchip/rk3506/fast-display-overlay/`**: 显示相关的覆盖（可能有显示优化配置）
    

### 2. **查找和修改脚本的流程**

```
# 第一步：查看目标系统中实际存在的脚本
ls -la /etc/init.d/S89*  # 在目标板上运行

# 第二步：根据脚本名查找源文件
# 假设您在目标板上看到的是 S89ML37init
cd /path/to/buildroot

# 第三步：按优先级从高到低查找
echo "在fast-display-overlay中查找："
find board/rockchip/rk3506/fast-display-overlay -name "S89ML37init" 2>/dev/null

echo "在fs-overlay中查找："
find board/rockchip/rk3506/fs-overlay -name "S89ML37init" 2>/dev/null

echo "在common/base中查找："
find board/rockchip/common/base -name "S89ML37init" 2>/dev/null
```

## 三、实际操作的完整示例

假设您的Buildroot在 `/home/user/buildroot`目录：

```
# 进入Buildroot目录
cd /home/user/buildroot

# 查看完整的覆盖目录配置
grep "BR2_ROOTFS_OVERLAY" .config

# 查找脚本的具体位置
SCRIPT_PATH=""
for dir in board/rockchip/common/base board/rockchip/rk3506/fs-overlay board/rockchip/rk3506/fast-display-overlay; do
    if [ -f "$dir/etc/init.d/S89ML37init" ]; then
        SCRIPT_PATH="$dir/etc/init.d/S89ML37init"
        echo "找到脚本在: $SCRIPT_PATH"
        break
    fi
done

# 如果没有找到，尝试更广泛的搜索
if [ -z "$SCRIPT_PATH" ]; then
    echo "未直接找到，尝试更广泛的搜索..."
    find board/rockchip -name "S89ML37init" 2>/dev/null
fi

# 如果找到脚本，进行修改
if [ -n "$SCRIPT_PATH" ]; then
    echo "正在编辑脚本: $SCRIPT_PATH"
    vi "$SCRIPT_PATH"
    
    # 修改GPIO引脚号（将138改为102）
    # 在脚本中找到类似这样的行并修改：
    # echo 138 > /sys/class/gpio/export  改为  echo 102 > /sys/class/gpio/export
    # 保存退出
fi
```

## 四、修改后的验证步骤

```
# 1. 重新构建系统
cd /home/user/buildroot
make

# 2. 检查修改是否生效
# 查看构建后的脚本内容
cat output/target/etc/init.d/S89ML37init | grep "gpio"

# 3. 部署到目标板并测试
# 将新镜像烧录到开发板
# 启动后检查GPIO引脚号是否正确
```

## 五、如果脚本不在覆盖目录中

如果通过上述方法没有找到脚本，可能有以下情况：

### 1. **脚本由软件包生成**

```
# 在package目录中搜索
find package -name "*ML37*" -o -name "*init*" 2>/dev/null | grep -v ".git"
```

### 2. **脚本在post-build脚本中生成**

```
# 查看post-build脚本
grep "BR2_ROOTFS_POST_BUILD_SCRIPT" .config
# 如果设置了，查看该脚本内容
cat board/rockchip/rk3506/post-build.sh 2>/dev/null
```

### 3. **检查最终生成的文件**

```
# 查看最终文件系统中的脚本
cat output/target/etc/init.d/S89ML37init

# 查看它的创建者（如果是在构建过程中生成的）
grep -r "S89ML37init" output/build/ 2>/dev/null
```

## 六、多覆盖目录的管理建议

由于有多个覆盖目录，建议：

1. **优先修改优先级高的目录**（`fast-display-overlay`> `fs-overlay`> `common/base`）
    
2. **保持目录结构清晰**：
    
    ```
    # 创建自己的覆盖目录（如果需要添加新文件）
    mkdir -p custom-overlay/etc/init.d/
    cp board/rockchip/rk3506/fs-overlay/etc/init.d/S89ML37init custom-overlay/etc/init.d/
    # 修改custom-overlay中的脚本
    
    # 在.config中添加自己的覆盖目录（放在最后，优先级最高）
    # BR2_ROOTFS_OVERLAY="原有路径 custom-overlay"
    ```
    

## 总结

根据您的配置文件，您需要在这3个目录中查找 `S89ML37init`脚本：

1. `board/rockchip/common/base/etc/init.d/`
    
2. `board/rockchip/rk3506/fs-overlay/etc/init.d/`
    
3. `board/rockchip/rk3506/fast-display-overlay/etc/init.d/`
    

**按照优先级从高到低查找**，优先修改 `fast-display-overlay`目录中的脚本，如果不存在再查找其他目录。

请执行查找命令并告诉我结果，我可以进一步指导您如何修改。

里面的脚本需要[[修改文件权限]]