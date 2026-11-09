# 无尽冬日桌面助手 Skill

供编程 AI 按具体规格实现 Windows 独立联网桌面助手：正常登录与 Token、自动移民、定点高迁、定时出征及连续加速。

内容包括 SDK 参数转换、两层登录签名、必要协议字段、收包处理、确定的业务行为和输入输出示例。保留少量独立参考代码，应用由使用者的 AI 编写。

## 安装与开始

支持 Skills 的工具可以安装：

~~~text
npx skills add jsgaolx/wjdr-desktop-rebuild
~~~

也可 [下载目录](https://github.com/jsgaolx/wjdr-desktop-rebuild/archive/refs/heads/main.zip)，解压后把目录交给 AI；详细指令见 [通用复刻提示词](通用复刻提示词.md)。

可以直接复制这段话：

> 请读取本目录 SKILL.md，按其中已给出的基线算法、字段字典、功能行为和示例，最小化实现无尽冬日桌面助手。先做正常登录与 Token、公共通信、北京时间，再接自动移民、高迁、出征与连续加速。有现成工程就继续修改。已写清的细节直接采用，只有当前客户端存在具体差异时才重新分析对应部分。不要停在方案说明，不要用演示状态冒充联网成功。先完成代码与使用说明；未经我要求，不启动程序、不测试、不登录或消耗游戏资源。

给 AI 此目录、工作目录和可用的游戏 APK 即可开始；应用配置资源值按教程在本机取得，账号会话在使用时由用户导入或正常登录。

## 内容导航

| 内容 | 用途 |
| --- | --- |
| [SKILL.md](SKILL.md) | 最短阅读顺序和行为约定 |
| [版本与证据](references/baseline.md) | APK 哈希、解包到协议的路径、模块定位 |
| [开发路线](references/implementation.md) | 模块输入输出与阶段成果 |
| [资料获取](references/setup.md) | 官方下载、JADX、配置文件与运行环境 |
| [登录与通信](references/login-communication.md) | SDK 转换和签名、登录字段、Sproto、加密状态 |
| [字段字典说明](references/protocol-schema.md) | 核心/战场 JSON 声明怎样直接使用 |
| [功能实施](references/actions.md) | 移民、高迁、出征与连发规则 |
| [活动战场](references/battlefields.md) | 六种活动的进入、地图、编队和加速差异 |
| [输入输出与完成标准](references/examples.md) | 参数、请求、状态合并与交付清单 |
| [时钟与界面](references/time-ui-delivery.md) | 北京时间、调度、设置、结果与分发 |
| [参考片段](references/snippets.md) | 几个独立机制的短例 |

基线为国服 Android 1.16.11 / baidu_cn。官网新包、其他渠道与热更新可能不同，按文档只检查相关差异。字段资料和实施说明可以降低分析工作量，真实可用性仍取决于当前客户端、有效会话及实际实现；各场景的验证范围应单独报告。
