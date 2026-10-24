# PopupGuard
Windows 弹窗拦截与流氓软件识别工具
PopupGuard
Windows 弹窗拦截与流氓软件识别/卸载工具（纯用户态，WinForms）。

目标：拦截流氓软件弹窗、扫描出可疑软件、在用户确认后按分级策略处置（L0 断源 / L1 官方卸载）。

下载
最新版本见 GitHub Releases 页面（本仓库 Releases）。推荐下载：

下载项	说明
PopupGuardSetup-offline.zip	推荐：离线安装包，免 .NET，双击按向导安装
PopupGuard-app.zip	免安装绿色版（自包含单文件），解压双击 PopupGuard.exe
PopupGuardSetup.zip / PopupGuard-app-framework.zip	框架依赖版，需 .NET 8 Desktop Runtime
SHA256SUMS.txt	校验值（Get-FileHash <文件> -Algorithm SHA256）
首次运行未签名程序可能触发 SmartScreen：点“更多信息 → 仍要运行”。

发布流程与清单见 docs/RELEASE-CHECKLIST.md，版本记录见 CHANGELOG.md。

功能
弹窗拦截：监听新窗口创建与前台切换，提取窗口/进程/签名特征，用 JSON 规则库加权评分；达到阈值时可自动隐藏窗口（默认，可恢复）或发送 WM_CLOSE，并在托盘气泡提示。
软件扫描：只读采集五路来源合成一份带风险分的清单——已安装软件（卸载注册表项）、运行中进程、启动项（Run/RunOnce + 启动文件夹）、服务、计划任务；可选用 WinVerifyTrust（含 catalog 目录签名回退）校验数字签名。
分级处置：
L0：禁用服务、禁用计划任务、移除启动项（先备份原始状态，可恢复；不删文件、不杀进程）。
L1：调用软件自带的官方 UninstallString / QuietUninstallString（自动识别 MSI / Inno Setup / NSIS / 自定义安装器，按静默→交互顺序尝试）。
L2：把残留目录移入隔离区（不删除文件）、卸载注册表键先 RegSaveKey 备份再删除；可移回/恢复。
L3：备份后结束勾选进程、强制隔离目录、删除服务 / 计划任务 / 注册表键（默认关闭；默认要求先创建系统还原点，可在设置中显式放开该要求并二次确认后执行）。
自动处置（可选，默认关闭）：命中拦截等级时自动执行 L0（仅禁用服务/任务、移除启动项，不删文件、不结束进程）。
安全护栏：所有处置必须经过计划生成 → SafetyPolicy 校验 → 用户逐项确认；不注入进程、不 hook user32.dll、不写内核驱动、不调用 File.Delete/Directory.Delete。
界面
WinForms 自绘的浅色现代界面（无第三方 UI 库）：深色侧边栏 + 圆角卡片 + 自绘按钮/开关/状态标签，四个页面。

页面	内容
仪表盘	本次拦截数、记录数、高风险软件数、规则引擎状态；最近拦截列表；快捷操作与引擎信息
弹窗记录	全部弹窗历史，支持按进程/标题搜索、按拦截/记录/白名单过滤；选中查看评分、动作、命中依据
软件扫描	五路来源勾选 + 签名校验上限；风险分彩色列表（严重/高/中/低）；L0/L1 处置；开启 L2/L3 后显示“高级清理（L2/L3）”按钮
设置	拦截开关、默认动作、托盘行为、扫描与历史上限、自动 L0 处置、L2/L3 高级开关；保存/重新载入
界面细节与配色见 docs/ui-design.md。

系统要求
Windows 10 1903+ / Windows 11（x64）。
绿色自包含包（PopupGuard-app*.zip）与离线安装包（PopupGuardSetup-offline.zip）不需要安装任何 .NET 运行时。
框架依赖包（PopupGuard-app-framework.zip、PopupGuardSetup.zip）需要 .NET 8 Desktop Runtime；目标机没有时会提示 "You must install or update .NET to run this application"。
编译需要 .NET 8 SDK（Windows + dotnet build）。
绿色版（免安装，推荐）
dist\ 下有三个可直接解压运行的包：

包	体积	说明
PopupGuard-app.zip	约 66 MB	推荐：自包含单文件，免 .NET，兼容 Windows 10 1903+
PopupGuard-app-compat.zip	约 70 MB	自包含多文件，个别杀软对单文件解压更敏感时可用
PopupGuard-app-framework.zip	约 0.3 MB	体积最小，但目标机必须已装 .NET 8 Desktop Runtime
解压后双击 Run-PopupGuard.cmd（推荐）或 PopupGuard.exe：

