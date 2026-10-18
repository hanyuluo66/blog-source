---
title: "10.18学习周报"
date: 2026-10-18
draft: false
slug: "10.18-weekly-report"
tags:
    - Go语言
    - 刷机
    - 学习周报
categories:
    - 学习
summary: "本周两件事：一是搞明白了小米10S（thyme）免解BL通过9008降级MIUI14的完整原理与流程，重点弄清了免授权引导和防回滚机制；二是正式入门Go语言，学完了环境搭建、变量与零值、常量与iota、控制流、函数多返回值与defer。"
---

# 10.18 学习周报

> 本周两条线：
>
> - **底层折腾线**：小米 10S 免解 BL 降级 MIUI 14。从"照着教程点鼠标"到搞明白免授权 9008 引导、Sahara/Firehose 协议、防回滚机制到底在干什么
> - **新语言线**：Go 语言入门，从 C/C++ 切过来，学完了环境搭建、变量与零值、常量与 iota、控制流、函数多返回值与 defer
>
> 两件事看着不相关，但有个共同点：**都在逼我搞清楚"底层到底发生了什么"**，而不是只记住操作步骤。

---

## 一、小米 10S 免解 BL 降级 MIUI 14

### 1.1 为什么选 9008 而不是常规线刷

常规线刷降级的前提是**已解锁 Bootloader**，而小米的 BL 解锁有几个硬门槛：

- 需要小米账号**绑定设备并连续登录满 168 小时**（7 天）
- 解锁过程会**强制清除全部数据**
- 新机型政策收紧，很多机器已经申请不到解锁权限

而 **9008（EDL 模式）** 是芯片级的紧急下载模式，**直接和高通 PBL 通信，绕过 Bootloader 校验**，所以理论上不需要解锁 BL 就能刷写分区。

> **关键认知**：9008 刷写**不会改变 BL 锁状态**。刷完之后 BL 依然是锁定的，只是系统版本变了。这也是"免解 BL 降级"这个说法的由来。

### 1.2 设备基本信息核对

刷机前必须把这几项对上，错一个就可能变砖：

| 项目 | 内容 |
|------|------|
| 机型 | 小米 10S |
| **机型代号** | **thyme**（这个最容易搞错） |
| SoC | 骁龙 870（SM8250-AC） |
| 存储类型 | UFS 3.0 |
| 目标版本 | MIUI 14 稳定版 |

**⚠️ 代号踩坑提醒**：网上有些教程把小米 10S 的代号写成 `fuxi`，那是**小米 13** 的代号。同系列代号完全不同，务必分清：

| 机型 | 代号 |
|------|------|
| 小米 10 | `umi` |
| 小米 10 Pro | `cmi` |
| 小米 10 至尊纪念版 | `cas` |
| **小米 10S** | **`thyme`** |
| Redmi K30S | `apollo` |

**下错包的后果**：MiFlash 会报 `Missmatching image and device`，运气好直接拒绝刷写；运气不好刷进去就是**彻底报废，无法修复**。

### 1.3 免解 BL 降级的原理

这部分是我这周最想搞明白的。整条链路是这样的：

```
① 进入 9008（EDL）模式
        ↓
② Sahara 协议握手
   PBL 主动向电脑请求文件，工具把【免授权 Firehose 引导文件】送进芯片 SRAM 执行
        ↓
③ Firehose 接管
   设备进入具备【全分区读写权限】的状态
        ↓
④ 按 rawprogram0.xml 的扇区映射刷写各分区镜像
        ↓
⑤ patch0.xml 修正 GPT 分区表备份头与 CRC
        ↓
⑥ 重启，BL 仍锁定，但系统已是 MIUI 14
```

**两个协议分工要记清楚：**

- **Sahara** — 负责"把 Loader 送进去"。Loader 是加载到 **SRAM** 里执行的，不是写进存储，因为此时存储还不可信
- **Firehose** — Loader 跑起来后接管设备，提供 XML 指令接口，做**原始扇区级读写**

### 1.4 关于"免授权引导文件"

正常的 Firehose 引导文件是**需要高通签名校验**的，只有厂商和授权售后有。所谓"免授权引导"，是社区针对特定平台提取或制作的、能通过校验的引导文件。

**兼容性有明确边界：**

- **骁龙 888 及以前的机型**（870、865、855…）目前基本都有可用的免授权引导方案，小米 10S 属于这一档
- **骁龙 8 Gen 1 之后的新平台**校验更严，方案少且不稳定
- 引导文件扩展名按平台分：
  - `.mbn` — 老设备，eMMC 存储
  - `.elf` — UFS 存储设备（小米 10S 属于这类）
  - `.melf` — 更新的平台
