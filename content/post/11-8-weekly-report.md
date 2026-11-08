---
title: "11.8学习周报"
date: 2026-11-08
draft: false
slug: "11.8-weekly-report"
tags:
  - C++
  - 洛谷
  - 项目实战
  - 学习周报
categories:
  - 学习
summary: "这周两条主线：一条是把'改配置文件'变成'点两下按钮'，另一条是让整套系统在账户没钱的时候还能活着。两条线最后撞在一起，暴露出同一类问题——看起来在跑，其实早就断了。"
---

# 11.8 学习周报

**周期**：2026-11-02 ～ 2026-11-08

这周两条主线：一条是**把"改配置文件"变成"点两下按钮"**，另一条是**让整套系统在账户没钱的时候还能活着**。两条线最后撞在一起，暴露出同一类问题——**看起来在跑，其实早就断了**。

C++ 只做了一道题（洛谷 P1125 笨小猴），但它把我这周在项目里反复见到的那个毛病又演了一遍：编译能过、样例能过、逻辑全错。

> 这周最值的一句结论：**"能跑"和"在正常工作"是两件事，中间差一次对账。**

* * *

## 一、小鲸鱼：把配置搬进控制台

### 1.1 监听设置上控制台

以前调"要不要回话、回复概率多少、多久最多回一次"，得手改两个 JSON。现在控制台多了一个「监听」页：

| 卡片 | 能改什么 |
| --- | --- |
| 监听会话 | 拉取真实群/好友，勾选要监听的 |
| 私聊 | 开关、回复概率、最短消息间隔 |
| 群聊 | 开关、只在被 @ 时回复、概率、间隔 |
| 主动开口 | 每 N~M 分钟想一次、会不会说话/发说说 |
| 最近判定 | 最近 12 次判定（通过 / 跳过原因） |

落地的关键是一个**闸门插件** `astrbot_plugin_listen_gate`。它的设计要点只有一条：**每条消息都重读配置**。

```python
async def gate(self, event, *args, **kwargs):
    if str(event.get_sender_id()) == str(event.get_self_id()):
        return                       # 自己发的消息不过闸门
    cfg = _load_cfg()                # ← 每来一条消息读一次
    ...
```

所以改完概率、间隔、开关，**不用重启 AstrBot**，下一条消息就生效。这一条省掉了后面无数次"改了没生效"的怀疑。

### 1.2 不再手写群号：直接拉会话

手写群号最容易错一个数字，然后永远不回复，日志里只留一句 `is not in the session allowlist`，很难查。

所以改成从 OneBot 拉真实列表，勾选即可：

```python
res = onebot_call("get_group_list", {}, timeout=10)
for g in ((res.get("data") or {}).get("data") or []):
    groups.append({"id": str(g["group_id"]), "name": g.get("group_name"),
                   "member_count": g.get("member_count")})
```

**这里踩了一个坑**：`onebot_call()` 返回的是 `{"ok": ..., "data": <整个 OneBot 响应>}`，真正的列表在 `data.data` 里。一开始按 `data` 取，拿到的是 dict 的键（字符串），报：

```
AttributeError: 'str' object has no attribute 'get'
```

顺手解决了另一个问题：白名单里同一个群会有三种写法（`snowluma:` / `aiocqhttp:` / 裸群号），界面上会显示成三个"群"。用保序去重解决：

```python
return list(dict.fromkeys(groups)), list(dict.fromkeys(friends)), other
```

### 1.3 每个会话单独一套规则

全局设置只是**默认值**，每个会话可以在行尾的 ⚙ 里单独覆盖：启用、是否必须 @、回复概率、最短间隔。

存进 `overrides`，键是 `<channel>:<id>`：

```json
"overrides": {
  "group:1079837378":   { "enabled": true, "only_when_at": true,
                          "reply_probability": 0.6, "min_interval_seconds": 120 },
  "private:2379871014": { "enabled": true, "reply_probability": 1.0,
                          "min_interval_seconds": 0 }
}
```

插件里做"默认 + 覆盖"合并，只覆盖出现的字段：

```python
def _rule_for(cfg, channel, chat_id):
    rule = dict(cfg[channel])
    override = (cfg.get("overrides") or {}).get("%s:%s" % (channel, chat_id))
    if isinstance(override, dict):
        for key in ("enabled", "only_when_at",
                    "reply_probability", "min_interval_seconds"):
            if key in override:
                rule[key] = override[key]
    return rule
```

