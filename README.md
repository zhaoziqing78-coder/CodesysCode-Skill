CODESYS 开发技能
概述
本技能为编写 CODESYS IEC 61131-3 代码提供结构化知识，涵盖三种编程语言（ST、LD、FBD）及多个应用领域：

结构化文本（ST）：复杂逻辑、算法、数据处理、状态机
梯形图（LD）：离散逻辑、联锁、电气式控制
功能块图（FBD）：信号流、模拟量处理、闭环控制
运动控制：凸轮表、编码器处理、卷绕/张力控制、伺服同步
通信协议：Modbus TCP/RTU、EtherCAT、PROFINET、OPC UA
可视化：WebVisu HMI 开发、仪表盘元素、报警显示
最佳实践：命名规范、代码结构、项目组织、注释标准
调试与测试：常见错误排查、在线调试、仿真方法
核心知识领域
1. IEC 61131-3 编程语言
1.1 结构化文本（ST）
ST 是编写复杂逻辑和算法的主要语言。关键语法规则：

变量声明：VAR_INPUT、VAR_OUTPUT、VAR_IN_OUT、VAR、VAR_CONSTANT
数据类型：BOOL、INT、UINT、DINT、LREAL、STRING、TIME、数组
ARRAY[0..N] OF LREAL — 定长数组（CODESYS 不支持动态数组）
VAR_IN_OUT 用于按引用传递大型数组（避免内存拷贝）
FUNCTION_BLOCK ... END_FUNCTION_BLOCK 结构
PROGRAM ... END_PROGRAM 结构
边沿检测模式：IF bEnable AND NOT LastEnable THEN ... LastEnable := bEnable;
RETURN 语句用于提前退出
CASE ... OF ... END_CASE 用于状态机
注释：// 单行，(* ... *) 块注释
类型转换：TO_LREAL()、TO_INT()，或 LREAL#nValue 强制转换语法
常见 ST 代码片段（边沿检测、定时器、状态机、环形缓冲区、类型转换），请参阅 references/st-code-snippets.md。

1.2 梯形图（LD）
LD 使用继电器逻辑符号进行离散控制。关键元素：

触点：常开 ---| |---，常闭 ---|/|---
线圈：---( )--- 标准，---(S)--- 置位，---(R)--- 复位
定时器：TON（通电延时），TOF（断电延时），TP（脉冲）
计数器：CTU（递增），CTD（递减），CTUD（增/减）
比较：---[>=]---、---[=]---、---[<>]--- 内联比较块
跳转/标号：---(JMP)--- 配合 Label: 目标
返回：---(RET)--- 用于提前退出 POU
LD 最适用于：安全联锁、电机启停电路、电气工程师易读的顺序逻辑。

LD 编程模式和最佳实践，请参阅 references/ld-fbd-patterns.md。

1.3 功能块图（FBD）
FBD 使用图形化块连接进行信号流逻辑。关键元素：

功能块：AND、OR、NOT、XOR、ADD、SUB、MUL、DIV、GT、LT、GE、LE、EQ、NE
选择块：MUX（多路复用器），SEL（选择器），LIMIT（限幅器）
类型转换块：INT_TO_LREAL、BOOL_TO_INT 等
定时器/计数器块：TON、TOF、TP、CTU 作为图形块
反馈回路：使用 VAR 中间变量打破循环连接
用户功能块：自定义 FB 显示为带命名引脚的图形块
FBD 最适用于：模拟量信号处理、PID 控制链、信号调理、信号流比时序更重要的多输入逻辑。

FBD 编程模式，请参阅 references/ld-fbd-patterns.md。

2. 功能块设计模式
模式 A：数据采集 FB
触发源：外部信号（角度、位置或时间）
数组填充模式：索引递增并检查溢出
边沿触发启停
自动读取当前值作为起点
双向运动检测（使用 ABS 差值，而非基于方向）
模式 B：数据处理 FB
通过 VAR_IN_OUT 获取源数组
通过 VAR_IN_OUT 输出结果数组
使用 bExecute 上升沿触发执行
单周期或多周期计算
无效输入时输出错误码
模式 C：状态机 FB
CASE state OF 模式
状态：IDLE、RUNNING、DONE、ERROR
显式状态转换
复位功能
模式 D：通信 FB
连接管理（连接/断开/重连）
请求-响应周期带超时
带重试计数的错误处理
状态输出供 HMI 显示
模式 E：可视化数据提供者 FB
收集并格式化 HMI 显示数据
提供报警/状态数组
处理来自可视化元素的用户输入
管理页面/画面导航状态
即用型 FB 代码模板，请参阅 references/fb-templates.md。

