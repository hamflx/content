# 用宏把函数自动提权：elevated 原理与实践

## 写在前面

在 Windows 桌面应用里，偶尔只需要让少量函数跑在管理员权限下。如果为此把整条执行链都换成“以管理员身份运行”，体验很差。`elevated` 提供了两个宏：在不改业务逻辑的情况下，把标记的函数自动拉到管理员权限进程中执行，调用方像普通函数一样收发参数与返回值。

## 适用场景与限制

- 需要零改动迁移到管理员权限的少量函数。
- 期望保持调用栈与业务代码结构不变。
- 只支持 Windows，依赖 `ShellExecute`、命名管道、UDP loopback、NT 系统调用。
- 参数和返回值都要能被 `serde` 序列化/反序列化。

## 核心设计概览

1) **宏包装**：`#[elevated::main]` 在入口插入提权监听逻辑；`#[elevated::elevated]` 为目标函数生成参数编解码与调度代码。宏实现见 `crates/elevated-derive/src/lib.rs`。  
2) **提权代理进程**：首次调用时通过 `ShellExecuteA + "runas"` 启动同一可执行文件的新实例，命令行带 `--elevate-token=...`，形成管理员代理进程，代码在 `crates/elevated/src/privilege.rs`。  
3) **Primary Token 替换**：代理进程收到普通进程的请求后，使用 `NtSetInformationProcess` 将目标子进程的 Primary Token 换成管理员令牌。  
4) **任务派发与 IPC**：普通进程启动一个“暂停的自克隆进程”，等待提权后通过命名管道传入要调用的函数指针与序列化参数，子进程执行后把结果写回。通信协调在 `task.rs` 和 `util.rs`。  
5) **生命周期守护**：代理进程会监听父进程句柄退出后自行退出；子进程挂起时有 `AutoTerminateProcess` 守护避免僵尸进程。

## 运行时流程（逐步拆解）

1. 启动：`#[elevated::main]` 生成的入口调用 `execute_elevation_and_tasks()`，如果发现命令行带 `--elevate-token`，进入代理/子任务模式，否则正常运行主逻辑。  
2. 触发：调用被标记的函数时，宏会先检查当前是否已是管理员。若是则直接执行；否则进入提权流程。  
3. 启动代理：第一次提权请求会通过 `ShellExecuteA` 以 `runas` 启动自身的管理员实例，并建立 UDP 通道。  
4. 准备子进程：普通进程用 `CreateProcessW` 克隆自身，设置 `CREATE_SUSPENDED` 让线程暂停。  
5. 请求提权：将子进程 PID 发给管理员代理，代理复制自己的 Token 并调用 `NtSetInformationProcess` 替换子进程 Primary Token。  
6. 建立管道：普通进程创建命名管道 `\\.\pipe\elevated_task_<pid>`，子进程连上后读取 `TaskInfo`（包含函数指针与 JSON 参数），执行并写回结果。  
7. 收尾：普通进程读取结果、恢复调用栈；子进程退出；代理进程继续等待下次请求。

## 快速上手

1. 在 `Cargo.toml` 添加依赖（本仓库已是 workspace，外部项目按需引入）：  

   ```toml
   elevated = "0.1"
   elevated-derive = "0.1"
   serde = { version = "1", features = ["derive"] }
   ```  

2. 标记入口与目标函数：  

   ```rust
   use elevated::is_elevated;
   use serde::{Deserialize, Serialize};

   #[elevated::main]
   fn main() {
       println!("普通权限：is_elevated={}", is_elevated());
       let pid = admin_right("Hello Rust".to_string());
       println!("管理员进程 pid={pid}");
   }

   #[derive(Serialize, Deserialize, Debug)]
   struct ComplexValue {
       pid: u32,
       args: Vec<String>,
   }

   #[elevated::elevated]
   fn admin_right(msg: String) -> ComplexValue {
       println!("管理员权限：msg={msg}, pid={}", std::process::id());
       ComplexValue { pid: std::process::id(), args: std::env::args().collect() }
   }
   ```  

3. 运行可执行文件，首次触发受控函数时会弹 UAC，同意后函数以管理员权限执行，返回值在原调用点直接可用。

## 代码细节解读

- **命令行封装**：`CommandLineBuilder` 处理空格与引号转义，保证通过 `ShellExecuteA` 传参正确。  
- **IPC 组合**：控制信号走 UDP，本机回环；大 payload 与结果走命名管道。命名管道缓冲 1 MiB，阻塞模式。  
- **进程/线程句柄管理**：`ProcessHandle`、`ThreadHandle` 封装 Win32 句柄的创建、复制、关闭，避免泄漏。`AutoTerminateProcess` 在作用域结束时尝试终止子进程，防止未恢复的挂起线程悬挂。  
- **序列化链路**：宏生成的 `caller` 使用 `serde_json` 编解码 `(args) -> ret`，因此参数和返回值必须可序列化。  
- **令牌操作**：`ProcessToken::duplicate` 拿管理员进程的主令牌，`replace_primary_token` 通过 `NtSetInformationProcess` 注入到目标进程。  
- **ASLR 假设**：任务信息里直接传递函数指针地址，假设同一映像加载基址一致；极端场景下 ASLR 差异可能导致崩溃，这是一项已知风险。

## 常见问题与注意事项

- **全局状态**：被 `#[elevated::elevated]` 标记的函数在新进程运行，全局可变数据可能尚未初始化或与主进程不同步，建议避免依赖。  
- **函数重名**：宏用函数名作为任务 ID，不可重名。  
- **UAC 体验**：第一次调用才会弹 UAC，后续复用同一管理员代理进程，关闭主进程时代理自动退出。  
- **序列化失败**：编解码错误会导致任务失败，确保参数与返回值类型实现 `Serialize`/`DeserializeOwned`。  
- **网络/管道失败**：UDP 仅在本地，通常不会被防火墙拦截；命名管道名为 `\\.\pipe\elevated_task_<pid>`，确保未被其他进程占用。  
- **平台限制**：仅 Windows；Linux/macOS 无法编译。  
- **日志与调试**：提权链路跨进程，建议在调用函数里加清晰日志（PID、is_elevated），或使用 `println!` 与调试器观察子进程。

## 可以扩展的方向

- 在 UDP/管道层增加认证或单向签名，降低同机攻击面。  
- 避免函数指针传递对 ASLR 的假设，可改为模块名 + 符号名解析。  
- 提供错误分级与可配置的超时时间。  
- 可选的“无 UAC”模式（需已在管理员会话内）或基于任务计划程序的持久化代理。

## 结语

`elevated` 用少量宏和 IPC 组合，把“偶尔需要管理员权限”的需求封装为透明调用。理解其运行流程后，能在保持业务代码简洁的同时把权限边界收紧到单个函数。欢迎在实际项目中试用，并结合自身安全要求做适配。***
