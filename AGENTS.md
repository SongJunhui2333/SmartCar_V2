# 项目协作说明

本文件适用于 SmartCar_V2 仓库及其子目录。默认使用中文沟通，先阅读相关实现，再做与请求相关的最小修改。本文记录的是初始化时的代码现状；后续以实际源码和工程配置为准。

## 项目概况

- 基于 NXP MIMXRT1064DVL6A（Cortex-M7）和逐飞 SEEKFREE 库的四轮智能车裸机工程。
- 主程序调用 `clock_init(SYSTEM_CLOCK_600M)`；未发现应用层 RTOS、CMake、Makefile 或自动化测试入口。
- MT9V03X 摄像头采集图像，大津法二值化后提取左右边界和中线，计算舵机转向；双轮速度闭环独立运行于 PIT 中断。
- 这是 NXP RT1064 工程，不要套用 TI MSPM0、SysConfig、STM32 HAL 等其他平台的配置与 API。

## 目录与修改入口

以下路径均相对于仓库根目录；工作终端可能位于 `demo/mdk`，不要将该目录误认为仓库根目录。

| 路径 | 职责 |
| --- | --- |
| `demo/user/src/main.c` | 初始化、图像处理调度、舵机与屏幕更新 |
| `demo/user/src/isr.c` | CSI、PIT、UART 等中断服务，双轮速度控制 |
| `demo/user/src/pid.c`、`demo/user/inc/pid.h` | PID 参数、状态、位置式与增量式算法 |
| `demo/user/src/img_process.c`、`demo/user/inc/img_process.h` | 直方图、大津法、二值化、赛道边界提取 |
| `demo/user/inc/main.h` | 应用层编码器、电机、舵机引脚与参数 |
| `demo/user/inc/isr.h` | 编码器测量结果等中断共享变量声明 |
| `demo/code` | 库附带说明建议的新增独立模块目录，当前工程已有对应 code 分组 |
| `demo/mdk/rt1064.uvprojx` | 主用 Keil 工程与源文件清单 |
| `demo/mdk/scf`、`demo/mdk/ini` | 链接布局与调试初始化脚本 |
| `demo/iar/rt1064.eww`、`demo/iar/program/rt1064.ewp` | 备用 IAR 工程 |
| `libraries/zf_common` | 时钟、调试输出、FIFO、公共工具 |
| `libraries/zf_driver` | UART、PWM、GPIO、编码器、PIT 等底层接口 |
| `libraries/zf_device` | 摄像头、屏幕、传感器等设备驱动及预编译依赖 |
| `libraries/sdk`、`libraries/components` | NXP SDK、CMSIS、USB、FatFs、SDMMC 等依赖 |
| `libraries/doc` | 库说明、版本记录与许可证说明 |

已有模块继续在原文件维护；新增独立模块可放在 `demo/code`，不要为小改动重新组织目录。新增 `.c` 文件不会被 Keil 自动发现，必须加入 `.uvprojx` 对应分组，并检查头文件搜索路径。涉及 IAR 支持时同步维护 `.ewp`，否则明确未验证 IAR。

## 构建方式

- 主工程：`demo/mdk/rt1064.uvprojx`。
- Target：`nor_sdram_zf_dtcm`。
- 工程声明的编译器：ARMClang 6.23；设备包：`NXP.MIMXRT1064_DFP.25.09.00`。
- 仓库已有构建日志记录过 Keil MDK 5.42 / ARMClang 6.23，但旧日志不能证明当前代码构建成功，也不能证明当前工具可执行。
- 首选使用 Keil 打开工程，选择上述 Target，执行 Build；涉及工程配置、源文件清单或链接布局时执行 Rebuild。
- 可使用本机实际安装的 UV4 命令行构建。从 `demo/mdk` 执行以下示例，先将 `$uv4` 替换为已确认存在的安装路径：

