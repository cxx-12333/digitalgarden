---
{"dg-publish": true, "permalink": "/Blog/pr-7061-vs-7182-philosophy/", "title": "我的 PR 败给了什么——一次 webhook 与官方 hook 引擎的哲学差距", "tags": ["架构", "安全", "开源", "工程哲学"], "created": "2026-08-20", "updated": "2026-08-20", "dg-note-properties": {"title": "我的 PR 败给了什么——一次 webhook 与官方 hook 引擎的哲学差距", "permalink": "pr-7061-vs-7182-philosophy", "created": "2026-08-20", "updated": "2026-08-20", "tags": ["架构", "安全", "开源", "工程哲学"], "type": "blog"}}
---



# 我的 PR 败给了什么
<p class="dg-date">✍️ 2026-08-20</p>

## 两个 PR

同一个月，同一个仓库（[multica](https://github.com/multica-ai/multica)），同一个功能方向（出站通知），两个 PR：

- **[#7061（我的）](https://github.com/multica-ai/multica/pull/7061)**：`add outbound webhook for action-required events`——issue 进入 in_review、任务失败、被分配时，向 workspace 配置的 webhook URL 发通知。**至今 open。**
- **[#7182（官方）](https://github.com/multica-ai/multica/pull/7182)**：`rebuild the plugin system (3/4) — hook engine`——同一批"需要人注意"的事件，走插件 hook 引擎出站。**已合并。**

我的先发。看完官方的实现和注释后，我明白它为什么赢——**不是代码量的差距，是每一个决定背后有没有"一条被命名的事实"**。这篇把对比整理成九组哲学，每组附官方原文的推理链。这是复盘，也是我今后写代码的标尺。

## 一、问题定性：设计的第零步

官方 PR 的源头是一句分类判断：**"A hook leaves"**——此前所有插件机制都是入站的（沙箱内的插件请求 host，host 用登录用户的会话行事），"nothing left our infrastructure on a plugin's say-so"；hook 是**第一个离开基础设施的调用**。于是"检查对象是目的地而非调用者""必须签名""必须不阻塞"全部自动推出。

我的 7061 把 webhook 分类成"通知系统的又一个 sink"——于是延迟要求、目的地验证、失败治理全部缺席。

> 写代码前先用一句话说出"这件事的本质类别是什么"。**分类错了，后面所有正确执行的细节都是在错误方向上的精益求精。**

### 方向不对称决定存储设计

两个凭证走同一条部署密钥，方向相反，存储方式相反：install token 是插件→host、host 只需验证，所以**存哈希**；signing secret 是 host→插件、host 必须重现它来签名，所以**从 deployment key 派生、不落库**（"Same deployment key, opposite directions"）。

> 不要问"这个秘密怎么存"，先问"它在哪个方向流动、谁需要重现它"——答案唯一确定存储方案。

## 二、安全：每条防御绑定一个具名威胁

官方防御清单的密度（每条都有具体攻击剧本）：

- HMAC 签 `timestamp.body` 用 `.` 分隔——因为"签名裸 body 可永久重放；签名裸拼接可以让伪造的 timestamp 和 body 互换字节"
- `ConstantTimeCompare`——因为"逐字节比较泄露猜对了多少，足够推出剩下的"
- 拨号时重解析并拒绝私网地址——防 **DNS rebinding**（先解析公网过审、TTL 后翻内网）
- 拒绝 redirect——因为"302 会把已签名的 body 和 callback token 原样重放到别处"

> 写不出攻击剧本的防御，删掉；写得出的，把剧本写进注释。**没有一条是"加了比较安全"。**

### 同意即边界

net: 白名单直接用 consent 屏上管理员批准的**同一份字符串**——"there is no second list to fall out of sync"。FindHook 从 installation 行读 manifest 而非插件当前服务的 URL——"管理员同意的是一组特定端点，插件事后改指向必须走升级流程"（防 TOCTOU）。

> 安全 UI 展示的内容和代码执行的内容必须是同一份数据。"展示一份、执行另一份"的结构在制造漂移漏洞。

### 最深刻的一条：看控件诱导的行为，不看名义强度

callback token 最初是单次有效——听起来更严格。但参考实现的真实形状是"读取 issue → 决策 → 写评论"，第二次调用就死在已花费的 token 上；作者被迫改用永不过期的 install token。原文：

> **"A control that pushes authors toward the stronger credential is worse than the looser one they will actually use."**

于是改成 invocation 级：几分钟、本次调用的 scope、actor 固定、调用返回即吊销。

> 控件的安全性 = 用户在它面前**实际会做什么**，不是它名义上禁止了什么。**逼用户绕过的控件比宽松控件更危险。**

## 三、执行与隔离：第三方不得进入关键路径

"an event hook NEVER blocks the host" 是 dispatch.go 的存在理由。总线是同步的，监听器跑在发布请求的 goroutine 上，所以监听器只做一次 channel 投递——"the listener does the least it possibly can: hand over the payload **unexamined**"。连"提取 issue id 要 JSON round-trip 整个 payload"这种间接成本都被专门移到 worker 上。

更极致的是对未来约束：agent 执行路径没有 hook——"每个 agent 回合前后必须运行的 hook，是第三方握着产品主循环不放手"；agent 是**选择**把 hook 当工具调用——"a call they can decline"。

> 架构里要有明确的"关键路径"清单。第三方代码出现在清单任何位置，都需要一个像 worker pool 这样的**结构性理由**。

配套三条：**不使用功能的人零成本**（flag 检查放 worker 上，DB 查询和 payload 解析都在 flag 之后）；**有界是一切后台资源的默认属性**（队列 512 深满了丢弃并计数——"无界队列把一个慢插件变成内存泄漏"；响应体 1MB 上限）；**生命周期不同的工作不继承调用者的 context**（"把出站 hook 绑在触发它的请求上，会在浏览器拿到响应那一刻取消 hook——而那正是 hook 刚开始的时候"）。

## 四、身份与归因：身份跟随触发者

ui/manual hook 的写回归属按下按钮的人（via_plugin_id 标注），event hook 的写归属 installation 本身——"an event hook has no person behind it and must not borrow the last one who happened to touch the issue"。为此新增第四种 comment author_type，并修掉 event 写入渲染成 "System" 的问题：

> "这是唯一一种**主动误导**的归因：它读起来像平台自己说了这句话。"

> 归因错误不是显示 bug，是一条说谎的数据。"没有归属"永远比"错误归属"好，"看起来像官方的归属"是最坏一档。

## 五、失败治理：拒绝是决定，不是故障

> "A refusal is a decision, not an outage: retrying a hook that is disabled, out of scope or rate limited just burns the budget."

失败分三档 refused / timeout / failed，只有后两档重试。把所有错误坍缩成一个"失败"桶，重试策略就只能在过度重试和漏重试之间二选一——**先建分类学，再建策略**。熔断的注释同样锋利："宕机一小时的端点不需要每个事件一次请求才注意到自己被放弃。"

## 六、数据与存储：遥测不是档案

plugin_invocation 表"刻意不是审计日志"：TTL 7 天、不存请求/响应体——"一个永久保存 payload 的表，是 workspace 内容的第二份副本，**却没有第一份的删除路径**"。错误字段只存 host 自己的描述、截断 500 字符——防端点 echo 输入把 issue 文本写进一张没有删除通道的表。

> secondary 数据默认不继承 primary 的治理。写表前回答：谁读它、活多久、随谁删除。

锁经济学同款具体性：迁移把 VALIDATE CONSTRAINT 拆出来单独跑，注释做历史对比——"107 当年在 ACCESS EXCLUSIVE 下整表重验，在那个时代的 comment 表上便宜，**今天不便宜**"。

## 七、契约演进 + 注释文化

- **两套词汇表，单点翻译**：内部事件名（会因内部原因漂移）和插件事件名（已发布契约）物理分开，中间一个可单测的翻译层
- **线上格式稳定且小，版本内嵌**：body 有 `version: 1`，签名方案独立的 `hookSignatureVersion = "v1"`，能拆开演进的维度各自版本化
- **注释记录被否决的方案**：sweeper"早期版本立即扫……然后它在 router 测试里 panic 了"。好注释回答的不是"这段代码做什么"，而是"将来谁会想改它、他会先想到什么错误方案"

## 八、测试：测性质，不测路径

- never-blocks 测试**在编写过程中发现了真 bug**（worker 裸 goroutine panic 保护缺失）——"Writing the never-blocks test found a real bug rather than confirming one"
- 背压测试"停下 workers 而不是和他们赛跑"，断言精确的溢出数量
- 那两个契约 bug（单次 token、body 缺 issue_id）"测试全都看不见，因为测试每个只调一次，而真实 handler 不是"
- 连文档都测：`TestShippedExamplesParseAndInstallOnThisHost`——"连文档都会漂移，所以测试文档"

> 如果一个测试从没让你意外过，它是橡皮图章。**上线路线本身是测试层**：实跑、性质测试、路径审计——三个正交探针各抓各的盲区，组合的覆盖才是覆盖。

## 九、AI 时代评审：约束必须活在代码里

官方评审抓到的最严重问题：callback token 的 issue 收窄"只存在于两条 doc 注释和 PR 描述里——`pluginTokenCaller` 从不读 `grant.IssueID`，所以一个 ui/manual token 在它存活的五分钟里值 actor 能看到的每一个 issue，读和写都行"。修复后用 404 而非 403（"你被收窄在别处"会确认 id 存在——又是一个具名威胁）。

> **写在注释里的安全性质是零。** 评审要专门找"声称的约束"和"执行的约束"之间的差集。

## 如果只带走五条

1. **先定性，再设计**——"A hook leaves"，分类决定约束集
2. **每条防御绑定一个具名威胁**——写不出剧本就删，写得出就写进注释
3. **控件看诱导的行为，不看名义强度**——单次 token 逼出更强永续 token，就是负安全
4. **跨越边界时逐条重验不变量**——panic 保护、context 生命周期、TOCTOU 都是同一条的不同面
5. **注释记录被否决的方案**——让下一个改动者不必重新踩一遍

## 尾声

这套哲学的共同底色只有一句话：**每个决定都锚定在一条具体的、被命名的事实上——攻击剧本、使用节奏、锁的级别、凭证的流动方向——没有悬空的"最佳实践"。**

这正是 #7061 和 #7182 之间的差距：我的 webhook "功能写完了"，它的 hook 引擎**每个决定都能回答"凭什么"**。功能会被替代，但这种做决定的方式不会。

败给这样的实现，不冤。

---

*关联：[文件即接口：agent自动化的调度解耦模式](Notes/文件即接口：agent自动化的调度解耦模式/) · [为什么 PR 里出现了别人早就合并过的提交——Git compare 的祖先真相](Blog/git-pr-parallel-history-trap/)*
