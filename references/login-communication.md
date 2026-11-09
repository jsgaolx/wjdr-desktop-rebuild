# 登录、Token 与通信实施规格

本文给出 [基线版本](baseline.md) 的具体输入、算法和字段。相同基线可直接按此实施；版本不同再比较对应资源和调用处。声明性字段见 [protocol-core.json](protocol-core.json)，读取方式见 [协议资料说明](protocol-schema.md)。

## 一、身份与本地文件

| 名称 | 来源/用途 |
| --- | --- |
| game_id / game_key | APK 的 SDK 应用参数，签 SDK HTTPS 请求 |
| fpid / session_key | 用户正常登录所得账号与会话 |
| uid / role.kid | 游戏角色 ID 与角色本国 |
| city_kid | 城市当前王国，来自当前角色状态，可能不同于本国 |
| battlefield_id | 部分活动的服务器实例 ID；没有此字段就不编造 |
| march.id | 本次实际行军 ID，出征响应或明确匹配的通知提供 |

采用 Python 最小方案时，统一约定项目根目录下 config/sdk-app.properties，UTF-8 内容为：

~~~properties
secret_android_game=<从当前 APK 的 strings.xml 取得的完整值>
~~~

这是开发者生成的配置文件，不是 APK 里原有同名文件。启动器以程序目录确定路径，支持一个显式配置路径覆盖；不要依赖启动时的工作目录，也不要把它固定到作者电脑。其他技术栈保持同一含义即可。

runtime 是所选语言的运行环境，与 SDK 参数无关。目录缺失分别按 [获取教程](setup.md) 处理，不要求找现成完整运行包。

## 二、SDK 参数转换：已给出算法

基线资源 secret_android_game 是十六进制密文。转换顺序为：

1. 取资源完整文本，首尾去空白；按十六进制转字节。
2. key = MD5(UTF8("com.gof.china..")) 的 **16 字节原始摘要**，不是 32 字符十六进制文本。
3. AES-128 / ECB 解密，无 IV。
4. 移除 PKCS#7 填充，再按 UTF-8 解码为 JSON。
5. 取 game_id、game_key，保留为应用参数；不打印真实值。

独立短例见 [参考片段](snippets.md)。不要把资源 ID、个人 Token、加密字符串本身当成 game_key。其他渠道若解码结果不同，回到 SDK 对该资源的引用核对。

## 三、SDK HTTPS 请求

### 固定结构

| 项目 | 基线值 |
| --- | --- |
| 主机 | https://passport-web-gof.campfiregames.cn |
| 客户端路径 | /client_api.php?ver=6 |
| 会话检查路径 | /server_api.php |
| 方法/编码 | POST，application/x-www-form-urlencoded; charset=UTF-8 |
| 公共参数 | game_id=str(解码值)、domain=funsdk、game_version=1.16.11、sdk_version=8.0.5.0 |
| 响应 | JSON status=1 时取 data，否则取 error；这不是游戏 code |

业务参数与公共参数合并后签名，所有值按实际发送的字符串处理：

~~~text
canonical = 按键名升序排列，逐项 key=value，用 & 连接
hex_text = HMAC-SHA256(UTF8(game_key), UTF8(canonical)).hexdigest()
signature = Base64(ASCII(hex_text))
Authorization = "FP " + game_id + ":" + signature
~~~

签名原文不做 URL 转义，空字符串也参与；最终请求体再做 URL 表单编码。不要 Base64 原始 HMAC 字节；此处 Base64 的对象是小写十六进制摘要文本。

### 三种业务参数

以下均与公共参数合并，不额外加入未知字段：

| 业务 | 路径 | 参数 |
| --- | --- | --- |
| 发短信 | client | method=sendCaptcha，mobile=用户手机号，length=验证码位数文本，fpid=""，session_key="" |
| 提交验证码 | client | method=loginWithPhoneCode，mobile=手机号，code=用户输入文本，session_key="" |
| 检查会话 | server | method=check_session，session_key=保存的会话，l=zh_CN，service_name=check_session |

最小界面验证码长度默认 4，对应已有桌面入口；用户需要时可改。发送验证码必须由用户点击，不能为调试自动重发。验证码保留字符串，前导零不丢失。

登录 data 中读取 fpid 与 session_key；check_session 返回的 fpid 应对应当前导入账号。正常会话失效则由用户重新登录，不把导入 Token 当作永久登录。

