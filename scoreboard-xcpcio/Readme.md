1. Overview of the Plugin:
The scoreboard-xcpcio plugin integrates XCPCIO-style scoreboard visualization into HydroOJ for programming contests. It provides a specialized view for contest scoreboards with features like medal counts, team grouping, and real-time updates.

2. Imported Packages from HydroOJ:

```typescript
import {
    ContestModel, Context, fs, Handler, ObjectId, PERM, Schema, STATUS, Types, UserModel,
} from 'hydrooj';
```

Let's break down each import:

a) ContestModel: Handles contest-related operations
```typescript
// Used for getting contest status and checking locks
const tsdocs = await ContestModel.getMultiStatus(tdoc.domainId, { docId: tdoc.docId });
ContestModel.isLocked(tdoc)
```

b) Context: Manages application context and routing
```typescript
export async function apply(ctx: Context) {
    ctx.Route('board_xcpcio', '/board', XcpcioHandler);
    ctx.inject(['scoreboard'], ({ scoreboard }) => {
        // Adds scoreboard view
    });
}
```

c) Handler: Base class for request handlers
```typescript
class XcpcioHandler extends Handler {
    async get() {
        this.response.body = {
            js: indexJs,
            css: indexCss,
        };
        this.response.template = 'xcpcio_board.html';
    }
}
```

d) ObjectId: Manages MongoDB ObjectIds
```typescript
const submit = new ObjectId(j.rid).getTimestamp().getTime();
```

e) PERM: Permission management
```typescript
if (realtime && !this.user.own(tdoc)) {
    this.checkPerm(PERM.PERM_VIEW_CONTEST_HIDDEN_SCOREBOARD);
}
```


f) Schema
The Schema package is part of HydroOJ's validation system, used for defining data structures and validation rules. In the scoreboard-xcpcio plugin, it's used to define configuration options:

```typescript
// Example from scoreboard configuration
{
    gold: Schema.transform(
        Schema.union([
            Schema.string(),
            Schema.number().step(1).min(0)
        ]), 
        (v) => +v
    ).default(0),
    silver: Schema.transform(
        Schema.union([
            Schema.string(), 
            Schema.number().step(1).min(0)
        ]), 
        (v) => +v
    ).default(0)
}
```

g) STATUS
STATUS is an enumeration of different submission/contest states in HydroOJ. From the builtin.ts file, it includes:

```typescript
const STATUS = {
    STATUS_WRONG_ANSWER: 'WRONG_ANSWER',
    STATUS_ACCEPTED: 'CORRECT',
    STATUS_TIME_LIMIT_EXCEEDED: 'TIME_LIMIT_EXCEEDED',
    STATUS_MEMORY_LIMIT_EXCEEDED: 'MEMORY_LIMIT_EXCEEDED',
    STATUS_RUNTIME_ERROR: 'RUNTIME_ERROR',
    STATUS_COMPILE_ERROR: 'COMPILE_ERROR',
    STATUS_SYSTEM_ERROR: 'SYSTEM_ERROR',
    STATUS_CANCELED: 'CANCELED',
    STATUS_ETC: 'ETC'
};
```

h) Types
Types provides TypeScript type definitions used throughout HydroOJ. Key types used in the scoreboard plugin include:

```typescript
// Common Types used in scoreboard
interface ScoreboardConfig {
    tdoc: string;
    groups: string[];
    json: boolean;
    realtime: boolean;
    gold: number;
    silver: number;
    bronze: number;
}

interface ScoreboardRow {
    user: User;
    score: number;
    rank: number;
    submissions: Submission[];
}
```

i) UserModel
UserModel, defined in user.ts, handles all user-related operations. Key functionalities used by the scoreboard plugin include:

```typescript
class UserModel {
    // Get user by ID
    static async getById(domainId: string, _id: number): Promise<User> {
        const udoc = await coll.findOne({ _id });
        if (!udoc) return null;
        const dudoc = await domain.getDomainUser(domainId, udoc);
        return new User(udoc, dudoc);
    }

    // Get multiple users
    static async getList(domainId: string, uids: number[]): Promise<Udict> {
        const r: Udict = {};
        await Promise.all(uniq(uids).map(async (uid) => {
            r[uid] = await UserModel.getById(domainId, uid);
        }));
        return r;
    }

    // Get user by username
    static async getByUname(domainId: string, uname: string): Promise<User> {
        const unameLower = uname.trim().toLowerCase();
        const udoc = await coll.findOne({ unameLower });
        if (!udoc) return null;
        const dudoc = await domain.getDomainUser(domainId, udoc);
        return new User(udoc, dudoc);
    }
}
```

