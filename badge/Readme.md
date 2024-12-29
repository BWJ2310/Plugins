# **Hydro OJ Badge Plugin Overview**

The **Badge Plugin** is designed to enhance Hydro OJ by enabling the following features:
- **Display of user badges**
- **Creation of new badges**
- **Management of existing badges**
- **Deletion of badges**

---

## **Imported Packages**

The plugin utilizes several core Hydro OJ packages:

```typescript
import {
    db, definePlugin, UserModel, Handler, UserNotFoundError, 
    NotFoundError, param, PermissionError, PRIV, Types,
} from 'hydrooj';
```

### **Package Breakdown**

#### **1. db**
The `db` service interacts with MongoDB for database operations.

```typescript
// Example from db.ts
class MongoService {
    public collection<K extends keyof Collections>(c: K) {
        let coll = this.opts.prefix ? `${this.opts.prefix}.${c}` : c;
        if (this.opts.collectionMap?.[coll]) coll = this.opts.collectionMap[coll];
        return this.db.collection<Collections[K]>(coll);
    }
}
```

---

#### **2. Handler**
The `Handler` class is the base for handling HTTP requests.

```typescript
// Example from badge plugin
class BadgeShowHandler extends Handler {
    async get() {
        const udocs = await UserModel.getMulti({ badge: { $exists: true, $ne: "" } }).toArray();
        this.response.template = 'badge_show.html';
        this.response.body = { udocs };
    }
}
```

---

#### **3. UserModel**
The `UserModel` is used for user-related database operations.

```typescript
// Example from badge plugin
await UserModel.setById(udoc._id, { badge: text + color + textColor });
const udocs = await UserModel.getMulti({ badge: { $exists: true, $ne: "" } }).toArray();
```

---

#### **4. param**
The `param` decorator validates HTTP request parameters.

```typescript
// Example from badge plugin
@param('uidOrName', Types.UidOrName)
@param('text', Types.String)
@param('color', Types.String)
```

---

#### **5. PRIV**
The `PRIV` constant defines permissions for access control.

```typescript
// Example usage in route definitions
ctx.Route('badge_show', '/badge', BadgeShowHandler, PRIV.PRIV_USER_PROFILE);
ctx.Route('badge_create', '/badge/create', BadgeCreateHandler, PRIV.PRIV_CREATE_DOMAIN);
```

---

## **UI Modifications**

The badge plugin modifies the UI through:
1. **Template Files**:
   - **`badge_show.html`**: Displays all users with badges.
   - **`badge_create.html`**: Provides a form for creating badges.
   - **`badge_manage.html`**: Lists users with badges and offers management options.

2. **Frontend JavaScript Integration**:

```typescript
// From frontend/foo.page.ts
import { $, addPage, NamedPage, UserSelectAutoComplete } from '@hydrooj/ui-default'

addPage(new NamedPage(['badge_create'], () => {
    UserSelectAutoComplete.getOrConstruct($('[name="uidOrName"]'), {
        clearDefaultValue: false,
    });
}));
```

---

## **Main Functionality Implementation**

### **1. Badge Display**

Displays all users with badges using `BadgeShowHandler`.

```typescript
class BadgeShowHandler extends Handler {
    async get() {
        const udocs = await UserModel.getMulti({ badge: { $exists: true, $ne: "" } }).toArray();
        this.response.template = 'badge_show.html';
        this.response.body = { udocs };
    }
}
```

---

### **2. Badge Creation**

Handles badge creation with custom text and colors using `BadgeCreateHandler`.

```typescript
class BadgeCreateHandler extends Handler {
    @param('uidOrName', Types.UidOrName)
    @param('text', Types.String)
    @param('color', Types.String)
    @param('textColor', Types.String)
    async post(domainId: string, uidOrName: string, text: string, color: string, textColor: string) {
        let udoc = await UserModel.getByUname(domainId, uidOrName);
        if (!udoc) throw new NotFoundError(uidOrName);
        // Validation and creation logic
        await UserModel.setById(udoc._id, { badge: text + color + textColor });
        this.response.redirect = "/badge";
    }
}
```

---

### **3. Badge Management**

Lists users with badges and provides management options using `BadgeManageHandler`.

```typescript
class BadgeManageHandler extends Handler {
    async get() {
        const udocs = await UserModel.getMulti({ badge: { $exists: true, $ne: "" } }).toArray();
        this.response.template = 'badge_manage.html';
        this.response.body = { udocs };
    }
}
```

---

### **4. Badge Deletion**

Removes badges from users using `BadgeDelHandler`.

```typescript
class BadgeDelHandler extends Handler {
    @param('uid', Types.Int)
    async get(domainId: string, uid: number) {
        await UserModel.setById(uid, { badge: "" });
        this.response.redirect = "/badge/manage";
    }
}
```

---

## **Route Registration**

The plugin registers its routes during initialization:

```typescript
export async function apply(ctx: Context) {
    ctx.Route('badge_show', '/badge', BadgeShowHandler, PRIV.PRIV_USER_PROFILE);
    ctx.Route('badge_create', '/badge/create', BadgeCreateHandler, PRIV.PRIV_CREATE_DOMAIN);
    ctx.Route('badge_manage', '/badge/manage', BadgeManageHandler, PRIV.PRIV_CREATE_DOMAIN);
    ctx.Route('badge_del', '/badge/manage/:uid/del', BadgeDelHandler, PRIV.PRIV_CREATE_DOMAIN);
}
```

---

## **Integration with Hydro OJ**

The badge plugin integrates seamlessly with Hydro OJ’s core infrastructure by:
1. **Using the database service (`db`)** for persistent storage.
2. **Implementing handlers (`Handler`)** for HTTP request processing.
3. **Leveraging the permission system (`PRIV`)** for access control.
4. **Adding frontend components** through Hydro OJ's UI framework.
5. **Managing user data (`UserModel`)** for badge assignments.
