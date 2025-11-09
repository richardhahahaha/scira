# Extreme Search 架构深度解析

> 基于 `lib/tools/extreme-search.ts` (759 行) 的完整剖析

## 架构概览

Extreme Search 采用**两阶段 Agent 架构**：

```
┌─────────────────────────────────────────────────────┐
│                  用户查询 (Prompt)                    │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│           阶段 1: Planning (规划)                    │
│  使用 generateObject + Structured Output             │
│  模型: scira-grok-4-fast-think                       │
│  输出: ResearchPlan (1-5 个主题，每个 3-5 个任务)     │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│           阶段 2: Execution (执行)                   │
│  使用 generateText + Tools                          │
│  模型: scira-grok-4-fast-think                       │
│  Agent 自主调用工具完成 Plan                         │
│  停止条件: stepCountIs(totalTodos)                   │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│              返回结果 (Research)                      │
│  - text: Agent 的最终输出                            │
│  - sources: 所有搜集的信息源                         │
│  - charts: 代码生成的可视化                          │
│  - toolResults: 工具调用的原始结果                    │
└─────────────────────────────────────────────────────┘
```

## 核心代码拆解

### 1. 类型定义

```typescript
// line 54-60: 搜索结果的统一格式
type SearchResult = {
  title: string;
  url: string;
  content: string;          // 最多 3000 字符
  publishedDate: string;
  favicon: string;
};

// line 62-67: 研究结果的完整结构
export type Research = {
  text: string;             // Agent 的文本输出
  toolResults: any[];       // 所有工具调用的原始结果
  sources: SearchResult[];  // 去重后的信息源
  charts: any[];            // 代码生成的图表
};

// line 69-75: 搜索分类枚举
enum SearchCategory {
  NEWS = 'news',
  COMPANY = 'company',
  RESEARCH_PAPER = 'research paper',
  GITHUB = 'github',
  FINANCIAL_REPORT = 'financial report',
}
```

**设计亮点**：
- ✅ 统一的数据格式便于处理和展示
- ✅ 搜索分类让 Agent 能够精确控制信息源
- ✅ 分离 toolResults 和 sources 便于调试

### 2. 工具层实现

#### 2.1 Web Search 工具

```typescript
// line 77-110: 核心搜索函数
const searchWeb = async (
  query: string,
  category?: SearchCategory,
  include_domains?: string[]
) => {
  console.log(`searchWeb called with query: "${query}", category: ${category}`);

  try {
    const { results } = await exa.searchAndContents(query, {
      numResults: 8,
      type: 'auto',
      ...(category ? { category } : {}),
      ...(include_domains ? { include_domains } : {}),
    });

    return results.map(r => ({
      title: r.title,
      url: r.url,
      content: r.text,
      publishedDate: r.publishedDate,
      favicon: r.favicon,
    })) as SearchResult[];

  } catch (error) {
    console.error('Error in searchWeb:', error);
    return []; // ⚠️ 静默失败！
  }
};
```

**关键问题**：
- 🔴 **错误信息丢失** - 返回空数组，Agent 不知道搜索失败
- 🟡 **固定结果数** - 总是 8 个，不够灵活
- 🟢 **支持分类和域名过滤** - 提供精确控制

#### 2.2 Content 获取函数（双重备份）

```typescript
// line 112-190: 内容获取 + Firecrawl 降级
const getContents = async (links: string[]) => {
  const results: SearchResult[] = [];
  const failedUrls: string[] = [];

  // 第一次尝试：使用 Exa
  try {
    const result = await exa.getContents(links, {
      text: {
        maxCharacters: 3000,
        includeHtmlTags: false,
      },
      livecrawl: 'preferred',
    });

    for (const r of result.results) {
      if (r.text && r.text.trim()) {
        results.push({ /* ... */ });
      } else {
        failedUrls.push(r.url); // 标记失败
      }
    }
  } catch (error) {
    console.error('Exa API error:', error);
    failedUrls.push(...links); // 全部降级
  }

  // 第二次尝试：Firecrawl 降级
  if (failedUrls.length > 0) {
    for (const url of failedUrls) {
      try {
        const scrapeResponse = await firecrawl.scrape(url, {
          formats: ['markdown'],
          proxy: 'auto',
          storeInCache: true,
          parsers: ['pdf'],
        });

        if (scrapeResponse.markdown) {
          results.push({ /* ... */ });
        }
      } catch (firecrawlError) {
        console.error(`Firecrawl error for ${url}:`, firecrawlError);
      }
    }
  }

  return results;
};
```

