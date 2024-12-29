# 1. Overview of Coin Plugin

The coin plugin is a module for HydroOJ that implements a virtual currency system. It allows:
- Administrators to distribute/deduct coins from users
- Users to view their coin balance and transaction history 
- Display of coin rankings across all users

# 2. Imported Packages from HydroOJ

```typescript
import {
    _, db, UserModel, SettingModel, DomainModel, Handler, 
    param, PRIV, Types, paginate, query
} from 'hydrooj';
```

Let's break down how each import is used:

1. `db`: Database interface for MongoDB operations
```typescript
const coll = db.collection('coin'); // Creates collection for coin transactions
```

2. `UserModel`: Handles user-related operations
```typescript
// Used to update user coin balances
await UserModel.inc(userId, 'coin_now', amount);
await UserModel.inc(userId, 'coin_all', amount);
```

3. `SettingModel`: Manages system settings
```typescript
// Adds coin settings to user account settings
SettingModel.Setting('setting_33oj', 'coin_now', 0, 'number', 'coin_now', null, 3)
```

4. `Handler`: Base class for request handlers
```typescript
class CoinShowHandler extends Handler {
    async get(domainId: string, page = 1) {
        // Handler implementation
    }
}
```

5. `param` & `query`: Decorators for parameter validation
```typescript
@param('uidOrName', Types.UidOrName)
@query('page', Types.PositiveInt, true)
```

6. `PRIV`: Permission constants
```typescript
ctx.Route('coin_inc', '/coin/inc', CoinIncHandler, PRIV.PRIV_CREATE_DOMAIN);
```

7. `Types`: Data type validators
```typescript
@param('amount', Types.Int)
```

8. `paginate`: Handles pagination
```typescript
const [dudocs, upcount, ucount] = await paginate(
    UserModel.getMulti({ coin_all: { $exists: true } }),
    page,
    50
);
```

# 3. Models and Functions

## 3.1 Coin Model

```typescript
interface Bill {
    _id: string;
    userId: number;
    rootId: number;
    amount: number;
    text: string;
}

const coinModel = {
    // Increment/decrement coins
    async inc(userId: number, rootId: number, amount: number, text: string) {
        await coll.insertOne({ userId, rootId, amount, text });
        await UserModel.inc(userId, 'coin_now', amount);
        if (amount > 0)
            await UserModel.inc(userId, 'coin_all', amount);
    },

    // Get total bill count
    async billCount() {
        return await coll.count();
    },

    // Get all transactions
    async getAll(ll: number, ss: number) {
        return await coll.find()
            .limit(ll)
            .skip((ss - 1) * ll)
            .sort({ _id: -1 })
            .toArray();
    },

    // Other functions...
};
```

# 4. Handlers

## 4.1 CoinShowHandler (Coin Rankings)
```typescript
class CoinShowHandler extends Handler {
    @query('page', Types.PositiveInt, true)
    async get(domainId: string, page = 1) {
        // Fetches and displays user coin rankings
    }
}
```

## 4.2 CoinIncHandler (Coin Distribution)
```typescript
class CoinIncHandler extends Handler {
    @param('uidOrName', Types.UidOrName)
    @param('amount', Types.Int)
    @param('text', Types.String)
    async post(domainId: string, uidOrName: string, amount: number, text: string) {
        // Handles coin distribution
    }
}
```

# 5. UI Modifications

The plugin adds three new pages to HydroOJ:

1. **Coin Show Page** (`/coin/show`)
```html
{% extends "layout/basic.html" %}
<div class="section">
  <div class="section__header">
    <h1>所有拥有硬币的学生</h1>
    <!-- Navigation buttons -->
  </div>
  <table class="data-table">
    <!-- Coin ranking table -->
  </table>
</div>
```

2. **Coin Distribution Page** (`/coin/inc`)
- Form for administrators to distribute coins
- Uses UserSelectAutoComplete for user selection

3. **Transaction History Page** (`/coin/bill/:uid`)
- Displays transaction history for specific user
- Includes pagination and filtering

The UI integration is done through:

1. Route Registration:
```typescript
export async function apply(ctx: Context) {
    ctx.Route('coin_inc', '/coin/inc', CoinIncHandler);
    ctx.Route('coin_show', '/coin/show', CoinShowHandler);
    ctx.Route('coin_bill', '/coin/bill/:uid', CoinBillHandler);
}
```

2. Template Integration:
- Templates are stored in `/coin/templates/`
- Uses Hydro's template engine
- Extends base layouts for consistent styling

3. Frontend Enhancement:
```typescript
addPage(new NamedPage(['coin_inc'], () => {
    UserSelectAutoComplete.getOrConstruct($('[name="uidOrName"]'));
}));
```

The plugin seamlessly integrates with HydroOJ's existing UI framework while adding new functionality for coin management. It follows HydroOJ's permission system and UI patterns to maintain consistency across the platform.
