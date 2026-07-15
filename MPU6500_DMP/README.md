# 002MPU6500_DMP

基于 STM32F103C8T6 HAL 库的 MPU6500 DMP 姿态角读取示例。工程通过 MPU6500 内部 DMP 输出四元数并换算为 Roll、Pitch、Yaw，结果显示在 0.96 寸 I2C OLED 上，同时通过 USART1 打印到串口。

## 功能

- MCU：STM32F103C8T6
- 传感器：MPU6500，使用 DMP 输出姿态角
- 显示：SSD1306/兼容 128x64 I2C OLED
- 调试输出：USART1，115200 8N1
- 构建方式：CMake + Ninja + arm-none-eabi-gcc

## 硬件连接

### MPU6500

| MPU6500 | STM32F103C8T6 | 说明 |
| --- | --- | --- |
| VCC | 3.3V | 不建议接 5V |
| GND | GND | 共地 |
| SCL | PB10 | I2C2_SCL |
| SDA | PB11 | I2C2_SDA |
| AD0 | GND | 7 位地址 0x68，代码中 HAL 地址为 0xD0 |
| INT | 悬空 | 当前工程未使用中断 |

### OLED

OLED 驱动使用软件 I2C，不走 CubeMX 配置的 I2C2。

| OLED | STM32F103C8T6 | 说明 |
| --- | --- | --- |
| VCC | 3.3V |  |
| GND | GND | 共地 |
| SCL | PB8 | 软件 I2C SCL |
| SDA | PB9 | 软件 I2C SDA |

### 串口

| USART1 | USB-TTL | 说明 |
| --- | --- | --- |
| PA9 TX | RX | 串口输出 |
| PA10 RX | TX | 串口输入，当前基本未使用 |
| GND | GND | 共地 |

串口参数：`115200 8N1`。

## 软件环境

需要安装并加入 `PATH`：

- CMake 3.22 或更高
- Ninja
- GNU Arm Embedded Toolchain，命令前缀为 `arm-none-eabi-`

可选工具：

- STM32CubeMX / STM32CubeIDE：用于查看或重新生成 `002MPU6500_DMP.ioc`
- STM32CubeProgrammer / OpenOCD / ST-LINK 工具：用于烧录 ELF/HEX

## 构建

```powershell
cmake --preset Debug
cmake --build --preset Debug
```

构建产物位于：

```text
build/Debug/002MPU6500_DMP.elf
```

Release 构建：

```powershell
cmake --preset Release
cmake --build --preset Release
```

## 烧录

本仓库当前没有固定烧录脚本，可以使用 STM32CubeProgrammer 或自己的 ST-LINK/OpenOCD 命令烧录：

```text
build/Debug/002MPU6500_DMP.elf
```

烧录后复位开发板。

## 运行现象

上电后程序会：

1. 初始化 OLED、I2C2、USART1。
2. 初始化 MPU6500 DMP，并执行自检。
3. 循环读取姿态角。
4. 在 OLED 上显示 `Roll`、`Pitch`、`Yaw`。
5. 通过 USART1 输出类似内容：

```text
angle:1.23,-0.45,89.67
```

主循环延时为 `20 ms`，显示和串口输出约 50 Hz；DMP 采样率定义在 `MPU6500_DMP/Inc/MPU6500.h` 的 `DEFAULT_MPU_HZ`，当前为 `100`。

## MPU6500_DMP 移植方法

下面以移植到另一个 STM32 HAL 工程为例。

### 1. 复制文件

把整个 `MPU6500_DMP` 目录复制到目标工程：

```text
MPU6500_DMP/
├── Inc/
│   ├── MPU6500.h
│   ├── inv_mpu.h
│   ├── inv_mpu_dmp_motion_driver.h
│   ├── dmpKey.h
│   └── dmpmap.h
└── Src/
    ├── MPU6500.c
    ├── inv_mpu.c
    └── inv_mpu_dmp_motion_driver.c
```

目标工程需要把 `MPU6500_DMP/Inc` 加入头文件路径，并把 `MPU6500_DMP/Src` 下的 3 个 `.c` 文件加入编译。

CMake 工程可参考本项目的写法：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    MPU6500_DMP/Src/MPU6500.c
    MPU6500_DMP/Src/inv_mpu.c
    MPU6500_DMP/Src/inv_mpu_dmp_motion_driver.c
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    MPU6500_DMP/Inc
)
```

如果使用 GCC，还要链接数学库 `m`，因为姿态角换算使用了 `asin`、`atan2`：

```cmake
target_link_libraries(${CMAKE_PROJECT_NAME} m)
```

### 2. 配置 CubeMX 外设

至少需要打开一个 I2C 外设，并确保 HAL I2C 驱动已启用：

- MPU6500 的 `SCL`、`SDA` 接到目标 I2C 外设。
- I2C 速度可先使用 `100 kHz`，稳定后再根据硬件情况提高。
- `Core/Inc/stm32xxxx_hal_conf.h` 中应启用 `HAL_I2C_MODULE_ENABLED`。
- 当前驱动没有使用 MPU6500 的 `INT` 引脚，移植时可以先不配置中断。

本工程使用的是：

```text
I2C2_SCL -> PB10
I2C2_SDA -> PB11
```

### 3. 适配 I2C 句柄

打开目标工程中的 `MPU6500_DMP/Src/inv_mpu.c`，修改顶部的 I2C 句柄和读写宏。

本工程当前使用 `hi2c2`：

```c
extern I2C_HandleTypeDef hi2c2;

