# **Hydro OJ Blog Plugin Overview**

The **Blog Plugin** introduces blogging functionality to HydroOJ, allowing users to:
- **Create, edit, view, and manage blog posts.**

---

## **Imported Package Usage**

The blog plugin utilizes several core Hydro OJ packages:

```typescript
import {
    _, Context, DiscussionNotFoundError, DocumentModel, Filter,
    Handler, NumberKeys, ObjectId, OplogModel, paginate,
    param, PRIV, Types, UserModel,
} from 'hydrooj';
```

### **Package Breakdown**

#### **1. DocumentModel**
Used to store blog posts as documents.

```typescript
// From blog/index.ts
static async add(owner: number, title: string, content: string, ip?: string) {
    const payload: Partial<BlogDoc> = {
        content,
        owner,
        title,
        ip,
        nReply: 0,
        updateAt: new Date(),
        views: 0,
    };
    // Uses DocumentModel to store blog post
    const res = await DocumentModel.add(
        'system', payload.content!, payload.owner!, TYPE_BLOG,
        null, null, null, _.omit(payload, ['domainId', 'content', 'owner']),
    );
    return res;
}
```

---

#### **2. UserModel**
Handles user authentication and permissions.

```typescript
// From blog/index.ts
class BlogDetailHandler extends BlogHandler {
    async get(domainId: string, did: ObjectId) {
        // Get user document for blog owner
        const udoc = await UserModel.getById(domainId, this.ddoc!.owner);
        // Check viewing permissions
        if (!this.user.hasPriv(PRIV.PRIV_USER_PROFILE)) {
            throw new PermissionError();
        }
    }
}
```

---

#### **3. OplogModel**
Records operations for audit trails.

```typescript
// From blog/index.ts
async postUpdate(domainId: string, did: ObjectId, title: string, content: string) {
    await Promise.all([
        BlogModel.edit(did, title, content),
        // Log blog edit operation
        OplogModel.log(this, 'blog.edit', this.ddoc),
    ]);
}
```

---

## **Models and Functions**

### **BlogModel**
Handles the main blog operations.

```typescript
export class BlogModel {
    // Add new blog post
    static async add(owner: number, title: string, content: string, ip?: string)
    
    // Get blog post by ID
    static async get(did: ObjectId): Promise<BlogDoc>
    
    // Edit blog post
    static edit(did: ObjectId, title: string, content: string)
    
    // Delete blog post
    static del(did: ObjectId)
    
    // Add reply to blog post
    static async addReply(did: ObjectId, owner: number, content: string, ip: string)
}
```

---

## **Handlers**

### **1. BlogHandler (Base Handler)**

Handles shared functionality for blog operations.

```typescript
class BlogHandler extends Handler {
    @param('did', Types.ObjectId, true)
    async _prepare(domainId: string, did: ObjectId) {
        if (did) {
            this.ddoc = await BlogModel.get(did);
            if (!this.ddoc) throw new DiscussionNotFoundError(domainId, did);
        }
    }
}
```

---

### **2. BlogDetailHandler**

Handles the display and view tracking of individual blog posts.

```typescript
class BlogDetailHandler extends BlogHandler {
    @param('did', Types.ObjectId)
    async get(domainId: string, did: ObjectId) {
        // Handle blog post viewing
        const dsdoc = this.user.hasPriv(PRIV.PRIV_USER_PROFILE)
            ? await BlogModel.getStatus(did, this.user._id)
            : null;
        const udoc = await UserModel.getById(domainId, this.ddoc!.owner);
        
        // Update view count
        if (!dsdoc?.view) {
            await Promise.all([
                BlogModel.inc(did, 'views', 1),
                BlogModel.setStatus(did, this.user._id, { view: true }),
            ]);
        }
        
        this.response.template = 'blog_detail.html';
        this.response.body = { ddoc: this.ddoc, dsdoc, udoc };
    }
}
```

---

## **UI Modifications**

### **1. User Dropdown Menu**
Adds a "Blog" link to the user dropdown menu.

```typescript
// From blog/index.ts
export async function apply(ctx: Context) {
    // Add blog link to user dropdown
    ctx.injectUI('UserDropdown', 'blog_main', 
        (h) => ({ 
            icon: 'book', 
            displayName: 'Blog', 
            uid: h.user._id.toString() 
        }),
        PRIV.PRIV_USER_PROFILE
    );
}
```

---

### **2. Blog Templates**

#### **Blog List Page (`blog_main.html`)**

Displays a list of blogs.

```html
{% extends "layout/basic.html" %}
{% block content %}
<div class="row">
    <div class="medium-9 columns">
        {% for blog in blogs %}
        <div class="blog-entry">
            <h2><a href="{{ url('blog_detail', {did: blog._id}) }}">
                {{ blog.title }}
            </a></h2>
            <div class="meta">
                {{ user.render_inline(blog.owner) }} @ {{ datetime(blog.updateAt) }}
            </div>
        </div>
        {% endfor %}
    </div>
</div>
{% endblock %}
```

---

#### **Blog Detail Page (`blog_detail.html`)**

Displays the content of an individual blog post.

```html
{% extends "layout/basic.html" %}
{% block content %}
<div class="blog-detail">
    <h1>{{ ddoc.title }}</h1>
    <div class="content">{{ ddoc.content|markdown }}</div>
    <div class="meta">
        Views: {{ ddoc.views }}
        {% if handler.user.own(ddoc) %}
        <a href="{{ url('blog_edit', {did: ddoc._id}) }}">Edit</a>
        {% endif %}
    </div>
</div>
{% endblock %}
```

---

### **3. Modified Pages**
- **User Profile Page**: Adds a "Blog" tab/link.
- **Navigation Menu**: Adds blog access in the user dropdown.

### **4. New Pages Added**
- **Blog List Page**: `/blog/:uid`
- **Blog Detail Page**: `/blog/:uid/:did`
- **Blog Edit Page**: `/blog/:uid/:did/edit`
- **Blog Create Page**: `/blog/:uid/create`

----

## **Internationalization**

The blog plugin supports multiple languages.

```typescript
// From blog/index.ts
ctx.i18n.load('en', {
    blog_main: 'Blog',
    blog_detail: 'Blog Detail',
    blog_edit: 'Edit Blog',
});
```

---

## **Integration with HydroOJ**

The blog plugin integrates seamlessly with HydroOJ through:
1. **Template inheritance** from base layouts.
2. **Consistent styling** with HydroOJ's design.
3. **Integration with user permissions** for access control.
4. **Multi-language support** for internationalization.
5. **Responsive design** matching HydroOJ's layout.

This ensures the blog functionality feels like a natural extension of the platform.
