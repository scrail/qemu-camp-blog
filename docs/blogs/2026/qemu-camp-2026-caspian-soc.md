# QEMU SOC 方向

!!! note "主要贡献者"

    - 作者：[@caspian](https://github.com/trace1729)

---

> 背景介绍 & 专业阶段 见 CPU 方向

## 功能实现

首先运行 `make -f Makefile.camp test-soc` 发现 test-board-g233 测试通过，g233 是如何配置设备而通过测试的呢？

## 设备初始化骨架：地址空间与 MemoryRegion

hw/riscv/g233.c 将地址空间映射定义在 vir_memmap 数组中，然后在 `virt_machine_init` 中按地址初始化设备：

```c
// hw/riscv/g233.c
static const MemMapEntry virt_memmap[] = {
    [VIRT_DRAM]  = { 0x80000000, 0x0 },     // size=0 → 运行时由 -m 决定
    [VIRT_UART0] = { 0x10000000, 0x100 },
    [VIRT_GPIO]  = { 0x10012000, 0x100 },
    // ...
};
```

注册地址空间有两个通用步骤：

1. `memory_region_init_*` — 创建一段地址空间但暂不挂接到全局
   - `memory_region_init_ram` — 可读写内存（DRAM）
   - `memory_region_init_rom` — 只读内存（MROM）
   - `memory_region_init_io` — **外设空间**，绑定读写回调（设备建模关键）

2. `memory_region_add_subregion` — 挂接到父容器的地址树
   - 设备通常用 `sysbus_mmio_map`（即 `add_subregion` 的马甲）

```c
// DRAM: 最简单，纯内存
memory_region_init_ram(machine->ram, NULL, "riscv_virt_board.ram",
                       machine->ram_size, &error_fatal);
memory_region_add_subregion(system_memory, s->memmap[VIRT_DRAM].base,
                            machine->ram);

// 外设: 需要绑定读写回调
memory_region_init_io(&s->iomem, OBJECT(s), &pl011_ops, s, "pl011", 0x1000);
sysbus_init_mmio(SYS_BUS_DEVICE(s), &s->iomem);  // 注册到 SysBus
// 之后 sysbus_mmio_map 将其挂到全局地址空间
```

---

## QOM 建模框架：以 OOP 视角理解

QEMU 用 C 实现了面向对象模型（QOM），每个外设都是一个**对象**

!!! tips inline "C++ 概念"

    - class 定义        
    - static 成员变量     
    - 构造函数 Constructor
    - 成员方法            
    - 工厂模式 Factory

!!! tips inline "QEMU 实现"

    - TypeInfo 结构体           
    - class_init 中设置的常量      
    - instance_init + realize 
    - read/write       
    - xxx_create 函数          

!!! tips "作用"

    - 类型名、继承关系...
    - 同类型所有实例共享       
    - 初始化内部状态 + 激活    
    - 通过 MMIO 访问触发    
    - 在 Board 中创建并连线  

### 四层抽象

```
① TypeInfo (动态类型 TypeImpl) → "这是什么类型？"
    .name, .parent, .instance_size
    .class_init, .instance_init

② class_init                  → "这类设备的公共特征"
    设置常量、属性 getter/setter
    dc->realize = xxx_realize / xxx_init 

③ instance_init               → "实例的初始状态"
    分配子对象、设置默认值
    初始化 MMIO/IRQ/Clock 等"先天配置"

④ realize                     → "激活这个实例"
    依赖用户配置的属性（如 chardev）
    分配硬件资源、注册中断
```

### 从 PL011 看四层展开

```c
// ① TypeInfo — 声明类型
static const TypeInfo pl011_arm_info = {
    .name          = TYPE_PL011,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(PL011State),
    .instance_init = pl011_init,
    .class_init    = pl011_class_init,
};

// ② class_init — 类特征
static void pl011_class_init(ObjectClass *oc, const void *data) {
    DeviceClass *dc = DEVICE_CLASS(oc);
    dc->realize = pl011_realize;
    device_class_set_legacy_reset(dc, pl011_reset);
    dc->vmsd = &vmstate_pl011;
}

// ③ instance_init — 先天配置
static void pl011_init(Object *obj) {
    memory_region_init_io(&s->iomem, ..., &pl011_ops, s, "pl011", 0x1000);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->iomem);
    sysbus_init_irq(SYS_BUS_DEVICE(obj), &s->irq);
    s->clk = qdev_init_clock_in(...);
}

// ④ realize — 激活（依赖用户已设置的属性）
static void pl011_realize(DeviceState *dev, Error **errp) {
    qemu_chr_fe_set_handlers(&s->chr, ...);  // 依赖 chardev property
}

// 工厂函数 — Board 调用
DeviceState *pl011_create(hwaddr addr, qemu_irq irq, Chardev *chr) {
    dev = qdev_new("pl011");
    qdev_prop_set_chr(dev, "chardev", chr);
    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, addr);
    sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, irq);
    return dev;
}
```

为什么 instance_init 和 realize 分开？

- **时序问题**：instance_init 执行时用户配置的属性（如 chardev）还未设置，realize 执行时属性已就位。MMIO/IRQ 这些"硬件固定"的放在 instance_init，依赖属性的放在 realize。

---

2: 内部状态 → C 结构体字段

```cpp
  struct G233xxxState {
      SysBusDevice parent_obj;     // QOM 基类
      MemoryRegion mmio;           // 地址空间
      qemu_irq irq;                // 中断线
      uint32_t regs[...];          // 寄存器值
      int64_t last_update_ns;      // 时间戳（延迟计算时使用）
  };
```

3: 状态转换 → read/write 回调

  每次 MMIO 访问触发 → 读取/更新内部状态 → 决定 IRQ 电平 → 驱动输出引脚

---

## IRQ 连接：两种机制

QEMU 中有**两种 IRQ 连接机制**，分别用于不同场景：

```
          sysbus_connect_irq              qdev_connect_gpio_out
 设备 ─────────────────────── PLIC ─────────────────────── CPU
      n = 中断源编号                    irq = IRQ_S_EXT/M_EXT

 外部设备 ──────────────────── GPIO ────────────────────── LED
  qdev_connect_gpio_out       qdev_connect_gpio_in
```

### 机制 1: sysbus_init_irq + sysbus_connect_irq

通过 SysBus 总线连接，用于**设备 → 中断控制器**：

```c
// 设备端 realize:
sysbus_init_irq(sbd, &s->irq);       // 注册 IRQ 输出（索引 0）

// Board 端:
sysbus_connect_irq(sbd, 0,            // 将设备的 IRQ[0]
    qdev_get_gpio_in(plic, 10));      // 连接到 PLIC 的 GPIO_IN[10]
```

**特点**：通过 SysBus 总线，按索引号匹配，一对一的设备→PLIC 连接。

### 机制 2: qdev_init_gpio_in/out + qdev_connect_gpio_out

**不经过任何总线**，板级直连，用于 PLIC → CPU、外设 ↔ 外设：

```c
// PLIC → CPU（实现板级中断通路）:
qdev_connect_gpio_out(plic_dev, hart_idx,
    qdev_get_gpio_in(DEVICE(cpu), IRQ_S_EXT));
//       ↑                    ↑
//  PLIC 的 GPIO_OUT[hart]     CPU 的 GPIO_IN[IRQ_S_EXT=9]

// 外设 → 外设（如按键→GPIO）:
qdev_connect_gpio_out(key_dev, 0,
    qdev_get_gpio_in(gpio_dev, 7));
```

**特点**：无总线介入，名称/索引匹配，多用于芯片内部引脚对接。

### 完整中断路径（以 GPIO 为例）

```
IRQ 路径:
GPIO.sysbus_irq[0] ──sysbus_connect_irq──→ PLIC.qdev_gpio_in[2]
                                              │
                                         qdev_connect_gpio_out
                                              │
                                              ▼
                                      CPU.qdev_gpio_in[IRQ_S_EXT=9]
                                              │
                                         riscv_cpu_set_irq
                                              │
                                         env->mip.SEIP = 1 → trap

数据路径（对比）:
GPIO.output[0] ──qdev_connect_gpio_out──→ LED.qdev_gpio_in[0]
```

---

## 外设建模通用范式

综上，我们可以对任何外设进行建模

1. 使用 OOP 建模外设
2. 使用 IRQ 拓扑连接

!!! tips inline "芯片引脚+状态机实现"

    - 寄存器
    - 状态变化
    - IRQ 输出          
    - GPIO 输出         
    - GPIO 输入         

!!! tips "QEMU 实现"

    - uint32_t
    - memory_region_init_io → read/write 回调
    - sysbus_init_irq + sysbus_connect_irq → PLIC
    - qdev_init_gpio_out → 驱动外部设备
    - qdev_init_gpio_in → 外部设备驱动

---

## 应用范式：从简单到复杂

### GPIO — 纯寄存器设备

```javascript
class g233_gpio: sys_bus_device{
    static ngpio = 32;
    struct G233GPIOState {
        SysBusDevice parent_obj;
        MemoryRegion mmio;
        qemu_irq irq;                    // → PLIC
        qemu_irq output[32];             // → 外部设备
        uint32_t dir, out, in, ext_in;   // 寄存器
        uint32_t ie, is, trig, pol;      // 中断配置
    }
    g233_gpio_read(offset)  { /* 返回寄存器值 */ }
    g233_gpio_write(offset) { /* 更新寄存器 + update() */ } 
    g233_gpio_update() {
        new_in = (dir & out) | (~dir & ext_in);
        // 检测跳变 → 更新 IS → 更新 IRQ
    }
}
g233_gpio_create(addr, plic_irq) {
    dev = qdev_new(TYPE_G233_GPIO);
    sysbus_realize_and_unref(sbd, &error_fatal);
    sysbus_mmio_map(sbd, 0, addr);
    sysbus_connect_irq(sbd, 0, plic_irq);    // IRQ → PLIC
}
```

**编程模型**：组合逻辑——每次 DIR/OUT/ext_in 变化时重新计算 IN、IS、IRQ。

### PWM — 延迟计算设备

**不同点**：计数器依赖虚拟时间推进，采用**延迟追赶（lazy catchup）**——不是实时递增计数器，而是在每次 MMIO 访问时按已消逝时间一次性计算出当前值。

```javascript
class g233_pwm: sys_bus_device{
    struct G233PWMState {
        SysBusDevice parent_obj;
        MemoryRegion mmio;
        qemu_irq irq;                     // DONE + INTIE → PLIC
        qemu_irq output[4];              // 方波输出 → 外部
        uint32_t glb;                    // CH_EN | CH_DONE
        struct { ctrl, period, duty, cnt, last_update_ns } ch[4];
    }
    g233_pwm_catchup() {
        elapsed = now - ch[i].last_update_ns;
        wraps = (cnt + elapsed) / period;
        if (wraps > 0) glb.DONE = 1;
        cnt = (cnt + elapsed) % period;
        ch[i].last_update_ns = now;
    }
    g233_pwm_read(offset) { catchup(); return reg; }
    g233_pwm_write(offset) { catchup(); save_reg(); }
}
```

**核心约束**：所有对"时间"敏感的寄存器访问必须先 catchup，所有会改变计数器状态的操作必须重置 `last_update_ns`。

### WDT — 倒计时设备

```javascript
class g233_wdt: sys_bus_device{
    struct G233WDTState {
        uint32_t ctrl;       // EN, INTEN, RSTEN
        uint32_t load;       // 倒计时初值
        uint32_t sr;         // TIMEOUT (w1c)
        int64_t last_update_ns;
        bool locked;
    }
    g233_wdt_catchup() {
        elapsed = now - last_update_ns;
        if (elapsed >= load) sr.TIMEOUT = 1;
        if (INTEN) IRQ↑;
        if (RSTEN) qemu_system_reset_request();
    }
    g233_wdt_write_KEY(value) {
        if (value == 0x5A5A5A5A) feed();    // 重置计时 + 清 TIMEOUT
        if (value == 0x1ACCE551) locked++;  // 锁定 CTRL
    }
}
```

**不同点**：写时保护（LOCK 后 CTRL/LOAD 只读）、复位输出（RSTEN=1 时超时触发系统复位）、倒计时方向。

---

## SPI — 总线桥接器

SPI 与之前所有设备**本质不同**——它不是终端设备，而是一个**总线桥接器**：

```
之前:        GPIO/PWM/WDT
             CPU ←MMIO→ [设备] ←IRQ→ PLIC
             单一交互对象: CPU

SPI:         CPU ←MMIO→ [SPI 主控] ←IRQ→ PLIC
                            │
                        SSI 总线
                        ├── W25X16 (CS0)
                        └── W25X32 (CS1)
             双重交互对象: CPU + Flash
```

### 特殊性体现在代码中

GPIO/PWM/WDT 都是自包含的，一行 `xxx_create` 就完成：

```c
// 终端设备 — 只管自己
g233_gpio_create(addr, plic_irq);
g233_pwm_create(addr, plic_irq);
g233_wdt_create(addr, plic_irq);
```

SPI 需要挂在两个 flash 芯片:

```c
// SPI 主控 — 返回 SSI 总线句柄
spi_dev = g233_spi_create(addr, plic_irq, &spi_bus);

// 在 SSI 总线上挂载从设备
flash = qdev_new("w25x16");
qdev_prop_set_uint8(flash, "cs", 0);          // CS0
ssi_realize_and_unref(flash, spi_bus, &error_fatal);

flash = qdev_new("w25x32");
qdev_prop_set_uint8(flash, "cs", 1);          // CS1
ssi_realize_and_unref(flash, spi_bus, &error_fatal);

// CS 信号: SPI 主控的 GPIO 输出 → Flash 的 GPIO 输入
qdev_connect_gpio_out(spi_dev, 0,
    qdev_get_gpio_in_named(flash0_cs_dev, SSI_GPIO_CS, 0));
qdev_connect_gpio_out(spi_dev, 1,
    qdev_get_gpio_in_named(flash1_cs_dev, SSI_GPIO_CS, 0));
```

### SPI 传输的数据流

```
CPU 写 DR(tx) ──→ 主控保存 tx_byte, 启动 timer(10μs)
                     │
                 timer fires
                     │
                     └── ssi_transfer(tx_byte) ──→ SSI 总线广播
                                                           │
                                            每个从设备检查自己的 CS:
                                            CS 选通 → 处理字节 → 返回数据
                                            CS 去选通 → 忽略 → 返回 0
                                                           │
                     ←── 接收 rx_byte ←───────────────────┘
                     SR.RXNE=1, SR.TXE=1

CPU 读 DR ←── 返回 rx_byte, SR.RXNE=0
```

**SSI 总线是广播的**——所有从设备同时收到同一字节，但只有 CS 被拉低的设备才响应。去选通设备的 `transfer_raw` 返回 0，不参与 MISO 驱动。

### 总线桥接器的编程模型

```
SPI 主控的内部职责:
┌─────────────────────────────────────────┐
│  G233_SPI                                │
│                                         │
│  MMIO 层: CR1/CR2/SR/DR ←── CPU         │
│      │                                   │
│  控制层:                                  │
│  ├─ g233_spi_update_cs:                  │
│  │   写 CR2 → 全部释放 CS → 选通目标     │
│  │   写 CR1.SPE=0 → 全部释放             │
│  │                                         │
│  └─ g233_spi_transfer_done (timer):      │
│       ssi_transfer → 更新 SR/rx_byte     │
│       OVERRUN 检测                       │
│                                         │
│  总线层: SSI Bus ←── 挂载 Flash (CS0, CS1)|          
│   GPIO CS 引脚 ←── 选通/去选通            │
└─────────────────────────────────────────┘
```

这也解释了为什么 SPI 的实现比其他设备多一个 `realize` 函数：

```c
// 其他设备: instance_init 就够了
static void g233_gpio_init(Object *obj) { ... }     // 无 realize

// SPI: 需要 realize 来创建 SSI 总线
static void g233_spi_init(Object *obj) { ... }
static void g233_spi_realize(DeviceState *dev, Error **errp) {
    s->ssi_bus = ssi_create_bus(dev, "ssi");          // ← 总线在 realize 时才创建
    s->transfer_timer = timer_new_ns(...);             // ← 定时器需要已 realize 的上下文
}
```

因为总线是设备与设备之间的**关系**，关系不能在 `instance_init`（构造函数）中建立——那时对象还没完全初始化。必须在 `realize`（激活）中创建总线，让从设备可以接上来。

---

## 总结

```
从芯片规格到 QEMU 代码的映射:
───────────────────────────

1. 芯片引脚
   ├── MMIO 寄存器 → memory_region_init_io + sysbus_init_mmio
   ├── IRQ 输出    → sysbus_init_irq + sysbus_connect_irq → PLIC
   └── GPIO 引脚   → qdev_init_gpio_in/out → 板级连线

2. 内部寄存器
   → C 结构体字段（uint32_t 映射每一位）

3. 状态转换
   → read/write 回调（每次 MMIO 访问触发）

4. 中断触发条件
   → g233_xxx_update_irq() → qemu_set_irq(s->irq, level)

5. 时间建模
   → 延迟追赶: last_update_ns + elapsed → 当前值

6. 总线互联（仅 SPI）
   → realize 中创建 SSI Bus
   → Board 中挂载从设备
   → GPIO 引脚连接 CS
```
