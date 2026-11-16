# STM32F103 点灯与按键作业

STM32F103C8T6 最小系统板 + STM32CubeMX 6.15 + Keil MDK（ARMCC V5）+ HAL 库。

## 一、硬件连接

| 功能 | 引脚 | 配置 |
| --- | --- | --- |
| LED1（外接） | PA9 | 推挽输出，高电平点亮 |
| LED2（外接） | PA8 | 推挽输出，高电平点亮 |
| 按键 1（题五） | PB14 | EXTI 下降沿触发，内部上拉 |
| 按键 2（题六） | PB13 | EXTI 下降沿触发，内部上拉 |

系统时钟 72MHz（HSE 8MHz 晶振 × 9），APB1 二分频，TIM2 时钟 72MHz。

## 二、工程目录

| 目录 | 内容 |
| --- | --- |
| `A_HAL_GetTick/` | 题2 的 GetTick 写法 + 题5 + 题6 |
| `B_TIM_Interrupt/` | 题2 的定时器中断写法 + 题5 + 题6 |

两个工程使用同一份 CubeMX 配置（`diandeng.ioc`），除 LED1 的计时方式外代码一致。
作业要求「多个 Keil 项目也可以，但提交时要放在一个仓库里」，因此合并为本仓库。

## 三、各题实现

### 题2：外接 LED 以 500ms 周期闪烁，全程不使用 HAL_Delay 和 GetTick

两版都实现 500ms 周期（亮 250ms + 灭 250ms），区别在时间来源。

**`A_HAL_GetTick`** —— 过渡版本，用 `HAL_GetTick()` 差值判断，不用 `HAL_Delay`：

```c
if ((HAL_GetTick() - led1time) >= 250)
{
    led1time = HAL_GetTick();
    HAL_GPIO_TogglePin(LED1_GPIO_Port, LED1_Pin);
}
```

**`B_TIM_Interrupt`** —— 符合作业要求的版本，由 TIM2 硬件定时：

- TIM2 参数：`PSC = 72`、`ARR = 999`
- 72MHz ÷ 72 = 1MHz，1MHz ÷ 1000 = 1kHz → **每 1ms 进一次中断**
- 中断里计数，满 250 次翻转一次 LED1

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim == &htim2)
    {
        if (++ms >= 250)          /* 250 × 1ms = 250ms */
        {
            ms = 0;
            HAL_GPIO_TogglePin(LED1_GPIO_Port, LED1_Pin);
        }
    }
}
```

LED1 这条通路上没有 `HAL_Delay`，也没有 `HAL_GetTick`，时间完全由硬件定时器产生。

### 题3：按键通过 EXTI 中断控制 LED 开关，软件消抖 20ms

EXTI15_10 中断服务中做时间戳消抖，20ms 内的重复下降沿直接丢弃：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if ((HAL_GetTick() - presstime) < 20) return;   /* 20ms 软件消抖 */
    ...
    presstime = HAL_GetTick();
}
```

本仓库中这条通道同时服务于题五、题六的按键输入，未单独再做一个「按一下开、再按一下关」的工程。

### 题5：按键短按（<1s）切换常亮/常灭，长按（≥1s）松手后进入 5Hz 快闪，再次长按退出

按键接 PB14。因为 EXTI 只配了下降沿，**松手测不到中断**，所以在主循环里读引脚判断释放时刻，再按按下时长分流：

- `interval < 1000ms` → 常亮/常灭切换
- `interval ≥ 1000ms` → 切换快闪状态

5Hz 快闪由主循环每 100ms 翻转一次实现（周期 200ms）：

```c
if ((HAL_GetTick() - led2time) >= 100)
{
    led2time = HAL_GetTick();
    HAL_GPIO_TogglePin(LED2_GPIO_Port, LED2_Pin);
}
```

### 题6：按键双击（间隔 <500ms）翻转 LED，单击不改变

按键接 PB13，用计数 + 时间窗判断：