Run-PopupGuard.cmd 启动后会检查进程是否存活；失败时自动运行 --diagnose 并把诊断结果显示在窗口里，方便反馈问题。
diagnose.cmd：单独查看环境诊断（程序/系统/字体/目录可写性/运行状态），文本同时写入 %TEMP%\PopupGuard\diagnose.txt。
启动日志：%TEMP%\PopupGuard\startup.log；运行日志：%LocalAppData%\PopupGuard\logs（不可写时自动回退到 %TEMP%\PopupGuard\logs）。
若 %LocalAppData% 不可写（受限账户/安全软件），配置、历史、日志会自动回退到 %TEMP%\PopupGuard，界面照常可用。
解压后运行不了的常见原因：① 下的是 framework 包但没装 .NET 8 Desktop Runtime（换 PopupGuard-app.zip）；② SmartScreen/杀软拦截（更多信息→仍要运行，或加白名单）；③ 处置相关操作需要管理员权限（右键以管理员身份运行）。仍无法启动时请把 diagnose.cmd 窗口截图发回。
看不到主界面 / 只看到托盘图标：程序是托盘常驻型，开机自启（--tray）或点窗口关闭按钮后只保留托盘图标。此时再次双击 PopupGuard.exe（或 Run-PopupGuard.cmd）即可把已运行实例的主窗口调到前台（单实例唤醒，不会再弹“已经在运行”的框）。也可以点任务栏右下的 ^ 展开隐藏图标，双击盾牌图标，或右键托盘图标 →「显示主窗口」。
双击后完全没反应（旧进程卡住）：极少数情况下旧版本/已卡死的 PopupGuard 进程会占用单实例，导致新窗口不能显示。处理办法：打开任务管理器 → 详细信息 → 结束所有 PopupGuard.exe 进程，再双击 Run-PopupGuard.cmd。新版本已改为“结束不了旧实例也会照常打开新窗口”，不会卡在看不见的确认框。
构建与运行
bat


rem 编译
scripts\build-app.cmd

rem 运行（开发方式）
dotnet run --project src\PopupGuard.App\PopupGuard.App.csproj

rem 发布单文件（输出 dist\PopupGuard.exe）
scripts\publish-app.cmd

rem 运行单元测试
scripts\run-tests.cmd
L0/L1 处置需要管理员权限：未提权时处置会中止并提示。建议右键 PopupGuard.exe →「以管理员身份运行」。
首次启动会自动加载 rules/default-rules.json（随 exe 复制到 rules\ 目录）。规则缺失时只提示、不拦截。
安装程序
可直接分发的安装 / 卸载程序（自定义 WinForms 安装器，按用户安装、无需管理员）。主程序载荷以 payload.zip 与安装器同目录分发：

bat


rem 框架依赖版：约 0.4 MB，目标机需安装 .NET 8 Desktop Runtime
scripts\build-installer.cmd

rem 离线自包含版：约 121 MB，目标机无需安装 .NET
scripts\build-installer-offline.cmd
产物	说明
dist\PopupGuardSetup-offline.zip	离线自包含安装包（推荐直接分发；解压后运行 exe，无需 .NET）
dist\offline\PopupGuardSetup.exe	离线版安装器（单文件，需与同目录 payload.zip 一起分发）
dist\PopupGuardSetup.zip	框架依赖安装包（需 .NET 8 Desktop Runtime）
dist\installer\PopupGuardSetup.exe	框架依赖版安装器（需同目录 dll 与 payload.zip）
artifacts\payload.zip	主程序载荷（替换后可重新构建安装包）
安装到 %LocalAppData%\Programs\PopupGuard，卸载程序为安装目录下的 PopupGuardSetup.exe（同时注册到「设置 → 应用」）。
卸载不删除任何内容：程序目录与快捷方式移入 %LocalAppData%\PopupGuard\Quarantine，注册表项先导出 JSON 再删除，用户配置/日志/历史保留。
只读自检：PopupGuardSetup.exe --dry-run（打印载荷、路径校验与计划，不写任何内容）。
限制：安装包未做代码签名（SmartScreen 会提示）；框架依赖版需要 .NET 8 Desktop Runtime，离线自包含版不需要；暂不支持 MSI/静默安装。
详细说明见 docs/installer.md。
使用流程
启动后程序常驻托盘，自动开始监听新窗口。
命中规则的弹窗会被隐藏（默认；可在「设置」改为关闭窗口或仅提示），并在托盘气泡显示进程名。
到「软件扫描」页勾选来源后点击「开始扫描」，列表按风险分排序；红色=严重、橙色=高。
勾选可疑项 →「生成处置计划并执行…」→ 在确认对话框中逐项确认 → 执行 L0/L1。
「弹窗记录」页可回看每次拦截的进程、标题、评分与判定依据（即“为什么判为广告”）。
数据与日志位置
内容	路径
配置	%LocalAppData%\PopupGuard\config.json
日志	%LocalAppData%\PopupGuard\logs\popupguard-yyyy-MM-dd.log
弹窗历史	%LocalAppData%\PopupGuard\history\history-yyyy-MM-dd.jsonl
L0/L1 备份	%ProgramData%\PopupGuard\Backups\<planId>\
结构化处置日志	%ProgramData%\PopupGuard\Logs\<yyyy-MM-dd>\<planId>.jsonl
架构
代码块