3. 运动控制
凸轮表 = (X, Y) 对数组，将主轴位置映射到从轴位置：

X 轴：通常为编码器位置或主轴位置
Y 轴：通常为从轴位置或角度
按角度间隔触发：卷绕轴每 N 度，采集编码器位置
平均斜率 = (X[end] - X[0]) / (Y[end] - Y[0])
线性拟合：FitX[i] = X[0] + AvgSlope * (Y[i] - Y[0])
移动平均平滑：N 个相邻点的平均值（N 必须为奇数）
数组最大尺寸：10000（编译时），运行时可通过 MaxArrayLength 配置
凸轮表设计模式、曲线拟合算法和双向采集细节，请参阅 references/cam-table-patterns.md。

4. 通信协议
CODESYS 支持多种工业通信协议。关键模式：

Modbus TCP/RTU
使用 NVL.ModbusTCP_Server / NVL.ModbusRTU_Server 进行设备仿真
使用 SysSocket 或 Modbus FB 库实现自定义客户端/服务端
寄存器映射：保持寄存器（40001+）、输入寄存器（30001+）、线圈（00001+）、离散输入（10001+）
功能码：03（读保持寄存器），06（写单个寄存器），16（写多个寄存器）
EtherCAT
通过设备树中的 EtherCAT Device 进行配置
使用 EtherCAT slaves 配合 ESI/EEPROM 文件
PDO 映射用于周期性数据交换
SDO 访问用于参数读写（非周期性）
PROFINET
通过设备树中的 PROFINET IO Device 进行配置
GSDML 文件描述设备能力
槽位/子槽位配置用于模块映射
实时（RT）和等时同步（IRT）模式
OPC UA
使用 OpcUaServer 设备启用内置 OPC UA 服务器
符号配置：选择要暴露的变量
命名空间映射：PLC 变量自动出现在命名空间下
通过 OPC UA 库中的 FB_OpcUaClient 进行客户端访问
详细通信 FB 模板、寄存器映射示例和协议特定代码模式，请参阅 references/communication-protocols.md。

5. 可视化（WebVisu）
CODESYS WebVisu 实现基于浏览器的 HMI 开发。关键概念：

可视化管理器
在设备树中的"Visualization Manager"下配置
设置 Web 服务器端口（默认 8080）
选择目标：HTML5（现代浏览器）、原生（CODESYS 应用）
可视化元素
基本图形：矩形、圆角矩形、椭圆、多边形、直线、位图
控件：按钮、文本输入、组合框、表格、滑块、仪表
显示：文本字段、柱状图、指示灯（LED 指示器）
特殊：报警表、趋势控件、配方管理
变量绑定
元素通过"Variable"属性绑定到 PLC 变量
使用 %s、%d、%f 格式字符串进行文本显示
动态属性（颜色、可见性、位置）通过变量表达式实现
切换输入："Tap"（点动），"Tap on / Tap off"（交替切换）
常见模式
电机控制：带反馈灯的按钮，绿色=运行，红色=停止
模拟量显示：带高/低限位着色的柱状仪表
报警显示：带确认按钮的报警表
趋势：可配置时间窗口的实时数据曲线
导航：带菜单按钮的页面选择器
详细 WebVisu 开发模式、元素配置示例和 HMI 设计最佳实践，请参阅 references/visualization-guide.md。

6. 最佳实践与规范
变量命名规范
前缀	含义	示例
b	BOOL	bEnable、bDone、bError
n	INT/UINT	nState、nCount
r	REAL/LREAL	rPosition、rAngle
s	STRING	sMessage、sAlarmText
t	TIME	tDelay、tTimeout
arr	ARRAY	arrCamTableX
st	STRUCT	stConfig、stStatus
e	ENUM	eMode、eState
fb	FB 实例	fbTimer、fbModbus
Step	可配置步长	StepInterval
Max	最大限制	MaxArrayLength
Last	上一周期值	LastEnable、LastCapturedY
Actual	当前有效值	ActualLength
Fit	拟合结果	FitLineX、FitSmoothY
Delta	差值	DeltaAngle、AbsDelta
POU 命名规范
前缀	类型	示例
FB_	功能块	FB_CamTableGenerator
FC_	函数	FC_CalculateSlope
PRG_	程序	PRG_Main
M_	方法	M_Reset、M_Start
P_	属性	P_IsRunning
错误处理模式
iecst
VAR CONSTANT
    ERR_OK    : UINT := 0;
    ERR_FULL  : UINT := 1;  // 数组已满
    ERR_PARAM : UINT := 2;  // 无效参数
    ERR_TIMEOUT : UINT := 3; // 超时
    ERR_COMM  : UINT := 4;  // 通信错误