- **不同芯片代之间互不兼容**，870 的引导文件刷不进 888

> **风险提示**：免授权引导文件属于第三方产物，来源渠道鱼龙混杂。光明网今年 5 月还专门发文提醒"秒解 BL 锁"的安全隐患——**不明来源的工具和固件包可能被植入后门**，刷机等于把设备底层完全交出去。所以：
>
> - 工具尽量从可信的开源项目或成熟玩机社区获取，**别用来路不明的"一键工具"**
> - 固件包必须来自小米官方发布的线刷包
> - 刷前备份数据，刷完不要在里面存敏感信息直到确认系统干净

### 1.5 进入 9008 模式

小米 10S 可用方法，按推荐度排序：

| 方法 | 操作 | 说明 |
|------|------|------|
| **按键组合法** | 完全关机 → 同时按住「音量+」和「音量-」→ 插入数据线连电脑 → 保持 5~12 秒 | 最省事，优先试这个 |
| **工程线法** | 用 9008 EDL 工程线，关机状态直接插电脑 | 免按键，成功率高，需要买线 |
| **ADB 命令法** | 系统能开机时执行 `adb reboot edl` | 需已开启 USB 调试 |
| **Fastboot 命令法** | `fastboot oem edl` | **多数国产新机已屏蔽此命令**，小米大概率无效 |
| **拆机短接法** | 拆机短接主板 EDL 测试点 | 最后手段，需要拆机能力 |

**成功标志**：设备管理器「端口 (COM 和 LPT)」下出现

```
Qualcomm HS-USB QDLoader 9008 (COMx)
```

对应的 USB 设备标识是 **VID:PID = 0x05C6:0x9008**。

**同时手机屏幕保持全黑 —— 这是正常的**，不是没反应。这周我就是在这里犹豫了一下差点去拔线。

### 1.6 完整操作流程

```
准备阶段
  ├─ 1. 核对机型代号是 thyme（不是 umi / cmi / fuxi）
  ├─ 2. 从小米官方渠道下载【thyme 对应】的 MIUI 14 线刷包
  ├─ 3. 解压到【纯英文、无空格】的路径，如 D:\rom\thyme_miui14
  ├─ 4. 安装高通 9008 驱动（Qualcomm HS-USB QDLoader 9008）
  ├─ 5. 安装 MiFlash 线刷工具
  ├─ 6. 关闭杀毒软件和 Windows Defender 实时防护
  ├─ 7. 手机电量充到 50% 以上
  ├─ 8. 备份全部数据（刷机会清空）
  └─ 9. 【重要】用 QFIL 的 QCN Backup 功能备份 NVRAM（含 IMEI）

刷机阶段
  ├─ 10. 进入 9008，确认设备管理器识别到 COM 端口
  ├─ 11. 打开 MiFlash，点 Refresh 确认设备被识别
  ├─ 12. 点「选择」，定位到解压后的刷机包目录（选到有刷机脚本的那一层）
  ├─ 13. 【先选好刷机脚本】再点刷机 ← 这步顺序很关键，见 1.7
  ├─ 14. 点击开始，全程不要碰数据线
  └─ 15. 等待进度条走完，设备自动重启

收尾阶段
  ├─ 16. 首次开机较慢（可能 3~10 分钟），耐心等
  ├─ 17. 开机后核对版本号是否为预期的 MIUI 14
  ├─ 18. 检查基带信号、IMEI 是否正常（拨号盘输入 *#06#）
  └─ 19. 恢复数据
```

### 1.7 MiFlash 三个脚本的区别（最容易选错）

刷机包里有几个 `.bat` 脚本，MiFlash 右下角可以选，含义完全不同：

| 脚本 | 用户数据 | 内置存储（照片等） | 适用场景 |
|------|---------|------------------|---------|
| `flash_all.bat` | **清除** | **清除** | 全清，最干净，救砖常用 |
| `flash_all_except_storage.bat` | **清除** | **保留** | **MiFlash 默认选项** |
| `flash_all_except_data_storage.bat` | 保留 | 保留 | 相当于卡刷效果 |
| `flash_all_lock.bat` | 清除 | 清除 | 全清 + 重新上锁 BL |

**⚠️ 一个真实的 MiFlash Bug**：

如果**刚打开 MiFlash 就去点选脚本**，程序可能仍然默认执行 `flash_all.bat`（全清），导致你以为保留了数据其实全没了。

**保险做法**：刷机前直接把不需要的脚本文件从目录里**删掉或改名**，只留下你要用的那一个，从根源上避免选错。

