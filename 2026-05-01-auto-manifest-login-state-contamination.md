---
title: 你以为给 LLM Agent 一个对的 URL 就够了？登录态污染如何吃掉 30% 字段准确率
subtitle: Auto-Manifest 实战 · W3 验收时发现的真实生产坑 + 我们怎么救回来的
author: 李怡霖（Yiling Li）
date: 2026-05-01
tags: [LLM Agent, Auto-Manifest, Playwright, 评测设计, 前端测试]
categories: [AI 工程, 评测方法学]
description: 当 Auto-Manifest 模块的 W3 验收脚本输出 errors=[] 时，我以为大功告成。打开生成的 manifest 文件后，我意识到自己在测一个登录前的落地页 — 而我们整个 8 样本对照测试集都中招了。
canonical: https://github.com/Yukibei/manifest-architecture-demo
---

> 配套上一篇：《[Manifest 架构设计哲学](./2026-05-01-manifest-architecture-design-philosophy.md)》
> 配套学术论文：CCF-B 期刊在投，本文内容是论文 §4.3 评测设置的实战初稿
> 配套开源 demo：[manifest-architecture-demo](https://github.com/Yukibei/manifest-architecture-demo)

---

## 一、那个让我冷汗下来的时刻

下午 14:30，U1 工程师在团队群里同步：

> *"W3 已完成，scout manifest 归档跑通，errors=[]，pageKey=scout，route=/ai-pose/scout，全部测试 7/7 通过。"*

我看了一眼，心想：稳。准备发 W3 PASS。

但作为验收人，我有个习惯——**不只看测试日志，要打开产物文件本身**。

`logs/auto-manifest/w3/w3-manifest.json` 打开。

```json
{
  "pageKey": "scout",
  "route": "/ai-pose/scout",
  "title": "球探对标",
  "description": "上传图片或视频，AI 分析动作特征并匹配最接近的球星",
  "intentTags": ["球探对标", "球探分析", "球星匹配", ...],
  
  "elements": {
    "button-product-capability": { "label": "产品能力", ... },
    "button-visual-archive-archive": { "label": "视觉档案", ... },
    "button-community-feedback": { "label": "社区反馈", ... },
    "button-enter-system": { "label": "进入系统", ... },
    "button-enter-login": { "label": "进入登录 / 系统", ... },
    "button-scroll": { "label": "SCROLL", ... },
    ...
  }
}
```

冷汗。

scout 是球探对标页面——应该有"上传区"、"开始分析"、"对标结果面板"。

但 elements 里全是「产品能力」「视觉档案」「进入系统」「进入登录」「SCROLL」——**这是登录前的落地页，不是 scout 工作台**。

URL 对，pageKey 对，title 对，description 对（因为是从既有手写 manifest 反推的）。但**真正从 DOM 抽出来的字段全是垃圾**。

---

## 二、问题的本质：你给 Agent 的 URL 不一定是它真正看到的页面

很多 LLM Agent 工程师踩过这个坑的变种：

```
你以为：
   URL https://app.com/dashboard
            ↓
       Agent 看到的 = Dashboard 页面 ✓

实际可能是：
   URL https://app.com/dashboard
            ↓
   路由守卫检测未登录 → 重定向到 /login
            ↓
       Agent 看到的 = Login 页面 ❌
```

或者更隐蔽：

```
   URL https://app.com/dashboard
            ↓
   服务端 SSR → 渲染了 landing 页占位
            ↓
   客户端水合 → 检测到未登录 → 渲染了 modal 提示
            ↓
       Agent 截到的 DOM = SSR 占位 + 半渲染 modal 混合体
```

**所有这些场景，URL 是对的，HTTP 200 也是对的，DOM 不空，但内容完全不是你期待的页面。**

我们就栽在第一种。

---

## 三、为什么这个错不容易被察觉

回头看这个 bug 链：

| 检查点 | 通过/失败 | 为什么没拦住 |
|---|---|---|
| URL 是否正确？| ✅ 通过 | URL ground truth 由我裁决统一过 |
| HTTP 响应是否 200？| ✅ 通过 | 落地页也是 200 |
| DOM 是否能解析？| ✅ 通过 | 落地页 DOM 完整 |
| W3 单元测试是否通过？| ✅ 通过 | 单测验证 schema 合法性，不验证内容语义 |
| Manifest 必填字段是否齐全？| ✅ 通过 | pageKey/route/title 是 CLI 参数覆盖的 |
| `errors` 数组是否空？| ✅ 通过 | 没有运行时报错 |

**6 个检查点全部通过，但产物的 elements 字段是垃圾**。

这就是评测设计经常踩的坑：**"我们检查了所有易查的事，但没检查重要的事"**。

具体到这里：所有易查检查（schema / 必填 / 不报错）都通过了，但 elements 内容的**语义合理性**没有自动检查——因为它需要你"知道这个页面应该长什么样"才能验证。

> 你以为你在测一个对的页面，**直到你看到测试数据本身**。

---

## 四、定量影响：这种污染对 13 字段准确率评测的杀伤

我们的 Auto-Manifest 论文（CCF-B 在投）核心评测是：8 样本 × 13 字段 = **104 个字段准确率比对**。

如果不修这个 bug 直接进 W4-W5 正式评测，每个字段会被打成什么样：

| Manifest 字段 | 来源 | 预期准确率（污染下）|
|---|---|:---:|
| `pageKey` | CLI 参数覆盖 | 100% ✅ |
| `route` | CLI 参数覆盖 | 100% ✅ |
| `title` | 既有手写 manifest 反推 | 80%+ ✅ |
| `description` | 既有手写 manifest 反推 | 70%+ ✅ |
| `intentTags` | 既有手写 manifest 反推 | 60%+ ✅ |
| `parentPageKey` | 既有手写 manifest 反推 | 80%+ ✅ |
| **`elements`** | **从 DOM 抽取** | **≤ 10% ❌❌❌** |
| **`capabilities`** | **从 DOM 抽取** | **≤ 10% ❌❌❌** |
| **`scripts`** | **基于 elements/capabilities** | **≤ 10% ❌❌❌** |
| `inputSchema` | 启发式推导 | 30%+ ⚠️ |
| `outputSchema` | 启发式推导 | 30%+ ⚠️ |
| `riskLevel` | 默认值 | 100% ✅ |
| `requiresConfirm` | 默认值 | 100% ✅ |

**3 个核心字段（elements / capabilities / scripts）直接归零**。

这意味着论文表 5 的 13 字段平均准确率，理论上限是 **(7×100% + 3×60% + 3×10%) / 13 ≈ 71.5%**——但 reviewer 一看 elements 字段 10% 就会问："你的方法在最关键的元素抽取上接近随机，凭什么发顶刊？"

**这是 paper-killing bug。**

---

## 五、解法第 1 步：建立登录态

最直接的想法：让自动化脚本登录后再截图。

但本平台登录页有**图形验证码**。Playwright 自动化无法识别图形验证码（除非接打码平台，但 reviewer 看到论文写"我们用打码服务过验证码"会皱眉）。

我们尝试了 3 个方案：

### 方案 A · 后端测试开关

在 Java 后端加 `?test_token=xxx` 跳过登录守卫。

**优点**：实现快  
**缺点**：动了业务代码 + 永远要维护这个测试开关 + 论文不好写（"我们改了业务后端的鉴权逻辑"听起来不严肃）

### 方案 B · 前端 publicPath 旁路

把 8 样本路径在测试构建时设为 `publicPath`，绕过 router guard。

**优点**：不动后端  
**缺点**：需要在测试构建里同步改 8 个页面的路由配置 + 论文要解释为什么 vs 生产 path 不同

### 方案 C · Playwright storageState 注入 ⭐

人手动登录一次浏览器 → 导出 cookies + localStorage → Playwright 加载 `storageState: 'auth.json'` → 后续所有自动化 session 直接是登录态。

**优点**：
- 0 行业务代码改动
- 论文可写："实验脚本通过 Playwright `storageState` 加载手动建立的登录态，模拟真实用户访问场景"——reviewer 听完点头
- 5 分钟人工成本（你登录一次就行）
- 处理验证码、二次因子、设备指纹等复杂登录场景的通用解法

**缺点**：cookie 失效需要重新生成（但这个频率每 7-30 天一次）

我们选了 C。具体步骤：

```
1. 浏览器手动登录一次目标系统
2. F12 → Application → 复制 cookies + localStorage
3. 组装成 Playwright storageState JSON 格式：

{
  "cookies": [
    { "name": "Admin-Token", "value": "...", "domain": "localhost", "path": "/" }
  ],
  "origins": [
    { "origin": "http://localhost", "localStorage": [...] }
  ]
}

4. capture 脚本启动浏览器：
   const context = await browser.newContext({
     storageState: 'paper-deliverables/U1-test-fixtures/auth-state.json'
   })

5. 把 auth-state.json 加到 .gitignore（含 token，不入仓库）
```

5 分钟解决。

---

## 六、解法第 2 步：声明式 setupSteps

修了登录态后跑了一次：scout 工作台终于出现了。

但发现新问题：**scout 进入工作台还需要点"开始"按钮 + 选模式**。

如果 capture 脚本只是登录后等待页面 idle，截到的还是「点开始之前」的中间态——上传区还没渲染出来。

ai-report 也一样：登录后还要选一场比赛 / 一个球员，元数据加载完才能看到"生成报告"按钮。

player-pk 更夸张：要选 2 个球员才能看到对比图表。

这不是 1 个页面的特例，是**全部 8 个样本的普遍现象**。真实业务页面几乎没有"URL 一进就工作台 ready"的——总有筛选、加载、模式选择、子模块进入。

### 设计：把"工作台准备态"声明化

我们扩展了 URL 清单文档，每个样本除了 URL 之外还要声明：

```
样本 4 · ai-report
- URL: http://localhost/basketball/ai-report
- Auth: 已登录
- setupSteps:
    1. 等待页面渲染完成（不再是 SSR 占位）
    2. 在比赛/球员选择器中选择第一个可选项
    3. 等待元数据加载完成
- 工作台就绪条件: 比赛/球员已选定 + "生成报告"按钮可点击
- Fixture: 依赖现有数据库的比赛/球员数据
```

8 个样本各有 3-4 步 setupSteps。capture 脚本严格按声明顺序执行 → 直到工作台就绪条件满足 → 才进 W1 截图阶段。

### 实现关键 1：避免脆弱等待

绝对不用 `setTimeout(2000)`。这种"睡 2 秒希望加载完"的代码是评测脚本最大的敌人——会导致 50% 的截图截到加载中间态。

替代方案：

```typescript
// 等待具体元素出现
await page.waitForSelector('[data-test="match-selector"]:not(:disabled)')

// 等待网络空闲
await page.waitForLoadState('networkidle')

// 等待自定义条件
await page.waitForFunction(() => {
  return window.__appReady__ === true
})
```

### 实现关键 2：失败立即终止，不静默退化

```typescript
// ❌ 错误做法（容易滚雪球）
try {
  await selectFirstPlayer()
} catch (e) {
  console.warn('选择失败，继续...')  // 然后截了个半页
}

// ✅ 正确做法（fail fast）
try {
  await selectFirstPlayer()
} catch (e) {
  await markSampleAsFailed(sampleName, 'setupSteps failed')
  throw e  // 立即终止该样本的 capture
}
```

任何样本进入工作台失败 → 立即标记「W4 准备失败」→ 不静默返回 landing 页或半渲染数据。**论文的可重复性就是从这里来的**——所有不可控状态被显式拒绝。

### 验收标准：5 次重试 5 次成功

我们对每个样本要求 5 次独立运行 5 次都进入工作台成功（87.5% 阈值实际操作时按 5/5 = 100%）。这个标准是从 [Playwright 自身测试稳定性指南](https://playwright.dev/docs/test-retries) 借的——**flaky test 不是测试，是噪声**。

---

## 七、让评测脚本"知道自己看的是什么"

这个 bug 还有一个深层教训：**自动化脚本应该有"语义自检"能力**，不只是"流程不报错"。

我们后续加了一个最简单的 sanity check：

```typescript
// W1 capture 后，验证当前 DOM 是不是 landing 页
async function assertNotLanding(page, sample) {
  const landingMarkers = ['进入系统', '进入登录', '产品能力', 'BASKETBALL INTELLIGENCE PLATFORM']
  const text = await page.textContent('body')
  
  for (const marker of landingMarkers) {
    if (text.includes(marker)) {
      throw new Error(
        `[${sample}] 检测到 landing 页特征文本「${marker}」, ` +
        `当前可能未进入工作台。请检查 storageState / setupSteps。`
      )
    }
  }
}
```

10 行代码，但它把"假绿光" bug 关在了 capture 阶段而不是 W5 评测阶段。

**评测脚本应该悲观假设自己有 bug**——不要相信"测试通过就 OK"，要主动产出反向证据证明"我看的不是垃圾"。

---

## 八、总账：这次踩坑的代价 vs 收益

**代价**：
- 我延期 W4 中期验收 1-2 天
- U1 工程师额外 2-3 小时实现 storageState 注入 + 8 样本 setupSteps
- 我额外写了一份 4500 字的 setupSteps 清单文档

**收益**：
- 8 样本对照测试不再被 landing 页污染
- elements / capabilities / scripts 三大字段不再归零
- 论文 §4.3 评测设置章节有了**比纯算法描述更高质量的实验严肃性论证**
- 我学到的"声明式 setupSteps + storageState 注入"已经决定推广到 ReID 算法的视频测试自动化（同样是登录态 + 数据加载问题）
- 写出了这篇博客 ⊙ ω ⊙

**如果当时直接 PASS W4**：
- 论文表 5 的 13 字段准确率会是垃圾数据
- reviewer 一查 evidence 就发现 elements 字段全是 landing 页元素
- 整篇论文的可信度归零，重写

**今天**这一个工程决定，避免了**论文层面的灾难**。

---

## 九、给在做类似事情的同行的 4 条建议

### 1. URL 不是页面 — 工作台准备态才是

**任何 Web 应用的自动化测试 / Agent 操作前**，都要明确声明"从用户访问到工作台 ready"经过哪些步骤。

URL ground truth 只是第一步，后面还有：登录态 / 路由守卫 / 数据加载 / 模式选择 / 子模块进入。

### 2. 评测脚本要带"反向证据"机制

不要相信"流程不报错就 OK"。主动加 sanity check 验证"我现在看的不是垃圾"。10 行代码救一篇论文。

### 3. setupSteps 应该声明式而非脚本式

不要把 setupSteps 写成 capture 脚本里的 hard-coded 流程，要写到独立的声明文档（如 markdown 表格）。这样：
- 8 样本各自的 steps 一眼可读
- reviewer 可以快速验证"研究者声明的 setup 是否合理"
- 后续加新样本不需要改测试脚本

### 4. storageState 是处理复杂登录场景的通用解法

不只是验证码——任何包含**二次因子、CAPTCHA、设备指纹、动态 token**的登录系统，都应该用 storageState 而非 "在脚本里写自动登录代码"。

实现 5 分钟，论文友好度满分。

---

## 十、写在最后：好的评测设计是论文的隐藏护城河

很多 ML / Agent 论文的方法描述写得很漂亮，但**评测设置 1 段话就含糊带过**。

我读论文时总是先翻评测章节——如果 evaluation setup 写得清楚（数据集来源、preprocessing、登录态、超参选择、随机种子、重复实验次数），方法层即使一般我也愿意相信。如果 evaluation 写得模糊，方法层吹得再天花乱坠我也持怀疑。

**Reviewer 也是这样的人**。他们经验丰富，知道"自动化评测"很容易出现"假绿光"。一篇评测设置严谨的论文，方法层 70 分能过 CCF-B；一篇评测含糊的论文，方法层 90 分也过不了。

这次踩坑让我对论文 §4.3 评测设置那一节重新动了刀。从原来 800 字的"我们用 Playwright 抓了 8 个样本"扩到了 2500 字，新增内容包括：

- 登录态建立方式（storageState 注入，附 JSON 格式范例）
- setupSteps 声明（每样本 3-4 步，覆盖率 100%）
- Sanity check 机制（10 行代码示例）
- 失败处理协议（fail fast + 不静默退化）
- 重试稳定性要求（5 次独立运行 5 次成功）

reviewer 看完这一节，应该会觉得"这个团队真的在做生产级评测，不是 toy demo"。

这就是评测设计的护城河。

---

## 配套资源

- 🌐 **生产环境**：[admin.hooppupil.me](https://admin.hooppupil.me) — 21 个 AI 页面的 Manifest 架构
- 🔧 **开源 demo**：[manifest-architecture-demo](https://github.com/Yukibei/manifest-architecture-demo)
- 📝 **上一篇博客**：[Manifest 架构设计哲学](./2026-05-01-manifest-architecture-design-philosophy.md)
- 🤝 **协议清单**：[awesome-llm-agent-protocols](https://github.com/Yukibei/awesome-llm-agent-protocols)
- 🏠 **作者**：[github.com/Yukibei](https://github.com/Yukibei)

---

## 写在博客之外

如果你也在做 LLM Agent × Web / 自动化评测 / Auto-Manifest 类工程，遇到了类似的 landing page contamination 问题，欢迎邮件交流：2747028274@qq.com。

CCF-B 论文（《基于用户意图认知行为的前端智能体页面调度诱导系统研究》）目前 5/7 章已完成正文（约 7500 字 / 12 数学公式 / 2 引理证明），第 4 章 Auto-Manifest 评测设置部分将以本文为蓝本扩写。投稿后我会同步更新本博客增加期刊与卷号。

— 李怡霖（Yiling Li）
2026 年 5 月 1 日下午
