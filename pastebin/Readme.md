# 1. Overview of Pastebin Plugin

The pastebin plugin adds functionality for users to create, edit, view and manage text snippets (pastes) in Hydro OJ. It's similar to services like pastebin.com but integrated into Hydro OJ.

# 2. Imported Packages from Hydro OJ

Let's examine how the plugin utilizes each imported package:

## 2.1 Database (db)
```typescript
const coll = db.collection('paste');
```
The plugin uses Hydro's database service to create and manage a 'paste' collection. From db.ts:
- Provides MongoDB operations
- Handles collection management
- Manages database connections

## 2.2 Handler
The plugin extends Hydro's base Handler class for request handling. From server.ts:
```typescript
class PasteCreateHandler extends Handler {
    async get() {
        this.response.template = 'paste_create.html';
    }
    
    @param('title', Types.Title)
    @param('content', Types.Content)
    @param('isprivate', Types.Boolean)
    async post(domainId: string, title: string, content: string, isprivate = false) {
        const pasteid = await pastebinModel.add(this.user._id, title, content, !!isprivate);
        this.response.redirect = this.url('paste_show', { id: pasteid });
    }
}
```

## 2.3 Error Handling
The plugin uses Hydro's error classes:
```typescript
if (!doc) throw new NotFoundError(id);
if (this.user._id !== doc.owner) {
    this.checkPriv(PRIV.PRIV_CREATE_DOMAIN);
}
```

## 2.4 Decorators (@param, @query)
Used for parameter validation:
```typescript
@param('title', Types.Title)
@param('content', Types.Content)
@param('isprivate', Types.Boolean)
async post(domainId: string, title: string, content: string, isprivate = false)
```

## 2.5 Privileges (PRIV)
Controls access permissions:
```typescript
ctx.Route('paste_create', '/paste/create', PasteCreateHandler, PRIV.PRIV_USER_PROFILE);
```

# 3. Plugin Models and Functions

## 3.1 Paste Interface
```typescript
interface Paste {
    _id: string;
    updateAt: Date,
    title: string;
    owner: number;
    content: string;
    isprivate: boolean;
}
```

## 3.2 PastebinModel Functions

### add()
```typescript
async function add(userId: number, title: string, content: string, isprivate: boolean): Promise<string> {
    const pasteId = String.random(16);
    const result = await coll.insertOne({
        _id: pasteId,
        updateAt: new Date(),
        title: title,
        owner: userId,
        content,
        isprivate,
    });
    return result.insertedId;
}
```

### edit(), get(), del() etc.
Similar CRUD operations for paste management.

# 4. Handlers

## 4.1 PasteCreateHandler
Handles paste creation:
```typescript
class PasteCreateHandler extends Handler {
    async get() {
        this.response.template = 'paste_create.html';
    }
    // ... post method
}
```

## 4.2 PasteShowHandler
Displays paste content:
```typescript
class PasteShowHandler extends Handler {
    @param('id', Types.String)
    async get(domainId: string, id: string) {
        const doc = await pastebinModel.get(id);
        // ... permission checks
        this.response.template = 'paste_show.html';
    }
}
```

# 5. UI Modifications

The plugin adds several new pages through HTML templates:

## 5.1 Create Page (paste_create.html)
```html
{% set page_name = "新建剪贴板" %}
{% extends "layout/basic.html" %}
{% block content %}
<div class="row">
  <div class="medium-9 columns">
    <div class="section">
      <div class="section__body">
        <form method="post">
          <!-- Title input -->
          <!-- Content textarea -->
          <!-- Private checkbox -->
        </form>
      </div>
    </div>
  </div>
</div>
```

## 5.2 Show Page (paste_show.html)
```html
{% set page_name = "查看剪贴板" %}
{% extends "layout/basic.html" %}
{% block content %}
<!-- Paste display -->
<!-- Markdown rendering -->
<!-- Edit/Delete options -->
```

## 5.3 Manage Page (paste_manage.html)
Lists all pastes with pagination.

# 6. Routes

The plugin registers these routes:
```typescript
export async function apply(ctx: Context) {
    ctx.Route('paste_create', '/paste/create', PasteCreateHandler, PRIV.PRIV_USER_PROFILE);
    ctx.Route('paste_manage', '/paste/manage', PasteManageHandler, PRIV.PRIV_USER_PROFILE);
    ctx.Route('paste_all', '/paste/all', PasteAllHandler,  PRIV.PRIV_CREATE_DOMAIN);
    ctx.Route('paste_show', '/paste/show/:id', PasteShowHandler);
    ctx.Route('paste_edit', '/paste/show/:id/edit', PasteEditHandler, PRIV.PRIV_USER_PROFILE);
    ctx.Route('paste_delete', '/paste/show/:id/delete', PasteDeleteHandler, PRIV.PRIV_USER_PROFILE);
}
```

The plugin integrates seamlessly with Hydro's existing UI by:
1. Extending the base layout template
2. Using Hydro's UI components and styles
3. Following Hydro's permission system
4. Implementing consistent navigation patterns

Each page provides specific functionality:
- Create: Text editor with markdown support
- Show: Rendered markdown with syntax highlighting
- Manage: List view with pagination
- Edit: Similar to create but pre-filled
- Delete: Confirmation page

The plugin demonstrates effective use of Hydro's framework components while adding valuable functionality for users to share and manage text content.
