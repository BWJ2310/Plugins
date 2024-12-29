# 1. Overview of Sonic Plugin

The Sonic plugin is a search service integration for Hydro OJ that provides fast and efficient problem searching capabilities. It uses the sonic-channel package for search operations.

# 2. Imported Packages from Hydro OJ

Let's examine how the plugin utilizes each imported package:

## Context
```typescript
// From /packages/hydrooj/src/context.ts
export interface Context extends cordis.Context {
    loader: Loader;
    check: CheckService;
    setImmediate: typeof setImmediate;
    addScript: typeof addScript;
    provideModule: typeof provideModule;
    injectUI: typeof inject;
    broadcast: Context['emit'];
}
```
The Context provides core functionality for plugin operations and service management. The Sonic plugin uses it to:
- Register event handlers
- Add scripts
- Access the sonic service

## ProblemModel
```typescript
// From /packages/hydrooj/src/model/problem.ts
export class ProblemModel {
    static PROJECTION_LIST = {
        docId: 1,
        domainId: 1,
        title: 1,
        content: 1,
        tag: 1,
        // ...
    };
    
    static async get(domainId: string, pid: number | string, projection = {}) {
        // Implementation
    }
}
```
Used for:
- Getting problem documents
- Accessing problem metadata
- Managing problem content

## Logger
```typescript
const logger = new Logger('sonic');
```
Provides logging functionality specific to the Sonic plugin.

# 3. Plugin Components and Functions

## SonicService
```typescript
// From /sonic/service.ts
export class SonicService extends Service {
    async query(collection: string, bucket: string, terms: string, options?: QueryOptions) {
        // Performs search queries
    }
    
    async push(collection: string, bucket: string, object: string, text: string) {
        // Indexes content for searching
    }
}
```

## Problem Search Function
```typescript
// From /sonic/index.ts
global.Hydro.lib.problemSearch = async (domainId, query, opts) => {
    const limit = opts?.limit || SystemModel.get('pagination.problem');
    let hits = await ctx.sonic.query('problem', `${domainId}@title`, query, { limit });
    // ... search implementation
};
```

## Event Handlers

### Problem Add Handler
```typescript
ctx.on('problem/add', async (doc, docId) => {
    Promise.all([
        ctx.sonic.push('problem', `${doc.domainId}@title`, `${doc.domainId}/${docId}`, 
            `${doc.pid || ''} ${doc.title} ${doc.tag?.join(' ')}`),
        ctx.sonic.push('problem', `${doc.domainId}@content`, `${doc.domainId}/${docId}`, 
            doc.content.toString()),
    ]).catch((e) => logger.error(e));
});
```

### Problem Edit Handler
```typescript
ctx.on('problem/edit', async (pdoc) => {
    const id = `${pdoc.domainId}/${pdoc.docId}`;
    Promise.all([
        ctx.sonic.flusho('problem', `${pdoc.domainId}@title`, id)
            .then(() => ctx.sonic.push('problem', `${pdoc.domainId}@title`, id, 
                `${pdoc.pid || ''} ${pdoc.title} ${pdoc.tag?.join(' ')}`)),
        // ... content update
    ]).catch((e) => logger.error(e));
});
```

# 4. UI Integration

The Sonic plugin primarily modifies the problem search functionality rather than directly modifying the UI. However, it enhances the search experience by:

1. Improving search results on problem listing pages
2. Enabling content-based search
3. Supporting tag-based filtering

The search functionality is integrated into existing UI components through Hydro's problem search API:

```typescript
// Example of how the search is used in UI components
const searchResults = await global.Hydro.lib.problemSearch(domainId, searchQuery, {
    limit: 20,
    skip: false
});
```

# 5. Search Index Management

The plugin includes a script for rebuilding search indices:

```typescript
async function run({ domainId }, report) {
    if (domainId) await ctx.sonic.flushb('problem', domainId);
    else await ctx.sonic.flushc('problem');
    
    let i = 0;
    const cb = async (pdoc) => {
        i++;
        if (!(i % 1000)) report({ message: `${i} problems indexed` });
        await Promise.all([
            pdoc.title && ctx.sonic.push(
                'problem', `${pdoc.domainid}@title`, 
                `${pdoc.domainId}/${pdoc.docId}`,
                `${pdoc.pid || ''} ${pdoc.title} ${pdoc.tag.join(' ')}`,
            ),
            // ... content indexing
        ]);
    };
    
    if (domainId) {
        await iterateAllProblemInDomain(domainId, ['title', 'content'], cb);
    } else {
        await iterateAllProblem(['title', 'content', 'tag'], cb);
    }
    return true;
}
```

This script can be triggered through Hydro's admin interface to rebuild search indices when needed.

# 6. Configuration

The plugin uses SystemModel for configuration:

```typescript
const config = {
    host: SystemModel.get('sonic.host') || '127.0.0.1',
    port: SystemModel.get('sonic.port') || 1491,
    auth: SystemModel.get('sonic.auth') || 'SecretPassword'
};
```

In summary, the Sonic plugin provides a powerful search enhancement to Hydro OJ by:
- Indexing problem titles, content, and tags
- Providing fast search capabilities
- Maintaining search indices through event handlers
- Integrating seamlessly with existing UI components

The plugin doesn't directly modify the UI but enhances the search functionality that's used throughout the platform's interface.
