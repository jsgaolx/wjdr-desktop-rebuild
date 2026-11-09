# 已分析版本与取材路线

## 从这里开始

先使用本目录给出的字段、算法和行为约定实施，再针对不同版本检查差异。无需在每个开发阶段把 APK 从头分析一遍。

本资料是国服 Android **1.16.11 / baidu_cn** 的实施基线，不代表官网当前下载包一定仍是这个版本，也不代表每项活动都已在真实战场验证。

| 项目 | 基线 |
| --- | --- |
| APK 文件名 | wjdr_1-16-11_146_11.apk |
| 文件大小 | 887529913 字节 |
| SHA-256 | F2EE6C4BAA508000F920B44C28159BCC326E549C752698963C7C158C7013CDF9 |
| SDK game_version / sdk_version | 1.16.11 / 8.0.5.0 |
| 登录 package_tag / ad_source | App_android_8_baidu_cn / baidu_cn |
| SDK 主机 | passport-web-gof.campfiregames.cn |
| 游戏网关 | gofcn-gateway-prod.campfiregames.cn:30101 |
| 任务时钟 | 本工具的产品约定：全部北京时间 UTC+8，毫秒控制本机写出时刻 |

官网入口与 JADX 操作见 [资料获取](setup.md)。本仓库不提供 APK、个人会话或真实应用配置值。取得的包与上述哈希不同，只说明不是同一文件；记录差异后检查 SDK 资源、登录、相关业务声明和调用处，不必重做所有模块。

## 可直接使用的内容

- [登录与通信](login-communication.md)：SDK 资源转换、签名、登录字段、帧格式、收包加密状态。
- [协议资料说明](protocol-schema.md)：按编号读取声明的方法、字段子集的含义。
- protocol-core.json：登录、资源、移民、高迁、普通出征与加速的字段声明摘录。
- protocol-battlefields.json：六种活动场景的进入、地图、出征/移动、加速及相关推送摘录。
- metadata.json：三类兵种 ID/等级/名称、英雄名称和必要枚举。它不是账号实际库存。
- [功能实施](actions.md)、[输入输出示例](examples.md)：确定行为与完成标准。
- [战场适配](battlefields.md)：不同活动的必要差异；没有活动需求时不用读取其大字典。

字段字典来自该基线的协议文本；行为说明结合客户端调用处及已有参考实现整理。时间、次数、间隔、手动配兵和不等待单条加速回包，是本工具的明确需求，不要误当游戏原按钮的全部行为。

## APK 到协议文本的具体路径

### 1. SDK 与游戏脚本分开处理

JADX 解码 Android 资源得到 res/values/strings.xml，取 secret_android_game；后续转换已写在登录教程，无需重新推导。

游戏 Lua 模块在 APK 的 assets/data/luapackage_64_* 中，外层为 UnityFS。先列 ZIP 项，再只提取匹配文件。Lua 字节码不交给 JADX 当作 Java 处理。

### 2. UnityFS → TextAsset

