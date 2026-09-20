# 第 0 章 从零搭建练习环境

假设你正在排查反作弊驱动处理程序启动事件时的卡顿。一个测试程序刚启动，目标 Windows 的桌面就没有响应。此时首先要分清，系统究竟发生了故障，还是被调试器暂停了。如果调试器也装在这台被暂停的系统里，你连继续观察的入口都没有；直接重启又会丢掉当前执行状态。这是本章的教学场景，后面会通过暂停已有系统来练习判断，无须运行自编驱动。

内核负责管理内存、调度程序并协调设备。内核反作弊驱动运行在这一层，错误可能影响整台 Windows 系统。学习它之前，先准备一台能观察、能还原的目标虚拟机。这样遇到启动配置错误时有办法退回原状态，也能把符号加载失败与系统故障分开判断。

本章完成后，你应能从宿主机连接 Windows 10 22H2 x64 来宾，暂停系统、查看版本和内核模块，再让它继续运行。你还会在宿主机编译一个独立的用户态小程序，确认断点和变量查看可用。这里不编译、加载或测试自编驱动。

以下步骤按 **2026 年 9 月 20 日** 查阅的微软与 Broadcom 官方资料编写。版本兼容关系来自文档，资源分配属于本教程的建议；预期现象供你核对，文中没有虚拟机实测记录。只阅读的读者可以先做思考题，操作项留待具备设备后完成。

## 0.1 先分清宿主与目标

虚拟机（Virtual Machine，VM）是一台由软件提供处理器、内存和设备接口的计算机。你会在里面单独安装 Windows，它有自己的桌面、系统盘和启动配置。

本章让一台实体电脑承担开发和调试工作，另一套 Windows 在虚拟机中接受观察。

| 名称 | 本章放在哪里 | 负责什么 |
| --- | --- | --- |
| 宿主机（host） | 你的 x64 Windows 实体电脑 | 运行虚拟化软件，保存虚拟磁盘和作业记录 |
| 开发机 | 与宿主机共用 | 安装 Visual Studio、SDK 和 WDK，编写后续用户态小实验 |
| 调试主机（debugger host） | 与宿主机共用 | 运行 WinDbg，向目标发出暂停、查看与继续命令 |
| 目标机（target）或来宾机（guest） | Windows 10 22H2 x64 虚拟机 | 运行被观察的 Windows 系统 |

Visual Studio 是编辑、编译和调试程序的开发环境。软件开发工具包（Software Development Kit，SDK）提供 Windows 程序所需的头文件、库和工具；Windows 驱动工具包（Windows Driver Kit，WDK）补充驱动开发接口和工具。WinDbg 是 Windows 调试器，能连接目标系统，读取其调试信息。[微软调试入门][debug-start]

普通应用主要在用户态（user mode）运行，访问系统资源时通过操作系统提供的接口；内核及内核模式驱动在内核态（kernel mode）运行，错误可能影响整个系统。管理员权限是账户权限，不能与内核态混为一谈。两种执行模式的细节会在系统基础章节展开。

如果你用过浏览器开发者工具，可以把“在一边发命令、观察另一边的执行状态”作为起点。这里的区别很大。内核调试暂停目标时，目标系统中的普通程序也无法照常执行，鼠标和桌面都可能停止响应；宿主上的 WinDbg 仍然可以工作。

后文每组操作都注明位置和权限。**管理员命令提示符**指在相应机器的开始菜单搜索 `cmd`，右键选择“以管理员身份运行”，确认窗口标题包含管理员身份。PowerShell 与命令提示符是两种命令环境，照着代码块上方的说明打开，避免把语法混用。

## 0.2 检查电脑与资源

本章宿主参考配置为 Windows 11 24H2 Pro x64，来宾固定为 Windows 10 22H2 x64。这里的 x64 指 Intel、AMD 常见的 64 位 x86 处理器架构，与 ARM64 分开；安装器中的 x86 通常指 32 位组件。宿主和来宾可以运行不同版本的 Windows。Hyper-V 路线要求宿主具有相应功能，本章选 Pro；Home 版读者可以选择 VMware 路线。

在**宿主机，普通用户权限**下依次检查。

1. 按 `Win+R`，运行 `winver`，记录宿主版本与完整操作系统构建号。
2. 再运行 `msinfo32`，在“系统摘要”记录系统型号、处理器、已安装物理内存和系统类型。本章要求系统类型为 `x64-based PC`。ARM64 电脑，包括 Apple Silicon Mac，不在这两条 x64 Windows 宿主路线内。
3. 打开任务管理器的“性能”页，选择 CPU，也就是中央处理器，查看“虚拟化”。若显示未启用，按电脑厂商说明进入固件设置，启用 Intel VT-x 或 AMD-V，有些固件把后者称为 SVM。保存原设置后重启，再回来核对。不要顺手修改启动模式或磁盘控制器模式。
4. 在普通命令提示符运行 `systeminfo`。Hyper-V 要求项应满足硬件虚拟化、二级地址转换（Second Level Address Translation，SLAT）等条件。若显示已检测到虚拟机监控器，相关要求不再逐项列出，这是宿主已经运行虚拟化组件的提示。[Hyper-V 硬件要求][hyperv-requirements]

固件负责操作系统启动前的硬件配置。SLAT 是处理器为虚拟机地址转换提供的支持，本章只检查是否具备，原理留到机器基础章节。

| 资源 | 本章建议 | 分配理由 |
| --- | --- | --- |
| 宿主内存 | 16 GB 起步，有条件使用 32 GB | 同时容纳 Windows、Visual Studio、WinDbg 和一台来宾 |
| 宿主可用磁盘 | 至少预留 150 GB，并另留备份位置 | 工具、安装介质、虚拟磁盘、符号缓存和快照都会占空间 |
| 目标虚拟处理器（vCPU） | 2 个 | 足够进行本章观察，也给宿主留出执行余量 |
| 目标内存 | 4 GB，先使用固定分配 | 目标只运行 Windows 和少量练习材料 |
| 目标虚拟磁盘 | 64 GB，按需增长 | 64 GB 是来宾可用容量上限，实际占用随写入增加 |

这些是教学预算，不能当成产品最低要求。微软列出的 Hyper-V 内存最低门槛不足以代表同时打开整套工具时的体验；Visual Studio 的安装空间也随组件变化，确认安装器显示的占用后再继续。[Visual Studio 系统要求][vs-requirements]

若宿主只有 8 GB 内存，可以先阅读和做用户态小实验。虚拟机频繁换页、宿主接近无响应时，先关闭其他程序或调整资源，不能直接把慢响应记成内核故障。

## 0.3 准备安装材料并固定版本

在**宿主机，普通用户权限**下建立 `C:\KernelLab`，用资源管理器新建 `Installers`、`Records`、`Symbols`、`Transfer` 和 `UserMode` 五个子目录。若创建根目录要求管理员确认，确认这一次即可。以后把读者记录保存在 `Records`，不要仅保存在目标虚拟机里。

| 材料 | 本章选用版本 | 获取与检查 |
| --- | --- | --- |
| 目标系统安装介质 | Windows 10 22H2 x64，按许可证选择 Pro 等版本 | 从[微软 Windows 10 下载页][windows-download]取得 ISO 光盘映像文件，或使用组织授权介质 |
| Hyper-V | Windows 11 24H2 Pro 内置版本 | 通过“启用或关闭 Windows 功能”安装，不另找下载包 |
| VMware Workstation Pro | 26H1，Windows x64 版 | 从[Broadcom 官方下载入口说明][vmware-download]进入支持门户，选择 Workstation Pro 与对应版本 |
| Visual Studio | 2022，17.14 系列，安装可用的维护更新 | 从[VS 2022 发布与下载说明][vs-release]进入对应版本下载，记录实际补丁号 |
| Microsoft Visual C++（MSVC）编译工具 | v143，14.44 工具集 | 通过 VS 2022 的 C++ 工作负载安装，用 `cl` 记录实际编译器版本 |
| Windows SDK | 10.0.26100.6584 | 从[微软 WDK 兼容表][wdk-versions]的 `26100.6584` 行进入 SDK 链接 |
| Windows WDK | 10.0.26100.6584 | 从同一行进入 WDK 链接 |
| WinDbg | 上述 SDK 的 Debugging Tools for Windows 中的 x64 WinDbg (Classic) | 使用固定的 SDK 安装包取得，在调试器 About 窗口记录其实际版本 |