#define i2c_write(dev_addr, reg_addr, data_size, p_data) \
HAL_I2C_Mem_Write(&hi2c2, dev_addr, reg_addr, I2C_MEMADD_SIZE_8BIT, p_data, data_size, 0x100)

#define i2c_read(dev_addr, reg_addr, data_size, p_data) \
HAL_I2C_Mem_Read(&hi2c2, dev_addr, reg_addr, I2C_MEMADD_SIZE_8BIT, p_data, data_size, 0x100)
```

如果目标工程使用 `hi2c1`，就改成：

```c
extern I2C_HandleTypeDef hi2c1;

#define i2c_write(dev_addr, reg_addr, data_size, p_data) \
HAL_I2C_Mem_Write(&hi2c1, dev_addr, reg_addr, I2C_MEMADD_SIZE_8BIT, p_data, data_size, 0x100)

#define i2c_read(dev_addr, reg_addr, data_size, p_data) \
HAL_I2C_Mem_Read(&hi2c1, dev_addr, reg_addr, I2C_MEMADD_SIZE_8BIT, p_data, data_size, 0x100)
```

`inv_mpu_dmp_motion_driver.c` 顶部也有一个历史遗留的 `extern I2C_HandleTypeDef hi2c1;`，当前通信并不靠它完成；真正要改的是 `inv_mpu.c` 里的 `i2c_write` 和 `i2c_read`。

### 4. 确认 I2C 地址

`inv_mpu.c` 里的 `.addr = 0xd0` 是 STM32 HAL 使用的 8 位设备地址，对应 MPU6500 的 7 位地址 `0x68`，也就是 `AD0` 接 GND。

如果 `AD0` 接高电平，7 位地址会变为 `0x69`，需要把 `.addr` 改为 `0xd2`。

### 5. 在主程序中调用

在 `main.c` 中包含头文件：

```c
#include "MPU6500.h"
```

外设初始化完成后初始化 DMP：

```c
int ret = 0;
do {
    ret = MPU6050_DMP_init();
} while (ret);
```

主循环中读取姿态角：

```c
float pitch, roll, yaw;

if (MPU6050_DMP_Get_Date(&pitch, &roll, &yaw) == 0) {
    printf("angle:%.2f,%.2f,%.2f\r\n", roll, pitch, yaw);
}
```

当前封装函数名仍叫 `MPU6050_DMP_*`，但底层 `inv_mpu.c` 已经定义为 `MPU6500`，不影响 MPU6500 使用。

### 6. 根据安装方向调整角度

如果 Roll、Pitch、Yaw 方向和实际安装方向不一致，修改 `MPU6500_DMP/Src/MPU6500.c` 中的安装方向矩阵：

```c
static signed char gyro_orientation[9] = {
    -1,  0, 0,
     0, -1, 0,
     0,  0, 1
};
```

这个矩阵决定传感器坐标系到机体坐标系的映射。更换传感器朝向、PCB 安装方向或希望改变正方向时，都应该优先检查这里。

### 7. 常见移植问题

- 编译提示找不到 `MPU6500.h`：检查是否加入了 `MPU6500_DMP/Inc` 头文件路径。
- 链接提示 `asin`、`atan2` 未定义：GCC 工程需要链接数学库 `m`。
- 初始化一直失败：检查 I2C 句柄是否改对、SCL/SDA 是否接反、AD0 地址是否匹配、模块是否 3.3V 供电。
- 自检失败：上电初始化期间保持 MPU6500 静止，并检查模块焊接、供电和 I2C 波形。
- 角度方向反了：修改 `gyro_orientation`。
- 串口打印浮点异常：GCC/newlib-nano 工程需要打开浮点格式化支持，例如链接参数加 `-u _printf_float`。

## 目录结构

```text
Core/                 CubeMX 生成的应用入口、中断和 HAL MSP 代码
Drivers/              STM32 HAL 与 CMSIS 驱动
MPU6500_DMP/          MPU6500 DMP 驱动和姿态角封装
OLED/                 OLED 显示驱动和字库
cmake/                CMake 工具链和 CubeMX 生成的 CMake 配置
002MPU6500_DMP.ioc    CubeMX 工程配置
CMakeLists.txt        顶层构建配置
```

## 注意事项

- DMP 初始化包含自检，上电或复位时尽量保持 MPU6500 静止。
- 如果程序卡在初始化阶段，优先检查 MPU6500 供电、SCL/SDA 接线、上拉电阻和 AD0 电平。
- 当前代码中 MPU6500 的外层函数名仍为 `MPU6050_DMP_*`，但底层 `inv_mpu.c` 已定义 `MPU6500`。
- 姿态安装方向由 `MPU6500_DMP/Src/MPU6500.c` 中的 `gyro_orientation` 决定。若显示角度方向不符合实际安装方向，需要修改该矩阵。
- OLED 使用 PB8/PB9 软件 I2C；MPU6500 使用 PB10/PB11 硬件 I2C2，二者不是同一组引脚。