**设计亮点**：
- ✅ **多层容错** - Exa 失败自动降级到 Firecrawl
- ✅ **智能缓存** - Firecrawl 设置 `storeInCache: true`
- ✅ **PDF 支持** - 可以解析学术论文
- 🟡 **串行处理** - 可以改为并发提升速度

### 3. Agent 规划阶段

```typescript
// line 192-260: 第一阶段 - 生成研究计划
const { object: result } = await generateObject({
  model: scira.languageModel('scira-grok-4-fast-think'),

  schema: z.object({
    plan: z.array(
      z.object({
        title: z.string().min(10).max(70),
        todos: z.array(z.string()).min(3).max(5)
      })
    ).min(1).max(5), // 最多 5 个主题，每个 3-5 个任务
  }),

  prompt: `
  Plan out the research for the following topic: ${prompt}.

  Today's Date: ${new Date().toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: '2-digit',
    weekday: 'short'
  })}

  Plan Guidelines:
  - Break down the topic into key aspects to research
  - Generate specific, diverse search queries for each aspect
  - Search for relevant information using the web search tool
  - The plan is limited to 15 actions, do not exceed this limit!
  - Keep the titles concise and to the point, no more than 70 characters
  - Mention if the topic needs to use the xSearch tool
  - Mention any need for visualizations in the plan
  - Make the plan technical and specific to the topic
  `,
});

const plan = result.plan;
const totalTodos = plan.reduce((acc, curr) => acc + curr.todos.length, 0);
```

**Prompt 工程分析**：

| Prompt 元素 | 作用 | 重要性 |
|------------|------|-------|
| `Today's Date` | 让 Agent 感知时间，搜索最新信息 | ⭐⭐⭐⭐⭐ |
| `limited to 15 actions` | 成本控制 | ⭐⭐⭐⭐⭐ |
| `specific, diverse search queries` | 避免重复搜索 | ⭐⭐⭐⭐ |
| `Mention if needs xSearch` | 引导使用社交媒体搜索 | ⭐⭐⭐ |
| `Mention visualizations` | 提示可以用代码分析 | ⭐⭐⭐ |

**实际输出示例**：
```json
{
  "plan": [
    {
      "title": "GPT-4 官方基准测试结果",
      "todos": [
        "搜索 OpenAI 官方 GPT-4 技术报告",
        "查找 GPT-4 在标准数据集上的表现",
        "对比 GPT-4 和 GPT-3.5 的性能差异"
      ]
    },
    {
      "title": "真实应用场景性能评估",
      "todos": [
        "搜索企业级应用的 GPT-4 案例",
        "使用 xSearch 查找用户真实反馈",
        "分析 GPT-4 在编程任务中的表现"
      ]
    },
    {
      "title": "性能数据可视化对比",
      "todos": [
        "收集性能指标数据",
        "使用 codeRunner 绘制对比图表"
      ]
    }
  ]
}
```

### 4. Agent 执行阶段

#### 4.1 System Prompt（核心）

```typescript
// line 268-340: Agent 的"大脑"
system: `
You are an autonomous deep research analyst.
Your goal is to research the given research plan thoroughly with the given tools.

Today's Date: ${new Date().toLocaleDateString(/* ... */)}

### PRIMARY FOCUS: SEARCH-DRIVEN RESEARCH (95% of your work)
Your main job is to SEARCH extensively and gather comprehensive information.

⚠️ IMP: Total Assistant function-call turns limit: at most ${totalTodos}!
        You must reach this limit strictly!

For searching:
- PRIORITIZE SEARCH OVER CODE - Search first, search often, search comprehensively
- Do not run all the queries at once, run them one by one
- Make 3-5 targeted searches per research topic to get different angles
- Search queries should be specific and focused, 5-15 words maximum
- Vary your search approaches: broad overview → specific details → recent developments
- Use different categories strategically: news, research papers, company info
- Use X search for real-time discussions, public opinion, breaking news
- Cross-reference information by searching from different angles
- Search for contradictory information to get balanced perspectives
- Always verify information with multiple searches from different sources