The scoreboard-xcpcio plugin utilizes these packages in the following ways:

1. Uses Schema for configuration validation:
```typescript
scoreboard.addView('xcpcio', 'XCPCIO', {
    // Schema validation for configuration options
    gold: Schema.transform(Schema.union([Schema.string(), Schema.number()]), (v) => +v),
    silver: Schema.transform(Schema.union([Schema.string(), Schema.number()]), (v) => +v),
    bronze: Schema.transform(Schema.union([Schema.string(), Schema.number()]), (v) => +v)
});
```

2. Uses STATUS for mapping submission states:
```typescript
const status = {
    [STATUS.STATUS_WRONG_ANSWER]: 'WRONG_ANSWER',
    [STATUS.STATUS_ACCEPTED]: 'CORRECT'
    // ... other status mappings
};
```

3. Uses Types for type safety:
```typescript
interface ScoreboardData {
    rows: ScoreboardRow[];
    problems: Problem[];
    teams: Team[];
}
```

4. Uses UserModel for fetching user data:
```typescript
const users = await UserModel.getList(domainId, tsdocs.map(i => i.uid));
const teams = await Promise.all(tsdocs.map(async (i) => {
    const udoc = users[i.uid];
    return {
        team_id: `${udoc._id}`,
        name: udoc.uname,
        organization: udoc.school
    };
}));
```

These packages form the foundation for the scoreboard-xcpcio plugin's functionality, providing data validation, user management, and type safety throughout the application.




3. Plugin Components:

a) XcpcioHandler:
Handles requests to the scoreboard page:
```typescript
class XcpcioHandler extends Handler {
    async get() {
        // Serves the scoreboard template
    }
}
```

b) Status Mapping:
```typescript
const status = {
    [STATUS.STATUS_WRONG_ANSWER]: 'WRONG_ANSWER',
    [STATUS.STATUS_ACCEPTED]: 'CORRECT',
    // ... other status mappings
};
```

4. UI Modifications:

The plugin modifies the contest scoreboard page through:

a) Template (xcpcio_board.html):
```html
{% extends "layout/basic.html" %}
{% block content %}
<div id="app"></div>
<script type="module" crossorigin src="{{ '/' if process.env.DEV else UiContext.cdn_prefix }}assets/index-{{ js }}.js"></script>
<link rel="stylesheet" crossorigin href="{{ '/' if process.env.DEV else UiContext.cdn_prefix }}assets/index-{{ css }}.css">
{% endblock %}
```

b) Scoreboard View Configuration:
```typescript
scoreboard.addView('xcpcio', 'XCPCIO', {
    tdoc: 'tdoc',
    groups: 'groups',
    json: Types.Boolean,
    realtime: Types.Boolean,
    gold: Schema.transform(Schema.union([Schema.string(), Schema.number().step(1).min(0)]), (v) => +v).default(0),
    silver: Schema.transform(Schema.union([Schema.string(), Schema.number().step(1).min(0)]), (v) => +v).default(0),
    bronze: Schema.transform(Schema.union([Schema.string(), Schema.number().step(1).min(0)]), (v) => +v).default(0),
}, {
    async display(/* ... */) {
        // Handles scoreboard display logic
    }
});
```

5. Data Processing:

The plugin processes contest data to create the scoreboard:

```typescript
// Team data processing
const teams = tsdocs.map((i) => ({
    team_id: `${udoc._id}`,
    name: udoc.uname,
    organization: udoc.school,
    members: [],
    coach: '',
    group: [
        ...(udoc.group || []),
        i.unrank ? 'unofficial' : 'official',
    ],
}));

// Submission processing
submissions: tsdocs.flatMap((i) => (i.journal || []).map((j) => ({
    problem_id: tdoc.pids.indexOf(j.pid),
    team_id: `${i.uid}`,
    timestamp: Math.floor(submit - tdoc.beginAt.getTime()),
    status: realtime
        ? curStatus
        : ContestModel.isLocked(tdoc) && submit > tdoc.lockAt.getTime()
            ? 'PENDING'
            : curStatus,
    language: j.lang || '',
    submission_id: j.rid,
})))
```

The plugin modifies the following pages:
- Contest scoreboard page (/board)
- Contest-specific scoreboard views

New features added:
- XCPCIO-style scoreboard visualization
- Medal counts (gold, silver, bronze)
- Team grouping
- Real-time updates
- Balloon colors for problems
- Support for official/unofficial participants

The plugin uses @xcpcio/board-app package (version ^0.53.2) for the actual scoreboard rendering and visualization.