合成事件测了 8 项全过。更值得记的是**真实群消息**也验到了：把某个群的单独概率设成 0 之后，群里来消息直接被记成

```
13:42:27  group  1079837378  dropped by probability
```

两个坑：

- **`hidden` 属性会被 CSS 覆盖**。`.lv-sess-cfg{display:flex}` 把 HTML 的 `hidden` 干掉了，导致所有行的规则面板默认全展开，把列表撑爆。必须显式补 `.lv-sess-cfg[hidden]{display:none}`。
- **测试写错 sender**。第一次测私聊覆盖，sender 用了 `105933959`——正好是**机器人自己的 QQ 号**，被插件里"忽略自己发的消息"那条守卫提前 return 了，看起来像"覆盖没生效"。换成普通好友号就对了。这条守卫是好的，错的是测试。

### 1.4 群聊旁观：不 @ 她也得知道你们在聊什么

「只在被 @ 时回复」开了以后有个副作用：**群里说什么她完全不知道**。有人问"群里刚才聊什么"，她一个字都答不上来。

于是加了旁观模式——因"没被 @"而跳过的群消息，仍然写进记忆：

```python
if (channel == "group" and reason == "group: not mentioned"
        and rule.get("observe", True)):
    _observe_group_message(cfg, event, chat_id, text[:200])
```

带限流（每群每分钟最多 20 条），避免活跃群把记忆库灌爆。验证记录：

```
群聊·施阳：今晚八点打副本 谁来     chat=qq:group:1079837378
被拦住(不回复)=True  关掉默认LLM=True
```

**记住但不插嘴**——这才是"旁观"该有的样子。

### 1.5 记忆实时同步：20 分钟 → 20 秒

原来的自动桥是：**20 分钟一轮 + 至少 2 条消息 + 至少 400 字**，再调一次大模型蒸馏成几条。所以这边刚聊完，QQ 那边至少要 20 分钟才知道。

加了条**不用大模型的快车道**：

| | 蒸馏（原） | 快车道（新） |
| --- | --- | --- |
| 间隔 | 20 分钟 | **20 秒** |
| 门槛 | ≥2 条、≥400 字 | 1 条就写 |
| 成本 | 每轮一次大模型调用 | **零调用** |
| 内容 | 模型提炼 45 字 | 原文，各截 240 字 |

这一块我连续踩了**两个 off-by-one**，都值得记：

**第一个**：为了不把"回复还在生成"的那一轮写成半截，我故意留最后一轮到下一跳，但消费进度写成了 `max_seq`——等于把留下的那轮也标成已读，于是**每个会话的最后一轮永久丢失**。

```python
# 错的
fast_state[sid] = {"last_seq": max_seq}
```

**第二个**：改成 `ready[-1]["seq"]` 之后还是漏。因为 `ready` 可能有 14 轮，而 `limit` 只写 10 轮，我却把进度推进到第 14 轮——**中间 4 轮被直接跳过**。

最终改成"只推进到这一批里真正写掉的那一轮"：

```python
batch = ready[:limit]
for turn in batch:
    remember(...)
consumed_seq = int(batch[-1]["seq"])      # ← 关键
fast_state[sid] = {"last_seq": consumed_seq}
```

写了诊断脚本对账（会话最后 15 轮的 seq vs 记忆库里有没有），结果：

```
最后 15 轮里没进记忆的：0 轮
dsh 条目 22 → 69
```

**这次的教训**：进度标记这种东西，一定要能和"实际写进去的东西"对账。不能只信计数器。

### 1.6 余额用完的那天：模型代理

某天，五个服务同时哑掉——记忆不会回忆、她不主动说话、日记不写、B站不挑视频。每个服务只在自己日志里留一行错误：

```
[rerank] 调用失败：HTTP Error 402: Payment Required
```

一查账户：

```json
{"is_available": false, "balance_infos": [{"total_balance": "-0.02"}]}
```

**欠 2 分钱。**

根因很清楚：`memory/rerank`、`companion`、`diary`、`bili`、`bridge` 各自读同一个 `deepseek.key` 直连 `api.deepseek.com`。一个 key 没钱，五个一起断。

