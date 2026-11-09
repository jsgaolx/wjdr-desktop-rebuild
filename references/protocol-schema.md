# 最小协议资料怎样使用

## 两个 JSON 字典

| 文件 | 使用范围 |
| --- | --- |
| [protocol-core.json](protocol-core.json) | SDK 之后的游戏登录、角色、库存、兵力、移民、高迁、普通出征与加速 |
| [protocol-battlefields.json](protocol-battlefields.json) | 活动进入、地图、行军/移动、加速和推送 |
| [metadata.json](metadata.json) | 兵种与英雄名称、必要枚举；不含个人状态 |

这些是声明性资料，AI 可以读取目标消息及依赖类型，或让自己实现的编解码器加载字典。无需把整个 JSON 展开到对话里。普通功能先只用 core；开发某个战场时再按消息名取 battlefields 对应条目。

## 字段含义

messages 按名称索引，每项有 id、direction，以及可选 request / response 类型名。notify_ 的 request 是**服务器推送体**的类型，不能因为它叫 request 就主动发给服务器。

types 按全限定类型名索引，每个字段有：

| 属性 | 含义 |
| --- | --- |
| name | 业务字段名 |
| tag | Sproto 字段编号，从 0 开始；省略字段不导致其他字段编号变化 |
| type | integer/string/boolean/binary/double，或另一结构的全名 |
| array | true 表示数组 |
| arg | 原声明括号内容；null 表示无括号，空字符串表示有空括号 |
| integer 的 arg | 精度位数，例如 "0" 保持整数，"3" 按 1000 缩放 |
| 结构数组的 arg | 映射键语义，例如 "kid"；线上依然是结构元素序列，不是 JSON 对象 |

例如 formation_build.hero_id 是 pair_int_int 数组。应用内部可用字典，但编码前变成 [{"k":1,"v":英雄ID}]；pair_int_int 的 k/v 均是 integer，tag 分别为 0/1。

无 request 定义表示 header 后无请求体；存在 request 定义但字段列表为空，表示一个空结构（u16LE 的 0），两者区别保留。没有传值的普通可选字段直接省略。

## 这是字段子集，不是整个游戏协议

omitted_fields 列出本资料主动省略的字段，通常是与任务无关的展示数据。登录请求只保留既有角色登录需要的字段，购买只保留本方案使用的字段，行军和地图只保留本工具跟踪目标需要的字段。

- 接收时仍然走完整 Sproto 结构解码/跳过流程，未选字段的长度也要消耗，不能让后续字段错位。
- 发送只编码这里声明且本功能确实使用的字段，不擅自补齐省略项。
- 部分结束通知仅靠消息类型判断，字段列表可以为空。
- 两个文件共享的类型声明相同；加载合并时按名字复用。
- 大整数保留 64 位精度。JavaScript 实现不要把所有 ID 无条件转成 Number。
- 需要新增功能时，从当前协议补出具体字段及依赖；不要根据其他场景猜字段。

## 最容易写错的地方

| 操作 | 必须保持的差异 |
| --- | --- |
| 游戏登录 | loginauth 编号 1；不是 SDK HTTP 请求 |
| 角色列表 3602 | list tag 0，code tag 1 |
| 心跳 22 | 响应 time tag 1，没有 code |
| 回本国高迁 5633 | 响应 code tag 3 |
| 普通出征 5701 | 没有 kid 字段 |
| 野怪出征 | extra.monsterid 来自 mapobj.map_monster.id |
| 钻石加速 | 2001 的 id 是道具 ID；加速的 id 才是行军 ID |
| 峡谷移动 15985 | 请求 path 是整数数组；推送行军的 path 是 src/dst 结构数组 |
| 服务器推送 | 通过 header.type 分发；响应通过 session 关联原请求 |

先实现通用 Sproto 编解码，再使用这些字典，不要为每个协议手写一套二进制拼接。帧、压缩、密钥和序号规则见 [登录与通信](login-communication.md)。