```powershell
$uv4 = 'C:\path\to\Keil_v5\UV4\UV4.exe'
& $uv4 -b '.\rt1064.uvprojx' -t 'nor_sdram_zf_dtcm' -o '.\Objects\agents_build.log'
```

- 等待构建真正结束并检查本次日志中的错误、警告和完成状态，不要只看已有 `.axf` 或进程启动成功。
- 输出目录为 `demo/mdk/Objects`、`demo/mdk/Listings`，可执行固件为 `Objects/rt1064.axf`。
- 链接脚本为 `demo/mdk/scf/MIMXRT1064xxxxx_flexspi_nor.scf`；本地调试选项使用 `demo/mdk/ini/evkmimxrt1064_flexspi_nor.ini`。
- 保留现有 XIP、启动头、DCD、时钟与内存分配配置；不要直接套用其他开发板的启动或链接文件。
- 不要根据旧注释安装/切换工具链。当前环境若无法访问 Keil 或缺少许可证/设备包，应明确报告验证停留在源码层面。

## 启动流程和实时边界

1. `main()` 初始化 600 MHz 时钟、调试串口，等待 300 ms。
2. `Init()` 初始化 Flash、TFT180、摄像头、编码器、电机 PWM、舵机和 PID，配置 PIT_CH0 为 10 ms。
3. 摄像头初始化失败会持续重试，阻塞 `Init()` 返回；排查主循环没有输出时先检查执行路径。
4. 初始化完成后调用 `interrupt_global_enable(0)`。当前主程序还单独初始化 UART4；它不是 `printf()` 的输出串口。
5. 主循环在 `mt9v03x_finish_flag` 置位后复制图像、处理、显示、更新舵机，完成后清除标志。
6. `PIT_IRQHandler()` 每 10 ms 读取并清空两轮编码器计数，调用左右轮 PID，再更新电机 PWM 与方向。

保持图像处理和电机速度环的职责边界。不要把图像处理、延时、等待外设或大量格式化输出放入控制中断。修改周期时同步检查速度单位、PID 参数及串口负载。保留中断标志清除、SDK 回调和 `__DSB()` 等现有同步操作。

## 硬件与接口约定

- 应用引脚以 `main.h` 为准；设备引脚还需要检查相应 `zf_device_*.h`，不能认为所有引脚都由 `main.h` 配置。
- 左编码器使用 C0/C1，右编码器使用 C2/C24。当前软件直接使用左计数，对右计数取负号以统一前进方向；不要未经硬件依据更改符号。
- 左电机 PWM/方向为 C6/C7，右电机为 C8/C9；当前前进方向电平均为 `GPIO_LOW`，PWM 频率为 17 kHz。
- `PWM_DUTY_MAX=10000` 表示 100%，不是速度单位。当前 PID 输出幅值限制为 5000，实际电机命令被截到 0～5000。
- 舵机为 C30，频率 50 Hz；左右标定端点当前为 100 和 85。保留 `SERVO_MOTOR_DUTY()` 的换算关系，不要把角度直接作为 PWM 值。
- 更改引脚、方向、供电相关假设、占空比上限或舵机范围时，应先核对实际接线与用户要求，不要凭通用例程猜测。

## UART 与上位机

- `debug_init()` 根据 `libraries/zf_common/zf_common_debug.h` 初始化 UART1：115200、TX B12、RX B13；串口助手使用 8N1、无流控。
- `printf()` 在 `zf_common_debug.c` 中通过 `fputc()` 输出到调试 UART；初始化其他 UART 不会自动改变其输出目标。
- 发送接口为 `uart_write_byte()`、`uart_write_buffer()`、`uart_write_string()`，也可用 `debug_send_buffer()`。
- 发送实现为轮询/阻塞。文本优先使用字符串接口，二进制使用显式长度；不要将字符串结尾的 `\0` 误计入有效文本长度。
- 当前 `isr.c` 在 PIT 中断里打印左右编码器计数和目标速度。这是现存行为，相关性能修改可考虑由中断保存快照、主循环降频发送，不要无关地改变上位机格式。
- 调试 UART 的接收中断默认启用，UART1 ISR 调用 `debug_interrupr_handler()`；更换调试串口时检查 ISR 转发。
- 板载 DEBUG USB 转串口与 MCU 原生 USB CDC 是不同链路，未确认连接方式前不要将其混用。

