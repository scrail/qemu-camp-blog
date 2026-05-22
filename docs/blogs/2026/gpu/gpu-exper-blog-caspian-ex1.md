# GPU 进阶

!!! note "主要贡献者"

    - 作者：[@caspian](https://github.com/trace1729)

---

> 背景介绍 & 专业阶段 见 CPU 方向

在进阶实验一中，我基于 arceos 来做软件栈的搭建。

## 引入

从 qtest 驱动测试到使用软件栈运行 gpu demo, 他们的差异在哪里，软件栈的概念是什么，为什么我们需要？

- qtest 方式：test 程序通过 Unix socket 直接对 QEMU 设备模型做 MMIO 读写和 VRAM 操作，不经过任何操作系统或驱动——test 自己就是"驱动"，手动填寄存器、手动管理 VRAM 地址。
- 软件栈方式：应用程序跑在 arceos 上，经过完整的软件层次：

  > 软件栈的概念：层层抽象，每层隐藏下层细节。应用不知道 PCI 配置空间、不知道 BAR 地址、不知道寄存器偏移——它只调用 gpgpu_launch_kernel()。
  >

```
app (gpgpu-demo)
       ↓ 调用
     libgpu (gpgpu_ops)      ← 提供gpgpu初始化、算子封装
       ↓ 调用
     driver (axgpgpu)        ← PCI BAR 映射、VRAM 分配、参数打包
       ↓ 调用
     bus/device (axdriver)   ← PCI 枚举、设备发现 （OS）
       ↓ 调用
     hal/runtime (axhal)     ← 平台初始化、地址空间、中断 (OS)
       ↓ MMIO/DMA
     QEMU GPGPU 设备模型
```

为 gpgpu 构建软件栈，整体上分为四个部分

- 首先是为 Qemu 模拟的 gpgpu 编程模型完善指令、添加分支发散的功能；(How to immitate gpu with software emulation)
- 其次是为 arceos 的 axdriver 注册 gpgpu 驱动；(how cpu comunicate with gpu)
- 然后为 arceos 增加 gpgpu 的运行时库；(config gpu in cpu)
- 最后是提供编程前端，实现一些算子 (program running on gpu)，然后用 rust 使用运行时库加载这些算子完成一个测试 demo：gpu-demo 完成三个算子的基本测试，lenent-demo 完成简单训练

## gpgpu 编程模型完善指令

> how to immitate gpu hardware in software, qemu

SIMT Stack 实现策略

初始想法：

```
1. SIMT stack 是 warp 内线程共享的
2. gpgpu_core_exec_warp 的实现中，如果是分支指令
3. 判断该分支指令是否会造成 divergence
    1. 如果不造成 divergence, 正常执行
    2. 如果造成 divergence,
    3. 设置 then_mask 和 if mask, 和聚合点 （后向跳转设置then, 前向跳转设置else）
    4. 如何在divergence 再次遇到 divergence, 那么就压栈，递归的执行下去
```

- 不足：不能确定嵌套分析的聚合点
  - 需要建立 CFG 图来分析聚合点

整体流程：

```
gpgpu_core_exec_kernel()
  ├─ gpgpu_core_build_cfg()    ← 一次 CFG 分析 (kernel 粒度)
  └─ for each warp:
       └─ gpgpu_core_exec_warp()
            ├─ 取指 → 执行 → 后检查汇合点
            ├─ 遇到发散: 查表确定汇合点 → 推栈 → 进 then 路径
            ├─ 到达汇合点: cascade pop 所有匹配 entry
            └─ ebreak → 完成
```

## Arceos 驱动适配

> how cpu comunicate with gpu

### 接入 arceos axdriver 体系

1. 实现 axdriver_base::BaseDriverOps trait，这样可以被接受为 device
2. 实现 DriverProbe trait trait, 实现 DriverProbe for GpgpuDriver
3. 加入条件编译：#[cfg(gpgpu_dev = "qemu-gpgpu")]，这样在 axdriver 在 pci probe 的时候会被自动展开
4. 更新配置文件
5. 为了支持 phys_to_virt() → 映射到虚拟地址，需要启用 arceos page 的特性

```
init_drivers()
       └─ AllDevices::probe()
            └─ probe_bus_devices()
                 ├─ PciRoot::new(MmioCam::new(ECAM_VA))
                 ├─ root.enumerate_bus(0) → for each (bdf, dev_info)
                 │    ├─ config_pci_device() → 配置 BAR、使能 PCI
                 │    └─ for_each_drivers!(type Driver, {
                 │         └─ GpgpuDriver::probe_pci()
                 │              ├─ match VID=0x1234, DID=0x1337
                 │              ├─ root.bar_info(bdf, 0) → BAR0 地址
                 │              ├─ root.bar_info(bdf, 2) → BAR2 地址
                 │              ├─ phys_to_virt() → 映射到虚拟地址
                 │              └─ AxDeviceEnum::from_gpgpu(GpgpuDevice::new(...))
                 │         })
                 │    └─ self.add_device(AxDeviceEnum::Gpgpu(dev))
                 │         └─ self.gpgpu.push(dev) → Option<GpgpuDevice>
                 └─ init 完成 → 应用取走设备
                 
     应用 main():
       └─ all_devices.gpgpu.take_one() → GpgpuDevice
            └─ upload_kernel().launch().wait().memcpy_d2h() → MMIO 读写 QEMU 设备
```

这个体系的关键设计是 `for_each_drivers!` 宏 + `build.rs cfg` 生成：新设备只需要定义一个 `Driver struct`+ 实现 `DriverProbe::probe_pci()`+ 在宏里加条件分支，就能自动接入 axdriver 的统一 PCI 发现流程。

### 封装 MMIO 读写操作

arceos 和 qemu 通过地址交互的方式如下：

1. QEMU 规定硬件地址空间（ECAM 0x3000_0000、PCI MMIO 0x4000_0000 等），定义在 `hw/riscv/virt.c`
2. ArceOS 根据 `.axconfig.toml` 读取这些地址范围，开启分页（Sv39），通过 `map_linear()` 为 MMIO 区域建立 VA→PA 的页表映射 (qemu 的 mmu 提供底层支持)
3. 应用执行 `sw` 指令访问设备的虚拟地址（如 `write_volatile(bar0 + 0x330, 1)`，bar0 是映射后的 VA）
4. QEMU 模拟的 RISC-V MMU 遍历页表 (2 中注册)，将 VA 翻译为 PA；发现该 PA 属于 MMIO 区域（非 RAM），查 MemoryRegion 树，触发对应的设备回调

在 `init_drivers` 时，会为 gpgpu 的 bar0 和 bar2(vram) 创建虚拟地址映射。下面将对这两块区域的读写封装为 api  

| 方法 | 说明 | 底层操作 |
|---|---|---|
| `write_reg(off, val)` | 写控制寄存器 | `write_volatile(bar0 + off, val)` |
| `read_reg(off) -> u32` | 读控制寄存器 | `read_volatile(bar0 + off)` |
| `vram_write(off, val)` | 写 VRAM 字 | `write_volatile(vram + off, val)` |
| `vram_read(off) -> u32` | 读 VRAM 字 | `read_volatile(vram + off)` |

基于对 MMIO 读写，进一步封装得到对 gpgpu 的数据和控制两个层面的接口

### 数据搬运 API

`gpgpu_malloc & gpgpu_memcpy_* & gpgpu_free` 维护了一个线性映射的内存链表。

| 方法 | 签名 | 说明 |
|---|---|---|
| `gpgpu_upload_kernel` | `(&self, bin: &[u8]) -> KernelObj` | 将 kernel 二进制逐字写入 VRAM kernel 区域，返回句柄 |
| `gpgpu_malloc` | `(&mut self, size: usize) -> VramPtr` | 分配 VRAM 缓冲区（bump allocator + free-list，64 字节对齐） |
| `gpgpu_free` | `(&mut self, ptr: VramPtr)` | 归还 VRAM 到 free-list |
| `gpgpu_memcpy_h2d` | `(&self, dst: VramPtr, src: &[u8])` | 主机 → VRAM：逐字 `write_volatile(vram + offset, word)` |
| `gpgpu_memcpy_d2h` | `(&self, dst: &mut [u8], src: VramPtr)` | VRAM → 主机：逐字 `read_volatile(vram + offset)` |

### 执行控制 API

| 方法 | 签名 | 说明 |
|---|---|---|
| `gpgpu_launch_kernel` | `(&self, kernel: &KernelObj, grid: Dim3, block: Dim3, args: &[u32])` | 依次写入 11 个 MMIO 寄存器后触发 DISPATCH |
| `wait` | `(&self)` | 轮询 `GLOBAL_STATUS & BUSY` 直至清除 |

## 运行时库的编写

> config gpu in cpu

功能：在驱动层之上提供类型安全、RAII 风格的 GPGPU 编程接口，屏蔽裸指针和 MMIO 细节。

### `GpgpuContext`

包装已就绪的 `GpgpuDevice` 实例：

```rust
pub fn new(device: GpgpuDevice) -> Self;
```

应用通过 `axdriver::init_drivers()` 或 `GpgpuDevice::probe_from_ecam()` 获得设备后，传入 `new()` 即可。内部通过 `device()` / `device_mut()` 把驱动层 API 暴露给同库的其他模块。

### `GpuBuffer<T: Copy>` — 类型化 VRAM 缓冲区

RAII 风格，生命周期绑定到 `GpgpuContext`：

| 方法 | 签名 | 说明 |
|---|---|---|
| `new` | `(ctx: &mut GpgpuContext, len: usize) -> Self` | 分配未初始化的 VRAM 缓冲区 |
| `from_host` | `(ctx: &mut GpgpuContext, data: &[T]) -> Self` | 分配并拷贝主机数据到 VRAM |
| `readback` | `(&self, ctx: &GpgpuContext) -> Vec<T>` | 从 VRAM 读回主机内存 |
| `as_offset` | `(&self) -> u32` | VRAM 字节偏移（作为 kernel 参数传入） |
| `len` | `(&self) -> usize` | 元素数量 |

### `Kernel` — 预加载的 kernel 对象

```rust
pub fn new(ctx: &mut GpgpuContext, _name: &str, bin: &[u8],
           block: Dim3, arg_count: u32) -> Self;
```

- 内部调用 `device.gpgpu_upload_kernel(bin)` 将二进制写入 VRAM
- `block` 参数指定每个 block 的线程布局（例如 `Dim3::new(32, 1, 1)` 表示每 block 32 线程）
- `arg_count` 指定期望的参数数量

### `KernelLauncher` — Builder 模式启动器

```rust
pub fn new(ctx: &GpgpuContext, kernel: &Kernel) -> Self;
pub fn grid(self, x: u32, y: u32, z: u32) -> Self;
pub fn block(self, x: u32, y: u32, z: u32) -> Self;
pub fn arg(self, val: u32) -> Self;            // 添加参数，最多 8 个
pub fn launch(self);                            // 执行 launch() + wait()
```

`launch()` 内部：

1. 将参数写入 VRAM 固定偏移 `0xF0`
2. 写 kernel 地址、参数地址、grid/block 维度到 MMIO 寄存器
3. 写 `DISPATCH=1` 触发 QEMU 执行
4. 轮询 `wait()` 直到完成

### 高层算子 `libgpgpu::operators`

每个算子内部嵌入对应的 kernel `.bin`（通过 `include_bytes!`），封装完整的 alloc → H2D → launch → wait → D2H 流程：

```rust
pub fn vec_add(ctx: &mut GpgpuContext, a: &[i32], b: &[i32]) -> Vec<i32>;
pub fn relu(ctx: &mut GpgpuContext, input: &[i32]) -> Vec<i32>;
pub fn matmul_4x4(ctx: &mut GpgpuContext, a: &[i32;16], b: &[i32;16]) -> [i32;16];
pub fn conv2d_3x3(ctx: &mut GpgpuContext, input: &[i32], weight: &[i32],
                  bias: &[i32], in_h: u32, in_w: u32, out_c: u32) -> Vec<i32>;
pub fn max_pool_2x2(ctx: &mut GpgpuContext, input: &[i32],
                    h: u32, w: u32, c: u32) -> Vec<i32>;
pub fn fc_layer(ctx: &mut GpgpuContext, input: &[i32], weight: &[i32],
                bias: &[i32], in_dim: u32, out_dim: u32) -> Vec<i32>;
```

## 测试编程前端

> program running on gpu

V 汇编编写（`gpgpu-kernels/*.S`），用 `riscv64-*-gcc -march=rv32imf -mabi=ilp32` 交叉编译为纯指令 `.bin` 文件。

### 线程身份 API（通过 `mhartid` CSR 编码）

每个 lane 通过读取 `mhartid` CSR 确定自己在执行网格中的位置：

```asm
csrr t0, mhartid          # 完整线程 ID
andi t0, t0, 31           # bit[4:0] = lane_id (0-31)
srli t0, t0, 5
andi t0, t0, 0xFF         # bit[12:5] = warp_id
srli t0, t0, 8            # bit 13+ = block_id (0..n)
```

编码格式：`mhartid = (block_id << 13) | (warp_id << 5) | lane_id`

### 参数传递 API

CPU 侧调用 `launch()` 时将参数写入 VRAM 地址 `0xF0`；QEMU 在执行 warp 前从此地址读取最多 8 个 u32 填入 `a0-a7`（RISC-V x10-x17）：

```rust
// CPU 侧
KernelLauncher::new(&ctx, &kernel)
    .arg(in_buf.as_offset())     // → a0
    .arg(w_buf.as_offset())      // → a1
    .arg(bias_val as u32)        // → a2
    .arg(out_buf.as_offset())    // → a3
    .launch();

// GPU 侧汇编
lw   a0, 0(a0)    # a0 = in_buf.offset → 读取 VRAM 中的输入数据
```

典型参数布局：`[input_offset, weight_offset, bias_value, output_offset, dim0, dim1, ...]`

### 数据寻址

每个 lane 根据 `lane_id` 计算自己在缓冲区的偏移：

```asm
# 每个 lane 处理一个元素，步长为 4 字节
slli t1, lane_id, 2        # offset = lane_id * 4
add  t2, buf_base, t1      # 地址 = buf_base + offset
lw   t3, 0(t2)             # 读取值
```

### kernel 终止

```asm
ebreak                      # 指令编码 0x00100073
```

每个 warp 执行到 `ebreak` 后退出循环，QEMU 标记该 warp 完成。

### 编译流程

```
vec_add.S → riscv64-elf-gcc -march=rv32imf -mabi=ilp32 -nostdlib -O2 -c → vec_add.o
vec_add.o → riscv64-elf-objcopy -O binary -j .text → vec_add.bin
           → include_bytes!("../../../gpgpu-kernels/vec_add.bin")
```

## lenet-demo：完整流水线

lenet-demo 实现简化 LeNet-5 的推理：`6×6 输入 → Conv2D → ReLU → MaxPool → FC → argmax`。

### 启动阶段 — 设备发现

```
main()
  ├─ axdriver::init_drivers()
  │    → 扫 PCI ECAM：匹配 0x1234:0x1337
  │    → GpgpuDriver::probe_pci() 读 BAR 地址
  │    → GpgpuDevice::new() 映射 MMIO + 使能设备
  │    → 存入 AllDevices.gpgpu
  │
  ├─ all_devices.gpgpu.take_one()      // 取走设备
  │
  └─ GpgpuContext::new(gpu)            // 包装成运行时上下文
```

这一步把 QEMU 模拟的 PCI 设备变成代码里可操作的 `ctx` 对象。之后所有 GPU 操作都通过 `ctx` 完成。

### Conv2D 算子执行

`libgpgpu::operators::conv2d_3x3(&mut ctx, input, weight, bias, 6, 6, 2)`

函数内部串联三个运行时 API：

**第一步 — 数据准备**：调用 `GpuBuffer::from_host()` 把 `input`、`weight`、`bias` 从 `Vec<i32>` 拷入 VRAM；调用 `GpuBuffer::<i32>::new()` 在 VRAM 分配输出缓冲区。每个 buffer 内部完成 `gpgpu_malloc` + `gpgpu_memcpy_h2d`两步，返回的 `GpuBuffer` 持有 `VramPtr` 供后续寻址。

**第二步 — kernel 上传**：调用 `Kernel::new()` 把 `include_bytes!` 嵌入的 conv2d.bin 通过 `gpgpu_upload_kernel()` 写到 VRAM kernel 区域。此后的 Conv2D 算子可复用同一个 kernel 对象。

**第三步 — 启动**：用 `KernelLauncher` 的 builder 模式组装参数：`.arg(in_buf.as_offset())` 传入各 buffer 的 VRAM 偏移，`.arg(IN_H)`, `.arg(IN_W)` 传入维度。`.launch()` 内部写 MMIO 寄存器触发 QEMU 执行，然后 `wait()` 轮询完成。

QEMU 收到 DISPATCH 后：

- 对整个 grid/block 空间枚举 block 和 warp
- 每个 warp 分配一个 `GPGPUWarp`，共享 PC 和 `active_mask`
- 进入 `gpgpu_core_exec_warp()` 取指解码循环
- 遇到分支发散时用 SIMT stack 处理
- 执行到 `ebreak` 后 warp 退出
- 所有 warp 完成后清 BUSY 状态

**第四步 — 读回**：调用 `out_buf.readback(&ctx)` 通过 `gpgpu_memcpy_d2h` 把 VRAM 结果逐字拷回 `Vec<i32>`，作为函数返回值。

### ReLU / MaxPool / FC 算子

三个算子重复同样的模式：

```
operator(ctx, inputs...)
  ├─ Kernel::new(embed bin)    // 每个 operator 有自己的 kernel binary
  ├─ GpuBuffer::from_host()    // 输入数据
  ├─ GpuBuffer::<i32>::new()   // 输出
  ├─ KernelLauncher::arg().launch()
  └─ readback() → Vec<i32>
```

差异仅在于 kernel 二进制文件不同、参数数量和含义不同。

### 验证方法

GPU 跑之前，demo 先在同一份输入上用纯 Rust 的 CPU 参考实现算出期望结果：

```rust
let ref_conv1 = cpu_conv2d_3x3(&input, &conv1_w, &conv1_b, CONV1_CH);
let ref_relu1 = cpu_relu(&ref_conv1);
let ref_pool1 = cpu_maxpool_2x2(&ref_relu1, OUT_H, OUT_W, CONV1_CH);
let ref_fc1 = cpu_fc(&ref_pool1, &fc_w, &fc_b, FC_OUT);
```

GPU 每层的输出与对应的 CPU 结果逐值比较。四个算子全部一致才算流水线通过：

```
[OK] Conv2D: 32 values match CPU reference
[OK] ReLU: 32 values match
[OK] MaxPool: 8 values match
[OK] FC: output [40, 40, 40, 80]
[OK] Classification: class 3 (GPU) == class 3 (CPU)
```

### 调用关系总览

```
应用 (main)               运行时 (libgpgpu)          驱动 (axgpgpu)           QEMU 设备
─────────               ──────────────────         ──────────────           ──────────
init_drivers()
take_one()
new(gpu) ──────────→    GpgpuContext::new()
                                  │
conv2d_3x3(ctx, ...) ──→ Kernel::new() ──→ gpgpu_upload_kernel(bin) ──→ vram_write(逐字)
                          GpuBuffer::from_host() ──→ gpgpu_malloc()
                                                      gpgpu_memcpy_h2d() ──→ vram_write(逐字)
                         KernelLauncher::new()
                            .arg().arg().launch() ──→ gpgpu_launch_kernel() ──→ write_reg(×11)
                                                                ──→ write_reg(DISPATCH,1)
                                                     wait() ────→ read_reg(BUSY)→loop
                                                                            │
                                                                      gpgpu_core_exec_kernel()
                                                                        CFG 分析
                                                                        for each warp:
                                                                          exec_warp()
                                                                            SIMT stack
                                                                            ebreak
                          GpuBuffer::readback() ──→ gpgpu_memcpy_d2h() ──→ vram_read(逐字)
                         ↓ Vec<i32>
relu / max_pool / fc    重复相同模式

验证: GPU result == CPU reference → [OK]
```

## 总结

在这个过程中，为 gpu 增加 simt stack，以及配置 arceos 的运行环境花费了比较多时间。

- 为 gpu 增加 simt stack:低估了复杂性，让 AI 帮我实现，但是实现有很多漏洞。反思：复杂特性直接找权威参考资料，看懂了再让 AI 帮忙实现。
- 算法和概念定义原问题，原问题找得好，用 AI 帮忙扩展是事半功倍，如果啥也不明白就直接让 AI 写，那可能最终的结果是在一个丑陋的状态机上做修修补补

## 概念辨析

- qemu 设备模型对于 pcie 的模拟和 操作系统中对于 pcie 的支持有什么区别：Qemu 的设备模型是模拟硬件，arceos 中对于 pcie 的配置是为了使用硬件
  - Crate 和 module 区别：Crate 是一个独立的编译单元，一个 cargo.toml 定义一个 crate, 通常有一个 lib.rs 或者 main.rs 和 若干个 module，Module 是文件内的代码组织，不独立编译
- mmio 与 pcie 的关系，他们到底是独立存在，还是主从关系
  - MMIO 只是对于 CPU 来说的一种访存方式，CPU 可以通过 MMIO 的方式来访问 PCIE 总线，发出 MMIO transactrion.
