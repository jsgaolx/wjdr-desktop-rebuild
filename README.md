# 无尽冬日桌面助手 Skill

供 AI 按方案实现 Windows 独立联网桌面助手，包括账号与 Token、自动移民、定点高迁、定时出征和连续加速。

## 使用

支持 Skills 的工具可以安装：

```text
npx skills add jsgaolx/wjdr-desktop-rebuild
```

也可以 [下载技能目录](https://github.com/jsgaolx/wjdr-desktop-rebuild/archive/refs/heads/main.zip)，解压后让 AI 读取 SKILL.md：

> 请使用这个 Skill，结合当前客户端资料，按最小方案实现无尽冬日桌面助手。从检查已有工程和客户端资料开始，按步骤完成登录、时钟、任务与界面。

## 内容

| 文件 | 用途 |
| --- | --- |
| [SKILL.md](SKILL.md) | 实施入口、阅读顺序与功能约定 |
| [开发路线](references/implementation.md) | 模块输入输出、开发顺序和阶段成果 |
| [资料获取](references/setup.md) | 游戏下载、APK 解包、SDK 配置和协议定位 |
| [登录与通信](references/login-communication.md) | 短信、Token、角色、收发与回包匹配 |
| [功能实施](references/actions.md) | 移民、高迁、出征、加速与战场适配 |
| [时钟、界面与交付](references/time-ui-delivery.md) | 北京时间、调度、控件、打包与排查 |
| [参考片段](references/snippets.md) | 三个独立机制的短示例 |
| [实施提示词](通用复刻提示词.md) | 交给其他 AI 的任务指令 |

技能提供详细方案与少量参考代码。AI 按当前客户端核对协议并编写程序；所需资料和配置的获取方法见教程。