于是加了个本地 LLM 代理（端口 8952），所有服务改成指向它，降级只写一处：

```
deepseek API  →  DSH 账号通道（网关 8326）  →  openrouter 免费  →  kilo 免费
    主通道              降级                        免费             免费
```

调用的服务**一行代码都没改**，只把 `base_url` 指过来。发来的 `model` 和 `Authorization` 会被忽略（模型由通道决定）。

实测各通道（这是整件事的前提）：

| 通道 | 结果 |
| --- | --- |
| `deepseek-api`（我的 key） | ❌ 402 Insufficient Balance |
| `deepseek-account/deepseek-flash` | ✅ 1.6 秒 |
| `our-free-model/openrouter/free` | ✅ 2.7 秒 |
| `our-free-model/kilo-auto/free` | ✅ 5.8 秒 |
| `our-free-model/mimo-*` | ❌ 502 |
| `our-free-model/opencode/*` | ❌ 401（要登录） |

`deepseek-account` 这条是 DSH 自己账号走的路，**和那个欠费的 API key 是两条不同的路**，所以 key 欠费它照样能用。

这里踩了四个坑，一个比一个隐蔽：

**坑①：推理模型会把 max_tokens 吃光**

重排给它 2500 token，它把预算全花在思考上，`content` 返回**空串**，还磨了 16 秒。给到 8000 之后：

```
✓ deepseek-account/deepseek-flash  2.5s  内容 332 字  推理 620 字
```

所以每个通道可以配 `min_max_tokens`，由代理把请求上限抬上去。

**坑②：重排自己的超时比降级链还短**

`rerank.timeout_seconds` 原来是 10 秒，而整条链（402 → 账号通道 → 免费档）根本跑不完 → 每次全超时。改成 30 秒，同时把候选数 20→12、片段 64→48，prompt 短了推理也快。

**坑③：空 content 判失败，会把工具调用丢掉**

这是最要命的一个。AstrBot 每次请求都带 `tools`，模型决定调函数时，**`content` 是空串**，只有 `tool_calls`。代理把"空 content"当失败 → 丢掉工具调用 → 换下一个通道。

后果：她的记忆检索、本机控制、发说说**全都调不出来**。就算她能说话了，也只是个只会聊天的哑巴。

```python
# 只调函数不产文本时 content 是空的 —— 那是成功
if not content and tool_calls:
    return True, parsed
```

修之前测出来是 `通道=free-openrouter`（被降级），修之后是 `通道=dsh-account, tool_calls=1, 降级 0`。

**坑④：思考内容漏进正文**

验证时她发了这么一段：

```
🤔 思考: The user repeated the same question. She asked "在吗？..."
```

`reasoning_content` 被原样转给 AstrBot，它当成正文片段存下来了。这要是发到 QQ，就是一大段英文内心戏。在代理里剥掉即可。

### 1.7 AstrBot 才是真正断掉的那个

我一开始以为"AstrBot 走的是另一个 provider，所以私聊还能回"——**这句是错的**。

查它数据库里每次调用的记录，时间线非常清楚：

```
11-04 06:43  status: completed      ← 最后一次成功
11-04 06:44  status: error
11-04 12:21  status: error  (连续 5 次)
11-04 14:10  status: error  (连续 3 次)
11-04 14:49  status: error
```

**她从早上 6:44 起就一条都回不出来了**，每次被叫到都固定回：

```
LLM 响应错误: All chat models failed: APIStatusError: 402 - Insufficient Balance
```

根因在配置里：

```json
"provider_sources": [{
  "id": "deepseek",
  "api_base": "https://api.deepseek.com/v1",   ← 直连，绕过了代理
  "key": ["sk-2da..."]
}]
```

改成 `http://127.0.0.1:8952/v1`、timeout 180，重启后她立刻恢复。

**端到端验证**：绕开 QQ，用 AstrBot 自己的 OpenAPI 注入一条消息（WebUI 登录拿 token → `POST /api/v1/chat`），再查它数据库里存的回复：

```
[43] 你: 你现在能调用工具吗？
[44] 她: 能呀
[45] 她: 想让我干啥
```

代理那边同步记录：3 次请求，降级 0 次，全走 `dsh-account`。

> 这段最值得记：**"应该是另一条路"这种推断，必须拿数据验一遍**。