### 1.8 防回滚机制（降级最大的风险）

这是降级操作**必须搞清楚**的一点，和升级完全不同。

**什么是防回滚（Anti-Rollback Protection）**：

- Android 8.0 起引入的安全机制，小米从 MIUI 9.5.17.0 稳定版开始启用
- 芯片里有一个**不可逆的版本计数器**，每次刷入带更高版本号的固件，计数器就 +1
- 一旦计数器升高，**再刷低于该版本的固件会触发启动拒绝**
- 后果：**手机无法启动系统，且不能通过常规刷机方式恢复**，只能送售后用工程手段处理

**这意味着什么**：

- 降级不是"随便挑个旧版本刷"，**能降到哪个版本受当前防回滚计数器限制**
- 9008 刷写虽然绕过了 BL 校验，但**防回滚检查发生在更底层**，不一定能绕过
- 如果 MiFlash 报 `Antirollback check error`，就是撞上了这个机制

**遇到 `Antirollback check error` 怎么办**：

1. 先确认刷机包路径是**纯英文短路径**（路径含中文也可能触发这个误报）
2. 检查目标版本是否**低于当前防回滚版本**，如果是，这个版本就是降不下去的，换更高的目标版本
3. 网上有教程说可以改 `flash_all.bat` 注释掉防回滚校验行 —— **强烈不建议**，这等于强行绕过安全机制，失败就是硬砖
4. 真要硬降，只能找有工程权限的售后，个人操作风险太高

> **我的结论**：降级前**先查清当前设备的防回滚版本**，确认目标 MIUI 14 版本不低于它。这一步不做，后面全是白费。

### 1.9 常见失败原因对照

| 现象 | 可能原因 | 处理 |
|------|---------|------|
| 插上电脑完全没反应 | 数据线是充电线（无数据芯） | 换原装数据线 |
| 识别成 `QHSUSB_BULK` | 驱动未正确安装 | 见 1.10 驱动排查 |
| 设备管理器有黄叹号 | 驱动签名被系统拦截 | 禁用驱动强制签名后重装 |
| COM 端口号频繁跳变 | USB 供电不稳 | 换机箱**后置** USB 口，或加带供电的 Hub |
| 卡在 Sahara 阶段 | Firehose 引导文件与平台不匹配 | 换对应骁龙 870 的引导文件 |
| `Missmatching image and device` | **刷机包代号不对** | 确认是 `thyme` 而非 `umi`/`cmi`/`fuxi` |
| `Antirollback check error` | 目标版本低于防回滚版本 | 见 1.8，换更高目标版本 |
| Firehose 传输中途失败 | 线材/接口接触不良或供电不足 | 换线换口，别用延长线和前置面板 |
| patch 写入失败 | xml 与实际分区表不匹配 | 确认刷机包完整、版本对应机型 |
| 刷完不开机 | 刷错包，或触发防回滚 | 重刷正确包；仍不行考虑硬件问题 |
| 刷完无信号 / IMEI 丢失 | 未备份 NVRAM 就全清刷写 | 只能找售后写回 QCN |

### 1.10 驱动装不上的排查顺序

设备管理器显示 `QHSUSB_BULK` 或黄色感叹号，按这个顺序来：

1. **驱动签名问题**（最常见）— Windows 10/11 默认强制驱动签名
   - 走「设置 → 恢复 → 高级启动 → 启动设置 → 重启 → 禁用驱动程序强制签名」
   - 或命令行 `bcdedit /set testsigning on` 后重启
   - **刷完记得 `bcdedit /set testsigning off` 关回去**，长期开测试模式有安全隐患
2. **杀毒软件拦截** — 安装前退出，装完再开
3. **驱动版本与平台不匹配** — 新平台需要新版驱动
4. **安装路径含中文或空格** — MiFlash/QFIL 对此敏感，路径用纯英文无空格
5. **残留驱动冲突** — 用 USBDeview 清理旧驱动后重装

### 1.11 命令行方案：edl

除了 MiFlash 图形工具，还有开源的 Python 工具 `edl`，好处是**流程透明、可脚本化、日志清晰**：

```bash
# 安装
pip install edlclient

# 先读取分区表，确认握手正常（这一步能验证引导文件对不对）
edl printgpt --memory=ufs

# 备份整个存储（降级前先备份，很重要）
edl rf dump.bin

# 刷写单个分区
edl w boot boot.img

# 按 xml 批量刷写
edl xml rawprogram0.xml
```

Linux 下用高通官方 QPST 组件的对应流程：

