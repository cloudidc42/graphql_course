# Part 098 — GraphQL + AI: LLM Integration 🤖

> **ระดับ:** World-Class | **เวลาเรียน:** 85 นาที | **ขั้นตอนที่:** 2886–2930

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL API สำหรับ AI applications
- Streaming responses กับ GraphQL
- AI-generated queries (NLP → GraphQL)
- Vector search integration
- RAG pattern ด้วย GraphQL
- AI tools/functions เป็น GraphQL mutations
- Rate limiting สำหรับ AI endpoints
- Cost tracking สำหรับ AI operations

---

## 📌 Step 2886: Streaming AI Responses

```graphql
# schema/ai.graphql

type Query {
  # Standard (non-streaming): รอ complete response
  askQuestion(question: String!): AIResponse!
}

type Mutation {
  # Streaming: ใช้ Subscription pattern
  startAIChat(sessionId: ID!, message: String!): AIChatStarted!
}

type Subscription {
  # Stream tokens as they arrive
  aiChatStream(sessionId: ID!): AIChatChunk!
}

type AIChatChunk {
  sessionId: ID!
  token: String!       # ส่งทีละ token
  isComplete: Boolean!
  metadata: AIChatMetadata
}

type AIChatMetadata {
  model: String!
  promptTokens: Int
  completionTokens: Int
  totalCost: Float     # USD
}

type AIResponse {
  text: String!
  sources: [Source!]!
  metadata: AIChatMetadata!
}

type Source {
  title: String!
  url: String
  relevanceScore: Float!
  excerpt: String!
}
```

---

## 📌 Step 2887: AI Streaming Resolver

```typescript
// src/resolvers/ai.resolver.ts
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const aiResolvers = {
  Mutation: {
    startAIChat: async (
      _: unknown,
      { sessionId, message }: { sessionId: string; message: string },
      { user, redis, pubsub }: AppContext
    ) => {
      requireAuth(user);
      
      // Rate limiting สำหรับ AI (expensive!)
      const key = `ai_limit:${user!.id}`;
      const count = await redis.incr(key);
      await redis.expire(key, 3600);  // per hour
      
      if (count > 50) {  // max 50 AI queries per hour
        throw new AppError('AI rate limit exceeded', ErrorCode.RATE_LIMITED);
      }
      
      // Start streaming ใน background
      void streamAIResponse(sessionId, message, user!.id, pubsub, anthropic);
      
      return { sessionId, started: true };
    },
  },
  
  Subscription: {
    aiChatStream: {
      subscribe: withFilter(
        (_, { sessionId }) => pubsub.asyncIterator(`AI_STREAM:${sessionId}`),
        (payload, args) => payload.aiChatStream.sessionId === args.sessionId
      ),
    },
  },
  
  Query: {
    // Non-streaming: for simple Q&A
    askQuestion: async (
      _: unknown,
      { question }: { question: string },
      { user }: AppContext
    ) => {
      requireAuth(user);
      
      // RAG: ค้นหา relevant documents ก่อน
      const sources = await vectorSearch(question);
      const context = sources.map(s => s.content).join('\n\n');
      
      const response = await anthropic.messages.create({
        model: 'claude-haiku-4-5-20251001',
        max_tokens: 1024,
        system: `You are a helpful assistant. Use the following context to answer questions:\n\n${context}`,
        messages: [{ role: 'user', content: question }],
      });
      
      const text = response.content[0].type === 'text'
        ? response.content[0].text
        : '';
      
      return {
        text,
        sources: sources.map(s => ({
          title: s.title,
          url: s.url,
          relevanceScore: s.score,
          excerpt: s.excerpt,
        })),
        metadata: {
          model: response.model,
          promptTokens: response.usage.input_tokens,
          completionTokens: response.usage.output_tokens,
          totalCost: calculateCost(response.model, response.usage),
        },
      };
    },
  },
};

async function streamAIResponse(
  sessionId: string,
  message: string,
  userId: string,
  pubsub: PubSubEngine,
  client: Anthropic
) {
  try {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-6',
      max_tokens: 2048,
      messages: [{ role: 'user', content: message }],
    });
    
    for await (const event of stream) {
      if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
        await pubsub.publish(`AI_STREAM:${sessionId}`, {
          aiChatStream: {
            sessionId,
            token: event.delta.text,
            isComplete: false,
          },
        });
      }
    }
    
    const finalMessage = await stream.finalMessage();
    
    // Send completion event
    await pubsub.publish(`AI_STREAM:${sessionId}`, {
      aiChatStream: {
        sessionId,
        token: '',
        isComplete: true,
        metadata: {
          model: finalMessage.model,
          promptTokens: finalMessage.usage.input_tokens,
          completionTokens: finalMessage.usage.output_tokens,
          totalCost: calculateCost(finalMessage.model, finalMessage.usage),
        },
      },
    });
    
    // Track cost
    await trackAICost(userId, finalMessage.model, finalMessage.usage);
    
  } catch (err) {
    await pubsub.publish(`AI_STREAM:${sessionId}`, {
      aiChatStream: {
        sessionId,
        token: 'Error: ' + (err as Error).message,
        isComplete: true,
      },
    });
  }
}

function calculateCost(model: string, usage: { input_tokens: number; output_tokens: number }): number {
  const pricing: Record<string, { input: number; output: number }> = {
    'claude-haiku-4-5-20251001': { input: 0.00025, output: 0.00125 },
    'claude-sonnet-4-6': { input: 0.003, output: 0.015 },
    'claude-opus-5-5': { input: 0.015, output: 0.075 },
  };
  
  const prices = pricing[model] ?? pricing['claude-sonnet-4-6'];
  return (usage.input_tokens / 1000) * prices.input +
         (usage.output_tokens / 1000) * prices.output;
}
```