### 1.8 健康页在数老错误

健康页长期挂着两条红字：

```
B站谷时刷   日志有近期硬错误 95 条
AstrBot    日志有近期硬错误 32 条
```

两个服务明明在跑、点开也正常。

根因：健康检查扫日志尾部 400 行，而 bili 的日志**不带时间戳**，所以"只看最近 10 分钟"那套过滤对它完全失效；几小时前记忆库重启时刷的 95 条 `WinError 10061` 就一直躺在窗口里，**每 5 秒被数一遍，永远消不掉**。

AstrBot 那 32 条更冤——是每次被 `kill` 重启时，`asyncio.run` 打的**退出堆栈**：

```
File "...\astrbot\main.py", line 176, in <module>
    asyncio.run(main_async(args.webui_dir))
```

那不是运行期故障，是进程在退出。

改法：给每个服务记一个"日志写到哪了"的标记，只统计标记之后的错误。

```python
rotate_log(log_path(self.id))
set_log_marker(self.id)      # data/log_markers.json 记下当前文件大小

# 检查时从标记处往后读
text = _tail_text_after(path, markers[sid]["offset"])
```

改完 `ok=9 warn=0 down=0`，九宫格全绿。

顺手加固了两处：`/api/health` 偶尔 500（已包兜底，连打 10 次 0 失败）；点「优先」时代理重启会让面板闪"读不到代理状态"（改成等服务就绪再返回，实测 2.4 秒完成）。

### 1.9 开机自启：两个开关合成一个

原来"开机自启"分散在两处：启动文件夹的快捷方式、以及 `console/config.json` 里的 `autoStart`。最容易只开一半。

现在合成一个开关，`enabled` 必须两半都满足：

```python
"enabled": exists and auto_start,
"shortcut_exists": exists,     # 供排查
"auto_start": auto_start,
```

所以"快捷方式还在但 autoStart 被关掉"这种半开状态，开关会显示成关，拨一下就把两半补齐。

**这里有个特别阴的坑**：开关做出来第一次就翻车——**页面一加载，自启自己变成关闭，快捷方式被删了**。

原因是浏览器**还原表单状态**时会改 checkbox 并触发 `change` 事件，前端把它当成了用户点击，直接发了 `POST {enabled:false}`。修法是记住服务器真状态做比对：

```js
let LAST_AUTOSTART = null;      // loadAutostart() 里同步

swAutostart.addEventListener('change', () => {
  if (LAST_AUTOSTART !== null && swAutostart.checked === LAST_AUTOSTART) return;
  toggleAutostart(swAutostart.checked);
});
```

和服务器一致就忽略——宁可漏一次点击，也不能让自启自己把自己关掉。

### 1.10 对话历史被注入内容撑爆

这一节是"隔壁工作区"踩过的同一个坑——**对话历史里存了大量注入内容**。

起因是查别的东西，顺手看了 AstrBot 的数据库，主对话（跟爸的 QQ 私聊）：

| 角色 | 字符 | 占比 | 条数 | 平均 |
| --- | --- | --- | --- | --- |
| **user** | **3,490,013** | **97.4%** | 150 | 23,266 |
| assistant | 49,119 | 1.4% | 213 | 230 |
| tool | 44,424 | 1.2% | 87 | 510 |

**450 条消息，3.6 MB。** 正常聊天不可能这么大，继续往下拆 user 消息的片段类型：

```
image_url   3,456,664 字符  (99.5%)
text           18,385 字符  (0.5%)
```

真相：每张 QQ 图片都被存成了完整的 `data:image/jpeg;base64,...`，**一条 17~36 万字符**。单条最大 **361,933 字符**。再外加每轮一条：

```html
<system_reminder>Current datetime: 2026-10-31 17:50 (CST), Weekday: Saturday</system_reminder>
```

150 轮就是 150 条。这些都跟对话内容无关，全是**注入物**。

这也解释了另一个早就觉得奇怪的现象：同一个会话的 `token_input_other` 从最初的 1.2 万，一路涨到 **4.4 万**。历史在背着自己长大。

**两个配置把这件事放大**：

