# Part 060 — AI/ML Integration กับ GraphQL 🤖

> **ระดับ:** Expert | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 1311–1350

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Schema สำหรับ AI features
- LLM API integration (OpenAI, Anthropic, Google)
- Streaming responses ผ่าน GraphQL Subscriptions
- Vector search กับ pgvector/Pinecone
- Recommendation engine
- AI-powered schema generation
- Tool calling / function calling pattern
- RAG (Retrieval-Augmented Generation)

---

## 📌 Step 1311: GraphQL Schema สำหรับ AI

```graphql
# src/schema/ai.graphql

type Query {
  # Semantic search
  semanticSearch(query: String!, limit: Int = 10): [SearchResult!]!
  
  # Product recommendations
  recommendations(userId: ID!, limit: Int = 10): [ProductRecommendation!]!
  
  # Similar products
  similarProducts(productId: ID!, limit: Int = 5): [Product!]!
}

type Mutation {
  # Chat with AI assistant
  sendMessage(input: ChatInput!): ChatResponse!
  
  # AI-powered product description
  generateProductDescription(productId: ID!): String!
  
  # Auto-categorize product
  categorizeProduct(input: ProductInput!): CategorySuggestion!
  
  # Sentiment analysis
  analyzeSentiment(text: String!): SentimentResult!
}

type Subscription {
  # Streaming LLM response
  chatStream(sessionId: ID!, message: String!): ChatChunk!
}

input ChatInput {
  sessionId: ID!
  message: String!
  context: ChatContext
}

type ChatContext {
  productId: ID
  orderId: ID
  userId: ID
}

type ChatResponse {
  id: ID!
  sessionId: ID!
  role: ChatRole!
  content: String!
  usage: TokenUsage
  finishReason: String
}

enum ChatRole {
  USER
  ASSISTANT
  SYSTEM
}

type ChatChunk {
  sessionId: ID!
  delta: String!
  isComplete: Boolean!
  usage: TokenUsage
}

type TokenUsage {
  promptTokens: Int!
  completionTokens: Int!
  totalTokens: Int!
}

type ProductRecommendation {
  product: Product!
  score: Float!
  reason: String!
}

type SentimentResult {
  sentiment: Sentiment!
  score: Float!
  aspects: [AspectSentiment!]!
}

enum Sentiment {
  POSITIVE
  NEGATIVE
  NEUTRAL
  MIXED
}

type AspectSentiment {
  aspect: String!
  sentiment: Sentiment!
  score: Float!
}
```

---

## 📌 Step 1312: OpenAI Integration

```bash
npm install openai
```