END_VAR

// 逻辑开始处：
IF StepInterval <= 0 THEN
    bError := TRUE;
    ErrorCode := ERR_PARAM;
    RETURN;
END_IF;
完整最佳实践指南（项目结构、注释标准、代码组织、安全注意事项），请参阅 references/best-practices.md。

7. 调试与测试
常见错误类别
类别	症状	典型原因
数组溢出	ActualLength 超出数组边界	写入前缺少边界检查
边沿触发失效	动作每个扫描周期都触发或从不触发	缺少 LastXxx 赋值或顺序错误
VAR_IN_OUT 不匹配	FB 调用时编译错误	调用方数组尺寸小于 FB 声明
类型转换	数值异常	隐式转换截断（LREAL 转 INT）
通信超时	设备无响应	IP/端口错误、网络问题、功能码错误
任务看门狗	PLC 停机，看门狗错误	扫描时间超过任务周期时间
内存溢出	PLC 崩溃或行为异常	单个任务中大型数组过多
在线调试技术
在线更改：PLC 运行时修改代码（无需下载）
写入/强制值：覆盖变量值进行测试
断点：在特定行暂停执行（部分运行时受限）
Trace 追踪：随时间采样记录变量值
调用栈：断点期间查看 POU 调用层次
逻辑分析仪：多通道数字信号记录
仿真与测试
CODESYS Control Win V3：基于 Windows 的软 PLC 仿真
单元测试：使用 CfUnit 库进行自动化 FB 测试
模拟输入：创建 PRG_Test 模拟传感器输入
Trace 对比：记录预期与实际行为进行对比
详细调试工作流、测试策略和常见陷阱解决方案，请参阅 references/debugging-testing.md。

参考文件
文件	内容
references/st-code-snippets.md	常见 ST 代码片段：边沿检测、定时器、状态机、环形缓冲区、类型转换、错误模式
references/fb-templates.md	即用型 FB 代码模板：数据采集、处理、状态机、通信和测试程序
references/cam-table-patterns.md	凸轮表设计模式：数据采集、曲线拟合、双向采集、SoftMotion 集成
references/ld-fbd-patterns.md	LD 和 FBD 编程模式：电机控制、联锁、模拟量处理、信号调理
references/communication-protocols.md	通信 FB：Modbus TCP/RTU、EtherCAT、PROFINET、OPC UA 代码示例和寄存器映射
references/visualization-guide.md	WebVisu 开发：元素配置、变量绑定、HMI 设计模式、报警表
references/best-practices.md	命名规范、项目结构、注释标准、代码组织、安全指南
references/debugging-testing.md	调试技术、仿真方法、常见错误、CfUnit 单元测试
使用工作流
识别任务：用户描述所需的 PLC 功能
确定语言：选择 ST（逻辑/算法）、LD（离散/联锁）或 FBD（信号流/模拟量）
选择模式：匹配相应的设计模式（采集、处理、状态机、通信、可视化）
生成代码：以参考文件中的匹配模板为起点
应用规范：遵循命名规范和错误处理模式
生成测试程序：创建 PRG_Test 模拟输入并验证输出
提供指导：包含参数调试、调试提示和集成说明
推荐调试方法：建议合适的调试技术（Trace、断点、在线更改）
局限性
无法直接编译或测试 CODESYS 代码（无 CODESYS 运行时访问权限）
代码需手动复制到 CODESYS Development System 中
数组尺寸在编译时固定（最大 10000 个元素）
FBD 和 LD 为图形化语言 — 只能提供文本描述和 ST 等效代码
可视化配置基于 GUI — 只能提供配置指导和变量绑定模式，无法提供视觉布局
EtherCAT/PROFINET 配置基于设备树 — 仅靠代码无法配置总线拓扑
复杂运动控制需要先在设备树中配置 SoftMotion，FB 代码才能正常工作
