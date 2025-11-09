# 硬编码 Workflow vs AI Agent：两种范式的本质差异

## 核心区别概览

### 硬编码 Workflow（传统方式）

```javascript
// 伪代码示例
async function deepResearch(query) {
  // 步骤 1: 固定的初始搜索
  const results1 = await search(query);

  // 步骤 2: 提取关键词
  const keywords = extractKeywords(results1);

  // 步骤 3: 扩展搜索
  const results2 = await search(keywords[0]);
  const results3 = await search(keywords[1]);

  // 步骤 4: 汇总
  return summarize([results1, results2, results3]);
}
```

**特征：**
- 🔒 **固定流程** - 代码决定执行路径
- 📏 **确定性强** - 相同输入 → 相同步骤
- ⚡ **可预测** - 性能和成本可控

### AI Agent（当前实现）

```typescript
// Scira 的实际代码简化版
async function extremeSearch(prompt) {
  // 步骤 1: AI 规划
  const plan = await generateObject({
    model: 'grok-4-fast-think',
    prompt: `Plan research for: ${prompt}`,
    schema: ResearchPlanSchema
  });

  // 步骤 2: Agent 自主执行
  const { text } = await generateText({
    model: 'grok-4-fast-think',
    tools: { webSearch, xSearch, codeRunner },
    stopWhen: stepCountIs(plan.totalSteps),
    // AI 自己决定何时调用哪个工具！
  });

  return text;
}
```

**特征：**
- 🧠 **AI 决策** - LLM 决定执行路径
- 🔀 **不确定性** - 相同输入 → 可能不同路径
- 🎯 **适应性强** - 根据中间结果调整策略

## 深度对比：控制流

### 1. 控制流的本质差异

#### 硬编码：命令式编程
```python
# 你（程序员）告诉计算机"怎么做"
def research_topic(topic):
    # 明确的步骤 1
    web_results = search_web(topic)

    # 明确的步骤 2
    if len(web_results) < 5:
        web_results += search_web(topic + " overview")

    # 明确的步骤 3
    for result in web_results[:3]:
        detailed = fetch_content(result.url)
        analyze(detailed)

    # 明确的步骤 4
    return generate_report()
```

**控制权**：程序员 100% 控制

#### AI Agent：目标导向编程
```typescript
// 你告诉 AI "做什么"，AI 决定"怎么做"
const result = await generateText({
  prompt: `Research topic: ${topic}`,
  tools: {
    webSearch: { /* ... */ },
    fetchContent: { /* ... */ },
    analyze: { /* ... */ }
  },
  // AI 自己决定：
  // - 调用哪个工具
  // - 何时调用
  // - 调用几次
  // - 何时停止
});
```

**控制权**：AI 70% 控制，程序员 30% 约束

### 2. 决策树对比

#### 硬编码决策树（静态）

```
查询: "分析 GPT-4 性能"
│
├─ 步骤 1: 搜索 "GPT-4 performance"
│   └─ 固定获取前 10 个结果
│
├─ 步骤 2: 提取关键词 ["benchmark", "comparison"]
│   └─ 基于简单规则（如 TF-IDF）
│
├─ 步骤 3: 搜索 "GPT-4 benchmark"
│   └─ 固定获取前 5 个结果
│
├─ 步骤 4: 搜索 "GPT-4 comparison"
│   └─ 固定获取前 5 个结果
│
└─ 步骤 5: 汇总并返回
```

**问题**：
- ❌ 如果步骤 1 已经找到完美答案，仍会执行步骤 2-4
- ❌ 如果结果不够，无法自动扩展搜索
- ❌ 无法处理意外情况（如某个 API 失败）

#### Agent 决策树（动态）

```
查询: "分析 GPT-4 性能"
│
├─ AI 规划阶段:
│   ├─ 主题 1: 基准测试结果
│   │   └─ 子任务: [搜索官方 benchmark, 第三方评测, 对比 GPT-3.5]
│   ├─ 主题 2: 实际应用性能
│   │   └─ 子任务: [用户反馈, 案例研究]
│   └─ 主题 3: 成本效益分析
│       └─ 子任务: [定价, ROI 计算]
│
├─ AI 执行阶段 (动态):
│   ├─ 调用 webSearch("GPT-4 official benchmarks")
│   │   └─ 发现 OpenAI 官方报告 ✅
│   │
│   ├─ 调用 webSearch("GPT-4 vs GPT-3.5 comparison")
│   │   └─ 结果不够详细 ⚠️
│   │
│   ├─ 调用 xSearch("GPT-4 user experience") ← AI 自主决定用 X 搜索
│   │   └─ 发现大量实际反馈 ✅
│   │
│   ├─ 调用 codeRunner(绘制性能对比图) ← AI 决定需要可视化
│   │   └─ 生成图表 ✅
│   │
│   └─ 停止 (信息已充足)
```

