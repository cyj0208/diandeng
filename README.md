# STM32F103 点灯与按键作业

## LED1时刻闪烁，LED2仅通过两个按钮实现不同功能

## 一、硬件连接

1. LED1（外接）  PA9   推挽
2. LED2（外接）  PA8   推挽
3. 按键 1（题五）  PB14  EXTI 下降沿，上拉 
4. 按键 2（题六）  PB13  EXTI 下降沿，上拉 

系统时钟 72MHz（HSE 8MHz 晶振 × 9），APB1 二分频，TIM2 时钟 72MHz。


## 二、各题阐述

### 题1：外接 LED 以 500ms 周期闪烁，可以使用 HAL_Delay

用  **HAL_GetTick**  做延时

### 题2：外接 LED 以 500ms 周期闪烁，全程不使用 HAL_Delay 和 GetTick

**`B_TIM_Interrupt`** ——  TIM2 定时器

- TIM2 参数：`PSC = 72`、`ARR = 999` 每 1ms 进一次中断

### 题3：按键通过 EXTI 中断控制 LED 开关，软件消抖 20ms

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if ((HAL_GetTick() - presstime) < 20) { return; }    

    .....
}
```

### 题5：按键短按（<1s）切换常亮/常灭，长按（≥1s）松手后进入 5Hz 快闪，再次长按退出

- `interval < 1000ms` → 常亮/常灭切换
- `interval ≥ 1000ms` → 切换快闪状态

### 题6：按键双击（间隔 <500ms）翻转 LED，单击不改变

- 第一次按下：记下时刻，计数为 1
- 500ms 内再次按下：即为双击，翻转 LED2
- 超过 500ms 未等到第二次：计数清零、退出该模式

                                                            |

