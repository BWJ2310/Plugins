# Elastic Plugin Overview

The elastic plugin integrates Elasticsearch functionality into Hydro OJ, primarily for enhanced problem searching capabilities. Here's a detailed breakdown:

## 1. Core Functionality

The plugin creates an Elasticsearch client and manages problem document indexing:

```typescript
// From /elastic/index.ts
const client = new Client({ 
  node: SystemModel.get('elastic-search.url') || 'http://127.0.0.1:9200' 
});
```

### Key Imported Packages Usage:

1. **SystemModel**:
- Used to retrieve Elasticsearch configuration settings
- Example:
```typescript
// Getting Elasticsearch URL and index size
const url = SystemModel.get('elastic-search.url')
const allowedSize = SystemModel.get('elasic-search.indexSize') || 10000;
```

2. **ProblemModel & ProblemDoc**:
- ProblemModel: Handles problem data operations
- ProblemDoc: Defines problem document structure
- Used in indexing and searching problems:
```typescript
// Example of using ProblemModel for getting problem data
let pdoc = await ProblemModel.get(domainId, +q || q, ProblemModel.PROJECTION_LIST);
```

3. **iterateAllProblem & iterateAllProblemInDomain**:
- Used to process all problems for indexing
```typescript
// Iterating through problems in a domain
if (domainId) {
  await iterateAllProblemInDomain(
    domainId, 
    ProblemModel.PROJECTION_PUBLIC, 
    cb
  );
} else {
  await iterateAllProblem(ProblemModel.PROJECTION_PUBLIC, cb);
}
```

4. **Context**:
- Manages plugin lifecycle and event handling
```typescript
export const apply = (ctx: Context) => {
  // Event handlers registration
  ctx.on('problem/add', async (doc, docId) => {
    await client.index({/*...*/});
  });
  // ... other event handlers
};
```

## 2. Plugin Functions & Handlers

### Document Processing
```typescript
const processDocument = (doc: Partial<ProblemDoc>) => {
  // Clean and format content
  doc.content &&= doc.content.replace(/[[\]ãã()ï¼ï¼]/g, ' ');
  doc.title &&= doc.title.replace(/[[\]ãã()ï¼ï¼]/g, ' ')
    .replace(/([a-zA-Z]{2,})(\d+)/, '$1$2 $1 $2');
  
  // Add tags based on problem ID
  if (doc.pid?.includes('-')) {
    const ns = doc.pid.split('-')[0];
    doc.tag.push(ns);
  }
  
  return _.omit(doc, indexOmit);
};
```

### Search Implementation
```typescript
global.Hydro.lib.problemSearch = async (domainId, q, opts) => {
  const size = opts?.limit || SystemModel.get('pagination.problem');
  const from = Math.min(allowedSize - size, opts?.skip || 0);
  
  // Elasticsearch query
  const res = await client.search({
    index: 'problem',
    size,
    from,
    query: {
      simple_query_string: {
        query: q.replace(/([a-z0-9]{2,})/gi, (i) => `${i}~5`),
        fields: ['tag^5', 'pid^4', 'title^3', 'content'],
      }
    },
    // ... additional query parameters
  });
  
  // Process results
  return {
    countRelation: typeof res.hits.total === 'number' ? 'eq' : res.hits.total.relation,
    total: typeof res.hits.total === 'number' ? res.hits.total : res.hits.total.value,
    hits: Array.from(new Set(hits)),
  };
};
```

### Event Handlers
```typescript
// Problem addition handler
ctx.on('problem/add', async (doc, docId) => {
  await client.index({
    index: 'problem',
    id: `${doc.domainId}/${docId}`,
    document: processDocument(doc),
  });
});

// Problem edit handler
ctx.on('problem/edit', async (pdoc) => {
  await client.index({
    index: 'problem',
    id: `${pdoc.domainId}/${pdoc.docId}`,
    document: processDocument(pdoc),
  });
});

// Problem deletion handler
ctx.on('problem/del', async (domainId, docId) => {
  await client.delete({
    index: 'problem',
    id: `${domainId}/${docId}`,
  });
});
```

## 3. UI Modifications

While the elastic plugin doesn't directly modify the UI through traditional means (like adding new components), it enhances the search functionality which affects how problems are displayed and searched:

1. **Problem List Page**:
- Enhanced search capabilities with better relevance ranking
- Support for fuzzy matching
- Search across multiple fields (title, content, tags)

2. **Search Results**:
- More accurate problem matching
- Better handling of partial matches
- Support for advanced query syntax

The plugin achieves this by replacing the default search implementation with Elasticsearch:

```typescript
// Enhanced search functionality
const searchResults = await client.search({
  index: 'problem',
  query: {
    simple_query_string: {
      query: q.replace(/([a-z0-9]{2,})/gi, (i) => `${i}~5`),
      fields: ['tag^5', 'pid^4', 'title^3', 'content'],
    }
  }
});
```

## 4. Configuration

The plugin uses a configuration file (`setting.yaml`) for basic settings:

```yaml
url:
  value: http://127.0.0.1:9200
indexSize:
  value: 10000
```

This configuration can be accessed through the SystemModel:
```typescript
const url = SystemModel.get('elastic-search.url');
const indexSize = SystemModel.get('elasic-search.indexSize');
```

The elastic plugin significantly enhances Hydro OJ's search capabilities by providing:
- Fuzzy matching
- Field-specific boosting
- Better relevance ranking
- Efficient indexing and searching of problem content

It integrates seamlessly with Hydro OJ's existing infrastructure while providing more powerful search capabilities through Elasticsearch.