**优势**：
- ✅ 根据中间结果调整策略
- ✅ 可以跳过不必要的步骤
- ✅ 能够探索意外的信息源

## 代码结构对比

### 硬编码版本的典型结构

```javascript
// 假设的旧版实现
class DeepResearchWorkflow {
  async execute(query) {
    try {
      // 阶段 1: 初始搜索
      const initialResults = await this.initialSearch(query);
      this.validateResults(initialResults, 'initial');

      // 阶段 2: 关键词提取
      const keywords = this.extractKeywords(initialResults);

      // 阶段 3: 扩展搜索
      const expandedResults = await Promise.all(
        keywords.map(kw => this.expandSearch(kw))
      );

      // 阶段 4: 深度内容获取
      const contents = await this.fetchContents(
        expandedResults.flat().slice(0, 10)
      );

      // 阶段 5: 分析和汇总
      return this.synthesize(contents);

    } catch (error) {
      return this.handleError(error);
    }
  }

  // 每个步骤都是明确定义的函数
  async initialSearch(query) { /* 固定逻辑 */ }
  extractKeywords(results) { /* 固定算法 */ }
  async expandSearch(keyword) { /* 固定策略 */ }
  async fetchContents(urls) { /* 固定流程 */ }
  synthesize(contents) { /* 固定模板 */ }
}
```

**特点**：
- 📦 每个函数职责明确
- 🔍 易于调试（可以在每步打断点）
- 📊 易于监控（每步都有明确的 metrics）
- ⚙️ 易于优化（针对性改进某个步骤）

### AI Agent 版本（当前实现）

```typescript
// lib/tools/extreme-search.ts (简化版)
async function extremeSearch(prompt, dataStream) {
  // 第一阶段：AI 生成计划（动态！）
  const { object: plan } = await generateObject({
    model: scira.languageModel('scira-grok-4-fast-think'),
    schema: z.object({
      plan: z.array(
        z.object({
          title: z.string(),
          todos: z.array(z.string()).min(3).max(5)
        })
      ).min(1).max(5)
    }),
    prompt: `Plan out the research for: ${prompt}

    Guidelines:
    - Break down into key aspects
    - Generate specific search queries
    - Plan is limited to 15 actions
    - Mention if needs visualizations
    `
  });

  // 第二阶段：Agent 自主执行（动态！）
  const { text } = await generateText({
    model: scira.languageModel('scira-grok-4-fast-think'),
    stopWhen: stepCountIs(totalTodos),

    system: `
    You are an autonomous research analyst.

    PRIMARY FOCUS: SEARCH-DRIVEN RESEARCH (95% of work)
    - Make 3-5 targeted searches per topic
    - Vary approaches: broad → specific → recent → expert
    - Cross-reference information
    - Only use code when absolutely necessary

    Research Plan: ${JSON.stringify(plan)}
    `,

    tools: {
      webSearch: tool({
        description: 'Search the web',
        inputSchema: z.object({
          query: z.string(),
          category: z.enum(['news', 'research', 'company']).optional()
        }),
        execute: async ({ query, category }) => {
          const results = await exa.searchAndContents(query, { category });
          // Fallback to Firecrawl if needed
          return results;
        }
      }),

      xSearch: tool({
        description: 'Search X (Twitter)',
        inputSchema: z.object({
          query: z.string(),
          startDate: z.string().optional(),
          xHandles: z.array(z.string()).optional()
        }),
        execute: async ({ query, startDate, xHandles }) => {
          // Use xAI's Grok with live search
          return await searchXPosts(query, { startDate, xHandles });
        }
      }),

      codeRunner: tool({
        description: 'Run Python code',
        inputSchema: z.object({
          code: z.string()
        }),
        execute: async ({ code }) => {
          return await daytona.runCode(code);
        }
      })
    },

    // AI 自己决定何时调用哪个工具！
  });

  return { text, sources, charts };
}
```

**特点**：
- 🌟 代码更简洁（从 500 行降到 ~100 行核心逻辑）
- 🔮 逻辑在 prompt 中（system message）
- 🎲 执行路径不确定
- 🐛 调试困难（AI 的决策过程是黑盒）

## 优缺点对比表

