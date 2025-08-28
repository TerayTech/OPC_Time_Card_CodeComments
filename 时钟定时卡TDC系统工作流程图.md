# 时钟定时卡TDC系统工作流程图

## 概述

本文档通过详细的Markdown流程图，展示时钟定时卡中TDC（Time-to-Digital Converter）系统的完整工作流程，包括信号触发、时间戳采集、处理和输出的全过程。

## 系统架构流程图

```mermaid
graph TB
    subgraph "输入信号源"
        GPS[GPS PPS信号]
        EXT[外部信号]
        SMA[SMA连接器信号]
    end
    
    subgraph "信号预处理"
        POL[极性选择器]
        BUF[输入缓冲器]
    end
    
    subgraph "TDC核心系统"
        HRCLK[高分辨率时钟域<br/>200MHz]
        SYSCLK[系统时钟域<br/>50MHz]
        SHIFT[移位寄存器TDC]
        CALC[时间戳计算器]
    end
    
    subgraph "时间基准"
        ADJCLK[可调时钟<br/>Adjustable Clock]
        LOCALTIME[本地时间基准<br/>秒+纳秒]
    end
    
    subgraph "处理与控制"
        PI[PI控制器]
        SERVO[伺服算法]
        COMP[延迟补偿]
    end
    
    subgraph "输出与接口"
        IRQ[中断输出]
        AXI[AXI4接口]
        CPU[CPU/软件]
    end
    
    GPS --> POL
    EXT --> POL
    SMA --> POL
    POL --> BUF
    BUF --> SHIFT
    HRCLK --> SHIFT
    SHIFT --> CALC
    SYSCLK --> CALC
    ADJCLK --> LOCALTIME
    LOCALTIME --> CALC
    CALC --> COMP
    COMP --> PI
    PI --> SERVO
    SERVO --> ADJCLK
    CALC --> IRQ
    IRQ --> CPU
    CALC --> AXI
    AXI --> CPU
```

## 详细工作流程

### 1. PPS Slave TDC工作流程

```mermaid
sequenceDiagram
    participant GPS as GPS信号
    participant HR as 高分辨率时钟域
    participant SYS as 系统时钟域
    participant TS as 时间戳处理
    participant PI as PI控制器
    participant ADJ as 可调时钟
    
    Note over GPS,ADJ: PPS信号处理流程
    
    GPS->>HR: PPS边沿信号
    Note right of HR: 200MHz采样
    
    HR->>HR: 移位寄存器记录
    loop 高分辨率采样
        HR->>HR: shift_reg <= shift_reg[N-2:0] & pps_input
    end
    
    HR->>SYS: 同步到系统时钟域
    Note right of SYS: 时钟域切换
    
    SYS->>TS: 检测边沿事件
    alt 检测到PPS边沿
        TS->>TS: 计算高分辨率延迟
        TS->>TS: 补偿输入延迟
        TS->>TS: 补偿电缆延迟
        TS->>TS: 生成精确时间戳
    end
    
    TS->>PI: 发送偏移误差
    PI->>PI: 计算PI控制输出
    PI->>ADJ: 发送调整命令
    
    Note over GPS,ADJ: 完成一次PPS处理循环
```

### 2. 信号时间戳器TDC工作流程

```mermaid
sequenceDiagram
    participant SIG as 输入信号
    participant HR as 高分辨率时钟
    participant SYS as 系统时钟
    participant TS as 时间戳器
    participant IRQ as 中断控制
    participant CPU as CPU
    
    Note over SIG,CPU: 信号时间戳处理流程
    
    SIG->>HR: 信号边沿触发
    
    HR->>HR: 高频采样记录
    Note right of HR: TimestampSysClkNx_EvtShiftReg记录
    
    HR->>SYS: 域同步传输
    Note right of SYS: 移位寄存器同步
    
    SYS->>TS: 边沿检测
    alt 检测到有效边沿
        TS->>TS: 计算寄存器延迟
        Note right of TS: 分析移位寄存器状态
        TS->>TS: 应用延迟补偿
        TS->>TS: 生成最终时间戳
        TS->>TS: 递增事件计数器
    end
    
    TS->>IRQ: 检查中断状态
    alt 中断未挂起
        IRQ->>IRQ: 设置中断标志
        IRQ->>CPU: 发送中断信号
        IRQ->>TS: 存储时间戳到寄存器
    else 中断已挂起
        TS->>TS: 设置丢弃标志
        Note right of TS: 时间戳被丢弃
    end
    
    CPU->>IRQ: 读取时间戳
    CPU->>IRQ: 清除中断标志
    
    Note over SIG,CPU: 准备处理下一个事件
```