```bash
# 第一步：Sahara 阶段，把 Firehose loader 送进设备
QSaharaServer -v 2 -p /dev/ttyUSB0 -s 13:prog_firehose_ddr.elf
#                    ↑ 设备节点        ↑ Firehose 默认 img_id 是 13

# 第二步：Firehose 阶段，按 xml 刷写
fh_loader --port=/dev/ttyUSB0 \
          --sendxml=rawprogram_unsparse0.xml \
          --search_path=${VERSION_PATH} \
          --noprompt --showpercentagecomplete \
          --memoryname=ufs
```

### 1.12 关键要点

- 9008 = EDL 模式，**直接与芯片 PBL 通信**，绕过 BL 校验，这是免解锁刷机的根本原因
- **9008 刷写不改变 BL 锁状态**，刷完依然是锁定状态
- 通信分两层：**Sahara**（送 Loader 进 SRAM）→ **Firehose**（拿全分区读写权限）
- 小米 10S 代号是 **`thyme`**，不是 `fuxi`（那是小米 13），也不是 `umi`（小米 10）
- 骁龙 870 属于"888 及以前"这一档，**免授权引导方案相对成熟**
- **防回滚是降级最大的拦路虎**，动手前必须查清当前计数器版本
- MiFlash 选脚本要**先选后刷**，保险起见直接删掉不用的脚本文件
- **刷前备份 NVRAM/QCN**，否则可能丢 IMEI 导致无信号
- 手机黑屏不代表失败，**以设备管理器的端口识别为准**
- 刷机过程中**绝不能断电、断线**，这是唯一可能真正搞坏设备的操作

> **一点体会**：9008 刷机本质上是在操作一个"没有文件系统、只有裸扇区"的设备。理解了这点，就明白为什么 xml 里全是扇区偏移量，也明白为什么 patch 阶段要修 GPT 备份头——因为主分区表被覆盖后，尾部那份备份已经对不上了。
>
> 另一层体会是：**降级比升级危险得多**。升级是顺着安全机制走，降级是在逆着走，防回滚、签名校验、版本匹配每一关都可能把机器卡死。这类操作"能做"和"该做"是两回事，动手前一定要想清楚有没有必要。

---

## 二、Go 语言入门

本周 Go 学到了**函数与 defer** 这一块，后面的数组/切片/映射、结构体、接口、并发都还没开始。

### 2.1 环境搭建与第一个程序

**安装**（Windows）：

1. 从 https://go.dev/dl/ 下载 `.msi` 安装包，一路下一步
2. 安装完 `go` 命令自动进 PATH
3. 验证：

```bash
go version
# go version go1.23.x windows/amd64

go env GOPATH GOROOT GOPROXY
```

**国内网络建议配置模块代理**（否则 `go get` 经常超时）：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn
```

**第一个程序**：

```go
// hello.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

```bash
go run hello.go      # 直接编译并运行（不产出可执行文件）
go build hello.go    # 编译成 hello.exe
```

**项目初始化**（Go Modules，现代 Go 的标准做法）：

```bash
mkdir myapp && cd myapp
go mod init myapp        # 生成 go.mod
go run .                 # 运行当前包
go mod tidy              # 自动整理依赖（补缺失的、删多余的）
```

> **对比 C/C++**：Go 没有 `.h` 头文件，也没有 CMake/Makefile 那一套。`go mod` 直接管依赖，`go build` 一条命令出可执行文件，编译速度快到不像话——这是我从 C++ 切过来最直观的感受。

### 2.2 变量与基本类型

```go
package main

import "fmt"

func main() {
    // 方式1：var 显式声明类型
    var a int = 10
    var b float64 = 3.14
    var c string = "hello"
    var d bool = true

    // 方式2：var 类型推导
    var e = 20          // 自动推断为 int
    var f = 2.5         // 自动推断为 float64

    // 方式3：短变量声明 := （只能在函数内部用，最常用）
    g := 30
    h := "world"

    // 批量声明
    var (
        name string = "张三"
        age  int    = 20
    )

    // 多变量同时赋值
    x, y := 1, 2

    fmt.Println(a, b, c, d, e, f, g, h)
    fmt.Println(name, age, x, y)
}
```

**零值机制**（和 C 的重大区别）：

```go
var i int        // 0        ← C 里是随机值，Go 保证是零值
var s string     // ""       ← 空字符串，不是 NULL
var fl float64   // 0
var bo bool      // false
var p *int       // nil
```

> **这点太舒服了**。C 里 `int arr[10];` 之后内容全是垃圾值，必须自己 memset。Go 里所有类型都有明确的零值，而且**零值本身是可用的**——空字符串能直接拼接和比较、nil 指针能直接判等。这是 Go 很重要的设计哲学：**让类型的零值就是合理的初始状态**。