| 维度 | 硬编码 Workflow | AI Agent |
|------|----------------|----------|
| **开发难度** | ⭐⭐⭐⭐ 需要深入理解业务逻辑 | ⭐⭐⭐ 主要是 prompt 工程 |
| **代码量** | ⭐ 500-1000 行 | ⭐⭐⭐⭐ 100-200 行 |
| **灵活性** | ⭐⭐ 需要修改代码才能改变逻辑 | ⭐⭐⭐⭐⭐ 修改 prompt 即可 |
| **可预测性** | ⭐⭐⭐⭐⭐ 100% 确定 | ⭐⭐ 每次可能不同 |
| **调试难度** | ⭐⭐ 可以打断点 | ⭐⭐⭐⭐ AI 决策难追踪 |
| **成本控制** | ⭐⭐⭐⭐⭐ 完全可控 | ⭐⭐ 可能超预算 |
| **处理简单任务** | ⭐⭐⭐⭐⭐ 快速高效 | ⭐⭐ 过度设计 |
| **处理复杂任务** | ⭐⭐ 可能遗漏边缘情况 | ⭐⭐⭐⭐⭐ 自适应探索 |
| **错误处理** | ⭐⭐⭐⭐ 明确的 try-catch | ⭐⭐⭐ 依赖 AI 理解错误 |
| **性能优化** | ⭐⭐⭐⭐ 可精确优化每个步骤 | ⭐⭐⭐ 只能通过 prompt 引导 |
| **可维护性** | ⭐⭐⭐⭐ 逻辑清晰 | ⭐⭐⭐ 逻辑分散在 prompts |
| **测试容易度** | ⭐⭐⭐⭐⭐ 单元测试简单 | ⭐⭐ 非确定性难测 |

## 实际案例对比

### 案例 1: 简单查询 "北京今天天气"

**硬编码：**
```javascript
// 3 秒，$0.01
1. 识别为天气查询
2. 调用天气 API
3. 格式化返回
```

**AI Agent：**
```typescript
// 8 秒，$0.05
1. AI 规划: "需要查天气"
2. AI 决定: 调用 webSearch("北京天气")
3. AI 可能还搜: "北京空气质量" (过度)
4. AI 汇总结果
```

**结论**：硬编码完胜 ⭐⭐⭐⭐⭐

### 案例 2: 复杂研究 "AI 监管的全球趋势对比"

**硬编码：**
```javascript
// 20 秒，$0.15
1. 搜 "AI regulation 2025"
2. 搜 "AI policy comparison"
3. 搜 "AI governance trends"
4. 汇总前 20 个结果
// 问题: 可能遗漏欧盟、中国、美国的具体政策差异
```

**AI Agent：**
```typescript
// 45 秒，$0.60
1. AI 规划:
   - 美国 AI 监管现状
   - 欧盟 AI Act 详解
   - 中国 AI 治理框架
   - 全球对比分析

2. AI 执行:
   - 搜 "US AI Executive Order 2024"
   - 搜 "EU AI Act implementation"
   - 搜 "China AI governance"
   - xSearch("AI regulation debate") ← 发现实时讨论
   - codeRunner(绘制对比表) ← 主动可视化

// 优势: 多维度，自动发现相关话题
```

**结论**：AI Agent 完胜 ⭐⭐⭐⭐⭐

## 技术演进的必然性

### 为什么从硬编码演进到 Agent？

1. **任务复杂度提升**
   - 早期：简单的"搜索 + 汇总"
   - 现在：多步推理、跨领域整合

2. **用户期望提升**
   - 早期：给我 10 个链接
   - 现在：给我完整的分析报告

3. **LLM 能力提升**
   - GPT-3 时代：只能做文本生成
   - GPT-4/Grok 时代：可以可靠地调用工具和规划

4. **框架成熟**
   - 早期：需要手写工具调用解析
   - 现在：Vercel AI SDK 提供开箱即用的 Agent 能力

### 从 Scira 的 Git 历史看演进

```bash
# 查看演进历史
git log --oneline --all | grep -i "research\|search\|agent"
```

可能的演进路径：
```
v1.0 (2023-Q1): 基础搜索 + 手写 workflow
v2.0 (2023-Q3): 引入 LLM 做结果汇总
v2.5 (2024-Q1): 实验性 Agent (不稳定)
v3.0 (2024-Q4): 完整 Agent 架构 (当前)
```

## 关键启示

### 🎯 不是非此即彼

最佳实践不是选择一种抛弃另一种，而是：

```typescript
function smartResearch(query, complexity) {
  if (isSimpleQuery(query)) {
    return hardcodedWorkflow(query);  // 快速、便宜、可靠
  } else if (isComplexResearch(query)) {
    return agentWorkflow(query);      // 灵活、深入、创新
  } else {
    return hybridApproach(query);     // 结合两者优势
  }
}
```

### 🎯 Agent 不是银弹

AI Agent 更适合：
- ✅ 探索性任务（不知道确切的搜索路径）
- ✅ 高度个性化（每次查询都很不同）
- ✅ 创造性工作（需要发现意外关联）

硬编码更适合：
- ✅ 重复性任务（天气查询、货币转换）
- ✅ 性能敏感（低延迟要求）
- ✅ 成本敏感（大量用户调用）

---

**下一步**: [Extreme Search 架构深度解析](./03-Extreme-Search架构深度解析.md)
