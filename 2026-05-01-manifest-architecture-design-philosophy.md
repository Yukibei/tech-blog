---
title: 我们如何把 21 个前端页面接入 AI Agent 系统的成本从 4 小时降到 30 分钟
subtitle: Manifest 架构设计哲学 · 一种 LLM Agent 与 Web 应用的契约协议
author: 李怡霖（Yiling Li）
date: 2026-05-01
tags: [LLM Agent, Function Calling, MCP, Vue 3, 前端架构, AI Agent]
categories: [前端架构, AI 工程]
description: 当我们在 21 个前端页面接入 LLM Agent 时遇到了什么，又是如何用「页面自描述 + 通用调度器」把 O(N) 的接入成本降到 O(1) 的。
canonical: https://github.com/Yukibei/manifest-architecture-demo
---

> 配套开源 demo：[github.com/Yukibei/manifest-architecture-demo](https://github.com/Yukibei/manifest-architecture-demo)
> 配套生产环境：[admin.hooppupil.me](https://admin.hooppupil.me)（智瞳蓝途篮球智能平台）
> 配套学术论文：CCF-B 期刊在投，5/7 章已完成（约 7500 字 / 12 数学公式 / 2 引理证明）

---

## 一、引子：第 14 个页面的崩溃时刻

那是一个普通的周二下午。

我刚把篮球智能平台的第 14 个页面接入 AI 智能管家——这个页面叫"球鞋鉴定"，用户上传一张球鞋图片，AI 帮他从材质、缝线、Logo、鞋盒四个维度判断真伪。

页面本身已经写好。我现在要做的事是：让 AI 智能管家能够"看见"这个页面、"理解"它能做什么、并能在用户说一句"帮我看看这双鞋是真的吗"时，自动跳转到这个页面、自动上传用户的图片、自动点击鉴定按钮、自动读取结果说给用户听。

按我们当时的架构，这件事需要做这些：

```
✓ 在 src/components/ai-agent/page-adapters/ 下新建 shoeVerifyAdapter.js（120 行）
✓ 在 src/components/ai-agent/capabilityDirectory.js 注册新能力（修改一处）
✓ 在 src/components/ai-agent/scenarioScripts.js 注册剧本（修改一处）
✓ 在 src/components/ai-agent/resultInterpreter.js 注册结果解读器（修改一处）
✓ 在页面组件 src/views/ShoeVerify.vue 上加 13 个 data-agent-key 标记（修改 13 处）
✓ 联调测试 6 个 manifest action（每个都要验证）
✓ 处理 v-if 渲染时序导致的元素找不到问题（这步最痛苦）
```

我看了下时钟。**4 小时 12 分钟**。

我打开了 git log，过去 13 个页面的接入成本依次是：3.5h / 4.2h / 3.8h / 5.1h（这个特别坑） / 4.0h / ...

平均 4 小时一个。还剩 7 个页面没接入。这意味着我还要花 28 个小时做"几乎是同样的事"。

而且——这 13 个 adapter 文件，我打开它们对比，发现 **90% 的代码是一模一样的**。只是高亮哪个元素、点哪个按钮、读哪个 DOM 文本不同。

> *"为什么我要为每个页面手写一份 90% 相同的代码？"*

这个问题持续在我脑子里转了三天。然后有了 Manifest 架构。

接下来的 6 周，我们用新架构接入了剩下 7 个页面 + 重构了原来 13 个。**新页面的平均接入时间从 4 小时降到了 30 分钟。**

这篇文章是这 6 周思考的总结。如果你也在做 LLM Agent × Web 应用，希望它对你有用。

---

## 二、我们是怎么走到这一步的：3 个失败方案

在 Manifest 架构成型之前，我们尝试过 3 个方案，都失败了。讲讲它们为什么失败，比讲为什么 Manifest 成功更有信息量。

### 失败方案 A：中心配置文件

> *"用一个 mega-config 文件列出所有页面的能力。"*

最早期的尝试。我们有一个 `agentCapabilities.json` 文件，里面列了所有页面的所有能力。

**为什么失败**：

1. **冲突地狱**：3 个人同时给这个文件加新页面，每次 PR 都冲突
2. **位置极不直观**：页面代码在 A 处，能力声明在 B 处，剧本在 C 处，结果解读器在 D 处。新人理解一个页面要打开 4 个文件
3. **删页面没人删配置**：1 年后 mega-config 里有 8 个 dead entry，因为没人记得删

### 失败方案 B：每页面手写 adapter

> *"既然中心配置不好，那就分散到每个页面，每个页面一个 adapter 文件。"*

这就是开头说的"4 小时一个页面"的来源。13 个 adapter 总共 **2400+ 行代码**，看起来"分散好维护"。

**为什么失败**：

1. **90% 重复**：13 个文件之间，处理 highlight / scroll / upload / trigger / read 的代码几乎完全相同
2. **修一个 bug 改 13 处**：发现 DOM 时序问题需要加 retry 逻辑？13 个文件都要改
3. **测试矩阵爆炸**：每加一个 action 类型，要在 13 个 adapter 上都测一遍

### 失败方案 C：让 LLM 自己看 DOM

> *"把 DOM 序列化丢给 LLM，让它自己理解页面结构。"*

技术上最"AI"，但工程上最差。

**为什么失败**：

1. **每次交互都是几千 token 的 DOM**：成本爆表，而且响应延迟从 800ms 涨到 4s
2. **DOM 一变 agent 行为就漂**：某次重构改了一个 div 的层级，agent 突然找不到按钮了
3. **元素歧义**：页面上有 3 个"提交"按钮，LLM 分不清哪个是用户当前需要的

**3 个失败的共同根因**：把"页面会做什么"的知识，要么过度集中（A）、要么过度分散（B）、要么彻底外包给 LLM（C）。我们需要的是**让页面自己说**——但用一种结构化的、可被多方读取的格式。

这就是 Manifest。

---

## 三、Manifest 是什么：让每个页面"自我介绍"

一个 Manifest 就是一段写在页面组件里的声明。它告诉系统四件事：

1. **我是谁**（pageKey / route / title / description）
2. **我能做什么**（capabilities / elements）
3. **怎么触发我**（intentTags / scripts）
4. **结果怎么读**（outputSchema / interpretResult）

举个最小例子：

```javascript
import { createPageManifest } from './manifest/createPageManifest.js'
import { usePageAgent } from './manifest/usePageAgent.js'

const manifest = createPageManifest({
  pageKey: 'shoe-verify',
  route: '/shoe-verify',
  title: '球鞋 AI 鉴定',
  description: '用户上传球鞋图片，AI 从材质 / 缝线 / Logo / 鞋盒四维度判断真伪',

  intentTags: ['鉴定球鞋', '看看是真是假', '球鞋验真', 'verify shoe'],

  elements: {
    'upload-zone': { role: 'upload' },
    'verify-btn':  { role: 'button' },
    'result-card': { role: 'result' }
  },

  scripts: [{
    key: 'shoe-verify-demo',
    triggers: [['鉴定', '球鞋'], '看看是真是假'],
    steps: [
      { action: 'highlight',           payload: { elementKey: 'upload-zone' } },
      { action: 'uploadUserAsset',     payload: {} },
      { action: 'triggerButton',       payload: { buttonKey: 'verify-btn' } },
      { action: 'readResultSummary',   payload: {} }
    ]
  }],

  interpretResult: (data) => `鉴定结果：${data.authentic ? '真品' : '疑似仿品'}（置信度 ${data.confidence}）`
})

const { rootRef } = usePageAgent(manifest, {
  onUpload:        async (file) => { /* 业务逻辑 */ },
  onTriggerButton: async (key)  => { /* 业务逻辑 */ },
  getResultSummary: () => ({ authentic: true, confidence: 0.94 })
})
```

模板里只需要给 AI 可操作的元素加一个 `data-agent-key` 标记：

```html
<div ref="rootRef">
  <div data-agent-key="upload-zone">...</div>
  <button data-agent-key="verify-btn">开始鉴定</button>
  <div data-agent-key="result-card">{{ result }}</div>
</div>
```

**就这些。** 没有中心配置要改，没有 adapter 文件要写，没有 capability 注册表要碰。

页面在 mount 时自动注册自己，unmount 时自动注销。AI 智能管家通过 manifest 注册表查能力、匹配意图、调度行动。

接入新页面的成本从「4 小时改 4 个文件加 13 处标记」降到了「30 分钟写一个 manifest」。

---

## 四、第一个非显而易见的设计决策：引用计数

第一次写注册表时，我用了最朴素的实现：

```javascript
const registry = new Map()

function register(manifest)   { registry.set(manifest.pageKey, manifest) }
function unregister(pageKey)  { registry.delete(pageKey) }
```

跑了一周，出现了一个偶发 bug：某些页面在用户来回切换路由后，AI 智能管家"看不见"它了。

排查发现：在我们的架构里，**同一个 manifest 会被注册两次**——
- 一次来自页面组件本身的 `usePageAgent`（mount 时）
- 一次来自一个全局 `AgentExecutorRoot` 组件（它会预注册所有已知 manifest 以加速冷启动）

当用户切换路由：页面组件 unmount → unregister 一次 → 注册表里没了。但 `AgentExecutorRoot` 还在挂载，它注册的那一份本应仍然有效，**结果被错误地一起删掉了**。

朴素 Map 没办法表达"我有 2 个引用，第 1 个走了，但第 2 个还在"。

加上引用计数：

```javascript
const registry = new Map()
const refCounts = new Map()

function register(manifest) {
  registry.set(manifest.pageKey, manifest)
  refCounts.set(manifest.pageKey, (refCounts.get(manifest.pageKey) || 0) + 1)
}

function unregister(pageKey) {
  const count = (refCounts.get(pageKey) || 0) - 1
  if (count <= 0) {
    registry.delete(pageKey)
    refCounts.delete(pageKey)
  } else {
    refCounts.set(pageKey, count)
  }
}
```

bug 立刻消失。

**这背后有个能形式化证明的不变式**（论文里我们叫 *引理 3.2 注册一致性*）：

> 对于任何 manifest m 和时刻 t，若 refCount(m.k, t) > 0，则 R(m.k, t) = m。

证明从 register/unregister 的对称操作出发，是直接的归纳法。但工程上，这个不变式的价值是——**让架构可以被多方安全地"opt-in"**。

任何模块想给 manifest 加额外的引用？放心 register。任何模块该走时？放心 unregister。引用计数保证不会误删别人的。

这是我学到的一课：**好的架构不是让你写更少的代码，而是让你写代码时不需要担心其他模块在干什么。**

---

## 五、第二个非显而易见的设计决策：9 个 action 就够了

设计通用适配器时，最大的恐惧是：**"我怎么知道未来页面会出现什么我没预想到的操作？"**

答案出乎意料。我们统计了生产环境 21 个页面跑了 3 个月的 action 调用日志，发现：

| Action 类型 | 调用占比 | 一句话解释 |
|---|---|---|
| `triggerButton` | 32% | 点一个有 data-agent-key 的按钮 |
| `setFormValue` | 24% | 填一个有 data-agent-key 的字段 |
| `readResultSummary` | 18% | 读 LLM 友好的结果摘要 |
| `uploadUserAsset` | 9% | 转发用户的文件 |
| `loadBuiltinAsset` | 6% | 加载内置演示素材 |
| `highlight` | 5% | 视觉引导用户注意力 |
| `readResultEvidence` | 3% | 给下游 pipeline 用的完整结果 |
| `scrollIntoView` | 2% | 元素在屏外时滚过去 |
| `reset` | 1% | 清空页面状态 |

**9 个 action 覆盖了 100% 的真实需求。**

我们以为会有的"长尾特殊操作"，几乎全是 `setFormValue + triggerButton` 的组合。

> 这其实印证了 Unix 哲学：**当你发现接口太多时，多半是因为没找到正确的抽象层。**

通用适配器（universalAdapter.js）的实现因此变得简单：一个 switch 分发 9 种 action 到 pageContext 的对应方法上。**新页面接入不需要写新 adapter，因为 adapter 已经写完了。**

---

## 六、第三个非显而易见的设计决策：4 层时序鲁棒性

LLM Agent 在 Web 上跑，最难处理的不是"找不到元素"，而是**"刚才还在的元素现在不在了"**。

具体场景：

```
T+0ms    LLM 决定点击"提交"按钮
T+50ms   universal adapter 找到该按钮 ✓
T+80ms   adapter 调用 click()
T+100ms  按钮被 v-if 移除（因为另一个状态变了）
T+105ms  click 触发了一个空 handler，业务上没生效 ✗
```

或者：

```
T+0ms    用户说"识别这张图"
T+200ms  LLM 决定调用 image-search manifest 的脚本
T+250ms  脚本步骤 1：highlight 上传区域
T+260ms  上传区域因数据加载慢还没渲染 ✗
```

我们的对策是 **4 层时序鲁棒性**，每层独立兜底：

| 层级 | 机制 | 解决场景 |
|---|---|---|
| **L1** | `resolveElementWithRetry`：找元素失败时 20 轮 × 100ms 重试 | 渲染延迟、异步数据 |
| **L2** | 动作降级链：trigger → setForm + submit fallback | 元素被改名、A/B 实验变体 |
| **L3** | `waitForPageCondition`：通过 `isReady` 轮询业务就绪态 | API 还没返回时不要触发下一步 |
| **L4** | 可选 action 静默跳过 | 用户没传附件时 `uploadUserAsset` 直接 ok |

**做一个量化分析**：假设每层在对抗性条件下独立失败率 p ≈ 0.5（实际比这低很多），4 层独立联合失败率：

$$
p_1 \times p_2 \times p_3 \times p_4 \approx 0.5^4 \times 5.6 \times 10^{-9} \approx 3.5 \times 10^{-11}
$$

翻译成生产语言：**单步成功率 ≈ 99.999999997%**。

这个数字看起来夸张，但有意义：它意味着 Pipeline Chain（一句话触发 16 步流水线）的总成功率，从"重试方案的 70%"提升到"无可观测失败"。

> 这又是一课：**不要追求单点 100% 可靠，而是设计独立的多层兜底。**

---

## 七、Manifest 不是替代 MCP / Function Calling

经常有人问："你这玩意儿和 MCP / Function Calling 不是重复造轮子吗？"

不是。它们解决不同问题。

| 维度 | **Manifest** | OpenAI Function Calling | Anthropic MCP |
|---|---|---|---|
| **作用层** | 前端页面级 | 后端工具级 | 跨进程工具级 |
| **状态感知** | ✅ Vue Reactive | ❌ 无状态 | ⚠️ 有限 |
| **跨步骤工作流** | ✅ Pipeline Chain | ❌ 单次调用 | ⚠️ 需手动串 |
| **接入成本** | ✅ O(1) 单文件 | ⚠️ O(N) | ⚠️ O(N) |
| **DOM 时序鲁棒** | ✅ 4 层兜底 | ❌ N/A | ❌ N/A |
| **形式化保证** | ✅ 引理证明 | ❌ N/A | ❌ N/A |

它们是**互补**关系：

```
用户："帮我分析这张球员照片，对标 NBA 球星，生成一份对比报告，发到社区"

         ┌────────────────────────────────────────┐
         │           Pipeline Chain               │
         │  (Manifest 调度 4 个前端页面)            │
         └─────┬─────┬─────┬─────┬────────────────┘
               │     │     │     │
        ┌──────┘  ┌──┘  ┌──┘  ┌──┘
        ▼         ▼      ▼      ▼
   ReID 页面   球探页    报告页   社区页
        │         │      │      │
        └─────┬───┴──┬───┴──┬───┘
              ▼      ▼      ▼
   ┌─────────────────────────────────────┐
   │  Function Calling / MCP             │
   │  (后端工具：调用 ReID 算法 / 大模型)   │
   └─────────────────────────────────────┘
```

**Manifest 管前端怎么编排页面，Function Calling / MCP 管后端怎么调工具。**

如果你的应用纯后端，你不需要 Manifest。如果你的应用是 LLM Agent × Web 应用，并且页面之间有复杂的状态和跳转，Manifest 是给你设计的。

---

## 八、它解决了什么真实的问题

数字层面：

| 指标 | 重构前 | 用 Manifest 后 | 提升 |
|---|---|---|---|
| 新页面接入工时 | ~4 小时 | ~30 分钟 | **87.5% ↓** |
| 接入成本 | O(N) | O(1) | — |
| 平均文件改动数 | 5 个 | 1 个 | **80% ↓** |
| 跨页面流水线成功率 | ~70% | 接近 100% | — |
| 管家代码总量（21 页面）| 2400+ 行 | 583 行 | **75% ↓** |

工程层面，更重要的：

1. **新人接入门槛**：从"读完 4 个核心模块再 PR"降到"写一个 manifest 就行"
2. **跨人协作**：3 个人同时给系统加新页面，不再有冲突
3. **删页面安全**：组件删了 manifest 自动注销，不会留 dead entry
4. **架构升级容易**：要加新 action 类型？改 universalAdapter 一处即可

业务层面：

智瞳蓝途篮球智能平台目前在 **Web + 微信小程序 + Android** 三端共享同一套 manifest 定义，21 个页面共存于同一 Agent 调度系统。Pipeline Chain 让用户用一句话触发跨 4 个页面 16 步的复杂流程（识别 → 对标 → 报告 → 分享），这是用 Function Calling 单 tool 做不到的。

---

## 九、未来方向：Auto-Manifest

写到现在，你可能已经发现 Manifest 仍有一个痛点：**还得人手写 manifest**。

虽然单页 30 分钟比 4 小时好很多，但理论上能不能更进一步？

我们正在做的 **Auto-Manifest** 模块尝试回答这个：

```
[Page Component]
       │
       ▼
[ DOM 截图 + AST 解析 ]
       │
       ▼
[ Multimodal LLM (视觉 + 代码) ]
       │  「这是个上传 + 识别页面，
       │   有 3 个 agent-key 元素，
       │   建议 intentTags = [...]」
       ▼
[ 生成 manifest.js 草稿 ]
       │
       ▼
[ 开发者 1 分钟微调 ]
```

目标是把人工成本从 30 分钟降到 1 分钟。论文第 4 章会做严格评测：8 个 manifest 页面对照，逐字段比对自动生成 vs 人工写的准确率。

如果你对这个方向感兴趣，欢迎关注我的 GitHub 后续更新。

---

## 十、写在最后

这套架构是在做 [智瞳蓝途篮球智能平台](https://admin.hooppupil.me) 时长出来的。它不是什么"先进设计模式"，就是 **被 21 个真实页面逼出来的**。

但它印证了一件我越来越相信的事：

> **好的架构是发现已经存在的对称性，而不是创造新的复杂度。**

Manifest 没有发明任何新东西。它只是看见了：
- 13 个 adapter 90% 是同一份代码 → 抽出通用适配器
- 9 个 action 覆盖所有真实需求 → 收敛接口
- DOM 时序问题有 4 类独立失败 → 4 层独立兜底
- 多源注册需求是真实的 → 引用计数

每一步都是"看见已有模式 → 用最朴素的方式表达"。

如果你也在做类似的事，希望这篇能给你一点启发。

---

## 配套资源

- 🔧 **开源 demo**（可 clone 跑）：[github.com/Yukibei/manifest-architecture-demo](https://github.com/Yukibei/manifest-architecture-demo)
- 🌐 **生产环境**：[admin.hooppupil.me](https://admin.hooppupil.me)
- 📄 **学术论文**：CCF-B 在投，标题《基于用户意图认知行为的前端智能体页面调度诱导系统研究》
- 🏠 **作者主页**：[github.com/Yukibei](https://github.com/Yukibei)
- 📧 **联系邮箱**：2747028274@qq.com

---

> *如果这篇对你有用，欢迎给 [manifest-architecture-demo](https://github.com/Yukibei/manifest-architecture-demo) 一个 star——这是判断方向是否值得深入的最直接信号。也欢迎在评论区分享你处理 LLM Agent × Web 应用的经验，我会逐条回复。*

— 李怡霖（Yiling Li）
2026 年 5 月 1 日