## 四、Token 导入、复制与导出

格式：fpid----session_key，四个短横线，只在第一次出现处拆分。整段首尾空白可去掉，会话内部字符不改写。

导入创建/更新本地账号，先显示“已导入”；仍用同一会话走正常检查和游戏连接。复制、导出文本是用户主动按钮操作，输出同种格式。会话在本机可用 Windows 当前用户 DPAPI 保存；DPAPI 文件不能当跨电脑账号包。

账号列表用于保存与切换，不默认实现批量登录。日志不写完整 Token、登录签名和应用配置值。

## 五、游戏登录字段和签名

### 5.1 建连

TCP 网关 gofcn-gateway-prod.campfiregames.cn:30101。新连接第一条请求 loginauth，type=1、session=1。

下表字段均来自基线声明；值的选择是已有正常角色登录实现所用参数。未列出的可选字段不发送。

| tag | 字段 | 类型 | 值 |
| --- | --- | --- | --- |
| 0 | sdkToken | string | session_key |
| 1 | account | string | fpid 的字符串 |
| 2 | serverid | integer | 选定角色 kid；仅取角色列表的初次登录可省略 |
| 3 | version | string | 1.16.11 |
| 4 | uid | integer | 选定角色 uid；初次取角色列表可省略 |
| 5 | is_create | boolean | false，不创建角色 |
| 6 | group | integer | 1 |
| 7 | device | string | 参考实现使用 ranking-collector；此为客户端标签，不是角色名 |
| 8 | device_os | string | android |
| 9 | ip | string | 空字符串 |
| 10 | country_code | string | CN |
| 11 | distinct_id | string | 本次生成的 UUID 字符串 |
| 13 | lan | string | zh_cn |
| 18 | package_tag | string | App_android_8_baidu_cn |
| 19 | ts | integer(0) | 当前 UTC Unix 秒，整数 |
| 20 | sign | string | 下述 MD5 小写十六进制 |
| 22 | ad_source | string | baidu_cn |
| 23 | device_id | string | 与 tag 11 相同 UUID |

签名顺序不能换成按字段名排序：

~~~text
parts = [
  "MoZ~+i0HI)2ooFLLC",
  account, sdkToken, uid或空串, device, device_os,
  country_code, distinct_id, lan, version, package_tag, ts
]
sign = MD5(UTF8("|".join(各项转字符串))).hexdigest()
~~~

即 tag 顺序 [1,0,4,7,8,10,11,13,3,18,19]，最前面加上述固定前缀。serverid、device_id 等其他字段不参与此基线签名。ts 转十进制文本，不带小数，不用北京时间字符串。

### 5.2 加载与角色选择

1. 收到登录响应，读取 code(tag0)、serverid(tag1)、uid(tag3)、secret_key(tag14，binary)。立即保存密钥后再处理同一缓冲区的后续加密包。
2. 发 21 req_loaddata，无请求体；响应 code=0 后进入已加载状态。
3. 获取角色列表：发 3602，无请求体；响应 list 在 tag0、code 在 tag1。role_info.name/kid/uid 的 tag 分别为 1/3/4。
4. 初次连接用于列角色，用户选择后关闭该连接，以同一账号会话和选定 uid/kid 新建角色连接，重复登录和加载。
5. 角色连接约每 4 秒发送 22 心跳，无请求体；响应 time 在 tag1，**没有 code 字段**。为心跳分配正常 sequence。

不把手机号、fpid、uid、王国 ID 混为一个值。21 成功后才显示角色已连接，资源初始化的完成状态另记。库存尚未知不冒充 0。

## 六、Sproto 帧与编码