**基本类型速查**：

| 分类 | 类型 |
|------|------|
| 整数 | `int` `int8` `int16` `int32` `int64` `uint` `uint8`(=`byte`) `uint16` `uint32` `uint64` `uintptr` |
| 浮点 | `float32` `float64`（默认） |
| 其他 | `string` `bool` `rune`(=`int32`，一个 Unicode 码点) `complex64` `complex128` |

**⚠️ 不同数值类型之间不会隐式转换**：

```go
var i int = 10
var f float64 = 3.14

// var sum = i + f          // 编译错误！mismatched types int and float64
var sum = float64(i) + f     // 必须显式转换
fmt.Println(sum)             // 13.14
```

这点比 C 严格得多。C 里 `int + double` 会悄悄提升成 double，容易埋精度 bug；Go 直接编译不过，**把问题暴露在编译期而不是运行期**。

> 另外注意：Go 的 `int` 大小是**平台相关**的（64 位机上是 64 位），不像 C 里 `int` 通常是 32 位。要精确控制位宽就用 `int32` / `int64`。

### 2.3 常量与 iota

```go
package main

import "fmt"

const Pi = 3.14159

// iota：常量计数器，在 const 块中从 0 开始，每行自动 +1
const (
    Sunday = iota    // 0
    Monday           // 1
    Tuesday          // 2
    Wednesday        // 3
)

// 用法1：位移生成 2 的幂
const (
    KB = 1 << (10 * (iota + 1))   // 1 << 10 = 1024
    MB                            // 1 << 20 = 1048576
    GB                            // 1 << 30
)

// 用法2：用 _ 跳过某个值
const (
    A = iota   // 0
    _          // 1，跳过不用
    C          // 2
)

func main() {
    fmt.Println(Pi, Sunday, Monday, Tuesday, Wednesday)
    fmt.Println(KB, MB, GB)
    fmt.Println(A, C)
}
```

> `iota` 可以看作 C 里 `enum` 的替代品，但更灵活——它能参与表达式，配合位移能一行生成一整套单位常量。而且 `iota` **只在当前 const 块内计数**，换一个新的 `const ()` 块会从 0 重新开始。

### 2.4 控制流：Go 只有 for 一种循环

**if 语句**（条件不加括号，但大括号必须有）：

```go
score := 85

if score >= 90 {
    fmt.Println("优秀")
} else if score >= 60 {
    fmt.Println("及格")
} else {
    fmt.Println("不及格")
}

// if 可以带初始化语句，变量作用域限制在 if/else 块内
if n := len("hello"); n > 3 {
    fmt.Println("长度是", n)
}
// fmt.Println(n)   // 编译错误：n 在这里已不可见
```

> 这个"初始化 + 判断"的写法在处理错误时特别常用，能把临时变量限制在最小作用域里。

**for 循环**（Go 没有 `while`，全靠 `for` 一个关键字）：

```go
// 形式1：经典三段式
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// 形式2：当 while 用（只留条件）
n := 0
for n < 5 {
    n++
}

// 形式3：无限循环
for {
    break      // 只能靠 break 出去
}

// 形式4：range 遍历
nums := []int{10, 20, 30}
for index, value := range nums {
    fmt.Println(index, value)
}

// 只要值不要索引 → 用 _ 忽略
for _, v := range nums {
    fmt.Println(v)
}

// 只要索引
for i := range nums {
    fmt.Println(i)
}
```

**switch**（Go 的 switch **默认自带 break，不会穿透**）：

```go
day := 3

switch day {
case 1:
    fmt.Println("周一")
case 2:
    fmt.Println("周二")
case 3:
    fmt.Println("周三")
    fallthrough          // 必须显式写 fallthrough 才会继续执行下一个 case
case 4:
    fmt.Println("周四")
default:
    fmt.Println("其他")
}

// switch 可以不带表达式，当多重 if 用（比一长串 else if 清晰）
switch {
case score >= 90:
    fmt.Println("优秀")
case score >= 60:
    fmt.Println("及格")
default:
    fmt.Println("不及格")
}

// 一个 case 可以匹配多个值
switch day {
case 6, 7:
    fmt.Println("周末")
default:
    fmt.Println("工作日")
}
```

> **和 C 的重要区别**：C 的 switch 忘写 break 会一路穿透下去，这是经典 bug 来源。Go 反过来——默认不穿透，需要穿透必须显式写 `fallthrough`。这个默认值选得很好。

### 2.5 函数：多返回值是 Go 的灵魂