### 3. 高分辨率时间戳生成详细流程

```mermaid
flowchart TD
    START([输入信号边沿]) --> HRCLK{高分辨率时钟域}
    
    HRCLK --> SHIFT1[移位寄存器采样<br/>shift_reg <= shift_reg N-2:0  & input]
    SHIFT1 --> SHIFT2[连续移位记录<br/>记录信号状态变化]
    
    SHIFT2 --> SYNC[同步到系统时钟域<br/>TimestampSysClk_EvtShiftReg <= TimestampSysClkNx_EvtShiftReg]
    
    SYNC --> DETECT{检测边沿?<br/>TimestampSysClk2_EvtReg = '1'<br/>and TimestampSysClk3_EvtReg = '0'}
    
    DETECT -->|是| ANALYZE[分析移位寄存器状态]
    DETECT -->|否| WAIT[等待下一个时钟周期]
    WAIT --> DETECT
    
    ANALYZE --> CALC_DELAY[计算高分辨率延迟]
    
    subgraph "延迟计算逻辑"
        CALC_DELAY --> LOOP_START[遍历移位寄存器位]
        LOOP_START --> CHECK_BIT{检查位i是否为'1'}
        CHECK_BIT -->|是,i >= N*2-3| DELAY1[RegisterDelay = 3*ClockPeriod]
        CHECK_BIT -->|是,i >= N-3| DELAY2[RegisterDelay = 2*ClockPeriod + 精细调整]
        CHECK_BIT -->|否| DELAY3[RegisterDelay = 2*ClockPeriod]
        DELAY1 --> DELAY_DONE[延迟计算完成]
        DELAY2 --> DELAY_DONE
        DELAY3 --> DELAY_DONE
    end
    
    DELAY_DONE --> COMPENSATE[应用延迟补偿]
    
    subgraph "延迟补偿计算"
        COMPENSATE --> TOTAL_DELAY[总延迟 = 输入延迟 + 寄存器延迟 + 电缆延迟]
        TOTAL_DELAY --> CHECK_UNDERFLOW{纳秒字段会下溢?}
        CHECK_UNDERFLOW -->|是| BORROW[从秒字段借位<br/>纳秒 = 1000000000 + 当前纳秒 - 总延迟<br/>秒 = 当前秒 - 1]
        CHECK_UNDERFLOW -->|否| SUBTRACT[纳秒 = 当前纳秒 - 总延迟<br/>秒 = 当前秒]
        BORROW --> TIMESTAMP_READY[时间戳准备就绪]
        SUBTRACT --> TIMESTAMP_READY
    end
    
    TIMESTAMP_READY --> OUTPUT[输出最终时间戳<br/>Timestamp_Second_DatReg<br/>Timestamp_Nanosecond_DatReg]
    OUTPUT --> TRIGGER[触发后续处理<br/>Timestamp_ValReg = '1']
    
    TRIGGER --> END([流程结束])
```

### 4. PI控制器处理流程

```mermaid
flowchart TD
    START([接收偏移/漂移数据]) --> CHECK_ENABLE{系统使能?}
    
    CHECK_ENABLE -->|否| RESET[重置PI积分器<br/>输出零调整]
    CHECK_ENABLE -->|是| CHECK_VALID{数据有效?}
    
    CHECK_VALID -->|否| ERROR[输出错误标志<br/>保持上次输出]
    CHECK_VALID -->|是| CHECK_JUMP{时间跳跃?<br/>秒字段 != 0}
    
    CHECK_JUMP -->|是| TIME_JUMP[直接时间设置<br/>重置积分器]
    CHECK_JUMP -->|否| PI_CALC[PI控制计算]
    
    subgraph "PI控制算法"
        PI_CALC --> UPDATE_INTEGRAL[更新积分项<br/>根据误差符号累加/相减]
        UPDATE_INTEGRAL --> CALC_P[计算比例项<br/>P_out = Kp × error]
        CALC_P --> CALC_I[计算积分项<br/>I_out = Ki × integral]
        CALC_I --> COMBINE[组合PI输出<br/>output = P_out + I_out]
        COMBINE --> LIMIT_CHECK{输出限幅检查}
        LIMIT_CHECK -->|超限| LIMIT[限制到最大值<br/>重置积分器]
        LIMIT_CHECK -->|正常| NORMAL[正常输出]
        LIMIT --> PI_OUTPUT[PI控制器输出]
        NORMAL --> PI_OUTPUT
    end
    
    TIME_JUMP --> DIRECT_OUTPUT[直接调整输出]
    PI_OUTPUT --> FORMAT[格式化输出]
    DIRECT_OUTPUT --> FORMAT
    ERROR --> FORMAT
    RESET --> FORMAT
    
    FORMAT --> OUTPUT_REGS[更新输出寄存器<br/>OffsetAdjustment_*<br/>DriftAdjustment_*]
    OUTPUT_REGS --> TRIGGER[触发调整信号<br/>Adjustment_ValOut = '1']
    
    TRIGGER --> END([发送到可调时钟])
```