优先使用符合 Sproto 格式的编解码器；游戏字段字典另行加载。[Sproto 官方项目](https://github.com/cloudwu/sproto) 提供格式与实现参考。本节规则也对应基线参考实现，不要求使用某个语言绑定。

### 6.1 帧与包头

~~~text
TCP帧 = uint16_BE(压缩包字节长度) + zero_pack(encode(header) + encode(body))
header:
  type     tag0 integer      请求/推送编号；响应通常省略
  session  tag1 integer      请求关联序号
  userdata tag2 struct:
    crypt_method tag0 integer
    crypt_length tag1 integer
~~~

无请求体时只编码 header；空结构编码为 00 00。外层长度不包含自身两个字节。TCP 接收缓冲处理分包与粘包，不能用一次 recv 代表一帧。

### 6.2 结构编码的必要规则

- 结构开头是 u16LE 的 header word 数量，随后同样数量的 u16LE word，最后是外部数据区。
- 字段 tag 计数器从 −1 开始，每处理一个 word 先加 1。奇数 word 表示再跳过 floor(word/2) 个 tag，没有值。
- 非零偶数 word 表示内联整数 word/2−1，支持 0～32766。
- word=0 表示从外部数据区读 u32LE 长度及该长度的字节，继续推进外部数据指针。
- 其他整数用 4 或 8 字节有符号小端；boolean 为 0/1；string 用 UTF-8；binary 保留原始字节；double 用 8 字节 IEEE754 小端；结构为递归编码。
- 整数数组外部内容是 1 字节元素宽度（4/8）＋连续有符号小端整数；空数组外部内容为空。
- boolean 数组每个元素 1 字节；结构/字符串数组逐元素 u32LE 长度＋内容。数组整体仍有字段外部长度。
- integer(n) 是定点精度，不是数组宽度；结构数组括号里的键不改变线上每个元素的结构布局。

解析 header 要返回实际消耗长度，用该位置切分 body，不能猜固定头长。未知字段仍按其头部和外部长度跳过。

### 6.3 零压缩

每 8 字节为一组，不足补零；mask 的第 i 位表示组内第 i 字节是否非零。普通 mask 后仅写非零字节。mask=255 时后面一字节 n，随后是 (n+1)×8 字节原样块；发送器可先使用 n=0 的单块方式，无需追求最优压缩。接收器必须支持多块。

静态演算例：请求 15858、session=2、kid=42。

~~~text
未压缩 header：02 00 e6 7b 06 00
未压缩 body：  01 00 56 00
零压缩结果：  5d 02 e6 7b 06 01 01 56
完整 TCP 帧： 00 08 5d 02 e6 7b 06 01 01 56
~~~

这只是格式示例，不是在线操作记录，不要为核对格式而发送真实迁入请求。

## 七、接收加密状态

先零解压，解析 header；只对 body 做对应处理。crypt_length 存在时，先截取有效密文字节再解密，末尾补零不能消耗流密码状态。

| crypt_method | 处理 |
| --- | --- |
| 缺失/0 | 明文 |
| 1 | 每包从 secret[0] 开始循环 XOR，out[i]=body[i] XOR secret[i%len(secret)] |
| 2 | 下述 RC4 变体 |

方法 2：

1. 新连接或 secret 改变时，以 secret 原始字节初始化 S=0..255。
2. KSA：j=0；i 遍历 0..255，j=(j+S[i]+secret[i%len])%256，交换 S[i]/S[j]。
3. 每个 body 开始时 i=j=0；每字节 i=(i+1)%256，j=(j+S[i])%256，交换，然后与 S[(S[i]+S[j])%256] 异或。
4. **S 在连续包之间保留，i/j 每包重新归零**。不是标准 RC4 对象一直连续读，也不是每包重建 S。
5. 登录密钥初始化发生在后续包解密之前；断开时清除。完整响应或通知解码前发生异常时，不猜测重新开始同一密钥流。

基线正常任务发送使用明文 body，header 不设置加密 userdata；不要把接收的 XOR/RC4 全局套到所有出站请求。这里的结论限于本文列出的登录与业务请求。

## 八、请求关联和并发

最小实现：一条角色连接、一个接收循环、一个实际写入位置。

1. 写出前分配唯一 sequence，登记操作名、响应类型和所属任务。
2. 写入完整帧，记录写入开始/结束，再增加“已发送”数量。
3. header.type 存在的包按通知/消息编号分发；响应按 header.session 查原请求，不用“下一包就是我的响应”。
4. send 返回 sequence；call 在 send 后等待该序号结果。出征使用 call 取得真实行军；加速循环用 send，结果异步统计。
5. 心跳、地图查询与任务共享序号分配和串行写入；不要让两个线程的帧字节交错。
6. 正常无 code 的响应按声明处理，不能一律要求 code=0。业务失败原样报告返回码。
7. 断开停止该连接尚未发出的任务，不自动重放迁入、高迁或整轮加速。

“连续发送不等返回”不等于关闭接收器；仍需更新背包、钻石、位置、行军和错误统计。
