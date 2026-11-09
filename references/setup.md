# 客户端下载、解包与资料定位教程

按以下步骤取得客户端和本地配置。先读 [已分析基线](baseline.md)：相同基线直接使用已有算法和字段摘录，仅在资源值缺失或版本不同处继续取材。

## 一、取得游戏 APK

1. 打开 [无尽冬日国服官网](https://wjdr.cn/#/)，选择“游戏下载”；也可使用 [官方安卓下载入口](https://wjdr-app.campfiregames.cn/download)。
2. 下载 Android APK，保留原文件。上号器 APK 只用来研究其 Token 文本格式，不能替代游戏 APK。
3. 记录下载来源、日期、文件名、版本号、渠道与 SHA-256。Windows PowerShell 7 可用 Get-FileHash -Algorithm SHA256 计算。
4. 在 JADX 的 AndroidManifest.xml 查看包名、版本信息，结合 SDK 初始化和渠道资源记录渠道。热更新脚本可能与 APK 内置版本不同，另记其版本/来源。

使用官方入口取得实际客户端，记录版本和渠道，协议分析以这份资料为准。

## 二、用 JADX 解码资源与查看代码

1. 到 [JADX 官方 Releases](https://github.com/skylot/jadx/releases) 取得 Windows GUI 包；带 JRE 的发行包可省去单独准备 Java。
2. 解压到自己的工具目录，启动发行包内的 GUI，打开游戏 APK。
3. 先看 Resources：AndroidManifest.xml、res/values/strings.xml、res/values/public.xml、assets、lib。
4. 用全局搜索找 SDK 方法、资源名称和协议调用；对目标函数使用交叉引用，追踪它的调用方。
5. 按需导出相关资源或函数，记录导出位置和对应 APK。不要看到一个 Java 反编译错误就断定资源也不能读取。

JADX 能反编译 DEX 并解码 Android 资源，具体支持范围和操作以 [官方说明](https://github.com/skylot/jadx) 为准。普通 ZIP 解压只得到原始 resources.arsc 等文件，不等于已经得到可读 strings.xml。

## 三、SDK 应用配置从哪里来

以 secret_android_game 为资源定位入口，具体操作：

1. 在 JADX 全局搜索 secret_android_game。
2. 打开 Resources 下解码后的 res/values/strings.xml，找同名 string，取标签内的完整字符串。
3. public.xml 的同名条目只有资源 ID；它不是配置值。若只找到 public.xml，继续查 strings.xml 或资源引用。
4. 基线的转换已给出：MD5(UTF-8 的 com.gof.china..) 原始摘要作 AES-128-ECB 密钥，解密资源十六进制字节，移除 PKCS#7 填充，解析 UTF-8 JSON 的 game_id/game_key。完整规则见 [登录规格](login-communication.md)，短例见 [参考片段](snippets.md)。
5. 仅当当前渠道或版本无法按上述规则解析时，再沿 SDK 资源引用查具体差异；不要反复推导已有基线。
6. 若 APK 中没有该名字，沿 SDK 初始化和签名函数向上找实际参数来源；不要写一个同名空文件假装完成。

### sdk-app.properties 是怎样生成的

采用以下本地配置约定：创建 UTF-8 文件 sdk-app.properties，保存取得的资源字符串。它不是 APK 中需直接解压出的同名文件，也不是个人 Token。格式为：

```properties
secret_android_game=<当前 APK 的同名 string 完整值，需自行替换>
```

上面只是格式示例，带尖括号的内容不能拿来登录。读取器取等号后字符串，再按登录规格的 SDK 方法转换为应用参数。不要把 public.xml 的资源编号、fpid 或 session_key 填进去。

最小目录约定为项目根目录 config/sdk-app.properties，读取器和启动说明使用同一路径。报文件不存在时检查实际路径与文件是否已生成，无需从 APK 中寻找同名文件。

不要把实际配置值印进日志或放进公共教程。这里提供来源与生成步骤，不分发某个版本的真实应用配置。

## 四、游戏协议和业务逻辑去哪里找

JADX 主要解决 Java/Android 层；基线游戏逻辑位于 assets/data/luapackage_64_* 的 UnityFS / LuaJIT 资源中。具体到模块和协议常量的提取步骤见 [基线路线](baseline.md)，无需从零猜格式。

| 想找的内容 | 先搜索什么 | 继续追踪什么 |
| --- | --- | --- |
| 短信与会话 | sendCaptcha、loginWithPhoneCode、check_session | HTTP 参数、签名、结果解析 |
| 游戏连接 | 网关域名、登录、socket、Sproto | 建连参数、包头、序号、压缩/加密 |
| 协议声明 | .sproto、c2s、s2c、请求名称 | 字段类型、tag、嵌套结构、响应结构 |
| 迁城/出征入口 | req_world_move_city、req_world_gohome_to_point、march、speedup | UI 选项到请求构造的调用链 |
| 发包公共入口 | NetMsg.SendMsg 或当前版本同职能方法 | 序号、场景通道、写入函数 |
| 道具/兵种 | 301901、610104、soldier_type、lv | 元数据及实际背包/兵力通知 |

用上述名称定位当前版本的对应实现。脚本若被压缩、编译或封装：

1. 找到资源加载器，确认文件头、压缩方式、脚本运行时和版本。
2. 沿加载链得到解包后的脚本或协议声明；Lua/LuaJIT 字节码需要匹配其版本的分析工具，不能交给 JADX 当 Java 反编译。
3. 若参数来自原生库，转到对应库的调用链查找。先确定格式，再决定工具，不凭扩展名套一个“通用解密脚本”。
4. 基线可直接使用附带协议字段摘录，不必先编写通用解包器；其他包的具体加载差异才记为待分析项。

## 五、从一个按钮整理出可实现说明

以“二阶行军加速”为例，沿以下顺序记录：

1. 按钮是否先检查背包，以及钻石选项调用哪个分支。
2. 传入的是行军 ID、角色 ID 还是队伍槽位，是否还需要 owner。
3. 普通地图与活动场景是否调用不同请求。
4. 请求声明中的字段名、tag、类型和嵌套结构。
5. 请求序号怎样与响应匹配，返回码哪里读取，行军到达时间如何更新。
6. 钻石路径是一条请求还是“快捷购买＋使用”；是否真的等待购买回包。
7. UI 防连点间隔与服务端频率限制分别是什么证据，不能互相代替。

整理成一行证据表：

| 版本/渠道 | 操作 | 调用位置 | 请求/响应声明位置 | 字段来源 | 结论/待确认 |
| --- | --- | --- | --- | --- | --- |
| 实际记录 | 二阶加速 | 真实函数位置 | 真实声明位置 | 行军与道具等 | 当前确认内容 |

本 Skill 已提供所需编号、字段、嵌套类型和调用行为；新增动作时按同样方式补齐。文档中的 JSON 风格字段示意用于表达含义，线上仍需按实际二进制协议编码。

## 六、runtime 从哪里来

runtime 是开发者为桌面程序准备的语言运行环境，**不从游戏 APK 解出来**。Python 方案可选择官方 Python 和 Tk 桌面组件，按实现引入必要依赖；另选 .NET 等技术栈则按相应分发方案准备。

先明确程序用哪个解释器/运行时启动，再写安装或打包说明。若做便携包，由开发者打包自己选定的运行环境和依赖；这个 Skill 本身无需附一个 runtime 才能被 AI 阅读。

继续阅读：[登录与通信](login-communication.md) → [业务功能](actions.md)。