```typescript
// src/ai/openai.ts
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Simple chat completion
export async function chat(
  messages: OpenAI.Chat.ChatCompletionMessageParam[],
  options?: { maxTokens?: number; temperature?: number }
) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages,
    max_tokens: options?.maxTokens ?? 1000,
    temperature: options?.temperature ?? 0.7,
  });
  
  return {
    content: response.choices[0]?.message.content ?? '',
    usage: response.usage,
    finishReason: response.choices[0]?.finish_reason,
  };
}

// Streaming response
export async function* streamChat(
  messages: OpenAI.Chat.ChatCompletionMessageParam[]
): AsyncGenerator<{ delta: string; isComplete: boolean }> {
  const stream = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages,
    stream: true,
  });
  
  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta.content ?? '';
    const isComplete = chunk.choices[0]?.finish_reason === 'stop';
    
    if (delta || isComplete) {
      yield { delta, isComplete };
    }
  }
}

// Generate embeddings สำหรับ semantic search
export async function embed(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
    dimensions: 1536,
  });
  
  return response.data[0]?.embedding ?? [];
}

// Tool calling pattern
export async function chatWithTools(
  message: string,
  context: { userId: string; sessionId: string }
) {
  const tools: OpenAI.Chat.ChatCompletionTool[] = [
    {
      type: 'function',
      function: {
        name: 'get_product_info',
        description: 'ดึงข้อมูล product จาก ID',
        parameters: {
          type: 'object',
          properties: {
            productId: { type: 'string', description: 'Product ID' },
          },
          required: ['productId'],
        },
      },
    },
    {
      type: 'function',
      function: {
        name: 'check_order_status',
        description: 'ตรวจสอบสถานะ order',
        parameters: {
          type: 'object',
          properties: {
            orderId: { type: 'string' },
          },
          required: ['orderId'],
        },
      },
    },
  ];
  
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: 'คุณเป็น customer service AI ของ E-commerce' },
      { role: 'user', content: message },
    ],
    tools,
    tool_choice: 'auto',
  });
  
  const choice = response.choices[0]!;
  
  // Handle tool calls
  if (choice.finish_reason === 'tool_calls' && choice.message.tool_calls) {
    const toolResults = await Promise.all(
      choice.message.tool_calls.map(async (toolCall) => {
        const args = JSON.parse(toolCall.function.arguments);
        let result: unknown;
        
        switch (toolCall.function.name) {
          case 'get_product_info':
            result = await getProductInfo(args.productId);
            break;
          case 'check_order_status':
            result = await checkOrderStatus(args.orderId, context.userId);
            break;
        }
        
        return {
          tool_call_id: toolCall.id,
          role: 'tool' as const,
          content: JSON.stringify(result),
        };
      })
    );
    
    // Continue conversation with tool results
    const finalResponse = await openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        { role: 'user', content: message },
        choice.message,
        ...toolResults,
      ],
    });
    
    return finalResponse.choices[0]?.message.content ?? '';
  }
  
  return choice.message.content ?? '';
}
```

---

## 📌 Step 1313: Vector Search กับ pgvector

```sql
-- PostgreSQL + pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Add embedding column
ALTER TABLE products ADD COLUMN embedding vector(1536);

-- Create index สำหรับ fast search
CREATE INDEX ON products USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```