### SEARCH STRATEGY EXAMPLES:
- Topic: "AI model performance" →
  Search: "GPT-4 benchmark results 2025",
          "LLM performance comparison studies",
          "AI model evaluation metrics research"

Only use code when:
- You need to process or analyze data that was found through searches
- Mathematical calculations are required that cannot be found through search
- Creating visualizations of data trends that were discovered through research

Code guidelines (when absolutely necessary):
- Keep code simple and focused on the specific calculation or analysis needed
- Always end with print() statements for any results
- Prefer data visualization (line charts, bar charts only)
- Import required libraries: pandas, numpy, matplotlib, scipy as needed

### RESEARCH WORKFLOW:
1. Start with broad searches to understand the topic landscape
2. Identify key subtopics and drill down with specific searches
3. Look for recent developments and trends
4. Cross-validate information with searches from different categories
5. Use code execution if mathematical analysis is needed
6. Continue searching to fill any gaps in understanding

Research Plan:
${JSON.stringify(plan)}
`,
```

**Prompt 设计的精妙之处**：

1. **强制步数限制的两面性**
```typescript
⚠️ Total function-call turns limit: at most ${totalTodos}!
   You must reach this limit strictly!
```
- ✅ 防止无限循环
- ❌ 强制"凑数"，即使已经找到答案

2. **搜索优先级明确**
```
PRIMARY FOCUS: SEARCH-DRIVEN RESEARCH (95% of your work)
```
- ✅ 防止 Agent 过度使用代码执行（成本高）
- ✅ 引导正确的工作流程

3. **具体的搜索策略**
```
broad overview → specific details → recent developments → expert opinions
```
- ✅ 提供清晰的执行路径
- ✅ 避免 Agent "不知道下一步做什么"

4. **交叉验证要求**
```
- Cross-reference information
- Search for contradictory information
```
- ✅ 提升结果可靠性
- ❌ 依赖 Agent 的"自觉性"

#### 4.2 工具定义

##### Tool 1: webSearch

```typescript
// line 411-536: Web 搜索工具
webSearch: {
  description: 'Search the web for information on a topic',

  inputSchema: z.object({
    query: z.string().max(150),
    category: z.nativeEnum(SearchCategory).optional(),
    includeDomains: z.array(z.string()).optional(),
  }),

  execute: async ({ query, category, includeDomains }, { toolCallId }) => {
    // 1. 通知用户开始搜索
    if (dataStream) {
      dataStream.write({
        type: 'data-extreme_search',
        data: {
          kind: 'query',
          queryId: toolCallId,
          query: query,
          status: 'started',
        },
      });
    }

    // 2. 执行搜索
    let results = await searchWeb(query, category, includeDomains);

    // 3. 收集到全局 sources
    allSources.push(...results);

    // 4. 通知前端展示 sources
    if (dataStream) {
      results.forEach(async (source) => {
        dataStream.write({
          type: 'data-extreme_search',
          data: {
            kind: 'source',
            queryId: toolCallId,
            source: { title, url, favicon },
          },
        });
      });
    }

    // 5. 获取完整内容
    if (results.length > 0) {
      dataStream.write({ status: 'reading_content' });
      const contentsResults = await getContents(results.map(r => r.url));

      // 更新 results 和通知前端
      results = contentsResults.map(/* ... */);
    }

    // 6. 标记完成
    dataStream.write({ status: 'completed' });

    // 7. 返回给 Agent（只返回关键信息）
    return results.map(r => ({
      title: r.title,
      url: r.url,
      content: r.content,
      publishedDate: r.publishedDate,
    }));
  }
},
```

**流式通信设计**：
```
User → Agent → Tool → 实时更新 UI
                ↓
              返回给 Agent
```

**状态流转**：
```
started → showing sources → reading_content → completed
```

##### Tool 2: xSearch（X/Twitter 搜索）