### 5. 中断处理流程

```mermaid
flowchart TD
    START([时间戳生成完成]) --> CHECK_ENABLE{时间戳器使能?}
    
    CHECK_ENABLE -->|否| DISABLE[清除所有中断<br/>重置计数器]
    CHECK_ENABLE -->|是| CHECK_IRQ{当前中断状态?}
    
    CHECK_IRQ -->|无中断挂起| NEW_IRQ[设置新中断]
    CHECK_IRQ -->|中断已挂起| DROP[设置丢弃标志<br/>TimestamperStatus_DropBit = '1']
    
    subgraph "新中断处理"
        NEW_IRQ --> SET_IRQ[设置中断标志<br/>TimestamperIrq_TimestampBit = '1']
        SET_IRQ --> STORE_TS[存储时间戳<br/>TimestamperTimeValueL/H_DatReg]
        STORE_TS --> STORE_COUNT[存储事件计数<br/>TimestamperCount_DatReg]
        STORE_COUNT --> UPDATE_TOTAL[更新总事件计数<br/>TimestamperEvtCount_DatReg++]
        UPDATE_TOTAL --> CHECK_MASK{中断屏蔽?}
        CHECK_MASK -->|未屏蔽| SEND_IRQ[发送中断到CPU<br/>Irq_EvtOut = '1']
        CHECK_MASK -->|已屏蔽| NO_IRQ[不发送中断]
    end
    
    DROP --> UPDATE_DROP[更新丢弃计数]
    SEND_IRQ --> WAIT_CPU[等待CPU响应]
    NO_IRQ --> WAIT_CPU
    UPDATE_DROP --> WAIT_CPU
    DISABLE --> END
    
    WAIT_CPU --> CPU_READ{CPU读取时间戳?}
    CPU_READ -->|是| CLEAR_IRQ[CPU清除中断标志]
    CPU_READ -->|否| WAIT_CPU
    
    CLEAR_IRQ --> READY[准备处理下一个事件]
    READY --> END([流程结束])
```

### 6. 系统级TDC集成流程

```mermaid
flowchart LR
    subgraph "信号输入层"
        GPS_IN[GPS PPS]
        EXT_IN[外部信号]
        SMA_IN[SMA信号]
    end
    
    subgraph "TDC处理层"
        direction TB
        PPSTDC[PPS Slave TDC<br/>GPS同步处理]
        SIGTDC[Signal Timestamper TDC<br/>通用信号时间戳]
        FREQTDC[Frequency Counter TDC<br/>频率测量]
    end
    
    subgraph "时间基准层"
        ADJCLK[可调时钟<br/>系统时间基准]
        CLKDET[时钟检测器<br/>时钟源选择]
    end
    
    subgraph "控制算法层"
        PI_OFFSET[偏移PI控制器]
        PI_DRIFT[漂移PI控制器]
        TOD[TOD Slave<br/>绝对时间获取]
    end
    
    subgraph "接口层"
        AXI_BUS[AXI4总线]
        IRQ_CTRL[中断控制器]
        PCIE[PCIe接口]
    end
    
    GPS_IN --> PPSTDC
    EXT_IN --> SIGTDC
    SMA_IN --> FREQTDC
    
    PPSTDC --> PI_OFFSET
    PPSTDC --> PI_DRIFT
    PI_OFFSET --> ADJCLK
    PI_DRIFT --> ADJCLK
    TOD --> ADJCLK
    
    CLKDET --> ADJCLK
    ADJCLK --> PPSTDC
    ADJCLK --> SIGTDC
    ADJCLK --> FREQTDC
    
    SIGTDC --> IRQ_CTRL
    PPSTDC --> AXI_BUS
    SIGTDC --> AXI_BUS
    FREQTDC --> AXI_BUS
    
    AXI_BUS --> PCIE
    IRQ_CTRL --> PCIE
```