| 键 | 原值 | 改成 | 作用 |
| --- | --- | --- | --- |
| `agent_runner.config.misc.sanitize_context_by_modalities` | false | **true** | 每次请求前按模型能力删掉不支持的图片（`deepseek-chat` 没视觉能力，发了也白占 token） |
| `agent_runner.config.compression.max_turns` | -1（不限） | **40** | 只保留最近 40 轮，超出按既有策略压缩 |
| `compression.trim_turns` | 1 | **10** | 一次丢 10 轮，别频繁触发压缩 |

**清理只做减法**（`tools/clean_chat_history.py`）：`image_url` 片段换成一行占位
`[图片：已从历史清理]`，文本里的 `<system_reminder>…</system_reminder>` 删掉。真实对话内容一个字不动。

```
对话 7ccb5b98: 3627276 → 109307 字符（清掉图片 20 张、提醒 150 条）
合计: 3653834 → 128912 字符（减少 3.36 MB）
残留 base64 图片 0 处，残留 system_reminder 0 处
```

还有一步容易忘：**SQLite 删了行不会自己缩文件**。要 `VACUUM`：

```
VACUUM 前 7.30 MB → 后 0.48 MB
```

清理后她照常回话（`平安呀` / `爸，你放心，休息好，晚安`）——因为**她记得的事在记忆库里**（705 条，每句话都会检索注入），不靠这段原始历史。

> 一句话：**"存下来"是最容易被忽略的成本。** 消息发出去那一刻就结束了，但历史会一直背着它。

### 1.11 本周数据

| 指标 | 数值 |
| --- | --- |
| 服务数 | 9 个（模型代理 8952 / 记忆库 8940 / B站 8945 / 日记 8946 / 本机控制 8947 / 自动桥 8948 / 主动开口 8951 / AstrBot 6185+6199 / SnowLuma 5099+6727） |
| 记忆条目 | 705 条（微信 285 / QQ 316 / 电脑 69 / 日记 34） |
| 主动开口 | 累计 85 轮，模型调用 80 次，说过 38 次话、发过 10 条说说 |
| 记忆同步 | 快车道 20 秒一轮，零模型调用 |
| 模型通道 | 4 档降级链，5/5 服务已接线 |
| 对话历史 | 主对话 **3.63 MB → 107 KB**（清掉 20 张 base64 图片） |
| 数据库 | **7.30 MB → 0.48 MB**（VACUUM 后） |
| 健康检查 | 9/9 全绿 |

* * *

## 二、C++：洛谷 P1125 笨小猴

### 2.1 题目

给定一个单词（只含小写字母，长度 < 100），统计每个字母出现次数。

设出现最多的次数是 `maxn`，最少的是 `minn`（**没出现的字母不算**）。若 `maxn - minn` 是质数，输出 `Lucky Word` 和这个差；否则输出 `No Answer` 和 `0`。

样例：`error` → 最多 r 出现 3 次，最少 1 次，3−1=2 是质数 → `Lucky Word` / `2`。

### 2.2 我写的第一版

```cpp
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int a[26];
    int maxn=0,minn=1;
    string s;
    cin>>s;
    for(int i=0;i<s.size();i++)
    {
        a[s[i]-97]++;
    }
    for(int i=0;i<26;i++)
    {
        if(a[i]>maxn) maxn=a[i];
        if(a[i]<minn && a[i]!=0) minn=a[i];
    }
    int t=maxn-minn,z=0;
    if(t==1||t==2||t==3) z=1;
    for(int i=2;i<t;i++)
    {
        if(t%i==0) {z=1;break;}
    }
    if(z==1) cout<<"Lucky Word"<<endl<<t;
    else cout<<"No Answer"<<endl<<"0";
    return 0;
}
```

两个样例都过。但我把它拷回本机编译跑，输出是：

```
输入 error        期望 Lucky Word/2   | 我的 Lucky Word/902311546
输入 olympic      期望 No Answer/0    | 我的 Lucky Word/902704763
```

**`902311546`** —— 这串数字就是第一节课。

### 2.3 三个 bug

**① `int a[26];` 没初始化**

栈上是垃圾值，`a[s[i]-97]++` 是在垃圾上累加，所以 `maxn - minn` 会变成九亿多。

```cpp
int a[26] = {0};
```

为什么在洛谷上"有时候看起来是对的"？因为那块栈恰好是 0。换个环境、换次编译就变。**能用是运气，不是正确。**

**② `int minn=1;` 初值太小**