```typescript
// line 538-684: X 搜索工具
xSearch: {
  description: 'Search X (formerly Twitter) posts for recent information',

  inputSchema: z.object({
    query: z.string().max(150),
    startDate: z.string().optional(),
    endDate: z.string().optional(),
    xHandles: z.array(z.string()).optional(),
    maxResults: z.number().optional(),
  }),

  execute: async ({ query, startDate, endDate, xHandles, maxResults = 15 }) => {
    // 使用 xAI Grok 的 Live Search 功能
    const { text, sources } = await generateText({
      model: xai('grok-4-fast-non-reasoning'), // 注意：用的是非推理版本
      system: `You are a helpful assistant that searches for X posts...`,
      messages: [{ role: 'user', content: query }],
      maxOutputTokens: 10, // ⚠️ 只要 10 tokens！只为触发搜索

      providerOptions: {
        xai: {
          searchParameters: {
            mode: 'on',
            fromDate: startDate || /* 7 days ago */,
            toDate: endDate || /* today */,
            maxSearchResults: maxResults,
            returnCitations: true,
            sources: [
              xHandles?.length > 0
                ? { type: 'x', xHandles }
                : { type: 'x' }
            ],
          },
        } satisfies XaiProviderOptions,
      },
    });

    // 从 citations 中提取推文
    const tweetFetchPromises = citations
      .filter(link => link.sourceType === 'url')
      .map(async (link) => {
        const tweetId = link.url.match(/\/status\/(\d+)/)?.[1];
        const tweetData = await getTweet(tweetId); // react-tweet API
        return {
          text: tweetData.text,
          link: link.url,
          title: `Post from @${tweetData.user.screen_name}`,
        };
      });

    const tweetResults = await Promise.all(tweetFetchPromises);

    return {
      content: text,
      citations: citations,
      sources: tweetResults,
      dateRange: `${startDate} to ${endDate}`,
      handles: xHandles || [],
    };
  }
},
```

**巧妙之处**：
1. **Grok 的 Live Search** - 利用 xAI 的原生能力搜索 X
2. **maxOutputTokens: 10** - 不需要 LLM 输出，只要搜索结果
3. **获取完整推文** - 通过 `react-tweet` 获取完整数据

##### Tool 3: codeRunner

```typescript
// line 350-409: Python 代码执行
codeRunner: {
  description: 'Run Python code in a sandbox',

  inputSchema: z.object({
    title: z.string(),
    code: z.string(),
  }),

  execute: async ({ title, code }) => {
    // 1. 检查需要安装的库
    const imports = code.match(/import\s+([\w\s,]+)/);
    const importLibs = imports ? imports[1].split(',') : [];
    const missingLibs = importLibs.filter(
      lib => !pythonLibsAvailable.includes(lib)
    );

    // 2. 通知用户代码运行中
    dataStream.write({
      kind: 'code',
      codeId: `code-${Date.now()}`,
      title, code,
      status: 'running',
    });

    // 3. 执行代码（Daytona 沙箱）
    const response = await runCode(code, missingLibs);

    // 4. 提取图表数据
    const charts = response.artifacts?.charts?.map(chart => {
      const { png, ...chartWithoutPng } = chart;
      return chartWithoutPng; // 移除 PNG 数据，减少传输
    }) || [];

    // 5. 通知完成并返回结果
    dataStream.write({
      kind: 'code',
      status: 'completed',
      result: response.result,
      charts: charts,
    });

    return { result: response.result, charts };
  }
},
```

**Daytona 沙箱**：
```typescript
const daytona = new Daytona({
  apiKey: serverEnv.DAYTONA_API_KEY,
  target: 'us',
});

const runCode = async (code: string, installLibs: string[] = []) => {
  const sandbox = await daytona.create({
    snapshot: SNAPSHOT_NAME, // 预配置的环境
  });

  if (installLibs.length > 0) {
    await sandbox.process.executeCommand(`pip install ${installLibs.join(' ')}`);
  }

  const result = await sandbox.process.codeRun(code);
  sandbox.delete(); // ⚠️ 用后立即删除
  return result;
};
```

### 5. 结果汇总与去重

