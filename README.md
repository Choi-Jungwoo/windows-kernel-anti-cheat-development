# Windows Kernel Anti-Cheat Development

如果你写过前后端程序，有 C/C++ 基础，却从未接触过 Windows 内核和驱动开发，这份中文教程就是为你准备的。课程会先带你搭好练习环境，再从机器与操作系统基础讲到驱动加载、用户态通信和反作弊设计。

项目的长期目标，是让读者掌握构建高水平 Windows 内核反作弊驱动所需的知识与工程方法。教程提供原理说明、伪代码和局部 C/C++ 示例，用来解释调用关系、判断条件与资源管理。本项目不提供完整驱动，也不编译、运行或测试驱动。

目前已编写 [第 0 章 从零搭建练习环境](src/chapter-00-environment.md)，包含 Hyper-V 与 VMware 两条路线、工具安装、内核调试、恢复练习和独立用户态小实验。第 1 至 32 章仍处于[详细大纲](src/windows-kernel-driver-intro-outline.md)阶段，正文尚待展开。实际内容可以查看 [教程目录](src/SUMMARY.md)。

[学习速查表](src/cheat-sheet.md) 已收录第 0 章的环境分工、工具与版本规则、调试命令和恢复要点，后续随章节正文补充。

## 开始前需要会什么

这里的“从零”指 Windows 内核知识。你需要能独立编译 C/C++ 程序，知道指针指向什么，能读懂结构体、函数指针和回调，也知道一块内存该由谁释放。这与微软驱动入门文档要求的 C 和回调基础相符。[驱动开发入门](https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/)

用户态与内核态、地址空间、中断请求级别（IRQL）这些知识会在课程里从头讲。你不需要提前会用 WDK、WinDbg 或配置内核调试。遇到新概念时，课程会借助请求处理、异步任务和资源管理等熟悉的经验帮助理解，再说明内核里的额外约束。

## 会学到哪些内容

课程先安排环境搭建，再讲语言、机器与 Windows 系统基础，随后进入驱动生命周期、用户态通信和观测保护机制。每个示例只处理当前要解释的问题，并标出依赖的条件和省略的部分。

| 学到这里 | 主要内容 | 应当理解的问题 |
| --- | --- | --- |
| 搭建练习环境 | Hyper-V 或 VMware、Windows 10、开发工具、WinDbg、符号与恢复 | 怎样安装和配置，如何确认调试连接可用，配置出错后怎样恢复 |
| 理解内核代码 | 执行上下文、内存、对象生命周期与同步 | 一次访问为什么无效，资源应该在何时释放 |
| 理解驱动生命周期 | 构建与签名流程、加载、卸载 | 初始化中途失败时，哪些资源需要清理 |
| 用户态通信 | 设备权限、I/O 控制请求（IOCTL）、缓冲区与并发 | 请求经过哪些检查，畸形输入应该在哪里被拒绝 |
| 反作弊机制 | 威胁模型、进程与映像事件、句柄访问、完整性检查 | 检测依赖哪些前提，什么正常行为也可能触发规则 |
| 系统协作 | 用户态服务、证据记录、检测规则与处置策略 | 检测结果如何影响后续决定，服务退出或事件丢失会造成什么影响 |
| 工程质量 | Driver Verifier、崩溃分析、性能、兼容性与更新 | 实际开发时需要怎样评估效果，以及为故障恢复准备什么 |

代码会注明是伪代码还是局部示例。比如讲用户态请求时，可以用一小段代码说明缓冲区长度不符合要求时如何返回错误，再解释这个检查为什么要放在读取数据之前。微软的驱动安全指南也把访问控制和缓冲区检查列为开发中需要处理的问题。[驱动安全指南](https://learn.microsoft.com/en-us/windows-hardware/drivers/driversecurity/driver-security-checklist)

到了反作弊部分，还要讨论判断能成立到什么程度。教程会给出明确的行为场景，分析正常程序是否可能触发同一条规则，以及攻击者权限改变后，原先的判断是否仍然成立。这些分析会写明假设，示意输出也会与实际运行记录区分开。

每章都会安排思考题与练习题。你会解释一段代码为什么有问题，推演条件变化后的结果，也会做环境操作、系统观察或局部代码练习。每道题的标准答案与解析默认折叠，先自己作答，再展开核对依据和常见错误。

## 教程采用的环境

教程以 Windows 10 22H2 x64 为目标环境基准，虚拟机可选 Hyper-V 或 VMware。[第 0 章](src/chapter-00-environment.md) 以 x64 Windows 宿主讲解两条搭建路线，给出工具版本、安装配置、调试连接和快照恢复步骤，并附有完成检查。正文中的预期现象供读者核对，不代表本项目已进行 Windows 虚拟机或驱动实测。

只阅读原理时可以暂不安装工具；动手练习前，需要完成对应环境步骤。环境练习和用户态小实验不要求编译或加载本项目的驱动。

调试章节会解释主机与目标系统的分工，连接方式参考 [微软的 Windows 调试入门](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windows-debugging)。

讲到驱动加载或系统配置时，正文会交代操作条件和恢复方法。Driver Verifier 可能主动触发系统崩溃，介绍它时也会说明这一点，供读者理解实际开发中的风险。[Driver Verifier 使用说明](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/driver-verifier)

## 本地阅读

也可以直接打开 [在线教程](https://choi-jungwoo.github.io/windows-kernel-anti-cheat-development/)。每次推送到 `main` 分支后，GitHub Actions 会构建并发布更新；在仓库 Actions 页面可以查看部署结果或手动重新发布。

本地使用与 CI 一致的 mdBook 0.5.4。按 [mdBook 安装说明](https://rust-lang.github.io/mdBook/guide/installation.html) 安装后，在仓库根目录运行下面的命令，就能在浏览器里预览。本项目不使用 mise。

```sh
mdbook serve --open
```

需要生成静态页面时运行下面的命令，结果写入 `book/`。[mdBook 命令说明](https://rust-lang.github.io/mdBook/cli/index.html)

```sh
mdbook build
```

## 参与编写

新章节放进 `src/`，并加入 `src/SUMMARY.md`。

提交前运行 `mdbook build`，检查引用的资料与示例逻辑，并确认伪代码和省略内容已经标明。这里只构建教程页面。