出现的字母计数**都 ≥1**，所以 `a[i] < 1` 永远不成立——**minn 永远停在 1**。

- `aaaabb`（a=4, b=2）真实 min 是 2，算出来 t = 4−1 = **3**
- `aabb`（2,2）真实 t=0，算出来 t=1

```cpp
int minn = 1000;        // 比最大可能计数（100）大就行
for (int i = 0; i < 26; i++) {
    if (a[i] == 0) continue;   // 没出现的字母不参与
    ...
}
```

**③ 质数判断整个反了**

```cpp
if(t==1||t==2||t==3) z=1;        // t=1 不是质数，这里也设成 z=1
for(int i=2;i<t;i++)
    if(t%i==0) {z=1;break;}      // 找到因子=合数，这里也设成 z=1
if(z==1) cout<<"Lucky Word";     // z=1 表示"是 Lucky Word"
```

两条规则互相矛盾，结果：

| t | 我的判断 | 正确 |
| --- | --- | --- |
| 1 | Lucky ❌ | No Answer |
| 2、3 | Lucky ✅ | Lucky |
| 4、9（合数） | Lucky ❌ | No Answer |
| 5、7、11（质数） | No Answer ❌ | Lucky |

**只有 2 和 3 恰好对。**

```cpp
bool prime = (t >= 2);
for (int i = 2; i * i <= t; i++)
    if (t % i == 0) { prime = false; break; }
```

试到 `i*i <= t` 就够，不用试到 `t-1`。

### 2.4 修好的一版

```cpp
#include<bits/stdc++.h>
using namespace std;
int main()
{
    int a[26] = {0};
    int maxn = 0, minn = 1000;
    string s;
    cin >> s;

    for (int i = 0; i < (int)s.size(); i++)
        a[s[i] - 'a']++;

    for (int i = 0; i < 26; i++)
    {
        if (a[i] == 0) continue;
        if (a[i] > maxn) maxn = a[i];
        if (a[i] < minn) minn = a[i];
    }

    int t = maxn - minn;
    bool prime = (t >= 2);
    for (int i = 2; i * i <= t; i++)
        if (t % i == 0) { prime = false; break; }

    if (prime) cout << "Lucky Word" << endl << t << endl;
    else       cout << "No Answer" << endl << 0 << endl;
    return 0;
}
```

### 2.5 为什么两个样例都过了却没发现

- `error`：真实 min 恰好是 1，我写死的 `minn=1` 正好撞对；t=2 又命中了 `t==2`
- `olympic`：t=0，两个判断都不进，z 保持 0 → `No Answer/0`，凑巧对

**两个错误互相抵消，样例全绿。** 这和项目里那几次静默失败是同一类问题：不报错、样例过、换个数据全崩。

### 2.6 三版对照（12 个用例）

| 输入 | 期望 | 我原版 | 只加数组初始化 | 完全改对 |
| --- | --- | --- | --- | --- |
| error | Lucky Word/2 | 902311546 ❌ | 2 ✅ | 2 ✅ |
| olympic | No Answer/0 | 902704763 ❌ | ✅ | ✅ |
| aaaabb | Lucky Word/2 | 902835834 ❌ | **3** ❌ | 2 ✅ |
| aabb | No Answer/0 | 903032442 ❌ | **Lucky Word/1** ❌ | ✅ |
| aaaaab | No Answer/0 | 903163514 ❌ | **Lucky Word/4** ❌ | ✅ |
| aaaaaab | Lucky Word/5 | 902114938 ❌ | **No Answer/0** ❌ | 5 ✅ |
| aaaaaaaaaab | No Answer/0 | 903556730 ❌ | **Lucky Word/9** ❌ | ✅ |

"只加数组初始化"那一列特别有价值——它证明了 bug ② 和 ③ 是**独立存在**的，不是被 ① 掩盖出来的。

* * *

## 三、本周踩的坑

### 项目相关