## PID 约定与已知问题

- 当前调用位置式 `PidLocCtrl()`，左右轮分别使用 `speed_pid_l`、`speed_pid_r`；`PidIncCtrl()` 已实现但未用于当前速度环。
- 当前参数为 Kp=50、Ki=0、Kd=0，目标为 60 个编码器计数/10 ms；不能直接解释为 rpm 或 cm/s。
- 调用中的 `t=1.f` 代表当前按采样周期使用参数，不能不调整增益就改为秒单位的 `0.01f`。
- `My_Pid_Init()` 将参数复制进两个 PID 实例；运行时只更改 `speed_KP` 等全局初始参数，不会自动更新实例增益。
- `speed_real`、`speed_pwm` 由左右轮复用，中断结束后保存的是右轮结果。分别观测两轮时读取独立状态或明确保存快照。
- `PidLocCtrl()` 中对 `out_i` 的限幅随后被覆盖，`integrator` 未真正限幅。启用积分前需要检查积分限制和与实际输出范围一致的抗积分饱和。
- 目标速度为零时，若将来启用积分，应设计停止/重启时的积分与输出状态处理；负输出目前仅截为零，不表示主动反转或保证立即停车。
- 上述问题是检查线索，不是初始化时已完成的修复。仅在任务涉及相应行为时修改，并报告行为变化。

## 修改规则

- 优先修改应用层，复用本项目真实存在的逐飞接口；调用前查看声明、实现和枚举，不猜 API、引脚或中断名。
- SDK 中包含逐飞定制，例如 PWM 精度、CSI 和内存配置；不要用未核对的上游 SDK 批量覆盖依赖。
- 保留 `libraries/zf_device/zf_device_config.lib` 和 `.a`，它们是预编译输入，不是编译垃圾。
- 保留源码及第三方依赖中的版权和许可证声明，许可证信息见 `libraries/LICENSE` 和 `libraries/doc`。
- 应用文件与库文件可能混用 UTF-8、GBK 和不同换行方式。先识别编码，保持原文件编码、换行和局部风格；避免全文件格式化或批量转码造成中文乱码。
- 对中断和主循环共享状态，检查 `volatile`、一致性及必要的短临界区；`volatile` 本身不保证多变量快照一致。
- 不要手改 `Objects`、`Listings`、IDE 日志、用户会话缓存或生成的 `.vscode/c_cpp_properties.json`。共享工程配置以 `.uvprojx` 为准。
- 保留根目录 `.gitignore` 对产物的忽略规则；不要全局忽略 `.lib`、`.a`、链接脚本或设备初始化文件。
- 仓库可能没有初始提交，`git diff` 无输出不代表没有改动；结合 `git status` 检查未跟踪文件。除非用户要求，不创建提交、不暂存全部文件。

## 验证与交付

- 文档或忽略规则修改：核对内容、引用路径和 Git 状态即可，不必触发固件编译。
- C/头文件/工程配置修改：优先通过现有 Keil Target 构建；若无法构建，明确说明原因和实际完成的静态检查。
- PID、图像处理等纯算法修改可按需要做小范围主机测试，但不要用算法测试替代目标工程构建或硬件验证。
- 修改控制算法应检查采样周期、符号、数组边界、输出范围、停止行为和状态初始化；不要顺便改变硬件标定值。
- 构建成功不代表实车行为已验证。最终说明修改内容、验证结果及尚未验证的硬件行为。
- 烧录、复位、运行和断点会影响实时控制；仅在用户任务明确包含相应硬件操作时执行，先确认实际探针和板卡，并确保测试条件适合电机/舵机动作。