本章选用的 SDK 与 WDK 对应官方兼容表中的 VS 2022 组合。下载页上的“Windows 11”标签描述工具包对应的发布系列，目标仍可选 Windows 10；具体驱动接口是否支持目标系统，还要在后续章节查各接口的最低版本。本章也不据此宣称任何驱动已通过兼容性测试。[WDK 安装与版本规则][wdk-install]

SDK 与 WDK 的 **build number 必须一致**。例如 `10.0.26100.6584` 中的 `26100` 是 build number，`6584` 是修订号。微软允许部分修订号不同的组合，本章选择相同版本以减少初次安装的变量。`Windows Kits\10\Include\10.0.26100.0` 这样的目录名不能证明修订号相同，要从“设置”的已安装应用列表记录完整版本。

MSVC 工具集的 `14.44` 与编译器输出中的 `19.44` 属于不同的版本编号方式，都对应这里的 VS 2022 17.14 系列，记录时保留各自原值。[MSVC 版本对应表][msvc-versions]

微软当前主下载页已经面向更新的 Visual Studio 系列，直接连点多个页面上的“最新版”可能得到另一组工具。用上表逐项核对。VS Community、Professional 和 Enterprise 选择与你使用资格相符的一种，不需要为了本章购买额外插件。

Broadcom 已发布 Workstation 26H1，官方宿主兼容表包含 Windows 11。门户可能要求注册账户并完成下载资料，下载按钮不可用时先检查账户状态，或选择 Hyper-V 路线。[26H1 发布说明][vmware-release]、[宿主兼容表][vmware-hosts]

Windows 10 的常规支持已在 2025 年 10 月 14 日结束。本教程保留 22H2 作为学习基准，使用只存放练习数据的来宾。需要联网更新时按你的更新资格处理，日常账户、工作文档和密码留在正常使用的系统中，不放进调试来宾。[Windows 10 生命周期说明][windows-lifecycle]

在 Windows 宿主运行下载工具时，选择“为另一台电脑创建安装介质”，取消沿用当前电脑配置，核对 Windows 10 与 64 位架构后保存 ISO。若网页直接提供 ISO，选择 22H2 x64。保存文件名和下载日期；安装后还要用 `winver` 再确认版本，不能只看 ISO 文件名。

## 0.4 安装并检查开发工具

这一节在**宿主机**完成。运行安装器需要管理员权限，日常编写和用户态调试使用普通用户权限。

1. 启动 Visual Studio 2022 安装器，勾选“使用 C++ 的桌面开发”（Desktop development with C++）。在“单个组件”确认选中 MSVC v143 的 x64/x86 14.44 工具，以及对应的 x64/x86 Spectre 缓解库。后者是带有特定处理器漏洞缓解措施的库，先按 WDK 安装要求备齐，不在本章讨论其实现。
2. 在“单个组件”搜索并勾选 **Windows Driver Kit**。这是 Visual Studio 的 WDK 扩展组件，负责集成驱动项目支持，不能替代完整 WDK 安装。VS 17.11 起，微软通过此入口提供 WDK VSIX 扩展。[WDK 扩展安装说明][wdk-install]
3. 安装上表指定的 SDK，保留 Windows SDK 的桌面 C++ 组件，同时勾选 **Debugging Tools for Windows**。若之前装过 SDK，可再次运行同一安装器，选择修改组件。
4. 关闭 Visual Studio，安装上表指定的 WDK，按提示重启后再打开 Visual Studio。若 WDK 安装器提示缺少扩展，回到第 2 步修改对应 VS 2022 实例。

安装结束后，先核对实物。

| 检查位置 | 应找到什么 | 没找到时怎样处理 |
| --- | --- | --- |
| Visual Studio 的“帮助 / 关于” | Visual Studio 2022 与 17.14.x | 确认打开的是对应版本，多个 VS 实例可以并存 |
| 开始菜单的 `x64 Native Tools Command Prompt for VS 2022` | 执行 `cl` 后显示面向 x64 的编译器版本 | 普通 `cmd` 没有开发环境变量，先换到这个专用窗口；入口缺失则修改 C++ 工作负载 |
| 设置中的已安装应用，搜索 `Kit` | SDK、WDK 的完整版本 | 核对安装器与实例；不要靠目录末尾的 `.0` 推断修订号 |
| `C:\Program Files (x86)\Windows Kits\10\Include\10.0.26100.0\um\Windows.h` | SDK 用户态头文件 | 缺失时补装 SDK 桌面 C++ 组件 |
| 同一 `Include` 版本下的 `km\ntddk.h` | WDK 内核接口头文件 | 缺失时检查完整 WDK 是否安装成功 |
| `C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\um\x64\kernel32.lib` 与 `km\x64\ntoskrnl.lib` | 用户态和内核态的对应库文件 | 按缺失的一侧修复 SDK 或 WDK |
| `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64` | `windbg.exe`、`kdnet.exe`、`VerifiedNICList.xml` | 修改 SDK，补选 Debugging Tools for Windows |

头文件提供接口声明，库文件参与链接。此时只确认位置与版本，暂不阅读内部声明。单独运行 `cl` 后因为没有输入源文件而报错，并不表示编译器未安装；需要保存的是它前面的版本与架构信息。[MSVC 命令行入门][msvc-cli]

还可以打开“创建新项目”，清除语言与平台筛选，搜索 `Kernel Mode Driver`，确认驱动模板能被找到，然后取消。若文件齐全而模板缺失，修复对应 VS 实例的 Windows Driver Kit 组件，重启 IDE 后再查。**找到模板只验证集成入口，本章不创建驱动工程。**

## 0.5 Hyper-V 路线

