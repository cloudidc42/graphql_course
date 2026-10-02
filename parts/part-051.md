# Part 051 — Full-Text Search Integration 🔍

> **ระดับ:** Advanced | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 951–990

---

## 🎯 สิ่งที่จะได้เรียนรู้

- PostgreSQL full-text search ใน GraphQL
- Elasticsearch integration
- Algolia integration
- Search result highlighting
- Faceted search (filters + counts)
- Auto-suggest / typeahead
- Multi-model search
- Search ranking strategies

---

## 📌 Step 951: PostgreSQL Full-Text Search

```typescript
// PostgreSQL มี built-in full-text search ด้วย tsvector/tsquery

// schema.prisma
// model Product {
//   id          String  @id @default(uuid())
//   name        String
//   description String?
//   searchVector Unsupported("tsvector")?
// }

// SQL migration สำหรับ search vector
// CREATE INDEX products_search_idx ON products USING gin(search_vector);
// CREATE TRIGGER products_search_update
//   BEFORE INSERT OR UPDATE ON products
//   FOR EACH ROW EXECUTE FUNCTION
//     tsvector_update_trigger(search_vector, 'pg_catalog.english', name, description);

// Prisma raw query สำหรับ full-text search
async function searchProducts(term: string, prisma: PrismaClient) {
  // tsquery: แปลง search term
  const tsQuery = term
    .trim()
    .split(/\s+/)
    .map(word => `${word}:*`) // prefix search
    .join(' & ');
  
  return prisma.$queryRaw<Array<{
    id: string;
    name: string;
    price: number;
    rank: number;
    headline: string;
  }>>`
    SELECT
      id,
      name,
      price,
      ts_rank(search_vector, to_tsquery('english', ${tsQuery})) AS rank,
      ts_headline(
        'english',
        name || ' ' || COALESCE(description, ''),
        to_tsquery('english', ${tsQuery}),
        'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20'
      ) AS headline
    FROM products
    WHERE search_vector @@ to_tsquery('english', ${tsQuery})
    ORDER BY rank DESC
    LIMIT 20
  `;
}
```

---

## 📌 Step 952: GraphQL Search Schema

```graphql
# src/schema/search.graphql

type Query {
  search(
    term: String!
    types: [SearchType!]
    filter: SearchFilter
    first: Int = 20
    after: String
  ): SearchConnection!
  
  suggest(
    term: String!
    types: [SearchType!]
    limit: Int = 5
  ): [Suggestion!]!
}

enum SearchType {
  PRODUCT
  USER
  POST
  TAG
}

input SearchFilter {
  minPrice: Float
  maxPrice: Float
  categoryIds: [ID!]
  dateFrom: DateTime
  dateTo: DateTime
}

union SearchResult = ProductResult | UserResult | PostResult | TagResult

type SearchConnection {
  edges: [SearchEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
  facets: SearchFacets!
}

type SearchEdge {
  node: SearchResult!
  cursor: String!
  score: Float!
  highlight: String
}

type SearchFacets {
  categories: [FacetCount!]!
  priceRanges: [PriceRangeFacet!]!
  types: [TypeFacet!]!
}

type FacetCount {
  value: String!
  label: String!
  count: Int!
}

type PriceRangeFacet {
  min: Float!
  max: Float!
  count: Int!
}

type TypeFacet {
  type: SearchType!
  count: Int!
}

type Suggestion {
  text: String!
  type: SearchType!
  score: Float!
}
```

---

## 📌 Step 953: Elasticsearch Integration

```bash
npm install @elastic/elasticsearch
```