```go
package main

import (
    "errors"
    "fmt"
)

// 基本函数
func add(a int, b int) int {
    return a + b
}

// 参数类型相同可以合并写
func sub(a, b int) int {
    return a - b
}

// 多返回值（Go 最核心的特性之一）
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("除数不能为零")
    }
    return a / b, nil
}

// 命名返回值
func swap(a, b int) (x, y int) {
    x = b
    y = a
    return        // 裸 return，自动返回 x 和 y
}

// 可变参数
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// 函数作为值 / 作为参数
func apply(f func(int, int) int, a, b int) int {
    return f(a, b)
}

func main() {
    fmt.Println(add(3, 4))         // 7
    fmt.Println(sub(10, 3))        // 7

    // 多返回值的标准用法：判断 error
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Printf("结果: %.2f\n", result)
    }

    _, err2 := divide(10, 0)       // 用 _ 丢弃不需要的返回值
    fmt.Println("err2:", err2)

    x, y := swap(1, 2)
    fmt.Println(x, y)              // 2 1

    fmt.Println(sum(1, 2, 3, 4))   // 10

    // 匿名函数
    double := func(n int) int { return n * 2 }
    fmt.Println(double(5))         // 10

    // 函数作为参数传入
    fmt.Println(apply(add, 3, 4))  // 7
}
```

**defer 延迟执行**（非常重要，用来替代 C 里手动 `free` / `fclose`）：

```go
func readFile() error {
    f, err := os.Open("data.txt")
    if err != nil {
        return err
    }
    defer f.Close()    // 注册：函数返回前自动执行，无论从哪个分支 return

    // ... 一堆操作，中间随时可以 return，都不用担心忘记关闭 ...
    return nil
}

// 多个 defer 按【后进先出】顺序执行
func demo() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
// 输出：3 2 1
```

> **defer 解决了我上周在 C 里的一个痛点**。写 C 的时候，函数有多个 return 分支，每个分支前都得记着 `free()` / `fclose()`，漏一个就是资源泄漏。Go 里在获取资源后**立刻写 defer**，之后无论怎么 return 都保证会释放，心智负担小很多。

**defer 的三个细节**：

```go
// ① 参数在【注册时】就求值，不是执行时
func a() {
    i := 0
    defer fmt.Println(i)   // 这里就把 0 存下来了
    i = 100
}
// 输出 0，不是 100

// ② defer 作用域是整个函数，不是代码块
func b() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)   // 注册了 3 次，函数结束时才一起执行
    }
    fmt.Println("循环结束")
}
// 输出：循环结束 → 2 → 1 → 0

// ③ 能修改命名返回值
func c() (r int) {
    defer func() { r *= 2 }()   // 在 return 赋值之后、真正返回之前执行
    return 10
}
// 返回 20
```

### 2.6 Go 与 C/C++ 对照（已学部分）

| 方面 | C/C++ | Go |
|------|-------|-----|
| 内存管理 | `malloc`/`free`、`new`/`delete`，手动 | **GC 自动回收**，没有 free |
| 空值 | `NULL` / `nullptr` | `nil` |
| 字符串 | `char[]` + `strcpy`/`strcat`/`strcmp` | `string` 内置类型，直接 `+`、`==`、`len()` |
| 未初始化变量 | **随机垃圾值**，必须 memset | **确定的零值**，且零值可用 |
| 隐式类型转换 | 允许（易埋精度 bug） | **禁止**，必须显式转换 |
| 函数返回值 | 只能返回一个，多值靠指针传出 | **原生多返回值** `(result, error)` |
| 枚举 | `enum` | `const` + `iota`，可参与表达式 |
| 资源释放 | 每个 return 分支手动 free/close | **`defer`** 注册一次，自动执行 |
| switch 穿透 | 默认穿透，忘写 break 就出 bug | **默认不穿透**，需显式 `fallthrough` |
| 循环关键字 | `for` / `while` / `do-while` | **只有 `for`**，四种写法覆盖全部场景 |
| 头文件 / 构建 | `.h` + `.c`，CMake/Makefile | **无头文件**，`go mod` + `go build` |
| 编译速度 | 慢（C++ 模板尤其） | **极快**，大项目也是秒级 |
| 交叉编译 | 需要配工具链 | `GOOS=linux GOARCH=amd64 go build` 一行搞定 |
| 大括号位置 | 可以换行写 | **必须同行**，否则编译报错 |
| 声明不用的变量 | 只是警告 | **编译错误**，直接不过 |

### 2.7 关键要点（已学部分）