```typescript
// src/ai/vector-search.ts
import { embed } from './openai.js';
import { prisma } from '../db.js';

// Index product embedding
export async function indexProduct(productId: string, content: string) {
  const embedding = await embed(content);
  
  await prisma.$executeRaw`
    UPDATE products
    SET embedding = ${JSON.stringify(embedding)}::vector
    WHERE id = ${productId}
  `;
}

// Semantic search
export async function semanticSearch(
  query: string,
  limit = 10
): Promise<Array<{ id: string; name: string; score: number }>> {
  const queryEmbedding = await embed(query);
  
  return prisma.$queryRaw<Array<{ id: string; name: string; score: number }>>`
    SELECT
      id,
      name,
      1 - (embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS score
    FROM products
    WHERE embedding IS NOT NULL
    ORDER BY embedding <=> ${JSON.stringify(queryEmbedding)}::vector
    LIMIT ${limit}
  `;
}

// RAG: Retrieval-Augmented Generation
export async function ragSearch(userQuery: string): Promise<string> {
  // 1. Retrieve relevant products
  const relevantProducts = await semanticSearch(userQuery, 5);
  
  // 2. Build context
  const productDetails = await prisma.product.findMany({
    where: { id: { in: relevantProducts.map(p => p.id) } },
    select: { id: true, name: true, description: true, price: true },
  });
  
  const context = productDetails.map(p =>
    `Product: ${p.name}\nPrice: ${p.price} THB\nDescription: ${p.description}`
  ).join('\n\n');
  
  // 3. Generate answer using retrieved context
  const response = await chat([
    {
      role: 'system',
      content: `คุณเป็น assistant ที่ช่วยแนะนำสินค้า ตอบคำถามโดยอ้างอิงจากข้อมูลสินค้าที่ให้มาเท่านั้น
      
ข้อมูลสินค้า:
${context}`,
    },
    {
      role: 'user',
      content: userQuery,
    },
  ]);
  
  return response.content;
}
```

---

## 📌 Step 1314: Streaming Subscription

```typescript
// src/resolvers/ai.resolver.ts

const aiResolvers = {
  Mutation: {
    sendMessage: async (_, { input }, ctx) => {
      requireAuth(ctx);
      
      const { sessionId, message, context: msgContext } = input;
      
      // Get conversation history
      const history = await getConversationHistory(sessionId, ctx);
      
      // Build system prompt with context
      let systemPrompt = `คุณเป็น AI assistant สำหรับ E-commerce platform
ช่วยตอบคำถามเกี่ยวกับสินค้า, orders, และ account`;
      
      if (msgContext?.productId) {
        const product = await ctx.prisma.product.findUnique({
          where: { id: msgContext.productId },
          select: { name: true, description: true, price: true },
        });
        if (product) {
          systemPrompt += `\nสินค้าที่กำลังดูอยู่: ${product.name} ราคา ${product.price} บาท`;
        }
      }
      
      const messages = [
        { role: 'system', content: systemPrompt },
        ...history,
        { role: 'user', content: message },
      ];
      
      const response = await chat(messages as OpenAI.Chat.ChatCompletionMessageParam[]);
      
      // Save to history
      await saveToHistory(sessionId, message, response.content, ctx);
      
      return {
        id: crypto.randomUUID(),
        sessionId,
        role: 'ASSISTANT',
        content: response.content,
        usage: response.usage,
        finishReason: response.finishReason,
      };
    },
  },
  
  Subscription: {
    chatStream: {
      subscribe: async function* (_, { sessionId, message }, ctx) {
        requireAuth(ctx);
        
        const history = await getConversationHistory(sessionId, ctx);
        
        const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
          { role: 'system', content: 'คุณเป็น AI assistant' },
          ...history,
          { role: 'user', content: message },
        ];
        
        let fullContent = '';
        
        for await (const chunk of streamChat(messages)) {
          fullContent += chunk.delta;
          
          yield {
            chatStream: {
              sessionId,
              delta: chunk.delta,
              isComplete: chunk.isComplete,
            },
          };
        }
        
        // Save complete response
        await saveToHistory(sessionId, message, fullContent, ctx);
      },
    },
  },
  
  Query: {
    semanticSearch: async (_, { query, limit }, ctx) => {
      const results = await semanticSearch(query, limit);
      return results.map(r => ({ ...r, __typename: 'ProductResult' }));
    },
    
    recommendations: async (_, { userId, limit }, ctx) => {
      // Collaborative filtering หรือ content-based
      const userHistory = await ctx.prisma.orderItem.findMany({
        where: { order: { customerId: userId } },
        select: { product: { select: { id: true, embedding: true } } },
        take: 10,
        orderBy: { createdAt: 'desc' },
      });
      
      if (!userHistory.length) {
        // Cold start: return popular products
        return ctx.prisma.product.findMany({
          orderBy: { salesCount: 'desc' },
          take: limit,
        }).then(products => products.map(p => ({
          product: p,
          score: 0.5,
          reason: 'Popular product',
        })));
      }
      
      // Compute average embedding of purchased products
      // ... vector arithmetic
      
      return [];
    },
  },
};
```

---

## 📌 Step 1315: สรุป Part 060

### เนื้อหาที่เรียนรู้

✅ GraphQL Schema สำหรับ AI features  
✅ OpenAI API integration  
✅ Streaming responses ผ่าน Subscriptions  
✅ Tool calling / Function calling  
✅ Vector embeddings กับ pgvector  
✅ Semantic search  
✅ RAG (Retrieval-Augmented Generation)  
✅ Recommendation systems  

### AI Integration Best Practices

```
1. Rate limiting AI endpoints (ค่า token แพงมาก)
2. Cache embeddings ใน DB (recompute ไม่บ่อย)
3. Stream responses สำหรับ LLM (latency สูง)
4. Validate user input ก่อนส่ง LLM
5. ไม่ส่ง sensitive data ไปยัง third-party LLMs
6. Monitor token usage (cost tracking)
7. Fallback เมื่อ AI ล้มเหลว (graceful degradation)
8. ตรวจสอบ AI responses ก่อน return (content safety)
```

### ในส่วนถัดไป

➡️ **[Part 061](./part-061.md)** — Schema Design Best Practices

---

*Part 060 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
