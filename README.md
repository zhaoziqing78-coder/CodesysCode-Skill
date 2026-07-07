WorkBuddy
CODESYS 开发能力
Skill 构成与安装指南
在全新 WorkBuddy 环境下，复现 CODESYS PLC 编程能力所需的全部 Skill 及其安装方法
文档类型 
Skill 安装指南
目标平台 
全新安装的 WorkBuddy（任意操作系统）
Skill 数量 
3 个（1 个自定义 + 2 个内置）
难度 
简单 — 复制文件夹 + 市场安装
WorkBuddy CODESYS 能力 Skill 安装指南    |    第 2 页
目录
序号 
章节 
页码
1 
概述：哪些 Skill 构成了 CODESYS 开发能力 
3
2 
Skill 一：codesys-development（自定义 Skill） 
4
3 
Skill 二：pdf（市场内置） 
7
4 
Skill 三：skill-creator（内置） 
8
5 
安装清单与快速部署 
9
6 
验证测试与故障排查 
10
WorkBuddy CODESYS 能力 Skill 安装指南    |    第 3 页
1. 概述：哪些 Skill 构成了 CODESYS 开发能力
当你在 WorkBuddy 中提出 CODESYS PLC 编程需求时，相关的回答能力由两部分构成：底层 AI 模型
+ 已安装的 Skill。要在另一台新装的 WorkBuddy 上完整复现这套能力，必须部署以下 3 个
Skill：
# 
Skill 名称 
类型 
来源 
用途
1 
codesys-development 
自定义 
手动复制 
CODESYS ST 编程知识、FB
设计模式、凸轮表算法
2 
pdf 
市场 
内置市场 
PDF 文档生成（reportlab），用于输出安装说
明和报告
3 
skill-creator 
内置 
内置 
创建/修改自定义 Skill，需要扩展能力时使用
能力调用流程
下表展示当用户提出 CODESYS 编程需求时，这些 Skill 如何协同工作：
步骤 
调用的 Skill 
动作
1 
codesys-development 
加载 IEC 61131-3 ST 语法规则、FB 设计模式、凸轮表算法
2 
（基础 AI 模型） 
结合用户需求与已加载知识，生成 ST 代码
3 
（内置 Write/Edit
工具） 
将代码写入磁盘（.txt 或 .st 文件）
4 
pdf 
生成 PDF 文档（安装说明、报告等）
5 
skill-creator 
（可选）用户希望扩展能力时创建新 Skill
WorkBuddy CODESYS 能力 Skill 安装指南    |    第 4 页
2. Skill 一：codesys-development（自定义 Skill）
这是最核心的自定义 Skill，提供 CODESYS 编程的全部领域知识，包括 IEC 61131-3 ST
语法、功能块设计模式、凸轮表算法以及可复用的代码模板。
2.1 Skill 文件结构
文件 
说明
SKILL.md 
主定义文件：触发词、知识领域、工作流
references/fb-templates.md 
可直接使用的 FB 代码模板（采集、处理、测试程序）
references/cam-table-patterns.
md 
凸轮表设计模式：采集、拟合、平滑算法
references/st-code-snippets.md 常用 ST 工具代码：边沿检测、状态机、数组操作、类型转换
2.2 覆盖的知识领域
领域 
内容
IEC 61131-3 ST 语法 
VAR_INPUT/OUTPUT/IN_OUT、数据类型、数组、边沿检测、RETURN、注释规范
FB 设计模式 
数据采集 FB、数据处理/拟合 FB、状态机 FB
凸轮表算法 
角度触发采集、平均斜率计算、线性拟合、移动平均平滑
命名规范 
b=BOOL、n=INT、r=REAL、arr=ARRAY、Last/Next/Actual 前缀约定
错误处理 
错误码常量、参数校验、溢出保护、状态标志
VAR_IN_OUT 模式 
大数组按引用传递、调用方声明与绑定方式
双向运动检测 
自动识别正反转、方向感知的触发条件
2.3 安装方法
这是用户级 Skill，在新机器上通过复制文件夹方式安装：
方式 A：直接复制文件夹（推荐）
1. 在源机器上定位 Skill 文件夹：
   C:\Users\<用户名>\.workbuddy\skills\codesys-development\
2. 复制整个 codesys-development 文件夹（含所有子目录）
3. 粘贴到目标机器的 Skill 目录：
   C:\Users\<新用户名>\.workbuddy\skills\codesys-development\
4. 如果 .workbuddy\skills\ 目录不存在，先创建它。
WorkBuddy CODESYS 能力 Skill 安装指南    |    第 5 页
5. 重启 WorkBuddy（或开启新对话），Skill 会被自动识别。
方式 B：压缩传输
1. 在源机器上将 codesys-development 文件夹压缩为 zip：
   右键 codesys-development → 发送到 → 压缩(zipped)文件夹
2. 通过 U 盘、局域网、网盘把 zip 文件传到新机器
3. 在新机器上解压到：
   C:\Users\<用户名>\.workbuddy\skills\
4. 确认目录结构：
   .workbuddy\skills\codesys-development\SKILL.md
   .workbuddy\skills\codesys-development\references\*.md
5. 重启 WorkBuddy。
方式 C：让 WorkBuddy 自动重建
如果无法直接传输文件，可以让新机器上的 WorkBuddy 重新生成该 Skill。
在新机器上开启新对话，输入：
"请帮我在我的用户 Skill 目录下创建一个名为 'codesys-development' 的 Skill。它应覆盖
CODESYS 的 IEC 61131-3 Structured Text 编程，包括 FB
设计模式、凸轮表数据采集与曲线拟合、边沿触发控制、双向运动检测、VAR_IN_OUT
数组绑定以及错误处理。请包含 FB 模板、凸轮表算法、ST 代码片段等参考文件。"
WorkBuddy 会调用内置的 skill-creator 自动生成该 Skill。
WorkBuddy CODESYS 能力 Skill 安装指南    |    第 6 页
3. Skill 二：pdf（市场内置）
该 Skill 提供 PDF 生成与处理能力（基于 Python 的 reportlab、pypdf、pdfplumber
库），用于生成安装说明、技术报告等 PDF 文档。
3.1 主要能力
能力 
依赖库 
用途
创建 PDF 
reportlab 
生成带表格、样式、中文字体的结构化 PDF 文档
合并 PDF 
pypdf 
把多个 PDF 文件合并为一个
拆分 PDF 
pypdf 
把一个 PDF 拆成单页或多页文件
提取文本 
pdfplumber 
从已有 PDF 中抽取文本和表格
加密保护 
pypdf 
为 PDF 添加密码加密
3.2 安装方法
该 Skill 属于 WorkBuddy 内置的 document-skills
市场包，默认应在所有安装中可用。如果没有，按以下方式获取：
方式 A：检查是否已安装
1. 打开 WorkBuddy
2. 输入：/skills
3. 在列表中查找 'pdf'
4. 如果已经存在，无需任何操作
方式 B：从市场安装
1. 在 WorkBuddy 中输入："安装 pdf 这个 Skill"
2. 或使用 find-skills 命令："找一个能生成 PDF 的 Skill"
3. WorkBuddy 会自动搜索市场并安装
方式 C：手动从市场安装
1. 打开 WorkBuddy 设置 → Skills / Connectors
2. 浏览内置市场（cb_teams_marketplace）
3. 找到 'document-skills' 包
4. 安装 'pdf' Skill
5. 重启 WorkBuddy
3.3 Python 依赖
pdf Skill 需要以下 Python 包，WorkBuddy 在首次使用时会自动安装。如需手动安装，执行：
# 