- `:=` 只能在**函数内部**用，包级变量必须用 `var`
- **零值可用**是 Go 的核心设计：`var s string` 是 `""` 而不是 NULL，不用像 C 那样先初始化
- 不同数值类型**不会隐式转换**，必须显式写 `float64(i)`；编译期报错总比运行期埋雷好
- Go 的 `int` 是**平台相关**的（64 位机上是 64 位），要精确控制位宽用 `int32` / `int64`
- `iota` 只在**当前 const 块内**计数，换新块从 0 重新开始
- `for` 是唯一的循环结构，四种写法覆盖 C 的 `for`/`while`/无限循环全部场景
- `range` 遍历时用 `_` 忽略不需要的索引或值
- switch **默认不穿透**，和 C 相反，需要穿透必须显式写 `fallthrough`
- **多返回值 + `error`** 是 Go 的错误处理范式，`if err != nil` 会成为肌肉记忆
- 可变参数 `nums ...int` 在函数内部可以直接用 `range` 遍历，调用时 `sum(1, 2, 3)` 传任意个
- `defer` 按**后进先出**执行，且**参数在注册时就求值**，这两点最容易搞错
- 裸 `return` 只能用在**命名返回值**的函数里

---

## 三、常见坑点整理

### 刷机相关

| 坑 | 现象 | 解决 |
|----|------|------|
| 用了充电线 | 插上电脑完全无反应 | 换原装数据线（要有数据芯） |
| 驱动装不上 | 设备管理器显示 `QHSUSB_BULK` 或黄叹号 | 禁用驱动强制签名 / 退出杀软 / 换纯英文路径 |
| **机型代号搞错** | `Missmatching image and device`，或刷完彻底报废 | 小米 10S 是 **`thyme`**，不是 `fuxi`（小米13）也不是 `umi`（小米10） |
| 跨机型复用引导文件 | 卡在 Sahara 阶段 | 引导文件必须严格匹配芯片平台 |
| **触发防回滚** | `Antirollback check error`，或刷完无法开机且常规手段救不回 | 降级前先查清防回滚版本，别强改脚本绕过 |
| MiFlash 脚本选错 | 以为保留了数据，实际全清了 | **先选脚本再点刷机**；保险做法是删掉不用的 .bat |
| 忘记备份 NVRAM | 刷完无信号、IMEI 丢失 | 刷前用 QFIL 的 QCN Backup 功能备份 |
| 刷机中途拔线 | 设备彻底无响应 | 全程别碰线，用机箱后置 USB 口保证供电 |
| 以为黑屏是失败 | 反复插拔把状态搞乱 | 手机黑屏是正常的，只看设备管理器端口 |
| 固件包路径含中文 | MiFlash 加载失败或误报 | 解压到纯英文无空格路径 |
| 用了不明来源的"一键工具" | 可能被植入后门 | 工具走可信开源项目，固件只用官方线刷包 |

### Go 相关

| 坑 | 现象 | 解决 |
|----|------|------|
| 声明了变量不用 | 编译错误 `declared and not used` | C 里只是警告，Go 是硬错误；用 `_` 接收或直接删掉 |
| 导入了包不用 | 编译错误 `imported and not used` | 删掉 import；只想执行 init 就写 `_ "pkg"` |
| `:=` 左边全是已存在的变量 | 编译错误 `no new variables on left side` | 已声明过的用 `=`，或保证左边至少有一个新变量 |
| 大括号换行写 | 编译错误 `syntax error: unexpected semicolon` | Go 的自动分号插入机制要求 `{` 必须和语句**同一行** |
| 不同数值类型直接运算 | 编译错误 `mismatched types` | 显式转换 `float64(i)` |
| 以为 int 是 32 位 | 大数计算结果和预期不符 | Go 的 `int` 平台相关（64位机上是64位），要精确控制用 `int32`/`int64` |
| `:=` 用在包级变量 | 编译错误 | 包级只能用 `var` |
| 以为 switch 会像 C 一样穿透 | 逻辑漏执行 | Go 默认不穿透，需要时显式写 `fallthrough` |
| 裸 return 用在非命名返回值函数 | 编译错误 `too few return values` | 只有 `func f() (x int)` 这种命名返回值才能裸 return |
| 以为 defer 的参数在执行时才求值 | 打印出来是旧值 | **注册时就求值**；要延迟取值用闭包 `defer func(){ fmt.Println(i) }()` |
| 在循环里 defer | 资源到函数结束才释放，可能耗尽文件句柄 | 把循环体抽成独立函数，让 defer 在每次调用结束时执行 |
| `errors.New` 忘了 import | 编译错误 undefined | 需要 `import "errors"`；或用 `fmt.Errorf("...")` |