选择 VMware 的读者跳到 [0.6](#vmware)。两条路线完成其一即可。

本路线使用第二代虚拟机（Generation 2）和 KDNET 网络调试。第二代虚拟机使用统一可扩展固件接口（Unified Extensible Firmware Interface，UEFI）启动，适合本章的 x64 来宾。代数在创建后不能直接切换，创建时就选好。[创建 Hyper-V 虚拟机][hyperv-create]

### 启用功能

在**宿主机，管理员权限**下按 `Win+R`，运行 `optionalfeatures`，勾选 Hyper-V 及其管理工具和平台子项，确认安装并重启。重启后在开始菜单打开“Hyper-V 管理器”，左侧应出现本机名称。

若没有 Hyper-V 项，先回 [0.2](#02-检查电脑与资源) 核对 Windows 版本与硬件条件；若只有管理工具可用，检查固件虚拟化。不要用来源不明的脚本为 Home 版补装功能。[Hyper-V 安装说明][hyperv-install]

### 创建用于调试的交换机

虚拟交换机决定来宾网卡接到哪里。微软区分外部、内部和专用交换机。

| 类型 | 默认连接范围 | 本章如何使用 |
| --- | --- | --- |
| 外部（External） | 来宾可接入宿主网卡所在的物理网络，可允许宿主共享该网卡 | 用于本路线，宿主和目标在同一个受信任的实验局域网 |
| 内部（Internal） | 宿主与接入的来宾之间 | 不自动提供互联网，本章不额外搭建路由或地址分配服务 |
| 专用（Private） | 接入的来宾之间 | 宿主默认不在其中，不适合直接照抄本章的宿主连接步骤 |

本路线按微软虚拟机 KDNET 文档采用外部交换机。准备一条接入受信任实验路由器的有线网络，并让路由器自动分配地址。若所处网络禁止额外设备、只有受限的无线网络，选用本章 VMware 的仅主机网络与串口路线，避免在公司网络上修改交换配置。[虚拟交换机说明][hyperv-switch]、[Hyper-V KDNET 配置][hyperv-kdnet]

在**宿主机，管理员权限**下操作。

1. 先保存工作。创建外部交换机可能短暂中断宿主网络，不在仅能远程连接的宿主上进行这一步。
2. 在 Hyper-V 管理器打开“虚拟交换机管理器”，新建“外部”交换机，命名 `KernelLab-External`。
3. 选择实际连接实验网络的以太网卡，勾选“允许管理操作系统共享此网络适配器”。不启用 VLAN 标识；VLAN 是网络分组配置，本章的普通实验网络不需要额外设置。
4. 应用后，在宿主普通命令提示符执行 `ipconfig`。应能找到 `vEthernet (KernelLab-External)`，并看到该网络的 IPv4 地址。

若宿主断网，回到交换机属性核对物理网卡与共享选项。选错了网卡时改回正确网卡；需要撤回时先断开使用这个新交换机的实验虚拟机，再删除刚建的 `KernelLab-External`，核对宿主恢复联网。不要删除其他程序正在使用的交换机。

### 创建目标机

仍在**宿主机的 Hyper-V 管理器，管理员权限**下，新建虚拟机。

1. 名称填写 `KernelLab-W10`，位置选择剩余空间充足的本地目录。
2. 选择“第二代”，内存填 `4096 MB`，取消“为此虚拟机使用动态内存”。
3. 网络选择 `KernelLab-External`，创建 64 GB 的 VHDX 虚拟硬盘。VHDX 是 Hyper-V 保存来宾磁盘内容的文件格式。
4. 安装选项选从映像文件安装，挂载 Windows 10 22H2 x64 ISO，完成向导。
5. 在虚拟机关闭时打开“设置”，把处理器数量设为 2。保留 UEFI 和安全启动的初始设置，安全启动模板使用 Microsoft Windows。安全启动（Secure Boot）检查启动组件是否符合信任要求，调试前会单独处理它。
6. 打开“连接”窗口，启动虚拟机，看到从 DVD 启动的提示时按任意键，进入 Windows 安装界面。

若直接进入网络启动或找不到系统，关机后检查 DVD 是否挂载 ISO，并在“固件”中把 DVD 调到安装期的优先启动位置。若提示无法验证映像，先核对官方 x64 ISO 与 Microsoft Windows 模板，不通过来回切换虚拟机代数试错。继续到 [0.7](#target-install)。

<a id="vmware"></a>

## 0.6 VMware Workstation 路线

本路线在**Windows x64 宿主**上使用 Workstation Pro 26H1，通过本机命名管道连接虚拟串口。命名管道（named pipe）是 Windows 提供的通信通道；这里把来宾看到的串口连接到宿主 WinDbg，不需要真实串口线。

### 安装与宿主兼容性

在**宿主机，管理员权限**下运行官方 Windows 安装器，完成安装并按提示重启。打开 Workstation，在“Help / About”记录版本与 build。若安装器明确提示宿主不支持，返回 [0.3](#03-准备安装材料并固定版本) 的官方宿主表核对，不能用“另一台电脑装过”作为兼容依据。

基于虚拟化的安全性（Virtualization-based Security，VBS）会使用 Windows 虚拟化能力。宿主启用 Hyper-V 或 VBS 时，Workstation 可能通过 Windows Hypervisor Platform（WHP，Windows 虚拟机监控程序平台）运行，性能和可用功能会与直接使用硬件虚拟化不同。Workstation 的状态信息和宿主 `msinfo32` 可帮助记录当前情况。Broadcom 对此有硬件与系统要求，并记录过 Windows 11 24H2 上旧版 Workstation 的性能问题。[共存条件][vmware-vbs]、[性能与运行模式说明][vmware-performance]

本章只运行一层 Windows 来宾，无须在虚拟机中再运行虚拟机。先保留宿主现有保护设置。若出现不支持虚拟化扩展的错误，检查虚拟机处理器选项是否误勾了向来宾暴露 VT-x/EPT 或 AMD-V/RVI，再核对宿主固件与 Workstation 版本。不要为了启动这个练习环境批量关闭宿主安全功能。

### 创建目标机

在**宿主机，普通用户权限**下打开 Workstation，选择“File / New Virtual Machine”。后文保留英文菜单名便于对照界面。

1. 选择 `Custom (advanced)`，硬件兼容性保留当前版本默认值。安装方式选择 `I will install the operating system later`，先手动配置，避免自动安装跳过你需要核对的项目。
2. 来宾类型选择 Microsoft Windows，版本选择 Windows 10 x64，名称填写 `KernelLab-W10`。
3. 固件选择 UEFI，记录安全启动复选框的初始状态。不要为本章额外添加虚拟 TPM 或启用来宾嵌套虚拟化。
4. 设置 1 个处理器、每处理器 2 个核心，总计 2 vCPU，内存设为 4096 MB。
5. 网络先选 NAT，磁盘控制器与虚拟磁盘类型保留向导推荐值。新建 64 GB 虚拟磁盘，不预先分配全部空间；可选择拆分文件以便复制。
6. 完成后打开“VM / Settings”，在虚拟 CD/DVD 中选择 ISO 映像，勾选 `Connect at power on`。确认处理器里的 `Virtualize Intel VT-x/EPT or AMD-V/RVI` 未勾选，再启动虚拟机。

预期进入 Windows 安装界面。若提示没有启动介质，先检查 CD/DVD 的 ISO 路径与开机连接选项，再从“VM / Power / Power On to Firmware”进入固件选择 DVD。调整前先关机，避免在挂起状态改硬件。虚拟串口在 Windows 装好并做完初始快照后配置。

<a id="target-install"></a>

## 0.7 安装并记录目标系统

以下在**目标虚拟机的控制台**进行。Windows 安装程序对来宾虚拟磁盘具有写入权限，先确认窗口确实属于 `KernelLab-W10`。

1. 选择语言、时间与键盘布局，点击安装。按授权输入产品密钥；安装器提供“我没有产品密钥”时可暂后处理，后续使用仍按许可证要求进行。所选版本应与授权一致。
2. 安装类型选择“自定义”，选择新建的 64 GB 虚拟磁盘上的未分配空间，交给安装器创建分区。这里不应该出现你的宿主工作磁盘；若容量或磁盘身份不符，先退出检查虚拟机设置。
3. 等待文件复制和重启。第一次重启后若再次出现 DVD 启动提示，不再按键。若反复回到安装界面，关闭来宾，断开 ISO，再从虚拟硬盘启动。
4. 完成初始设置，使用专门的实验账户。Pro 版安装界面提供“脱机账户”或“改为域加入”等本地账户入口时可选用；不同介质的文字可能不同。只按界面提供的账户方式操作，不使用绕过脚本。进入桌面后，为实验管理员账户设置密码。
5. 在需要联网的阶段完成适用更新，重启到没有待处理的安装任务。若所用介质实际装成了其他版本，换回正确的 22H2 x64 介质重新安装，尚未开始练习时重建最省事。

接着在**目标机，普通用户权限**下运行 `winver` 与 `msinfo32`，记录 Windows 10、22H2、完整 `19045.x` 构建号、x64 架构、BIOS 模式和安全启动状态。`19045` 是 22H2 的系统版本标识，末尾修订号随更新变化。[Windows 10 发布信息][windows-releases]

在“Windows 安全中心 / 设备安全性 / 内核隔离详细信息”记录“内存完整性”状态。内存完整性（Memory integrity，也称 HVCI，hypervisor-protected code integrity）利用虚拟化隔离保护内核代码完整性检查；它和安全启动、磁盘加密分别处理不同问题。选项不可用时记录不可用及提示，不把未显示写成已关闭。[内存完整性说明][memory-integrity]

最后在**目标机，管理员命令提示符**中运行。

```bat
manage-bde -status C:
bcdedit /enum {current}
bcdedit /dbgsettings
```

`manage-bde` 用来查看 BitLocker 磁盘加密状态。启动配置数据（Boot Configuration Data，BCD）保存 Windows 的启动选项，`bcdedit` 用来查看和修改它。这里的 `/enum` 与 `/dbgsettings` 都只读取配置，`{current}` 指当前运行的启动项。保存输出时记下 `debug`、`testsigning` 是否出现及其值，缺失项写“未显式设置”。[BCDEdit 调试开关][bcd-debug]

把版本截图和配置记录复制或手工抄到**宿主** `C:\KernelLab\Records`。若 BitLocker 已启用，在后续改安全启动前先从来宾控制面板的“BitLocker 驱动器加密 / 备份恢复密钥”保存恢复密钥到安全的来宾外位置，不能把密钥交进公开作业。

## 0.8 配置网络与材料传递

### 先确认地址属于哪台机器

IP 地址是网络中的寻址信息。IPv4 通常写成四段数字；本章所用地址应从本机输出读取。动态主机配置协议（Dynamic Host Configuration Protocol，DHCP）可以自动分配地址，因此重启或恢复快照后地址可能改变。

在**宿主与目标，各自的普通命令提示符**运行 `hostname` 和 `ipconfig`，在宿主记录一张表。

| 项目 | Hyper-V 路线应记录的接口 | VMware 路线应记录的接口 |
| --- | --- | --- |
| 宿主名称与 IPv4 | `vEthernet (KernelLab-External)` | 对应 VMnet8 或 VMnet1 的 VMware 虚拟网卡 |
| 目标名称与 IPv4 | 目标的以太网卡 | 目标的以太网卡 |
| 网络模式 | 外部交换机，与宿主位于实验局域网 | 下载时 NAT，下载后可改 Host-only |
| 调试通道 | UDP 端口与 KDNET 密钥 | 本机命名管道路径，无需 IP 作为连接参数 |

网络地址转换（Network Address Translation，NAT）让 VMware 来宾通过宿主出网，默认通常对应 VMnet8。仅主机网络（Host-only）通常对应 VMnet1，供宿主和来宾互通，默认不提供互联网。[VMware 网络类型][vmware-network]

VMware 来宾更新完毕后，在**宿主 Workstation，来宾关机时**打开“VM / Settings / Network Adapter”，改为 Host-only。启动后重新运行 `ipconfig`。若拿不到正常地址，在宿主“Edit / Virtual Network Editor / Change Settings”中以管理员权限检查 VMnet1 是否连接宿主虚拟适配器、是否启用 DHCP；不要为了修一台实验机直接恢复全部虚拟网络默认值。

Hyper-V 来宾应从实验网络获得地址。若显示 `169.254.x.x`，通常表示没有获得预期 DHCP 地址，先检查交换机连接和路由器。来宾能上网也不能证明调试已通，KDNET 还要有宿主入站规则和正确密钥。

需要辅助判断时，可在目标执行 `ping` 加上记录的宿主 IPv4。无响应也可能是宿主防火墙阻止 ICMP 回显，不能单凭它判定网络断开。最终以 WinDbg 的实际连接与暂停操作为准，不关闭整台宿主防火墙来试探。

### Hyper-V 的材料复制

在**宿主，管理员权限**下打开目标的“设置 / 集成服务”，勾选“来宾服务”。让目标正常运行后，在宿主**管理员 PowerShell**逐条执行。

```powershell
Copy-VMFile -Name 'KernelLab-W10' -SourcePath 'C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\kdnet.exe' -DestinationPath 'C:\KDNET\kdnet.exe' -FileSource Host -CreateFullPath
Copy-VMFile -Name 'KernelLab-W10' -SourcePath 'C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\VerifiedNICList.xml' -DestinationPath 'C:\KDNET\VerifiedNICList.xml' -FileSource Host -CreateFullPath
```

`Copy-VMFile` 通过 Hyper-V 来宾集成服务复制文件，`Host` 指来源是宿主，`CreateFullPath` 允许创建目标目录。两条命令分别复制配置工具和它使用的网卡清单。[Copy-VMFile 参数说明][copy-vmfile]

预期在目标 `C:\KDNET` 找到两个文件。失败时核对虚拟机名称、源路径、来宾是否正在运行以及“来宾服务”是否启用。若服务仍不可用，在目标管理员 PowerShell 运行 `Get-Service vmicguestinterface` 查看来宾服务，再用 `Start-Service vmicguestinterface` 启动后重试；不要通过关闭防火墙修复这条非普通网络共享的复制路径。[来宾集成服务管理][integration-services]

以后传小材料可沿用同一命令结构，把源文件与目标文件路径换成实际值。此命令用于宿主到来宾，作业结果另存在宿主记录中。

### VMware 的材料复制

VMware Tools 是改善来宾设备支持和宿主来宾交互的一组组件。它不承担本章串口调试连接，但共享文件夹需要它。

1. 在**宿主 Workstation**的 CD/DVD 设置中移除 Windows 安装 ISO，选择使用物理驱动器并自动检测。目标已启动并登录后，选择“VM / Install VMware Tools”，让 Workstation 挂载 Tools 安装光盘。在**目标机，管理员权限**下打开光驱运行 `setup64.exe`，采用典型安装并重启。菜单不可用时先检查目标运行状态与光驱设置；若提示产品不附带 Tools ISO，按弹窗或 [Broadcom 的 Tools 下载说明][vmware-tools-download]取得支持 Windows 10 x64 的版本，记录版本后挂载其 `windows.iso`，再运行安装器。不要把宿主 Workstation 安装器拿到来宾里安装。
2. 在**宿主 Workstation**打开“VM / Settings / Options / Shared Folders”，添加宿主的 `C:\KernelLab\Transfer`，名称设为 `Transfer`，启用并勾选只读。这里只共享这个专用目录。
3. 在宿主 `Transfer` 中放一个普通文本文件。然后在**目标，普通用户权限**下打开资源管理器，输入 `\\vmware-host\Shared Folders\Transfer`，把文件复制到来宾自己的文档目录，核对内容。
4. 验证后关闭共享，后续传材料时再启用。路径不存在时先检查 Tools 是否安装并重启、共享是否启用；只读共享不能写入是预期行为。[Workstation 设备与共享操作手册][vmware-manual]

## 0.9 在改启动配置前练习还原

检查点（checkpoint，Hyper-V 用语）和快照（snapshot，VMware 用语）保存虚拟机在某一时刻的可恢复状态。它们通常依赖原虚拟磁盘和增量文件，宿主磁盘损坏时不能代替独立备份。

本章统一**先在来宾内正常关机，再创建保存点**，这样不需要恢复一段正在运行的内存状态。Hyper-V 还区分保存内存状态的标准检查点，以及使用来宾数据一致性机制的生产检查点；名称不同不意味着自动成为独立备份。[Hyper-V 检查点说明][hyperv-checkpoints]

1. 在**目标机**保存工作并正常关机，等管理界面显示关闭，不使用挂起或保存状态。
2. 在**宿主**创建保存点。Hyper-V 中右键目标，选择“检查点”，重命名为 `00-clean`；VMware 中选择“VM / Snapshot / Take Snapshot”，名称填 `00-clean`。说明栏记录来宾版本、网络模式、磁盘加密和安全启动状态。
3. 启动目标，在来宾本地 `C:\Users\Public\Documents` 新建 `restore-marker.txt`，写入一行任意文字。这个位置用于避开桌面同步与宿主共享目录。检查文件已保存，再正常关机。
4. 在**宿主**选择刚才的保存点。Hyper-V 选择“应用”，确认要丢弃本次标记文件练习产生的状态；VMware 打开“VM / Snapshot / Snapshot Manager”，选 `00-clean` 并 `Go To`，确认还原。
5. 启动目标，再检查标记文件。它应消失。重新核对 `winver`、安全配置和 `ipconfig`，记录还原后的状态。此时还没配置内核调试，后面配置成功后还要练习一次连接。

若文件仍存在，检查是否选错保存点、文件是否位于同步或共享位置，以及虚拟磁盘是否被排除在保存范围外。VMware 本章使用普通新建磁盘，不使用独立磁盘模式。不要反复点击还原来碰运气。

需要留存较长期副本时，在来宾关机后，用 Hyper-V 的“导出”保存到另一处存储；VMware 关闭后复制完整虚拟机目录及它引用的磁盘文件，不只复制一个基础 `.vmdk`。本章新建磁盘放在虚拟机目录里，保持这一默认安排即可。检查点链和快照文件由管理器维护，不在资源管理器里手动删除。[Hyper-V 导出][hyperv-export]、[Workstation 备份与快照说明][vmware-manual]

<a id="symbols"></a>

## 0.10 打开 WinDbg 并准备符号

调试符号保存二进制中的名称、类型和源码对应信息，常见文件格式是程序数据库（Program Database，PDB）。正确符号能把某个地址解释成有意义的名称；错误符号可能导致错误解释。它们必须匹配目标模块，不能拿宿主版本相近的文件代替。[符号路径与匹配][symbol-path]

本章使用 **WinDbg (Classic)**，统一启动这个路径下的 x64 程序。

```text
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe
```

在**宿主，普通用户权限**下打开它，通过“Help / About”记录版本。没有这个文件时回到 SDK 安装器补选 Debugging Tools for Windows。微软也提供新版 WinDbg，两者获取方式和部分菜单不同，本章按 Classic 的路径与界面讲解。[调试工具的获取方式][debug-tools]

在“File / Symbol File Path”中填写下面一行并确认。宿主 `C:\KernelLab\Symbols` 必须可写；可先用资源管理器在那里创建再删除一个临时文本文件验证权限。

```text
srv*C:\KernelLab\Symbols*https://msdl.microsoft.com/download/symbols
```

`srv*` 表示使用符号服务器，后面依次是本地缓存和微软公共符号服务器地址。符号由**宿主**下载并缓存，目标不必为了下载符号单独联网。微软公共符号只包含公开发布的内容，不能据此期待得到完整 Windows 源码。[符号服务器用法][symbol-server]、[微软公共符号][public-symbols]

新建调试会话后需要再次检查路径。等 [0.13](#first-observation) 暂停目标时，可以在 WinDbg 命令窗口输入下面两条命令。

```text
.symfix C:\KernelLab\Symbols
.sympath
```

`.symfix` 把符号路径设为微软公共服务器并指定缓存，`.sympath` 不带参数时显示当前符号路径。此时只是配置好查找位置，尚不能宣称目标符号已经加载成功。[.symfix 命令][symfix]

<a id="connect"></a>

## 0.11 建立内核调试连接

连接前要有 `00-clean`，并已完成一次标记文件还原。以下修改仅作用于**目标虚拟机**，不会让宿主进入内核调试。

### 两条路线共用的准备

1. 在**目标机，管理员命令提示符**再次运行 `manage-bde -status C:`。若保护已启用，确认恢复密钥已保存在来宾之外，再执行 `manage-bde -protectors -disable C: -RebootCount 0` 暂停保护器。它不会解密整盘；参数 `0` 表示持续暂停，直到你明确恢复，因而要在记录里写下这项变更。未启用 BitLocker 时跳过。[BitLocker 保护器命令][bitlocker-protectors]
2. 正常关闭目标。在**宿主虚拟机设置**中记录安全启动原值，然后临时关闭来宾安全启动。Hyper-V 位于“设置 / 安全”；VMware 位于“VM / Settings / Options / Advanced”的 UEFI 安全启动设置。保持固件类型为 UEFI，不把它改成 BIOS。
3. 启动目标，再次记录安全启动状态。若触发 BitLocker 恢复界面，使用之前保存的恢复密钥；没有密钥就先退回原安全启动设置或 `00-clean`，不要继续叠加修改。

微软调试文档提示，写入调试启动配置时可能需要暂时处理 BitLocker 和安全启动。本章把这些变化限制在实验来宾，调试结束按 [0.12](#recovery) 恢复。内存完整性按原状态保留，本章不设置测试签名，也不更改宿主的安全启动。[Hyper-V 调试前提][hyperv-kdnet]、[虚拟串口调试前提][serial-debug]

### Hyper-V 使用 KDNET

KDNET 是 Windows 的网络内核调试传输方式，使用用户数据报协议（User Datagram Protocol，UDP）传送调试数据，端口号用于区分接收端的通信入口。`kdnet.exe` 帮你检查目标支持情况并配置连接，普通网卡能联网并不自动表示它支持 KDNET。

1. 在**宿主，普通命令提示符**运行 `ipconfig`，重新读取 `vEthernet (KernelLab-External)` 的 IPv4。记录实际值，下面以 `192.168.50.10` 举例。若你的输出不同，必须替换它。
2. 在**目标，管理员命令提示符**运行。

   ```bat
   cd /d C:\KDNET
   kdnet.exe
   ```

   应看到支持的调试网卡或 Hyper-V 来宾调试支持信息。若报告不支持，先确认两个文件来自同一套 x64 Debugging Tools、目标使用本章 Hyper-V 网卡与交换机。此时不要继续写入配置，也不要凭设备名字猜支持情况。

3. 检查通过后，在同一目标命令窗口执行以下配置命令，将地址换成第 1 步的实际宿主地址。`50005` 是本章选用的宿主 UDP 端口，每个同时调试的目标使用独立端口。

   ```bat
   kdnet.exe 192.168.50.10 50005
   bcdedit /enum {current}
   bcdedit /dbgsettings
   ```

   保存工具生成的密钥，检查当前项的调试开关与网络参数。密钥用于这次调试连接，不复制网上的示例密钥，也不要放到公开截图里。暂不重启。

4. 在**宿主，管理员 PowerShell**增加本章的入站规则。下面的规则只允许同一子网向本章调试器的 UDP 50005 端口发送数据，适用于前面约定的实验局域网。只创建一次；若已有同名规则，用高级防火墙界面检查它。

   ```powershell
   New-NetFirewallRule -Name 'KernelLab-KDNET' -DisplayName 'KernelLab KDNET' -Direction Inbound -Action Allow -Protocol UDP -LocalPort 50005 -RemoteAddress LocalSubnet -Program 'C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe' -Profile Any
   ```

   `Profile Any` 使规则不因实验网卡被识别为公共或专用网络而失效，程序、端口和来源范围仍有限制。记录这条规则，调试结束可以只移除它。[防火墙规则参数][firewall-rule]

5. 在**宿主，管理员命令提示符**启动 WinDbg。下面是带占位符的命令模板，先把 `PASTE_YOUR_KEY` 换成目标刚生成的完整密钥，再执行。

   ```bat
   "C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k net:port=50005,key=PASTE_YOUR_KEY
   ```

6. 宿主调试窗口进入等待连接状态后，在**目标**保存工作，从开始菜单选择重启。回到宿主观察 WinDbg。连接建立后选择“Debug / Break”，应进入可输入命令的内核调试状态，再转到 [0.13](#first-observation)。

以上是微软的自动 KDNET 配置流程，本章选用端口位于微软建议的 `50000` 至 `50039` 范围。[KDNET 自动配置与故障排查][kdnet-auto]

如果一直等待，按顺序检查宿主地址是否变化、目标是否实际重启、两侧端口与密钥是否相同、宿主规则是否生效。KDNET 启用后可能出现 Microsoft Kernel Debug Network Adapter，重新记录接口，不把它当作陌生故障网卡删除。宿主的 VPN 或组织安全策略也可能限制连接，此时记录具体限制，回到可控实验网络处理。Hyper-V 控制台使用基本会话，避免增强会话在目标暂停期间超时。

### VMware 使用虚拟串口

这条路线使用 Windows 的串口内核调试传输 KDCOM。串口通常显示为 COM1、COM2，数字必须与目标实际使用的端口一致。

1. 在**目标机**正常关机。进入**宿主 Workstation 的“VM / Settings / Hardware / Add”**，添加 Serial Port，完成添加后选择 `Output to named pipe`。路径填 `\\.\pipe\KernelLab-W10`，选择 `This end is the server` 与 `The other end is an application`，勾选 `Connect at power on`。服务端由 Workstation 提供，应用一端是宿主 WinDbg。再勾选 `Yield CPU on poll`，减少来宾串口轮询时占用处理器。[Workstation 虚拟串口说明][vmware-serial]
2. 启动目标，在设备管理器的“端口（COM 和 LPT）”检查通信端口编号。本章新建机器的第一个虚拟串口按 COM1 配置；如果实际为 COM2，就把下条命令的 `debugport:1` 改为 `debugport:2`。若无对应端口，先回到关闭状态检查虚拟串口确已添加，不在宿主设备管理器找它。
3. 在**目标，管理员命令提示符**执行。

   ```bat
   bcdedit /debug on
   bcdedit /dbgsettings serial debugport:1 baudrate:115200
   bcdedit /enum {current}
   bcdedit /dbgsettings
   ```

   `serial` 选择串口传输，`debugport` 指来宾 COM 编号，`baudrate` 是串口速率参数。命令成功后应能读到对应配置，若报安全启动策略限制，返回共用准备步骤核对安全启动，而非修改签名设置。

4. 保持目标运行，让 Workstation 创建管道。在**宿主，管理员命令提示符**执行。

   ```bat
   "C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k com:pipe,port=\\.\pipe\KernelLab-W10,baud=115200
   ```

5. 调试器等待后，从**目标 Windows 开始菜单**正常重启。连接成功后在 WinDbg 选择“Debug / Break”，继续 [0.13](#first-observation)。

微软的通用虚拟串口示例还列出了 `reconnect` 与 `resets=0`，但参数说明明确要求 VMware 不使用它们。本章命令已按 VMware 的说明取舍。若使用 Workstation 的重置按钮或关闭再启动了虚拟机，退出这次 WinDbg 会话，待目标重新运行后再启动调试器。[微软虚拟串口参数说明][serial-debug]

若出现管道打不开，先核对目标已开机、串口开机连接已勾选、两侧管道名字一致以及 WinDbg 使用管理员权限。若能打开管道却一直等待，核对 COM 编号、`debug` 开关与重启步骤。Host-only 网络没有互联网不会阻断这条本机管道连接。

VMware 也可以尝试 KDNET，但必须先由 `kdnet.exe` 确认该虚拟网卡受支持，再设计对应网络与防火墙规则。本章的可跟做路线到串口为止，无须为了更快的连接同时配置第二套传输。

<a id="recovery"></a>

## 0.12 启动配置的含义与恢复

现在你已经接触了几个容易混淆的设置。

| 设置 | 它控制什么 | 本章处理 |
| --- | --- | --- |
| `debug` | 当前 Windows 启动项是否启用内核调试 | 连接练习需要开启 |
| `dbgsettings` | 调试使用的传输与连接参数 | 选择 NET 或 SERIAL 其中一条路线 |
| `testsigning` | 是否启用测试签名代码相关加载策略 | 保持原值，本章无须开启 |
| Secure Boot | 固件对启动链的验证 | 只在实验来宾暂时关闭，调试结束恢复原值 |
| BitLocker 保护器 | 磁盘加密密钥的保护与解锁条件 | 仅在已启用时暂停，保留恢复密钥并在结束后恢复 |
| 内存完整性 / HVCI | 基于虚拟化的内核代码完整性保护 | 记录原值，不为本章主动关闭 |

打开 `debug` 不能证明某个驱动可以加载。测试签名也有签名、安全启动和代码完整性等条件，相关机制留到驱动加载章节。本章没有需要执行的 `testsigning on` 命令。[测试签名选项][testsigning]

### 能正常进入目标 Windows 时

需要结束本章调试配置时，先在 WinDbg 输入 `g` 让目标继续运行。然后在**目标，管理员命令提示符**执行。

```bat
bcdedit /debug off
```

从目标开始菜单正常关机，在宿主虚拟机设置中把安全启动恢复为原值，再启动目标。若之前暂停过 BitLocker，确认正常启动后在**目标，管理员命令提示符**恢复保护。

```bat
manage-bde -protectors -enable C:
manage-bde -status C:
bcdedit /enum {current}
```

预期调试开关关闭，BitLocker 保护状态恢复。再核对安全启动与内存完整性记录。原本未开启的保护项不要为了“恢复”擅自改成开启。[BCDEdit 调试开关][bcd-debug]、[BitLocker 恢复保护][bitlocker-protectors]

Hyper-V 路线若不再使用本章端口，在**宿主，管理员 PowerShell**移除本章规则。

```powershell
Remove-NetFirewallRule -Name 'KernelLab-KDNET'
```

保留的 `dbgsettings` 参数不会自行重新打开 `debug`。若需要回到修改前的完整 BCD 与虚拟机硬件状态，使用 `00-clean`；仅关闭调试开关和恢复保护项不等于逐字恢复全部配置。

### 目标看似卡住或无法启动时

先看宿主 WinDbg。如果命令窗口已有内核提示符，尝试 `g`。暂停状态下目标桌面不动是预期现象；如果是实际系统崩溃，`g` 不能修复其原因。

若调试器一直等待且目标停在启动过程中，先检查 [0.11](#connect) 的通道参数。仍无法恢复、来宾又没有需要抢救的新增作业时，在**宿主管理界面**关闭这台实验目标，应用 `00-clean`，再启动。强制关闭会丢失未保存内容，所以日常记录一直放在宿主。

还原后按保存点的网络模式重新核对地址与安全状态，再重新配置调试。宿主防火墙规则、共享文件夹中的真实文件以及来宾外保存的记录不会因为来宾还原而一起回退，需要分别核对。此处不要求初学者在无法启动的来宾上猜测系统分区并离线改 BCD。

<a id="first-observation"></a>

## 0.13 完成第一次内核观察

前置条件是 WinDbg 已连接，并已通过“Debug / Break”暂停目标。以下命令都输入到**宿主 WinDbg 的命令窗口**，不输入 Windows 的 `cmd`。内核提示符可能形如 `0: kd>`，前面的数字取决于当前上下文，不需要照抄。

按顺序执行，每条返回后再输入下一条。

```text
.logopen /t C:\KernelLab\Records\kernel-session.txt
.symfix C:\KernelLab\Symbols
vertarget
lm m nt
.reload /f nt
lmvm nt
.logclose
g
```

| 命令 | 用途 | 应检查的现象与边界 |
| --- | --- | --- |
| `.logopen /t ...` | 把后续调试输出写入宿主文件，`/t` 给文件名附加时间信息 | 查看提示的实际文件名；目录必须已存在且可写 |
| `.symfix ...` | 为本次会话设置微软符号服务器与本地缓存 | 仍可用 `.sympath` 核对路径 |
| `vertarget` | 查看正在调试的目标信息 | 应对应 x64 Windows 10 目标，不能把 WinDbg 自身版本当成目标版本 |
| `lm m nt` | 在模块列表中按名称筛选 Windows 内核模块 `nt` | 应能找到对应模块；`deferred` 表示符号延迟加载 |
| `.reload /f nt` | 要求立即加载 `nt` 的符号 | 首次可能需要下载，等待完成，不强制忽略匹配错误 |
| `lmvm nt` | 详细显示 `nt` 模块信息 | 核对 PDB 加载状态与路径；只有导出符号或仍报加载失败时不能算通过 |
| `.logclose` | 关闭日志文件 | 在宿主确认文件已保存 |
| `g` | 恢复目标执行 | 目标桌面应重新响应，此时一般不能继续输入需要暂停的观察命令 |

命令含义可对照微软的 [vertarget][vertarget]、[lm][lm]、[.reload][reload]、[日志命令][logopen] 与 [g][go] 参考。这里只观察已有系统，没有安装自编驱动。

Windows 10 的一些发布版本共享基础组件，调试器里的内核 build 或模块版本可能出现 `19041` 系列信息。确认 22H2 时同时保留来宾 `winver` 的 `19045.x` 记录，不能要求每个模块都显示 `19045`，也不能因字段不同就强塞另一套符号。[Windows 10 22H2 共享系统核心说明][windows-enablement]

符号失败时，依次检查这几处。

1. **缓存能否写入**。在宿主资源管理器验证 `Symbols` 目录权限与磁盘空间。不能写时更换到当前账户可写的目录，并更新 `.symfix`。
2. **宿主能否下载**。检查宿主网络、时间与代理设置。不要在目标机上反复安装证书来修宿主的符号下载问题。
3. **符号路径是否被其他路径覆盖**。执行 `.sympath` 核对，再依次执行 `!sym noisy`、`.reload /f nt`，阅读失败原因，最后执行 `!sym quiet` 关闭详细输出。公共符号下载可能暂时不可用，保留错误并稍后重试。[符号诊断命令][sym-diagnostics]
4. **文件是否匹配**。遇到不匹配提示，检查是否手工指定了其他版本的 PDB。恢复微软符号路径，移走自己错误指定的文件后重新加载，不使用忽略匹配的选项把错误压下去。

完成观察后，再做一次“Debug / Break”和 `g`，确认暂停与恢复可以重复。记录结果时，写“本次能连接并观察目标内核”就足够。它不能证明 SDK、WDK 配置正确，也不能证明任何反作弊产品会接受这个调试环境。

<a id="usermode"></a>

## 0.14 准备用户态小实验与排错记录

前面检查的是目标内核观察通道。后续章节还要写独立的 C/C++ 小程序，因此需要另外验证编译器、PDB 和源码断点。

### 编译一个独立小程序

这段是**完整的独立用户态 C++ 小实验**，用于检查工具链与断点，符合本教程的用户态练习范围。它不含驱动代码、不与内核驱动通信，也不依赖虚拟机。程序只计算两个整数的和，不涉及需要手动释放的资源；编译或运行失败时按下文检查工具与路径。

在**宿主，普通用户权限**下用编辑器新建 `C:\KernelLab\UserMode\hello.cpp`，确认扩展名确实是 `.cpp`，内容如下。

```cpp
#include <cstdio>

int main()
{
    int left = 20;
    int right = 22;
    int total = left + right;
    std::printf("total=%d\n", total);
    return 0;
}
```

打开**宿主的 x64 Native Tools Command Prompt for VS 2022，普通用户权限**，逐条执行。

```bat
cd /d C:\KernelLab\UserMode
cl /nologo /W4 /Zi /Od /EHsc hello.cpp /Fe:hello.exe /link /DEBUG /PDB:hello.pdb
hello.exe
```

`/W4` 提高编译警告等级，`/Zi` 生成调试信息，`/Od` 关闭优化以便逐行观察，`/EHsc` 是此用户态 C++ 程序的异常处理选项。`/Fe` 指定可执行文件名，`/link` 后面的参数交给链接器，`/DEBUG` 与 `/PDB` 生成并命名最终调试符号。不要把这套用户态选项直接抄到后面的内核代码中。[MSVC 命令行编译][msvc-cli]、[调试信息选项][msvc-zi]

按这段代码计算，预期输出为 `total=42`，目录应出现 `hello.exe` 与 `hello.pdb`。这是预期结果，你的作业记录需保留自己的编译和运行结果。

### 让断点真正停住

在**宿主 Visual Studio，普通用户权限**下操作。

1. 选择“文件 / 打开 / 项目或解决方案”，选中刚生成的 `hello.exe`，以可执行文件方式打开；如果筛选器未显示文件，切换为所有项目文件。
2. 再打开 `hello.cpp`，在 `std::printf` 所在行按 `F9` 设置断点，按 `F5` 开始调试。黄色执行箭头应停在该行执行之前。
3. 打开“调试 / 窗口 / 局部变量”，应能看到 `left` 为 20、`right` 为 22、`total` 为 42。在“调试 / 窗口 / 模块”中查看 `hello.exe` 的符号状态与 `hello.pdb` 路径。
4. 按 `F10` 执行当前语句，再按 `F5` 继续至程序结束。若需要保留输出，回到命令行运行一次。

空心断点通常需要检查源文件是否对应刚构建的程序、PDB 是否匹配、实际启动的 EXE 是否正确。先停止调试，重新编译，再打开同一目录中的 EXE。若提示 `cl` 找不到，回到专用开发命令窗口；若 `Windows.h` 或库缺失，返回 [0.4](#04-安装并检查开发工具)。微软也支持直接调试已有可执行文件。[调试已有程序][vs-exe]、[C++ 调试入门][vs-debug]

### 保存能帮助排错的材料

在宿主 `C:\KernelLab\Records` 新建 `environment.md`，按下表填入真实值。未完成的项目写“未完成”及原因，不把正文预期现象复制成记录。

| 记录项 | 需要保存的内容 |
| --- | --- |
| 宿主与来宾身份 | 两侧主机名、系统版本、完整构建号、架构 |
| 工具组合 | VS、MSVC、SDK、WDK、WinDbg 和虚拟化软件的实际版本与路径 |
| 虚拟机配置 | 代数或 UEFI、vCPU、内存、磁盘、网络模式与接口地址 |
| 安全与启动配置 | 修改前后安全启动、内存完整性、BitLocker 状态，`debug` 与传输类型 |
| 恢复记录 | `00-clean` 创建时间、标记文件还原结果、作业备份位置 |
| 调试记录 | 连接方式、暂停与恢复结果、符号状态、宿主日志文件位置 |
| 用户态检查 | 编译结果、输出、断点处变量与匹配的 PDB 位置 |

密钥和恢复密码单独保管，对外求助只给脱敏记录。遇到问题时先写发生在哪台机器、在哪一步、完整错误是什么，再回到对应位置处理。

| 现象 | 首先回到哪里 | 先查什么 |
| --- | --- | --- |
| 虚拟机不能启动或虚拟化不可用 | 0.2、所选虚拟化路线 | 架构、Windows 版本、固件开关，是否误开嵌套虚拟化 |
| 驱动模板缺失 | 0.4 | VS 实例的 WDK 扩展，与磁盘上完整 WDK 分开核对 |
| WinDbg 一直等待 | 0.11 | 本路线的地址或管道、端口、密钥、权限与重启 |
| 能暂停但符号失败 | 0.10、0.13 | 宿主缓存权限、下载条件、路径和二进制匹配 |
| 目标桌面不响应 | 0.12 | WinDbg 是否暂停目标，能否通过 `g` 恢复 |
| 用户态断点不停 | 0.14 | EXE、源码与 PDB 是否来自同一次编译 |

## 思考题

### 题 1 调试连上了，究竟证明了什么

**前置条件**　读完 0.1、0.4 和 0.13，无须实际搭建环境。

**输入材料**　一个教学场景中，读者能用 `vertarget` 查看目标，`lmvm nt` 显示内核 PDB 已加载，`g` 后目标恢复。但 SDK 的 build number 为 `26100`，WDK 为 `22621`，用户态小程序尚未编译。

**任务**　分别判断内核观察、SDK/WDK 组合和用户态工具链检查是否完成。再解释为何不能把“目标装的是 Windows 10”作为必须安装 `19041` 工具包的理由。

**完成标准**　对三项各给出结论和依据，并指出仍需补哪两类记录。

<details><summary>标准答案与解析</summary>

内核观察已完成题目给出的连接、符号与恢复检查。SDK/WDK 组合不符合 build number 一致的要求，应先按 0.3 选定配对版本，再记录实际安装版本。用户态工具链仍待检查，应补编译、运行与断点记录。

目标 Windows 版本和开发工具版本承担不同职责。WDK 支持的目标范围需要看官方兼容说明，具体接口还要查最低系统要求；目标是 Windows 10 并不要求 SDK、WDK 的数字等于目标系统构建号。常见错误是拿一个成功的调试会话替代所有环境检查，或只看目录名就认为修订号一致。

</details>

### 题 2 桌面停住时先做哪件事

**前置条件**　读完 0.1、0.9、0.12 与 0.13。

**输入材料**　假设目标在点击“Debug / Break”后桌面不响应，宿主 WinDbg 仍能执行 `lm m nt`。另一种情况是目标启动失败，WinDbg 也无法连上，但已有 `00-clean`，新增作业已保存在宿主。

**任务**　分别写出下一步处理方法，说明为何两种情况不采用同一种操作。再推演把调试器也放进目标机后，第一种操作会遇到什么问题。

**完成标准**　区分主动暂停与启动故障，说明 `g`、保存点恢复的作用范围，以及恢复不会回退的至少两项宿主状态。

<details><summary>标准答案与解析</summary>

第一种情况由主动中断引起，先用 `g` 恢复目标，再检查桌面响应。它无须丢弃当前虚拟机状态。若调试器也在目标系统中，目标暂停会妨碍继续操作，所以本章把 WinDbg 放在宿主。

第二种情况先核对连接配置，仍不可恢复时从宿主关闭实验来宾并应用 `00-clean`。它会丢弃保存点之后的来宾变更，已另存宿主的作业不受来宾磁盘还原影响。宿主防火墙规则、宿主上的真实共享文件和符号缓存都不会一起回退，需要单独核对。常见错误是见到目标桌面停住就强制断电，或认为快照能恢复宿主的全部状态。

</details>

## 练习题

### 题 1 留下一份可核对的环境与还原记录

**前置条件**　完成 0.1 至 0.10，选择其中一条虚拟化路线。来宾可正常关机，当前没有需要保留而尚未备份的文件。

**输入材料**　你的真实宿主、目标虚拟机和 0.14 的记录表。

**任务**　填写目标版本、工具版本与安装位置。使用 0.9 已创建的 `00-clean`，在指定来宾本地目录创建标记文件，再关机还原。记录文件还原前后的状态、来宾地址与安全设置；前面已完成还原练习的读者可沿用自己的记录。把修改一次工具版本或网络模式之后需要重新检查的项目也写出来。

**完成标准**　宿主上的记录能区分两台机器，SDK/WDK build number 可核对，标记文件结果有自己的截图或文字记录；说明改版本或改网络后不能直接沿用哪些旧结果。

<details><summary>标准答案与解析</summary>

合格记录包含 Windows 10 22H2 x64 与实际 `19045.x`，以及所选工具的实际版本。标记文件在创建后存在，应用此前保存点后应消失。没有消失时检查保存点时间和文件是否实际在来宾本地磁盘，不能把失败记成通过。

更换 SDK 或 WDK 后要重新检查版本配对、头文件和库位置，再检查用户态编译所选环境。调整网络模式后重新记录两侧地址与接口，Hyper-V KDNET 还要核对宿主地址、规则与连接。符号缓存里已有文件或曾经连上过，都不能替代变化后的检查。

</details>

### 题 2 完成观察，再证明自己能恢复连接

**前置条件**　练习 1 完成，已按 0.11 连上目标，来宾外保存了本路线连接记录和必要恢复信息。

**输入材料**　WinDbg、目标虚拟机与 0.13 的命令序列。

**任务**　记录一次暂停、目标版本、`nt` 符号加载状态和继续执行的结果。随后保存作业、正常关闭来宾，在宿主创建 `01-debug-ready` 保存点。重新启动并连接。最后关机还原到 `00-clean`，先读取启动配置，再按 0.11 重新配置本路线并连接一次。

**完成标准**　留下初次连接与还原 `00-clean` 后重新连接的观察结果，说明哪些调试配置需要重做、哪些宿主文件仍在。日志中至少包含目标版本和符号检查，密钥不出现在公开作业里。

<details><summary>标准答案与解析</summary>

第一次应能暂停目标、加载匹配的内核符号，并在 `g` 后恢复响应。`01-debug-ready` 保留了当时的来宾配置，恢复后仍需核对地址和重开调试会话；VMware 经过关机后要重新打开管道会话。

`00-clean` 创建于调试配置之前，恢复后不能直接假定 `debug` 仍开启。重新读取 BCD 和安全配置，再执行所选路线。Hyper-V 重新生成密钥时，宿主必须使用新值；VMware 若串口是在快照后才添加，还要检查并按需重新添加串口。宿主日志、符号缓存与此前的防火墙规则仍在。

如果只有模块列表而符号未加载成功，环境记录应写“连接完成，符号检查未完成”，按 0.13 排查。不能把 `deferred` 当成匹配已确认，也不能从正文复制一段输出代替本机记录。

</details>

### 题 3 用一次变化检查源码与程序是否对应

**前置条件**　完成 0.14 的编译和断点操作，不需要启动目标虚拟机。

**输入材料**　`hello.cpp`、对应的 `hello.exe` 和 `hello.pdb`。

**任务**　第一次保留断点处三个变量值与命令行输出。停止调试，把 `right` 从 22 改为 23，先保存源码但不编译，在命令行运行旧 `hello.exe` 并记录结果；再执行原编译命令，重新调试并记录变量和输出。

**完成标准**　记录修改源码但未编译、重新编译两种条件下的区别，指出在哪个阶段产生了新的 EXE 与 PDB。程序仍为独立用户态小实验，不增加任何驱动工程。

<details><summary>标准答案与解析</summary>

初次断点处应为 20、22、42，输出 `total=42`。只改源码不会改变已有二进制，所以运行旧 EXE 的预期输出仍为 `total=42`。成功重新编译后，断点处应为 20、23、43，输出 `total=43`，EXE 与链接器生成的 PDB 一起更新。

如果重新编译后还得到旧值，先确认编译成功，再检查运行路径是否指向 `C:\KernelLab\UserMode` 中的新程序。若源码不匹配或断点空心，检查是否仍打开旧的 EXE，或拿了其他构建的 PDB。这个变化只验证本次用户态编译调试对应关系，不能代替内核环境检查。

</details>

## 环境完成检查

在自己的记录中逐项确认即可。需要操作的项目没有完成时，保留未完成标记。

- [ ] 能区分宿主和目标，目标确认为 Windows 10 22H2 x64，并记录完整构建号。
- [ ] 已记录实际工具版本，SDK/WDK build number 一致，头文件、库和 WinDbg 路径可核对。
- [ ] 所选虚拟化路线可用，网络和调试通道参数均来自本机，知道常见失败的检查位置。
- [ ] 已实际还原标记文件，知道怎样应用 `00-clean`，作业保存在来宾还原范围之外。
- [ ] 能暂停目标、检查匹配的内核符号并恢复执行，已保存自己的观察日志。
- [ ] 独立用户态程序可以编译、运行、命中源码断点并检查变量，已完成数值变化练习。
- [ ] 知道怎样关闭目标调试并恢复原安全配置，已记录仍为练习保留的配置变化。

## 本章资料

安装时先看 [WDK 与 VS 兼容表][wdk-versions]，网络调试看 [KDNET 自动配置][kdnet-auto] 与 [Hyper-V 虚拟机说明][hyperv-kdnet]，串口参数看 [微软虚拟机调试说明][serial-debug]。需要回查 VMware 的设备、共享和快照菜单时，使用 [Workstation 官方手册][vmware-manual]。正文各处的链接对应相关条件与命令。

[debug-start]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windows-debugging
[hyperv-requirements]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/host-hardware-requirements
[vs-requirements]: https://learn.microsoft.com/en-us/visualstudio/releases/2022/system-requirements
[windows-download]: https://www.microsoft.com/en-us/software-download/windows10
[vmware-download]: https://knowledge.broadcom.com/external/article/368734
[vs-release]: https://learn.microsoft.com/en-us/visualstudio/releases/2022/release-notes
[wdk-versions]: https://learn.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads
[wdk-install]: https://learn.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk
[vmware-release]: https://blogs.vmware.com/cloud-foundation/2026/05/14/announcing-vmware-workstation-and-fusion-26h1/
[vmware-hosts]: https://knowledge.broadcom.com/external/article/315653
[windows-lifecycle]: https://learn.microsoft.com/en-us/lifecycle/announcements/windows-10-22h2-end-of-support-update
[hyperv-install]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/install-hyper-v
[hyperv-create]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-machine-in-hyper-v
[hyperv-switch]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-switch-for-hyper-v-virtual-machines
[hyperv-kdnet]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-network-debugging-of-a-virtual-machine-host
[vmware-vbs]: https://knowledge.broadcom.com/external/article/315616
[vmware-performance]: https://knowledge.broadcom.com/external/article/417896
[windows-releases]: https://learn.microsoft.com/en-us/windows/release-health/release-information
[memory-integrity]: https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-hvci-enablement
[bcd-debug]: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/bcdedit--debug
[vmware-network]: https://knowledge.broadcom.com/external/article?legacyId=1006480
[copy-vmfile]: https://learn.microsoft.com/en-us/powershell/module/hyper-v/copy-vmfile
[vmware-manual]: https://techdocs2-prod.adobecqms.net/content/dam/broadcom/techdocs/us/en/pdf/vmware/desktop-hypervisors/workstation/vmware-workstation-pro-26h1.pdf
[hyperv-checkpoints]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/checkpoints
[hyperv-export]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/deploy/export-and-import-virtual-machines
[debug-tools]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools
[symbol-path]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/symbol-path
[symbol-server]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/using-a-symbol-server
[public-symbols]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/microsoft-public-symbols
[symfix]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-symfix--set-symbol-store-path-
[bitlocker-protectors]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-protectors
[serial-debug]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/attaching-to-a-virtual-machine--kernel-mode-
[kdnet-auto]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection-automatically
[firewall-rule]: https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule
[testsigning]: https://learn.microsoft.com/en-us/windows-hardware/drivers/install/the-testsigning-boot-configuration-option
[vertarget]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/vertarget--show-target-computer-version-
[lm]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/lm--list-loaded-modules-
[reload]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-reload--reload-module-
[logopen]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-logopen--open-log-file-
[go]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/g--go-
[windows-enablement]: https://support.microsoft.com/en-us/servicing/os/windows-10/2022/06/kb5015684-featured-update-to-windows-10-version-22h2-by-using-an-enablement-package
[msvc-cli]: https://learn.microsoft.com/en-us/cpp/build/walkthrough-compiling-a-native-cpp-program-on-the-command-line?view=msvc-170
[msvc-zi]: https://learn.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format?view=msvc-170
[vs-exe]: https://learn.microsoft.com/en-us/visualstudio/debugger/how-to-debug-an-executable-not-part-of-a-visual-studio-solution?view=vs-2022
[vs-debug]: https://learn.microsoft.com/en-us/visualstudio/debugger/getting-started-with-the-debugger-cpp?view=vs-2022
[msvc-versions]: https://learn.microsoft.com/en-us/cpp/overview/compiler-versions?view=msvc-170
[integration-services]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/manage-hyper-v-integration-services
[sym-diagnostics]: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-sym
[vmware-serial]: https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/26H1/using-vmware-workstation-pro/configuring-and-managing-devices/configuring-virtual-ports/add-a-virtual-serial-port-to-a-virtual-machine.html
[vmware-tools-download]: https://knowledge.broadcom.com/external/article/443308