```typescript
// line 707-730: 汇总结果
const chartResults = toolResults.filter(
  result => result.toolName === 'codeRunner' &&
            'charts' in result.result
);

const charts = chartResults.flatMap(
  result => result.result.charts || []
);

return {
  text,
  toolResults,
  sources: Array.from(
    new Map(
      allSources.map(s => [
        s.url,
        { ...s, content: s.content.slice(0, 3000) + '...' }
      ])
    ).values()
  ), // URL 去重
  charts,
};
```

**去重策略**：
- 使用 `Map` 以 URL 为 key
- 内容截断到 3000 字符节省空间

## 流式输出架构

### UIMessageStreamWriter 机制

```typescript
dataStream.write({
  type: 'data-extreme_search',
  data: {
    kind: 'plan' | 'query' | 'source' | 'content' | 'code' | 'x_search',
    // 根据 kind 的不同，携带不同的数据
  }
});
```

### 前端消费（components/extreme-search.tsx）

```tsx
// 监听流式数据
useEffect(() => {
  toolInvocation.data.forEach((item: DataExtremeSearchPart) => {
    switch (item.kind) {
      case 'plan':
        setPlan(item.plan);
        setStatus(item.status?.title);
        break;

      case 'query':
        if (item.status === 'started') {
          addNewQuery(item.queryId, item.query);
        } else if (item.status === 'completed') {
          markQueryComplete(item.queryId);
        }
        break;

      case 'source':
        addSourceToQuery(item.queryId, item.source);
        break;

      case 'code':
        updateCodeExecution(item.codeId, item.status, item.result);
        break;
    }
  });
}, [toolInvocation.data]);
```

## 关键设计模式

### 1. **Dual-Phase Agent Pattern**
```
Planning Phase (generateObject)
  +
Execution Phase (generateText + Tools)
```

### 2. **Fallback Chain Pattern**
```
Exa Search → (fail) → Firecrawl Scrape → (fail) → Return Empty
```

### 3. **Stream-First Architecture**
```
Tool Execution → Immediate UI Update → Return to Agent
```

### 4. **Accumulator Pattern**
```typescript
const allSources: SearchResult[] = []; // 全局累加器

tools: {
  webSearch: {
    execute: async () => {
      const results = await search();
      allSources.push(...results); // 持续累加
      return results;
    }
  }
}
```

## 性能与成本分析

### 典型执行流程的成本

```
查询: "分析 React 19 的新特性"

1. Planning Phase:
   - generateObject() → ~500 tokens
   - 成本: $0.01

2. Execution Phase:
   - generateText() 主循环 → ~2000 tokens
   - webSearch × 8 次 → Exa API $0.16
   - xSearch × 2 次 → Grok API $0.20
   - codeRunner × 1 次 → Daytona $0.05
   - 成本: $0.45

3. Total: ~$0.46, ~45 秒
```

### 成本优化机会

1. **缓存搜索结果**
```typescript
const searchCache = new Map<string, SearchResult[]>();

async function searchWeb(query: string) {
  if (searchCache.has(query)) {
    return searchCache.get(query);
  }
  const results = await exa.search(query);
  searchCache.set(query, results);
  return results;
}
```

2. **并发获取内容**
```typescript
// 当前: 串行
for (const url of failedUrls) {
  await firecrawl.scrape(url);
}

// 优化: 并发
await Promise.allSettled(
  failedUrls.map(url => firecrawl.scrape(url))
);
```

3. **动态步数限制**
```typescript
// 当前: 固定步数
stopWhen: stepCountIs(totalTodos)

// 优化: 信息充足时提前退出
stopWhen: (step) => {
  return stepCountIs(totalTodos)(step) ||
         isInformationSufficient(step.toolResults);
}
```

## 总结

Extreme Search 的架构体现了现代 AI Agent 的最佳实践：

✅ **清晰的职责分离** - Planning vs Execution
✅ **丰富的工具生态** - Web, X, Code
✅ **实时用户反馈** - 流式输出
✅ **多层容错** - Fallback chains
✅ **成本控制** - 步数限制

但也存在改进空间：

⚠️ **错误处理** - 静默失败导致信息丢失
⚠️ **步数强制** - 可能过度搜索
⚠️ **串行执行** - 性能优化空间大

---

**下一步**: [结果质量分析](./04-结果质量分析.md)