优先尝试 [UnityPy 官方项目](https://github.com/K0lb3/UnityPy) 的 bundle / TextAsset 读取功能，保留模块名及原始脚本字节。不要先把字节码当 UTF-8 文本保存。

需要自行定位格式时，基线读取顺序如下：

1. 读取零结尾 UnityFS 标识、u32 大端格式版本、两个零结尾版本字符串。
2. 读取 totalSize(u64BE)、compressedInfoSize(u32BE)、rawInfoSize(u32BE)、flags(u32BE)；格式版本 ≥ 7 时按 16 字节对齐。
3. flags & 0x80 表示块信息在文件末尾，否则紧随头部；flags & 0x3f 表示压缩类型，2/3 使用原始 LZ4 block 解压，不是 LZ4 frame。
4. 解出的块信息先跳过 16 字节哈希，再读块数；每块是原始长度 u32BE、压缩长度 u32BE、标志 u16BE。随后是目录条目数和 offset(u64BE)、size(u64BE)、flags(u32BE)、零结尾名称。
5. flags & 0x200 时数据起点对齐到 16 字节；逐块按记录的长度解压并拼接。
6. 在序列化资源里定位 TextAsset。基线可见的局部布局为：名称长度 u32LE、名称字节、4 字节对齐、脚本长度 u32LE、脚本原始字节。优先用对象读取器；字节扫描只能用于定位，需核对名称及长度边界。

某个块不能解开时，记录该块和原因。用零填充损坏块只能用于局部分析，不能把这种结果宣称为完整解包；本 Skill 的既有字段摘录仍可供其他模块继续实施。

### 3. 两个 LuaJIT 模块 → Sproto 文本

查找下列 TextAsset 名称：

- game_proto_game_proto_c2s
- game_proto_game_proto_s2c

基线脚本头为十六进制 1B 4C 4A 02，即 LuaJIT dump v2。协议正文位于字节码的字符串常量中，**取得声明不需要先恢复全部 Lua 源码**。

解析线索：

- 文件头后 flags 为 ULEB128；未设置 stripped 标志时还含块名长度和块名。
- 每个 prototype 以 ULEB128 长度起始，长度 0 结束。
- prototype 先有 4 个单字节值，再有 GC 常量数、数值常量数、指令数的 ULEB128；调试字段是否存在取决于 stripped 标志。
- 跳过指令区（每条 4 字节）和 upvalue 区（每项 2 字节），按常量种类解码；GC kind ≥ 5 时字符串长度是 kind − 5。表常量、子 prototype 和数值常量仍要按格式跳过，不能简单搜索所有长字符串后拼接。
- 收集两个模块的字符串常量，分别取包含大段 .类型声明和请求编号的协议文本，按 UTF-8 保存为 c2s.sproto、s2c.sproto。基线抽出的长度分别为 1038066、644057 字节，长度仅用于定位同一基线。

字节码格式的权威实现见 [LuaJIT lj_bcread.c](https://github.com/LuaJIT/LuaJIT/blob/v2.1/src/lj_bcread.c)。如包内模块布局变化，按该版本加载器继续定位，不套用固定偏移。

### 4. 更新本 Skill 的字段摘录

从协议文本解析 .类型、嵌套类型、消息编号、request/response；把嵌套类型名展平成唯一全名。只导出本工具需要的字段及依赖类型，保留原 tag、类型、数组属性和参数，不重新编号。

对应协议名字和编号已经在 JSON 中。升级时逐条比较字段和调用函数即可；不需要重新制作完整游戏协议库。

## 客户端证据索引

名称均为基线资源模块名，解出后按名称检索；不使用某台电脑的绝对路径。

| 内容 | 模块/函数定位入口 |
| --- | --- |
| 游戏登录构造与收发 | game_module_login_ProtoHandler、game_module_login_SocketTCP、game_module_login_NetMsg |
| 收包加密 | game_proto_sproto_crypt |
| 迁入 | game_module_logic_module_immigrant_ImmigrantCtrl / ReqMigrantMigrate |
| 迁城与道具购买使用 | game_helper_ItemUseHelper / BuyAndUse；game_module_logic_module_world_WorldPlayerCtrl |
| 行军请求及场景分支 | game_helper_WorldMarchHelper_Request；game_module_logic_module_world_march_ctrl_WorldMarchCtrl |
| 加速界面分支 | game_module_ui_view_world_WorldMarchSpeedUpView |
| 兵力初始值与增减 | game_module_logic_module_soldier_SoldierCtrl / NotifySoldierInit、NotifySoldierUpdate、GetIdleCount |
| 野怪标识 | game_module_ui_view_world_popup_WorldMonsterPopupView / OnBtnAttackClick；game_module_logic_module_world_mapobj_data_MapMonsterData / GetId |

记录升级差异时写“模块＋函数＋字段变化”，不要仅记“协议可能变了”。相同版本实施时直接引用此处结论；只有具体证据冲突的部分才继续分析。