## 关键时序参数

### 时间精度参数

| 参数 | 数值 | 说明 |
|------|------|------|
| 系统时钟频率 | 50MHz | 基准时钟频率 |
| 高分辨率时钟频率 | 200MHz | TDC采样频率 |
| 时间戳分辨率 | 5ns | 理论最高精度 |
| 寄存器延迟 | 2-3个时钟周期 | 时钟域切换延迟 |
| 输入延迟 | 可配置 | 缓冲器延迟补偿 |
| 电缆延迟 | 可配置 | 传输延迟补偿 |

### 处理延迟分析

```
总处理延迟 = 高分辨率采样延迟 + 域同步延迟 + 计算延迟 + 输出延迟

其中:
- 高分辨率采样延迟: 0-5ns (取决于信号到达时间)
- 域同步延迟: 20ns (1个系统时钟周期)
- 计算延迟: 40-60ns (2-3个系统时钟周期)
- 输出延迟: 20ns (1个系统时钟周期)

总延迟范围: 80-105ns
```

## 软件接口流程

### CPU读取时间戳流程

```mermaid
sequenceDiagram
    participant CPU as CPU软件
    participant AXI as AXI总线
    participant TS as 时间戳器
    participant IRQ as 中断控制器
    
    Note over CPU,IRQ: 时间戳读取流程
    
    IRQ->>CPU: 中断信号
    CPU->>AXI: 读取中断状态寄存器
    AXI->>TS: 访问TimestamperIrq_Reg
    TS->>AXI: 返回中断状态
    AXI->>CPU: 中断状态数据
    
    alt 有时间戳中断
        CPU->>AXI: 读取时间戳低32位
        AXI->>TS: 访问TimestamperTimeValueL_Reg
        TS->>AXI: 返回纳秒字段
        AXI->>CPU: 纳秒数据
        
        CPU->>AXI: 读取时间戳高32位
        AXI->>TS: 访问TimestamperTimeValueH_Reg
        TS->>AXI: 返回秒字段
        AXI->>CPU: 秒数据
        
        CPU->>AXI: 读取事件计数
        AXI->>TS: 访问TimestamperCount_Reg
        TS->>AXI: 返回计数值
        AXI->>CPU: 计数数据
        
        CPU->>AXI: 清除中断标志
        AXI->>TS: 写TimestamperIrq_Reg
        TS->>TS: 清除中断位
        TS->>IRQ: 取消中断信号
    end
    
    Note over CPU,IRQ: 准备处理下一个中断
```

## 故障处理流程

### 时间戳丢弃处理

```mermaid
flowchart TD
    START([新时间戳事件]) --> CHECK_IRQ{检查中断状态}
    
    CHECK_IRQ -->|无中断挂起| NORMAL[正常处理流程]
    CHECK_IRQ -->|中断已挂起| DROP_EVENT[丢弃当前事件]
    
    NORMAL --> PROCESS[处理时间戳]
    PROCESS --> SET_IRQ[设置中断标志]
    SET_IRQ --> STORE[存储时间戳数据]
    
    DROP_EVENT --> SET_DROP[设置丢弃标志<br/>TimestamperStatus_DropBit = '1']
    SET_DROP --> COUNT_DROP[递增丢弃计数]
    COUNT_DROP --> LOG[记录丢弃事件]
    
    STORE --> SUCCESS[处理成功]
    LOG --> DROPPED[事件已丢弃]
    
    SUCCESS --> END([等待CPU读取])
    DROPPED --> END
```

## 总结

时钟定时卡的TDC系统通过以下关键技术实现高精度时间测量：

1. **双时钟域设计**: 高分辨率时钟域(200MHz)进行精确采样，系统时钟域(50MHz)进行处理
2. **移位寄存器TDC**: 利用移位寄存器记录信号状态变化，实现亚纳秒级精度
3. **多级延迟补偿**: 补偿寄存器延迟、输入延迟、电缆延迟等系统性误差
4. **PI控制反馈**: 通过PI控制器实现闭环时间同步
5. **中断驱动处理**: 高效的中断机制确保实时响应
6. **软硬件协同**: AXI接口提供灵活的软件控制和监控能力

整个系统从信号输入到最终输出，实现了完整的TDC功能，为高精度时间同步提供了可靠的硬件基础。
