引脚编号

IO测试脚本
```
#!/bin/bash

# GPIO引脚编号
GPIO_PINS=(11 12 13 14)

# 导出GPIO并设置为输出
for pin in ${GPIO_PINS[@]}; do
    echo $pin > /sys/class/gpio/export
    echo out > /sys/class/gpio/gpio$pin/direction
    echo 0 > /sys/class/gpio/gpio$pin/value
done

echo "开始GPIO测试..."
echo "每个LED应依次点亮1秒"

# 依次测试每个引脚
for pin in ${GPIO_PINS[@]}; do
    echo "测试GPIO$pin..."
    echo 1 > /sys/class/gpio/gpio$pin/value
    sleep 1
    echo 0 > /sys/class/gpio/gpio$pin/value
    sleep 0.5
done

# 同时闪烁所有LED
echo "同时闪烁所有LED 5次..."
for i in {1..5}; do
    for pin in ${GPIO_PINS[@]}; do
        echo 1 > /sys/class/gpio/gpio$pin/value
    done
    sleep 0.5
    
    for pin in ${GPIO_PINS[@]}; do
        echo 0 > /sys/class/gpio/gpio$pin/value
    done
    sleep 0.5
done

# 清理
for pin in ${GPIO_PINS[@]}; do
    echo $pin > /sys/class/gpio/unexport
done

echo "测试完成！"
```
`
[[libgpio]]