---

## 四、本周学习总结

| 知识点 | 掌握程度 | 备注 |
|--------|---------|------|
| 9008 / EDL 原理 | 熟练 | 搞懂了 Sahara → Firehose 两层协议的分工 |
| 免解 BL 降级流程 | 了解 | 完整流程和风险点清楚了，实操还需谨慎 |
| 防回滚机制 | 了解 | 理解了为什么降级比升级危险得多 |
| MiFlash 使用 | 基本掌握 | 四个脚本的区别和选择顺序搞清了 |
| 命令行 edl 工具 | 了解 | 知道原理，还没实际上手 |
| Go 环境搭建与 go mod | 熟练 | GOPROXY 配好了，编译运行顺畅 |
| Go 变量与零值 | 熟练 | 零值可用的设计很省事 |
| Go 常量与 iota | 熟练 | 位移生成单位常量那套写法很实用 |
| Go 控制流 | 熟练 | 只有 for 一种循环，switch 默认不穿透 |
| Go 函数与 defer | 熟练 | 多返回值 + defer，比 C 里手动管理舒服太多 |

### 本周最大的收获

**1. 明白了"降级"和"升级"完全不是一个风险等级**

升级是顺着系统的安全机制走，降级是在逆着走。防回滚计数器、签名校验、版本匹配，每一关都可能把机器卡死。尤其是防回滚——**那个计数器是不可逆的**，一旦升高就回不去了，触发之后连常规刷机都救不回来。

所以这类操作"能做"和"该做"是两回事。动手前必须先想清楚：真的有必要降吗？降下去能不能升回来？数据备份了吗？

**2. 机型代号这种细节，错了就是废一台机器**

小米 10S 是 `thyme`，但网上有些教程写成 `fuxi`（那其实是小米 13）。同系列里 `umi`、`cmi`、`cas`、`thyme`、`apollo` 五个代号长得毫无规律，靠猜必错。

这件事让我意识到：**刷机这类不可逆操作，所有参数都要亲自核对到官方来源，不能信二手教程**。轻则报 `Missmatching image and device` 拒绝刷写，重则直接刷进去把机器搞报废。

**3. Go 让我重新审视了前面学 C/C++ 的价值**

从 C/C++ 切到 Go，最大的感受是"很多事不用我管了"——没有 `free`、没有头文件、没有隐式类型转换的坑、变量保证有零值。

但反过来说，**正是因为前面啃过内存四区和手动内存管理，才能体会到 Go 这些设计到底省了多少事**。比如 C 里 `int arr[10];` 之后内容全是垃圾值，必须自己 memset；Go 的零值机制直接把这类 bug 从根上消灭了。

再比如 `defer`。上周写 C 的文件操作时，函数有好几个 return 分支，每个分支前都得记着 `fclose()`，漏一个就是资源泄漏。Go 里获取资源后立刻写一行 `defer f.Close()`，之后怎么 return 都不怕——**这种设计上的安心感，只有被坑过的人才懂**。

还有一点体会：Go 的编译器比 C 严格得多。变量声明了不用、包导入了不用，在 C 里最多是个警告，在 Go 里直接编译不过。一开始觉得烦，写完几个小程序后发现——**它逼着我删掉所有无用代码，最后留下来的都是真正需要的**。

> 所以前几周啃 C 语言的苦没白吃。语法可以速成，但**对内存和底层的理解是要靠踩过坑才有的**。

### 下周计划

- **Go 继续往后学**：
  - 数组、切片与映射（**重点**，切片的"指针+len+cap"底层结构必须搞懂，`append` 的扩容机制是高频坑点）
  - 结构体与方法（值接收者 vs 指针接收者）
  - 指针（和 C 指针对比，理解 Go 为什么没有指针运算）
  - 接口（Go 的隐式实现机制）
  - 错误处理进阶（`error` 接口、`errors.Is` / `errors.As`、错误包装 `%w`）
  - goroutine 与 channel（Go 的招牌特性，重点）
- **Go 实战**：把学的东西串起来，写一个命令行小工具（比如待办清单），练结构体 + 切片 + map 的组合使用
- **刷机知识补齐**：
  - 搞清楚怎么用 `edl printgpt` 读取分区表，理解分区布局
  - 学习如何做全分区备份，这是玩机最重要的保险
  - 了解防回滚版本号具体怎么查
- 继续用 Hugo 记录，保持每周一篇

---

> 本周 Go 代码均在本地编译运行验证。刷机部分为原理学习与流程整理，涉及不可逆操作，实操前请务必核对机型代号、目标版本与防回滚限制，并做好完整备份。
