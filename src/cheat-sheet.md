# 学习速查表

这里收录正文已经解释的知识，用于复习和查找。操作前仍需阅读对应章节的前置条件、执行位置和恢复步骤。

## 第 0 章 从零搭建练习环境

### 环境分工与工具

| 名称 | 含义与用途 | 关键限制与正文 |
| --- | --- | --- |
| 宿主机 / host | 承载虚拟机，保存虚拟磁盘和作业 | 本章兼任开发机与调试主机；来宾还原不等于宿主还原。[0.1](chapter-00-environment.md#01-先分清宿主与目标) |
| 目标机 / target、来宾机 / guest | 被观察的 Windows 10 22H2 x64 系统 | 内核暂停时桌面也会停住，WinDbg 留在宿主。[0.1](chapter-00-environment.md#01-先分清宿主与目标) |
| 用户态 / user mode、内核态 / kernel mode | 普通应用与内核组件的不同执行模式 | 管理员账户权限不能等同于内核态，内核错误可能影响全系统。[0.1](chapter-00-environment.md#01-先分清宿主与目标) |
| x64、x86、ARM64 | 本章的 x64 是常见 Intel、AMD 64 位 x86 架构，安装器中的 x86 通常指 32 位组件 | ARM64 不属于本章两条宿主路线，选择工具与介质时核对架构。[0.2](chapter-00-environment.md#02-检查电脑与资源) |
| VM、vCPU | Virtual Machine，虚拟机；虚拟处理器 | 本章分配 2 vCPU、4 GB 内存和 64 GB 磁盘，属于教学预算。[0.2](chapter-00-environment.md#02-检查电脑与资源) |
| SLAT | Second Level Address Translation，二级地址转换 | Hyper-V 的硬件条件之一，不靠安装软件补齐。[0.2](chapter-00-environment.md#02-检查电脑与资源) |
| Visual Studio、MSVC | 开发环境与 Microsoft Visual C++ 编译工具 | 本章使用 VS 2022 17.14、v143 14.44 工具集；编译器使用 19.44 编号，实际维护版本另记。[0.3](chapter-00-environment.md#03-准备安装材料并固定版本) |
| SDK、WDK | Software Development Kit，软件开发工具包；Windows Driver Kit，Windows 驱动工具包 | 本章选择 10.0.26100.6584。build number 必须配对，目标版本与工具版本分别核对。[0.3](chapter-00-environment.md#03-准备安装材料并固定版本) |
| WDK VSIX | Visual Studio 的 WDK 扩展 | 模板集成与完整 WDK 文件分别检查，不能互相替代。[0.4](chapter-00-environment.md#04-安装并检查开发工具) |
| WinDbg (Classic) | Windows 调试器，本章使用 SDK 附带的 x64 版本 | 用实际文件路径区分不同 WinDbg 安装，记录版本。[0.10](chapter-00-environment.md#symbols) |
| `ntddk.h`、`ntoskrnl.lib` | WDK 的内核接口头文件与库文件 | 查看 `km` 路径只验证安装内容，不表示编译或运行过驱动。[0.4](chapter-00-environment.md#04-安装并检查开发工具) |
| `Windows.h`、`kernel32.lib` | SDK 的 Windows 接口头文件与用户态库文件 | 在对应版本的 `um` 目录检查，x64 库不能与其他架构混用。[0.4](chapter-00-environment.md#04-安装并检查开发工具) |

### 网络、调试通道与恢复

| 名称 | 含义与反作弊学习用途 | 关键限制与正文 |
| --- | --- | --- |
| UEFI、Secure Boot | Unified Extensible Firmware Interface，统一可扩展固件接口；安全启动检查启动信任 | 记录原值，只对实验目标做必要变更，不把关闭安全启动等同于改成 BIOS。[0.5](chapter-00-environment.md#05-hyper-v-路线) |
| Hyper-V 外部交换机 | 把来宾接到物理实验网络，允许宿主共享网卡 | 本章 KDNET 使用受信任、有地址分配服务的实验局域网。[0.5](chapter-00-environment.md#05-hyper-v-路线) |
| NAT、Host-only | Network Address Translation，网络地址转换；仅主机网络 | VMware 下载材料时使用 NAT，Host-only 默认不出网，串口调试不依赖互联网。[0.8](chapter-00-environment.md#08-配置网络与材料传递) |
| DHCP | Dynamic Host Configuration Protocol，动态主机配置协议 | 地址可能在还原或重启后变化，连接前重查宿主地址。[0.8](chapter-00-environment.md#08-配置网络与材料传递) |
| UDP | User Datagram Protocol，用户数据报协议 | KDNET 使用它传输调试数据，宿主防火墙规则要匹配 UDP 端口。[0.11](chapter-00-environment.md#connect) |
| KDNET、`kdnet.exe`、`VerifiedNICList.xml` | 网络内核调试传输、配置工具及网卡清单 | 先验证目标支持，再设置宿主地址与端口；工具和清单来自同一套 x64 调试工具。[0.11](chapter-00-environment.md#connect) |
| KDCOM、COM1、named pipe | 串口内核调试传输、来宾串口编号、命名管道 | VMware 由宿主提供管道，调试器使用管理员权限；编号以实际目标为准。[0.11](chapter-00-environment.md#connect) |
| `reconnect`、`resets=0` | 通用虚拟串口命令中的重连与同步参数 | 微软说明要求 VMware 不使用它们，不能把通用示例原样套用。[0.11](chapter-00-environment.md#connect) |
| BCD | Boot Configuration Data，启动配置数据 | `debug`、`dbgsettings` 与 `testsigning` 职责不同。[0.12](chapter-00-environment.md#recovery) |
| BitLocker | Windows 磁盘加密功能 | 改来宾启动安全设置前保存恢复密钥；暂停保护器不等于解密磁盘。[0.11](chapter-00-environment.md#connect) |
| VBS、HVCI | Virtualization-based Security，基于虚拟化的安全性；hypervisor-protected code integrity，虚拟机监控程序保护的代码完整性 | 内存完整性使用虚拟化隔离保护内核代码完整性，状态与安全启动分别记录。[0.6](chapter-00-environment.md#vmware)、[0.7](chapter-00-environment.md#target-install) |
| WHP | Windows Hypervisor Platform，Windows 虚拟机监控程序平台 | Workstation 与宿主 Hyper-V/VBS 共存时可能经由它运行，不能据此假定支持来宾嵌套虚拟化。[0.6](chapter-00-environment.md#vmware) |
| 检查点 / checkpoint、快照 / snapshot | 保存虚拟机可恢复状态，便于练习启动配置与故障恢复 | 先用来宾本地标记文件验证，不能替代独立备份；作业放在还原范围外。[0.9](chapter-00-environment.md#09-在改启动配置前练习还原) |
| PDB | Program Database，程序数据库，保存调试符号 | 必须匹配对应二进制；符号解释错误会影响后续故障判断。[0.10](chapter-00-environment.md#symbols) |

### 常用检查与配置命令

下表保留命令名称和关键参数，完整可跟做步骤见链接。带占位含义的描述不能直接粘贴执行。

| 命令或工具 | 在哪里用、用来做什么 | 关键限制与正文 |
| --- | --- | --- |
| `winver`、`msinfo32`、`systeminfo` | Windows 中查看版本、架构、固件与虚拟化条件 | 区分宿主和目标；22H2 用目标 `winver` 核对。[0.2](chapter-00-environment.md#02-检查电脑与资源)、[0.7](chapter-00-environment.md#target-install) |
| `hostname`、`ipconfig`、`ping` | 分别确认机器名称、接口地址与 ICMP 回显 | `ping` 无响应可能是回显受阻，也不验证 KDNET 的 UDP 端口。[0.8](chapter-00-environment.md#08-配置网络与材料传递) |
| `Copy-VMFile` | 宿主管理员 PowerShell，通过来宾服务把材料复制进 Hyper-V 目标 | `-FileSource Host` 规定来源，`-CreateFullPath` 创建目标路径。[0.8](chapter-00-environment.md#08-配置网络与材料传递) |
| `Get-Service vmicguestinterface`、`Start-Service vmicguestinterface` | 目标管理员 PowerShell，检查并按需启动 Hyper-V 来宾服务 | 宿主还需启用目标的“来宾服务”集成项。[0.8](chapter-00-environment.md#08-配置网络与材料传递) |
| `bcdedit /enum {current}`、`bcdedit /dbgsettings` | 目标管理员命令提示符，读取当前启动项与调试参数 | 不带设置参数时用于检查，先留原值再修改。[0.7](chapter-00-environment.md#target-install) |
| `bcdedit /debug on`、`bcdedit /debug off` | 目标管理员命令提示符，开关内核调试 | 涉及重启；不能证明驱动可以加载，恢复步骤要同时检查安全配置。[0.11](chapter-00-environment.md#connect)、[0.12](chapter-00-environment.md#recovery) |
| `bcdedit /dbgsettings serial debugport:1 baudrate:115200` | 目标管理员命令提示符，选择 COM1 串口调试 | COM 编号必须与目标实际端口一致。[0.11](chapter-00-environment.md#connect) |
| `manage-bde -status C:` | 目标管理员命令提示符，读取系统盘 BitLocker 状态 | 状态未知时先核对，不把缺少记录当作未启用。[0.7](chapter-00-environment.md#target-install) |
| `manage-bde -protectors -disable C: -RebootCount 0`、`-enable C:` | 目标管理员命令提示符，暂停或恢复保护器 | `0` 持续暂停，必须有恢复密钥并明确恢复；完整恢复命令见正文。[0.11](chapter-00-environment.md#connect)、[0.12](chapter-00-environment.md#recovery) |
| `New-NetFirewallRule`、`Remove-NetFirewallRule` | 宿主管理员 PowerShell，添加或移除本章 KDNET 规则 | 限制程序、UDP 端口与实验子网，不关闭整个防火墙。[0.11](chapter-00-environment.md#connect) |
| `windbg.exe -k net:...`、`windbg.exe -k com:...` | 宿主启动内核调试会话 | `...` 是省略标记，须使用正文中的本路线参数；密钥不可照抄示例。[0.11](chapter-00-environment.md#connect) |

### WinDbg 观察命令

这些命令在宿主 WinDbg 中输入。先建立连接并暂停目标，不能当作 Windows shell 命令执行。

| 命令 | 用途 | 关键限制与正文 |
| --- | --- | --- |
| `.symfix C:\KernelLab\Symbols`、`.sympath` | 配置微软符号服务器与缓存，查看当前路径 | 设置路径不表示已经加载匹配符号。[0.10](chapter-00-environment.md#symbols) |
| `srv*缓存目录*https://msdl.microsoft.com/download/symbols` | 符号路径的服务器与缓存语法 | 缓存要在宿主可写，替换其中的目录描述。[0.10](chapter-00-environment.md#symbols) |
| `vertarget` | 查看目标信息 | 某些内核组件共享基础版本，结合来宾 `winver` 判断 22H2。[0.13](chapter-00-environment.md#first-observation) |
| `lm m nt`、`lmvm nt` | 筛选 Windows 内核模块，查看详细信息 | `nt` 是模块名；`deferred` 是延迟加载，不能等同于符号验证成功。[0.13](chapter-00-environment.md#first-observation) |
| `.reload /f nt` | 立即加载内核符号 | 首次可能下载，不能强制忽略匹配错误。[0.13](chapter-00-environment.md#first-observation) |
| `!sym noisy`、`!sym quiet` | 开启或关闭符号详细诊断输出 | 用失败原因区分缓存权限、下载与匹配问题。[0.13](chapter-00-environment.md#first-observation) |
| `.logopen /t`、`.logclose` | 打开带时间信息的日志，关闭日志 | 日志保存宿主，目录须存在；完整参数见正文。[0.13](chapter-00-environment.md#first-observation) |
| `g` | 继续执行目标系统 | 主动暂停后可恢复，实际崩溃原因不会因此修复。[0.12](chapter-00-environment.md#recovery)、[0.13](chapter-00-environment.md#first-observation) |

### 用户态小实验

| 名称或操作 | 用途 | 关键限制与正文 |
| --- | --- | --- |
| `cl`、`/W4`、`/Zi`、`/Od` | MSVC 编译器、警告等级、调试信息、关闭优化 | 在宿主 x64 开发命令窗口执行，选项用于本章用户态程序。[0.14](chapter-00-environment.md#usermode) |
| `/EHsc`、`/Fe`、`/link /DEBUG /PDB` | C++ 异常选项、输出文件名、传给链接器的调试与符号选项 | EXE 与 PDB 要来自同次构建，不能直接照搬为内核构建配置。[0.14](chapter-00-environment.md#usermode) |
| `std::printf` | 按格式向标准输出写出计算结果 | 本章用它核对修改源码与重新编译的区别，无驱动通信。[0.14](chapter-00-environment.md#usermode) |
| `F9`、`F5`、`F10` | 在 Visual Studio 设置断点、开始或继续调试、逐过程执行 | 空心断点时核对源码、EXE 和 PDB 是否对应。[0.14](chapter-00-environment.md#usermode) |

能连接并暂停目标，只完成了环境检查的一部分。版本配对、还原能力和用户态断点都需要各自的记录，参见[环境完成检查](chapter-00-environment.md#环境完成检查)。

<a id="chapter-01"></a>

## 第 1 章 重新看 C/C++ 中的数据与内存

### 整数、指针与对象

| 名称或表达式 | 含义与用途 | 关键限制与正文 |
| --- | --- | --- |
| bit、byte、`0x` | 位、字节与十六进制前缀；本章一个字节为 8 位 | 读消息字段前先确认位宽，十六进制每位对应 4 个二进制位。[1.1](chapter-01-data-and-memory.md#integers) |
| `std::uint8_t`、`std::uint16_t`、`std::uint32_t`、`std::uint64_t` | `<cstdint>` 中的无符号固定宽度整数，分别为 8、16、32、64 位 | 本章平台均可用；字段宽度不同时，转换可能丢失高位。[1.1](chapter-01-data-and-memory.md#integers) |
| `std::int32_t`、`std::int64_t` | 32、64 位有符号整数 | 有符号溢出是未定义行为，不能依赖回绕；负数先检查再转无符号长度。[1.1](chapter-01-data-and-memory.md#integers) |
| `std::size_t`、`void*`、`long` | 本地大小类型、指针类型与整数类型 | MSVC x64 下前两者占 8 字节，`long` 占 4 字节；不能把它们当作跨平台消息宽度。[1.1](chapter-01-data-and-memory.md#integers) |
| 补码 / two's complement、截断 / truncation、符号扩展 | 解释本章有符号表示、缩窄与扩宽 | 扩宽结果变量不能修复已经在较窄表达式中发生的溢出。[1.1](chapter-01-data-and-memory.md#integers)、[1.4](chapter-01-data-and-memory.md#lengths) |
| `static_cast<T>`、`auto`、`constexpr` | 显式转换、从初始化表达式推导类型、声明编译时常量 | 转换不会自动拒绝超范围输入；`u` 后缀用于无符号字面量。[1.1](chapter-01-data-and-memory.md#integers) |
| `&`、`\|`、`^`、`~`、`<<`、`>>` | 按位与、或、异或、取反、左移和右移，用于掩码与字节解码 | 至少一位置位与全部位置位条件不同；移位次数须小于提升后左操作数的位宽。[1.1](chapter-01-data-and-memory.md#integers) |
| buffer、容量、有效长度 | 缓冲区、容纳上限与实际允许使用的长度 | 元素数与字节数分别标明，接收容量不能替代实际收到的长度。[1.2](chapter-01-data-and-memory.md#pointers) |
| `nullptr`、dangling pointer | 空指针常量与悬空指针 | 非空不保证对象仍存活，指针副本不延长生存期。[1.2](chapter-01-data-and-memory.md#pointers) |
| array-to-pointer decay、尾后指针 | 数组转首元素指针，以及同一数组末尾之后的位置 | 指针不携带数组长度；尾后位置可用于范围比较，不可解引用。[1.2](chapter-01-data-and-memory.md#pointers) |
| scope、object lifetime、ownership | 作用域、对象生存期与所有权 | 分别回答名字在哪可见、对象何时存在、谁负责释放；排队地址前须保证数据有效期。[1.2](chapter-01-data-and-memory.md#pointers) |

### 布局与安全长度

| 名称或条件 | 含义与用途 | 关键限制与正文 |
| --- | --- | --- |
| offset、alignment、padding | 偏移、对齐与填充 | MSVC x64 默认条件下，本章 `LocalRecord` 偏移为 0、4、8、12，总大小为 16。[1.3](chapter-01-data-and-memory.md#layout) |
| `sizeof`、`alignof`、`offsetof` | 求大小、对齐要求与成员偏移 | 结构体大小包含填充；指针的 `sizeof` 不等于缓冲区长度，`offsetof` 不用于位域。[1.3](chapter-01-data-and-memory.md#layout) |
| little-endian、big-endian | 小端序与大端序，约定多字节数值的字节顺序 | 小端把最低有效字节放在最低偏移；协议应明确约定，不能靠强转推定。[1.3](chapter-01-data-and-memory.md#layout) |
| `union`、bit-field、`#pragma pack` | 联合体、位域与打包对齐控制 | 联合体跟踪当前有效成员；位域布局依赖实现，打包不能代替长度检查。[1.3](chapter-01-data-and-memory.md#layout) |
| `version`、`flags`、`count`、`payload_bytes` | 本章消息头字段，分别为版本、标志、条目数和条目区字节数 | 消息头 12 字节，每条 8 字节，最多 1024 条；版本为 1，标志只定义位 0，禁止尾随字节。[1.3](chapter-01-data-and-memory.md#layout) |
| `length >= H` | 在读取固定头之前确认真实可读长度足够 | 先检查，才能做无符号减法 `length - H`。[1.4](chapter-01-data-and-memory.md#lengths) |
| `count <= (length - H) / E` | 用剩余空间约束条目数，之后再做乘法 | `E` 必须非零；还要核对协议声明长度与业务上限。[1.4](chapter-01-data-and-memory.md#lengths) |
| `b <= M - a`、`a <= M / b` | 以目标类型最大值 `M` 预查无符号加法、乘法是否能表示 | 操作数先在目标类型范围内，第二式要求 `b != 0`；能表示不等于能分配。[1.4](chapter-01-data-and-memory.md#lengths) |
| `ReadU16Le`、`ReadU32Le`、`ValidateMessage` | 本章自定义的 2 字节、4 字节小端解码与消息结构校验函数 | 不是 Windows API；要求真实缓冲区在调用期间有效且不变，校验通过不代表内容可信。[1.4](chapter-01-data-and-memory.md#lengths) |
| `ntintsafe.h` | WDK 安全整数运算与转换函数所在头文件 | 函数报告溢出后调用方要处理失败，本章只介绍用途。[1.4](chapter-01-data-and-memory.md#lengths) |
| `assert`、`NDEBUG`、`/std:c++17`、`%zu` | 实验断言、禁用断言的宏、语言版本选项、大小类型打印格式 | 实验不定义 `NDEBUG`，正式输入检查保留在校验函数内；仅编译独立用户态程序。[1.4](chapter-01-data-and-memory.md#lengths) |

### 保存与处理记录

| 名称 | 用途与反作弊场景 | 关键限制与正文 |
| --- | --- | --- |
| 数组、`O(1)`、`O(n)` | 数组按下标访问为常数级，按值查找通常随项数增长 | 下标小于有效元素数，复杂度不表示具体耗时。[1.5](chapter-01-data-and-memory.md#structures) |
| intrusive linked list、`LIST_ENTRY`、`Flink`、`Blink` | 侵入式链表、Windows 双向链接结构、后继和前驱链接 | 已知节点可快速摘除；摘除不等于可释放，仍需确认借用者结束访问。[1.5](chapter-01-data-and-memory.md#structures) |
| FIFO、queue、ring buffer | First In, First Out，先进先出；队列与环形缓冲区，用于有上限的事件排队 | 本章 `head` 取出、`tail` 写入、`used` 区分空满；满时拒绝新项并记录丢弃。[1.5](chapter-01-data-and-memory.md#structures) |
| lookup table、hash table | 查找表与哈希表，按键定位已保存记录 | 哈希查找平均可为常数级，冲突会恶化；容量、删除与对象生存期仍需管理。[1.5](chapter-01-data-and-memory.md#structures) |

消息长度合法，只证明当前检查覆盖的结构关系。地址生存期、条目含义、权限与并发条件还需要分别成立；本章实验的范围见[用户态示例](chapter-01-data-and-memory.md#lengths)，复习题见[思考题](chapter-01-data-and-memory.md#thinking)与[练习题](chapter-01-data-and-memory.md#exercises)。