```typescript
// src/search/elasticsearch.ts
import { Client } from '@elastic/elasticsearch';

const es = new Client({
  node: process.env.ELASTICSEARCH_URL ?? 'http://localhost:9200',
  auth: {
    username: process.env.ELASTICSEARCH_USER ?? 'elastic',
    password: process.env.ELASTICSEARCH_PASSWORD ?? '',
  },
});

// Index mapping
const PRODUCT_INDEX = 'products';

export async function createProductIndex() {
  await es.indices.create({
    index: PRODUCT_INDEX,
    mappings: {
      properties: {
        id: { type: 'keyword' },
        name: {
          type: 'text',
          fields: {
            keyword: { type: 'keyword' }, // สำหรับ exact match
            suggest: { type: 'search_as_you_type' }, // typeahead
          },
        },
        description: { type: 'text' },
        price: { type: 'double' },
        categoryId: { type: 'keyword' },
        categoryName: { type: 'keyword' },
        tags: { type: 'keyword' },
        stock: { type: 'integer' },
        createdAt: { type: 'date' },
        updatedAt: { type: 'date' },
      },
    },
    settings: {
      analysis: {
        analyzer: {
          thai_analyzer: {
            type: 'custom',
            tokenizer: 'standard',
            filter: ['lowercase', 'asciifolding'],
          },
        },
      },
    },
  });
}

// Index product
export async function indexProduct(product: {
  id: string;
  name: string;
  description?: string | null;
  price: number;
  categoryId: string;
  categoryName: string;
  tags: string[];
  stock: number;
  createdAt: Date;
  updatedAt: Date;
}) {
  await es.index({
    index: PRODUCT_INDEX,
    id: product.id,
    document: product,
  });
}

// Search products
export async function searchProducts(
  term: string,
  filter?: { minPrice?: number; maxPrice?: number; categoryIds?: string[] },
  pagination?: { size: number; from: number }
) {
  const must: object[] = [
    {
      multi_match: {
        query: term,
        fields: ['name^3', 'description^1', 'tags^2'],
        type: 'best_fields',
        fuzziness: 'AUTO', // ยืดหยุ่นสำหรับ typos
      },
    },
  ];
  
  const filter_clauses: object[] = [];
  
  if (filter?.minPrice || filter?.maxPrice) {
    filter_clauses.push({
      range: {
        price: {
          gte: filter.minPrice,
          lte: filter.maxPrice,
        },
      },
    });
  }
  
  if (filter?.categoryIds?.length) {
    filter_clauses.push({
      terms: { categoryId: filter.categoryIds },
    });
  }
  
  const response = await es.search({
    index: PRODUCT_INDEX,
    query: {
      bool: {
        must,
        filter: filter_clauses,
      },
    },
    highlight: {
      fields: {
        name: { pre_tags: ['<mark>'], post_tags: ['</mark>'] },
        description: { pre_tags: ['<mark>'], post_tags: ['</mark>'], number_of_fragments: 2 },
      },
    },
    aggs: {
      categories: {
        terms: { field: 'categoryName', size: 20 },
      },
      price_ranges: {
        range: {
          field: 'price',
          ranges: [
            { to: 100 },
            { from: 100, to: 500 },
            { from: 500, to: 1000 },
            { from: 1000 },
          ],
        },
      },
    },
    size: pagination?.size ?? 20,
    from: pagination?.from ?? 0,
  });
  
  return {
    hits: response.hits.hits.map(hit => ({
      id: hit._id,
      source: hit._source,
      score: hit._score ?? 0,
      highlight: hit.highlight,
    })),
    total: (response.hits.total as { value: number }).value,
    aggregations: response.aggregations,
  };
}

// Typeahead suggestion
export async function suggestProducts(term: string, limit = 5) {
  const response = await es.search({
    index: PRODUCT_INDEX,
    query: {
      multi_match: {
        query: term,
        type: 'bool_prefix',
        fields: ['name.suggest', 'name.suggest._2gram', 'name.suggest._3gram'],
      },
    },
    _source: ['id', 'name'],
    size: limit,
  });
  
  return response.hits.hits.map(hit => ({
    text: (hit._source as { name: string }).name,
    score: hit._score ?? 0,
  }));
}
```

---

## 📌 Step 954: Algolia Integration

```bash
npm install @algolia/client-search algoliasearch
```

```typescript
// src/search/algolia.ts
import algoliasearch from 'algoliasearch';

const client = algoliasearch(
  process.env.ALGOLIA_APP_ID!,
  process.env.ALGOLIA_ADMIN_KEY!
);

const productsIndex = client.initIndex('products');

// Configure index settings
export async function configureAlgolia() {
  await productsIndex.setSettings({
    searchableAttributes: [
      'name',           // highest priority
      'description',
      'categoryName',
      'tags',
    ],
    attributesForFaceting: [
      'categoryId',
      'categoryName',
      'filterOnly(price)',
      'filterOnly(stock)',
    ],
    ranking: [
      'typo',
      'geo',
      'words',
      'filters',
      'proximity',
      'attribute',
      'exact',
      'custom',
    ],
    customRanking: ['desc(salesCount)', 'desc(rating)'],
    highlightPreTag: '<mark>',
    highlightPostTag: '</mark>',
  });
}

// Index/update product
export async function saveProductToAlgolia(product: {
  id: string;
  name: string;
  description?: string | null;
  price: number;
  categoryId: string;
  categoryName: string;
  tags: string[];
  stock: number;
  rating?: number;
  salesCount?: number;
}) {
  await productsIndex.saveObject({
    objectID: product.id,
    ...product,
  });
}

// Search
export async function searchAlgolia(
  term: string,
  options?: {
    filters?: string; // Algolia filter syntax
    facetFilters?: string[][];
    numericFilters?: string[];
    page?: number;
    hitsPerPage?: number;
  }
) {
  return productsIndex.search(term, {
    filters: options?.filters,
    facetFilters: options?.facetFilters,
    numericFilters: options?.numericFilters,
    page: options?.page ?? 0,
    hitsPerPage: options?.hitsPerPage ?? 20,
    attributesToHighlight: ['name', 'description'],
    attributesToSnippet: ['description:50'],
  });
}

// Usage:
// await searchAlgolia('laptop', {
//   numericFilters: ['price >= 500', 'price <= 2000'],
//   facetFilters: [['categoryId:electronics']],
// });
```