---

## 📌 Step 2888: Vector Search Integration

```typescript
// src/ai/vector-search.ts
// ใช้ pgvector สำหรับ semantic search

import { Pinecone } from '@pinecone-database/pinecone';
import Anthropic from '@anthropic-ai/sdk';

const pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
const anthropicClient = new Anthropic();

export interface SearchResult {
  id: string;
  title: string;
  content: string;
  excerpt: string;
  url?: string;
  score: number;
}

export async function vectorSearch(
  query: string,
  topK = 5
): Promise<SearchResult[]> {
  // 1. Embed query
  const embeddingResponse = await anthropicClient.messages.create({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 1,
    messages: [{
      role: 'user',
      content: `Generate embedding for: ${query}`,
    }],
  });
  
  // หรือใช้ OpenAI embeddings
  // const { data } = await openai.embeddings.create({
  //   model: 'text-embedding-3-small',
  //   input: query,
  // });
  // const embedding = data[0].embedding;
  
  // 2. Search ใน Pinecone
  const index = pinecone.index('knowledge-base');
  const searchResult = await index.query({
    vector: new Array(1536).fill(0),  // placeholder: ใช้ embedding จริง
    topK,
    includeMetadata: true,
  });
  
  return searchResult.matches.map(match => ({
    id: match.id,
    title: String(match.metadata?.title ?? ''),
    content: String(match.metadata?.content ?? ''),
    excerpt: String(match.metadata?.content ?? '').slice(0, 200),
    url: match.metadata?.url as string | undefined,
    score: match.score ?? 0,
  }));
}

// GraphQL resolver สำหรับ search
const searchResolvers = {
  Query: {
    semanticSearch: async (
      _: unknown,
      { query, limit = 10 }: { query: string; limit: number },
      { user }: AppContext
    ) => {
      requireAuth(user);
      return vectorSearch(query, limit);
    },
  },
};
```

---

## 📌 Step 2889: AI Tools เป็น GraphQL Mutations