- 第一次按下：记下时刻，计数置 1
- 500ms 内再次按下：判定双击，翻转 LED2
- 超过 500ms 未等到第二次：计数清零、退出该模式，单击不动作

```c
if (botton_press_ci == 1 && (HAL_GetTick() - firsttime) > 500)
{
    botton_press_ci = 0;        /* 单击，作废 */
    flag = 0;
}
if (botton_press_ci == 2)
{
    if ((presstime - firsttime) < 500) HAL_GPIO_TogglePin(LED2_GPIO_Port, LED2_Pin);
    botton_press_ci = 0;
}
```

## 四、按键消抖说明

只靠 20ms 时间窗挡不住一种情况：**松手瞬间的触点弹跳**。

松开时引脚先跳高、再弹回低，这个「高→低」也是一次下降沿；而它距离按下已经一两百毫秒，20ms 窗口早就过期，会被当成「第二次按下」，导致单击被判成双击。

为此在主循环里持续记录按键电平，中断里额外判断「这个下降沿之前，引脚是不是真的回到过高电平」：

```c
/* 主循环每轮采样 */
t6_botton = HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13);

/* 中断里 */
if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13) == GPIO_PIN_RESET)
{
    flag = 2;
    if (t6_botton) botton_press_ci++;      /* 只有真正松开过才算一次按下 */
}
```

松手弹跳时那个高电平只存在几百纳秒，主循环采样不到，`t6_botton` 仍为 0，该边沿被拒绝，单击不会再误触发翻转。

## 五、运行环境要求

| 项 | 要求 | 说明 |
| --- | --- | --- |
| Keil MDK | 5.x，编译器 **ARMCC V5（AC5）** | 工程按 AC5 配置（`uAC6 = 0`）。若本机只有 AC6，在 Options for Target → Target 里切到 v6 也能编译，HAL 代码兼容，可能有少量 warning |
| 器件包 | `Keil.STM32F1xx_DFP` **2.4.0**、`ARM::CMSIS` **5.0.1** | CubeMX 生成的 Keil 工程依赖这两个 pack。版本不一致时 Keil 会提示 Migrate，按提示操作即可 |
| 芯片 | STM32F103C8T6 | 64KB Flash / 20KB RAM |
| **外部晶振** | **必须有 8MHz 晶振** | 两个工程都用 HSE 8MHz × 9 = 72MHz。**板上没有 8MHz 晶振会卡在 `Error_Handler()` 的死循环里，现象是灯全不亮、像没烧进去** |
| 下载器 | ST-Link | 工程调试器已设为 ST-Link，Flash 算法配置随 `MDK-ARM/RTE` 一起提交 |

## 六、编译与下载

1. 用 Keil MDK 打开对应目录下的 `MDK-ARM/diandeng.uvprojx`
2. **先 Rebuild（F7）**——仓库里没有提交编译产物，必须自己编一次，0 error 0 warning 后再下载
3. 下载（F8，ST-Link）
4. 如需修改引脚或外设，用 STM32CubeMX 打开目录下的 `diandeng.ioc`

> 编译产物（`MDK-ARM/diandeng/`）和 Keil 的 `*.uvguix.*` 已加入 `.gitignore`，未提交。
> `Drivers/CMSIS` 下的 DSP、NN、RTOS 等与本作业无关的大体积目录也已剔除，不影响编译。

## 七、完成情况

| 题号 | 状态 | 说明 |
| --- | --- | --- |
| 题1 500ms 闪烁（可用 HAL_Delay） | 待补 | 本仓库未包含该工程 |
| 题2 500ms 闪烁（不用 Delay/GetTick） | 完成 | 见 `B_TIM_Interrupt/` |
| 题3 EXTI 控制 LED + 20ms 消抖 | 部分 | 消抖已实现，未单独做开关灯工程 |
| 题4 提交到 GitHub 仓库 | 完成 | 本仓库 |
| 题5 短按/长按 + 5Hz 快闪 | 完成 | 见两个工程的 PB14 分支 |
| 题6 双击翻转，单击不变 | 完成 | 见两个工程的 PB13 分支 |