---

## 📌 Step 955: Search Resolver

```typescript
// src/resolvers/search.resolver.ts
import { searchProducts as searchPg } from '../search/postgres.js';
import { searchProducts as searchEs, suggestProducts } from '../search/elasticsearch.js';

const searchResolvers = {
  Query: {
    search: async (_, args, ctx) => {
      const { term, types, filter, first = 20, after } = args;
      
      // Decode cursor
      let from = 0;
      if (after) {
        const cursor = JSON.parse(Buffer.from(after, 'base64url').toString());
        from = cursor.from;
      }
      
      // ค้นหาจาก Elasticsearch
      const results = await searchEs(term, filter, { size: first + 1, from });
      
      const hasNextPage = results.hits.length > first;
      const hits = hasNextPage ? results.hits.slice(0, first) : results.hits;
      
      const edges = hits.map((hit, i) => ({
        node: { ...hit.source, __typename: 'ProductResult' },
        cursor: Buffer.from(JSON.stringify({ from: from + i + 1 })).toString('base64url'),
        score: hit.score,
        highlight: hit.highlight?.name?.[0] ?? hit.highlight?.description?.[0] ?? null,
      }));
      
      // Build facets จาก aggregations
      const aggs = results.aggregations as Record<string, { buckets: Array<{ key: string; doc_count: number }> }>;
      const facets = {
        categories: (aggs?.categories?.buckets ?? []).map(b => ({
          value: b.key,
          label: b.key,
          count: b.doc_count,
        })),
        priceRanges: [],
        types: [{ type: 'PRODUCT', count: results.total }],
      };
      
      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: from > 0,
          startCursor: edges[0]?.cursor ?? null,
          endCursor: edges[edges.length - 1]?.cursor ?? null,
        },
        totalCount: results.total,
        facets,
      };
    },
    
    suggest: async (_, { term, limit = 5 }) => {
      const suggestions = await suggestProducts(term, limit);
      return suggestions.map(s => ({
        text: s.text,
        type: 'PRODUCT',
        score: s.score,
      }));
    },
  },
  
  SearchResult: {
    __resolveType: (obj: { __typename?: string }) => obj.__typename ?? 'ProductResult',
  },
};

// Keep Elasticsearch in sync เมื่อ product เปลี่ยน
// (ใน product mutation resolvers)
// await indexProduct(updatedProduct);
```

---

## 📌 Step 956: สรุป Part 051

### เนื้อหาที่เรียนรู้

✅ PostgreSQL full-text search (tsvector/tsquery)  
✅ Elasticsearch multi-field search + facets  
✅ Algolia integration  
✅ Search result highlighting  
✅ Typeahead/suggestion  
✅ GraphQL search schema  

### เลือก Search Solution

```
PostgreSQL Full-Text:
✅ ไม่ต้องการ infra เพิ่ม
✅ Transaction consistency
✅ เหมาะสำหรับ < 1M documents
❌ ไม่รองรับ typo tolerance
❌ facets ต้องเขียนเอง

Elasticsearch:
✅ Powerful, flexible
✅ Real-time analytics
✅ เหมาะสำหรับ > 1M documents
❌ ต้องจัดการ infra
❌ ซับซ้อนกว่า

Algolia:
✅ ง่ายมาก, เร็วมาก
✅ Typo tolerance ในตัว
✅ Managed service
❌ ราคาสูงที่ scale
❌ Data out of your control
```

### ในส่วนถัดไป

➡️ **[Part 052](./part-052.md)** — Batch Operations & Bulk Mutations

---

*Part 051 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