```typescript
// src/ai/tools.ts
// ใช้ GraphQL mutations เป็น AI tools (function calling)

const AI_TOOLS = [
  {
    name: 'search_products',
    description: 'ค้นหาสินค้าตาม keyword และ filter',
    input_schema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'คำค้นหา' },
        maxPrice: { type: 'number', description: 'ราคาสูงสุด (บาท)' },
        category: { type: 'string', description: 'หมวดหมู่สินค้า' },
      },
      required: ['query'],
    },
  },
  {
    name: 'get_order_status',
    description: 'ดูสถานะคำสั่งซื้อ',
    input_schema: {
      type: 'object',
      properties: {
        orderId: { type: 'string', description: 'เลขที่คำสั่งซื้อ' },
      },
      required: ['orderId'],
    },
  },
];

// AI Agent resolver
const agentResolvers = {
  Mutation: {
    runAIAgent: async (
      _: unknown,
      { goal }: { goal: string },
      { user, prisma }: AppContext
    ) => {
      requireAuth(user);
      
      const messages: Anthropic.MessageParam[] = [
        { role: 'user', content: goal },
      ];
      
      // Agentic loop
      let response = await anthropic.messages.create({
        model: 'claude-sonnet-4-6',
        max_tokens: 4096,
        tools: AI_TOOLS,
        messages,
      });
      
      const toolResults: unknown[] = [];
      
      while (response.stop_reason === 'tool_use') {
        const toolUse = response.content.find(b => b.type === 'tool_use');
        if (!toolUse || toolUse.type !== 'tool_use') break;
        
        // Execute tool (GraphQL mutation)
        let toolResult;
        switch (toolUse.name) {
          case 'search_products':
            toolResult = await prisma.product.findMany({
              where: {
                name: { contains: (toolUse.input as Record<string, string>).query },
                price: (toolUse.input as Record<string, number>).maxPrice
                  ? { lte: (toolUse.input as Record<string, number>).maxPrice }
                  : undefined,
              },
              take: 5,
            });
            break;
            
          case 'get_order_status':
            toolResult = await prisma.order.findFirst({
              where: {
                id: (toolUse.input as Record<string, string>).orderId,
                userId: user!.id,
              },
            });
            break;
        }
        
        toolResults.push({ tool: toolUse.name, result: toolResult });
        
        // Continue conversation with tool result
        messages.push({ role: 'assistant', content: response.content });
        messages.push({
          role: 'user',
          content: [{
            type: 'tool_result',
            tool_use_id: toolUse.id,
            content: JSON.stringify(toolResult),
          }],
        });
        
        response = await anthropic.messages.create({
          model: 'claude-sonnet-4-6',
          max_tokens: 4096,
          tools: AI_TOOLS,
          messages,
        });
      }
      
      const finalText = response.content
        .filter(b => b.type === 'text')
        .map(b => b.type === 'text' ? b.text : '')
        .join('');
      
      return {
        answer: finalText,
        toolsUsed: toolResults.map(r => (r as Record<string, unknown>).tool as string),
      };
    },
  },
};
```

---

## 📌 Step 2890: สรุป Part 098

### เนื้อหาที่เรียนรู้

✅ AI streaming ด้วย Subscriptions  
✅ RAG pattern ด้วย vector search  
✅ GraphQL mutations เป็น AI tools  
✅ Cost tracking  
✅ Rate limiting สำหรับ AI  

### AI + GraphQL Pattern Guide

```
Streaming:
→ Subscription + Server-Sent Events
→ Token-by-token delivery
→ isComplete flag

RAG (Retrieval-Augmented Generation):
→ Vector search → context → LLM prompt
→ Sources ใน response
→ Semantic search endpoint

AI Agent:
→ GraphQL mutations เป็น tools
→ Agentic loop (tool_use → execute → continue)
→ Structured output

Rate & Cost:
→ Rate limit per user/hour (expensive)
→ Track tokens + cost per operation
→ Budget alerts ถ้า cost สูง
→ Model selection ตาม task complexity
```

### ในส่วนถัดไป

➡️ **[Part 099](./part-099.md)** — GraphQL Performance Deep Dive: Advanced Optimization

---

*Part 098 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
