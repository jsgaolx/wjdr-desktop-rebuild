# 活动战场适配表

只在用户需要活动场景时读取本篇和 [protocol-battlefields.json](protocol-battlefields.json)。普通地图先按 [功能实施](actions.md) 完成；不要为了活动扩展把所有任务做成复杂插件系统。

## 一、入口和实例身份

| 场景键/名称 | 进入 | 地图 | 出征/移动 | 加速 |
| --- | --- | --- | --- | --- |
| arsenal / 兵工厂争夺战 | 11101 | 11118 | 11103 | 11104 |
| league / 兵工厂联赛 | 17201 | 17217 | 17203 | 17204 |
| supremacy / 霜龙霸主 | 16601 | 16622 | 16603 | 16605 |
| frostfire / 霜火矿区 | 13301 | 13302 | 13305 | 13309 |
| brawl / 欢乐大乱斗 | 17431 | 17432 | 17434 | 17435 |
| triangle / 峡谷会战 | 15981 | 15982 | 15985 | 15987 |

六种进入请求在基线均**无请求体**，不是填写王国和自造战场号。服务器按照当前角色的可进入场次处理。

进入成功后保存 city_point、活动时间、state 等实际返回字段；11101 另有 battlefield_id 字符串。其他进入响应没有这个字段时，保留场景键和该次连接的本地上下文 ID，用于区分缓存，不能把它显示成“服务器战场编号”。

进入响应中的 marchs/march_info 是已有行军，可装入本场缓存。改变场景清空旧目标与旧行军引用，不用旧场景行军继续加速。最小实现返回普通地图可直接断开重连；无需自动退出/重新匹配活动。

## 二、地图如何读取

| 场景 | 请求体 | 地图内容在哪里 |
| --- | --- | --- |
| arsenal | 无 | 11118 响应 objs/marchs |
| league | 无 | 17272 推送 objs/marchs；send_begin/send_end 标记分批 |
| brawl | 无 | 17461 推送 objs/marchs |
| supremacy | startpoint、endpoint、session、fulldata=true、lod=0 | 16651 推送 session/datalist/end_flag |
| frostfire | startpoint、endpoint、session、fulldata=true | 13352 推送 session/datalist/end_flag |
| triangle | session | 16021 推送目标，16022 推送行军，均含 session/datalist/end_flag |

区域查询的 startpoint 可取目标 x/y 各减 2（最低 0），endpoint 各加 2，读取目标附近即可。body.session 可用本次请求 sequence，同一原子发送步骤分配并登记；用它筛选对应地图推送，避免把旧查询数据混进新目标。

只有带结束标记的场景才按标记判断完成；普通 code=0 可能只表示查询受理，地图数据尚在推送。地图对象的 point/type/id 字段见字典。

普通地图定点查询仍使用 5617 {kid,point}，无需为了一个坐标实现整张大地图浏览器。

## 三、目标类型到出征动作

数字左侧为地图对象 type，右侧为出征请求的 type，二者不是同一个枚举。

| 场景 | 映射 |
| --- | --- |
| arsenal / league | 201～207→601（建筑）；208→605（武器车间）；1→2（城市） |
| supremacy | 501→1301（宫殿）；502→1304（守卫）；504→1307、505→1309（资源）；1→2 |
| frostfire | 301→803（矿点）；302→805（迸发矿点）；303→804（中心） |
| brawl | 601～604→1401（建筑）；605→1403（资源）；606→1405（空投）；1→2 |
| triangle | 不用此表构造新出征；对已有队列发送建筑路径 |

这张表只覆盖列出的动作，不代表所有活动玩法。霜龙集结不是表中的单队进攻；需要集结时还涉及 prepare_time_index 等输入，不默认加上一个动作编号就宣称完成。

## 四、编队和加速的差异

| 场景 | 出征体 | 加速体 |
| --- | --- | --- |
| arsenal、league、supremacy | type、endpoint、formation；有需要才传 schema 中的 extra | id、itemid、owner=本角色 uid |
| frostfire | type、endpoint、formation 只放 hero_id | id、itemid；没有 owner |
| brawl | type、endpoint、formation；无 extra 字段 | id、itemid；没有 owner |
| triangle | id=本场已有队列，path=建筑 ID 整数数组 | id、itemid；没有 owner |

除 triangle 外，响应 code=0 后从 march_info.id 取真实行军，然后使用本场加速协议。triangle 移动成功仍使用用户选择的已有队列 ID，不等待不存在的新建 march_info。

**峡谷请求 path 是整数数组**，例如 [101,102,105]；推送 triangle_war_march.path 才是带 src/dst 等字段的结构数组。它的目标是建筑路径，不能把王国坐标直接塞入，也不能宣称这里只实现的移动就是创建战场队列。

二阶 301901 用于通常行军；301951 是霜龙集结的另一道具，适用行军 type 为 1303/1306。选择什么道具就发什么 ID，不把两种库存合并。

## 五、推送和时间

| 场景 | 行军新增/更新/删除 | 补充 |
| --- | --- | --- |
| arsenal | 11156 / 11157 / 11158 | 11166 结果通知 |
| league | 17255 / 17256 / 17257 | 17263 结果通知 |
| supremacy | 16663 / 16664 / 16665 | 本人行军 16670/16671/16672；16694 含退出 uid |
| frostfire | 13358 / 13359 / 13360 | 本人行军 13371/13372/13373；13375 结果 |
| brawl | 17455 / 17456 / 17457 | 17459 结果 |
| triangle | 16024 / 16025 / 16026 | 16027 批量更新；16028 结果；16032 含退出 uid |

更新体按 id 合并 change，不能把部分更新覆盖整条记录。存在 owner 时只把本角色队列放进可操作列表；其他玩家的行军可忽略。带 uid 的退出通知先看是否是当前角色。

triangle 的 endtime 是毫秒，转为秒后再统一计算剩余时间；其 state=4 表示战斗。普通 world_march.endtime 使用秒，state=4 为返回，勿共用这一状态文案。

不管活动倒计时采用什么字段，**任务输入、触发和日志仍统一北京时间**。场景倒计时只是状态显示，不覆盖用户设定的本机发送截止点。

活动适配的声明、界面和调用流程完成后，逐项注明实际验证范围；没有真实参加活动，就不要写“所有战场已验证可用”。