- **`onebot_call` 的返回套了两层**：列表在 `data.data`，不在 `data`。
- **`hidden` 属性会被 `display:flex` 覆盖**：面板全展开把列表撑爆，必须显式写 `[hidden]{display:none}`。
- **测试数据踩到自己的守卫**：拿机器人自己的 QQ 号当 sender，被"忽略自己消息"提前 return，看起来像功能没生效。
- **进度标记不能只信计数器**：`max_seq` / `ready[-1]` 两次 off-by-one，都要靠"抓出来对账"才发现。
- **推理模型吃 max_tokens**：2500 → 空内容 + 16 秒；8000 → 2.5 秒正常。
- **空 content 不等于失败**：只调函数时 `content` 就是空的，判失败会把 `tool_calls` 丢掉。
- **跨服务的超时要不小于降级链总时长**：重排 10 秒超时 vs 30 秒降级链，必然全超时。
- **"应该是另一条路"这种推断必须验**：我以为 AstrBot 走别的 provider，实际它和五个服务用的是同一个欠费 key。
- **重启被杀时的 Traceback 是噪声**：`asyncio.run` 打出来的退出堆栈，不代表运行期故障。
- **对话历史会把 base64 图片和系统提示一起存下来**：主对话 3.6 MB / 450 条，其中 97.4% 是 user 消息、99.5% 是图片 data URI。发一张图 = 往历史里塞几十万字符。
- **SQLite 删了行不会缩文件**：要 `VACUUM`，7.30 MB → 0.48 MB。

### C++ 相关

- **数组不初始化就是垃圾**：`int a[26];` 与 `int a[26] = {0};` 是两个世界。
- **求最小值时初值要"足够大"**：`minn=1` 在计数都 ≥1 的场景下永远不会被更新。
- **布尔变量的语义要写死**：`z=1` 到底是"是"还是"不是"，要在两个分支上保持一致，否则自己和自己打架。
- **质数判断**：`t >= 2` 且试到 `i*i <= t`。

* * *

## 四、本周总结

### 四点收获

**一、能让系统自己降级的，就别让它只有一个出口。**

五个服务共用一个 key，key 没钱就五个一起哑。加一层代理之后，降级只写一处，上层的代码一行没动。而且这次才发现，真正断掉的不只是那五个服务——**AstrBot 也一直在用同一个 key**，她却是我最后一个发现的。

**二、"报错"和"没在工作"是两种状态，得分别对账。**

- 健康页那两条红字，服务是好的，是检查逻辑在数老错误
- AstrBot 那堆 `status: error` 记录，服务在跑，是每次调用都失败
- 快车道那两个 off-by-one，脚本没报错，只是安静地少写了几轮

三种情况都**不报错**，都只能靠"拉出来数一遍"发现。

**三、样例过 ≠ 正确，这一次是在 C++ 里亲手验证的。**

两个样例全过，换 12 个用例后原版全错。而且把 bug 一个个隔离出来之后发现：两个错误会**互相抵消**，抵消之后就再也看不出来了。

**四、"存下来"是最容易被忽略的成本。**

一张图发出去就结束了，但它以 base64 的形式在对话历史里住了下来，一条几十万字符。等发现"她怎么越来越贵、越来越慢"的时候，账已经记了三个月。

这类问题的共同点是：**它不会报错，只会慢慢变差**。所以得有一句能随时跑的复查 SQL：

```sql
SELECT conversation_id, length(content) FROM conversations ORDER BY 2 DESC;
```

### 下周计划

**小鲸鱼项目**

- [ ] 记忆重排的候选裁剪：现在 12 条候选仍需要 11 秒，想办法压到 5 秒内
- [ ] 模型代理加"用量统计"：每个通道每天消耗多少，避免把 DSH 账号额度也用光
- [ ] 群聊旁观要不要按群单独开关（现在是一次性全开）
- [ ] 白名单补丁抽成 `tools/apply_whitelist_patch.py`，升级后一条命令补回来
- [ ] 给「监听」页加一个"最近 24 小时她回了几条"的曲线
- [ ] 把"对话历史大小 / 数据库大小"也放进健康页，超 1 MB 就报警

**C++ 学习**

- [ ] 质数专题：筛法（埃氏筛）与 `i*i <= n` 的边界
- [ ] 字符与数组：`char` 的 ASCII 运算、`toupper`/`tolower`
- [ ] 洛谷继续：P5734 文字处理软件、P1308 统计单词数、P1012 拼数
- [ ] 把 P1125 用 `map<char,int>` 再写一遍，对比数组写法

* * *

> 本文由 Hugo 生成。上周周报：[11.1 学习周报](/p/11.1-weekly-report/)
