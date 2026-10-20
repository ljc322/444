# 实验 4 定时器与 PWM 输出实验报告

## 一、实验目的

1. 深入理解 STM32 通用定时器的工作原理与寄存器架构
2. 掌握定时器定时中断的配置方法与中断回调机制
3. 掌握脉宽调制（PWM）的生成原理与占空比控制方法
4. 实现定时器控制 LED 周期闪烁与 PWM 呼吸灯效果

## 二、实验环境

- **硬件平台**：STM32F103C8T6 最小系统板
- **开发软件**：STM32CubeMX 6.x、Keil MDK5
- **实验外设**：板载 LED（PC13）、外接 LED（可选）

## 三、实验原理

### 3.1 STM32 通用定时器原理

STM32F103 的 TIM2~TIM5 为 16 位通用定时器，挂载于 APB1 总线，定时器时钟为 72MHz（APB1 预分频为 2 时，定时器时钟倍频为 72MHz）。核心功能包括：

- 16 位自动重装载计数器（CNT）、16 位预分频器（PSC）
- 4 个独立通道，可配置为输入捕获、输出比较、PWM 输出模式
- 支持更新中断、捕获 / 比较中断等多种中断触发

定时时间计算公式：

\(T_{定时} = \frac{(PSC+1) \times (ARR+1)}{f_{TIM}}\)

其中\(f_{TIM}=72\mathrm{MHz}\)为定时器时钟频率，PSC 为预分频值，ARR 为自动重装载值。

### 3.2 定时中断控制 LED 原理

配置定时器为计数模式，当计数器计数到 ARR 值时产生**更新中断**，在中断回调函数中翻转 LED 引脚电平，即可实现 LED 周期性亮灭。

### 3.3 PWM 呼吸灯原理

脉宽调制（PWM）通过改变一个周期内高电平的占空比，控制 LED 的平均电流，从而实现亮度变化。

将定时器配置为 PWM 输出模式，硬件自动生成固定周期、可调占空比的方波；在主循环中逐步修改比较寄存器 CCR 的值，即可实现 LED 亮度渐变的呼吸灯效果。

## 四、实验内容与步骤

### 4.1 定时器中断控制 LED 2s 周期闪烁

要求：LED 以 2s 为周期亮灭（亮 1s、灭 1s）。

#### 4.1.1 STM32CubeMX 配置

1. **芯片选择**：新建工程，选择`STM32F103C8T6`

2. **系统时钟配置**：

   - RCC → HSE 选择`Crystal/Ceramic Resonator`
   - Clock Configuration：HSE 8MHz，PLL 倍频至 72MHz 系统时钟，APB1 预分频为 2

3. **调试接口**：SYS → Debug 选择`Serial Wire`

4. **LED 引脚配置**：

   - PC13 设置为`GPIO_Output`，GPIO 模式为推挽输出，初始电平高电平（板载 LED 低电平点亮）

5. **定时器配置**（以 TIM2 为例）：

   - TIM2 → Clock Source 选择`Internal Clock`

   - 参数设置：`PSC=7199`，`Counter Period (ARR)=9999`

     

     （代入公式：

     \(T=\frac{(7199+1)\times(9999+1)}{72000000}=1\mathrm{s}\)

     ，即每 1 秒触发一次中断）

   - NVIC Settings → 勾选`TIM2 global interrupt`，开启定时器全局中断

6. 生成 MDK-ARM 工程。

#### 4.1.2 核心代码编写

在`main.c`中添加如下代码：

1. 启动定时器中断（放在`MX_TIM2_Init()`之后、`while(1)`之前）





```
HAL_TIM_Base_Start_IT(&htim2);
```

1. 重写定时器周期溢出中断回调函数





```
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if(htim->Instance == TIM2)
    {
        // 翻转PC13电平，实现LED亮灭切换
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    }
}
```

### 4.2 PWM 模式实现呼吸灯

要求：LED 以 1~2 秒为周期渐亮渐暗，实现呼吸灯效果。

#### 4.2.1 STM32CubeMX 配置

基于上述工程修改，或新建工程：

1. **定时器 PWM 配置**（以 TIM2 通道 1，对应 PA0 引脚为例）：

   - TIM2 → Clock Source 选择`Internal Clock`

   - Channel1 选择`PWM Generation CH1`

   - 参数设置：`PSC=719`，`Counter Period (ARR)=999`

     

     （PWM 频率：

     \(f=\frac{72000000}{(719+1)\times(999+1)}=100\mathrm{Hz}\)

     ，周期 10ms，保证呼吸顺滑）

   - PWM 模式选择`PWM mode 1`，脉冲极性为高电平有效

2. 引脚 PA0 自动复用为`TIM2_CH1`，无需额外 GPIO 配置

3. 生成 MDK-ARM 工程。

#### 4.2.2 核心代码编写

在`main.c`的`main()`函数中添加：

1. 启动 PWM 输出（放在`MX_TIM2_Init()`之后、`while(1)`之前）



```
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
```

1. `while(1)`循环中实现占空比渐变



```
uint16_t pulse = 0;    // 占空比对应值，范围0~999
uint8_t dir = 0;       // 0：亮度增加，1：亮度降低

while (1)
{
    if(dir == 0)
    {
        pulse++;
        if(pulse >= 999)
            dir = 1;
    }
    else
    {
        pulse--;
        if(pulse == 0)
            dir = 0;
    }
    // 修改比较寄存器，更新占空比
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pulse);
    HAL_Delay(1);  // 调速，总周期约2秒
}
```

## 五、实验结果与现象

1. **定时中断闪烁实验**：

   板载 PC13 LED 以 2 秒为周期稳定亮灭，亮、灭各持续 1 秒，符合实验要求。

2. **PWM 呼吸灯实验**：

   外接 LED（PA0 引脚）呈现平滑的呼吸效果，从完全熄灭逐渐变为最亮，再逐渐熄灭，完整周期约 2 秒；

   通过 Keil 内置逻辑分析仪 / 虚拟示波器观察 PA0 引脚波形，可观测到占空比从 0% 到 100% 再回到 0% 的连续渐变 PWM 方波，周期稳定为 10ms。

## 六、实验总结

1. 通过定时中断实验，掌握了通用定时器 PSC、ARR 参数的计算方法，以及 HAL 库中断回调函数的使用流程。
2. 通过 PWM 呼吸灯实验，理解了 PWM 的硬件生成机制，学会通过修改 CCR 寄存器动态调整占空比。
3. STM32 定时器的 PWM 模式由硬件自动输出波形，无需 CPU 持续翻转引脚，相比软件模拟效率更高、波形更稳定。

附视频演示：

<video src="C:\Users\ljc1\Desktop\0.mp4" controls=""></video>

<video src="C:\Users\ljc1\Desktop\1.mp4" controls=""></video>