WindowWatcher (WinEvent 钩子, 只输出 HWND)
      ↓
FeatureExtractor (进程/路径/签名/几何/类名/标题, 失败字段置 null + 错误码)
      ↓
RuleEngine (JSON 规则 + 白名单 exceptions + 加权评分 → ScoreLevel)
      ↓ 达到阈值
WindowActionService (ShowWindow(SW_HIDE) / PostMessage(WM_CLOSE)) + 托盘提示 + 历史

InventoryScanner (只读: 卸载项/进程/启动项/服务/计划任务/签名)
      ↓ InventoryRiskEvaluator (启发式风险分，仅排序提示)
用户勾选 → DisposalPlanner (L0/L1 计划) → SafetyPolicy (硬约束校验) → 确认对话框
      ↓
Disposer (L0: 备份 → 禁用服务/任务/移除启动项 → 读回校验)
          (L1: 安装器识别 → 静默尝试 → 交互回退 → 卸载项复查)
      ↓
DisposalJournal / DisposalBackupStore (可追溯、可回滚)
模块（src/）：

项目	说明
PopupGuard.Core	数据模型、规则引擎、风险评分、处置计划与安全策略（纯逻辑，无系统调用）
PopupGuard.Windows	Win32 P/Invoke、窗口监听、特征提取、签名校验、Scanner、Disposer
PopupGuard.App	WinForms 托盘程序（UI 编排，不含业务判定逻辑）
tests/PopupGuard.Core.Tests	纯逻辑单元测试（自建 [Fact] 运行器，无 NuGet 依赖）
tests/*.TestApp	各模块的命令行验收工具
移动目录遇到“错误码 32 / 5（被占用）”
L2/L3 移动目录前，程序会用 Restart Manager 检测占用进程并自动结束可安全结束的进程（跳过系统进程、Microsoft 签名进程、本程序自身），然后重试移动。若 Restart Manager 被安全软件拦截，会退化为“按进程镜像路径检测”——凡是可执行文件位于目标目录内的进程都会被视为占用者。仍失败时会中止并回滚，日志给出具体进程与错误码。

安全边界（不可违反）
不调用 File.Delete / Directory.Delete；删除一律改为移入隔离区（文件）或备份后删除（注册表键）。
不操作 C:\Windows 及其子目录；不对 Microsoft 签名的文件执行写操作。
L2/L3 默认关闭；任何处置都需用户逐项勾选确认。
默认执行 L2/L3 前必须创建系统还原点，创建失败即中止；设置里可显式开启「无还原点时仍允许 L2/L3」，此时改为风险弹窗二次确认（默认关闭，确认后才执行）。
不注入其他进程、不 hook user32.dll 导出函数、不写内核驱动。
只读模块（WindowWatcher / FeatureExtractor / RuleEngine / Scanner）只申请 PROCESS_QUERY_LIMITED_INFORMATION，不写注册表、不停服务、不删文件。
错误不吞：失败字段置 null 并记录 Win32 错误码或 FeatureError。
验证状态（重要）
已用 .NET SDK 8.0.426 完成编译验证：dotnet build PopupGuard.sln -c Release → 0 错误、0 警告；Core 单元测试 81/81 通过；dotnet publish 已产出 PopupGuard\dist\PopupGuard.exe。
有 .NET 8 SDK 的机器可直接运行 scripts\build-app.cmd；若编译出错，请把错误信息反馈回来修复。
已提供三种免安装绿色包与两种安装包：自包含单文件（66 MB，推荐）、自包含多文件（70 MB）、框架依赖（0.3 MB）；安装包有离线自包含版（127 MB）与框架依赖版（0.4 MB）。详见上文「绿色版」与「安装程序」。
已在干净环境（清空 PATH/DOTNET_ROOT）实测：dist\portable\PopupGuard.exe 可启动并显示主窗口「PopupGuard · 弹窗拦截与流氓软件清理」；离线安装器 --dry-run 退出码 0。
发布为自包含时目标机器无需 .NET；框架依赖包才需要 .NET 8 Desktop Runtime。
已知需在真机验证的点：schtasks /Query /XML 的编码、ChangeServiceConfigW 的 SERVICE_NO_CHANGE 语义、GetWindowLongPtrW(GWLP_HINSTANCE) 对部分 Electron 进程返回 0、UIPI 导致隐藏高权限窗口失败、WinVerifyTrust 对 catalog 签名文件的回退等，详见 docs/。
更新记录
2026-10-24：发布准备 v1.0.0 —— 移除主界面侧边栏左下角的两行文字（版本号与“纯用户态…”说明）；新增 LICENSE（MIT）、CHANGELOG.md、SECURITY.md、GitHub Actions 工作流（CI 构建 + tag 自动发 Release）、docs/RELEASE-CHECKLIST.md 与 docs/release-notes-v1.0.0.md；修复 --diagnose 在单文件发布下的 IL3000 警告（不再读取 Assembly.Location）；重新打包全部绿色包与安装包。
2026-10-23：修复「界面部分文字重合/压字」的问题。界面所有自绘文字（标题/说明、统计卡片、侧边栏、头部、设置行、扫描页与记录页）改为按 Font.Height 动态排版，不再使用固定像素坐标；窗口在高缩放下也不再压字。新增 --page <0-3> 与 --screenshot <路径> 两个自查参数（无交互环境下逐页出图验收）。已重新构建并重新打包全部绿色包与安装包。
2026-10-23：修复「主窗口只显示侧边栏、右侧内容区空白」的问题（MainForm.BuildUi 中 _content 未设置 Dock = DockStyle.Fill，内容区被压缩为默认 200×100）；同时修正侧边栏导航顺序、Logo 文字截断与扫描页两个按钮文字截断。已重新构建并重新打包全部绿色包与安装包。
许可与免责
本项目采用 MIT 许可证。

仅用于个人电脑上识别与清理不想要的软件。处置前请确认目标；L1 会调用第三方卸载程序，其行为由该软件决定。作者不对误操作负责。安全问题请按 SECURITY.md 反馈。

L2/L3 强制清理的前置条件（系统还原点）
默认按硬性约束：L2/L3 必须先创建系统还原点，创建失败会中止且不执行任何删除/移动。

Windows 11 全新安装和虚拟机默认关闭「系统保护」，遇到“创建系统还原点失败（错误码 0）”时，优先按下面的方式开启：

Win+R 运行 SystemPropertiesProtection，打开「系统属性 → 系统保护」；
选中系统盘（通常是 C:）→ 点「配置」→ 选择「启用系统保护」→ 确定；
检查服务（services.msc）：「卷影复制 (VSS)」「Microsoft Software Shadow Copy Provider (swprv)」「系统还原 (SDRSVC)」不要处于“已禁用”；
回到 PopupGuard 重试「高级清理」。
程序会在前置检查中提示是否打开系统保护设置；若创建仍然失败，结果里会附带环境摘要（系统还原开关、SDRSVC/VSS 启动类型、系统盘可用空间），可据此排查。

无系统还原点时继续（可选，默认关闭）
某些机器（虚拟机、精简系统、禁用 VSS 的 OEM 系统）无法使用系统还原。若你确认要在这种环境下执行 L2/L3：

打开 PopupGuard →「设置」→「高级处置（L2/L3）」→ 开启「无还原点时仍允许 L2/L3」→ 保存；
再执行高级清理时，若还原点创建失败，程序会弹出风险确认（默认按钮为“否”），明确告知：
无法通过 Windows 系统还原回滚；
文件仍会移入隔离区、注册表/服务/任务仍会备份，提供程序内回滚；
终止进程、删除服务/计划任务/注册表键属于不可逆部分。
只有你点「是」后才会继续；日志会记录 RestorePointSkipped 警告。
该开关是对原始硬性约束“L2/L3 前必须创建还原点，失败则中止”的显式用户覆盖，默认关闭。不确认风险时请保持关闭，或改用 L0/L1（不需要还原点，可正常使用）。

一键启动 / 环境检查与自动修复工具
打包产物里自带 一键启动-环境检查修复.cmd（调用 PopupGuard-Repair.ps1），用于解决“双击没反应/看不到主界面/环境缺东西”的常见问题：

解压 PopupGuard-app.zip → 双击 一键启动-环境检查修复.cmd；
工具会自动：请求管理员权限（UAC 点“是”）→ 结束卡住的旧 PopupGuard 进程 → 解除文件 Internet 锁定（SmartScreen 误拦）→ 创建数据/隔离区/备份目录 → 检查 .NET 运行时（框架依赖版才需要）→ 检查并尝试开启系统保护（L2/L3 需要还原点）→ 检查系统盘空间 → 启动主程序并在 30 秒内确认主窗口出现；
若主窗口没出现：自动运行 --diagnose 并打印 startup.log 尾部，同时把完整过程写入 %LocalAppData%\PopupGuard\logs\repair-*.log。
命令行用法：powershell -NoProfile -ExecutionPolicy Bypass -File PopupGuard-Repair.ps1 [-AutoFix] [-NoLaunch] [-NoElevate] [-AppPath <exe>]

安全说明：该工具只做环境层修复（结束旧进程、解除文件锁定、建目录、开启系统还原），不删除用户文件、不修改被处置软件的注册表。
