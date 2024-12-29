# Overview of onsite-toolkit Plugin

The onsite-toolkit plugin is designed to enhance HydroOJ's contest functionality by providing:
1. IP-based auto-login functionality
2. Contest resolver visualization 
3. Contest data export in CDP format
4. Custom scoreboard views

Let's break down each component:

## 1. Imported Packages Usage

```typescript
import {
    AdmZip, avatar, ContestModel, ContestNotEndedError, Context, db, findFileSync,
    ForbiddenError, fs, ObjectId, parseTimeMS, PERM, ProblemConfig, ProblemModel,
    STATUS, STATUS_SHORT_TEXTS, STATUS_TEXTS, Time, UserModel,
} from 'hydrooj';
```

Here's how each imported package is utilized:

### Database Related
- `db`: Used to create and manage the IP login collection
```typescript
const coll = db.collection('iplogin');
```

### Models
- `ContestModel`: Handles contest data and operations
- `ProblemModel`: Manages problem data and configurations  
- `UserModel`: Handles user data and operations

Example usage:
```typescript
// Get contest status
const teams = await ContestModel.getMultiStatus(tdoc.domainId, { docId: tdoc.docId });

// Get problem details 
const pdict = await ProblemModel.getList(tdoc.domainId, tdoc.pids);

// Get user info
const udict = await UserModel.getList(tdoc.domainId, teams.map((i) => i.uid));
```

### File System
- `AdmZip`: Creates ZIP archives for CDP export
- `fs`: File system operations
- `findFileSync`: Locates files in the plugin directory

Example:
```typescript
const zip = new AdmZip();
zip.addFile('event-feed.ndjson', Buffer.from(eventfeed.concat(submissions).concat(endState)));
zip.addFile('contest/logo.png', fs.readFileSync(findFileSync('@hydrooj/onsite-toolkit/public/logo.png')));
```

### Status & Permissions
- `STATUS`: Contest submission status codes
- `STATUS_SHORT_TEXTS`: Short status text representations
- `STATUS_TEXTS`: Full status text descriptions
- `PERM`: Permission constants

Example:
```typescript
if (!this.user.own(tdoc) && ContestModel.isLocked(tdoc)) {
    this.checkPerm(PERM.PERM_VIEW_CONTEST_HIDDEN_SCOREBOARD);
}
```

## 2. Plugin Components

### IP Login Handler
```typescript
ctx.on('handler/init', async (that) => {
    const iplogin = await coll.findOne({ _id: normalizeIp(that.request.ip) });
    if (iplogin) {
        that.user = await UserModel.getById(that.domain._id, iplogin.uid);
        // Auto-login user based on IP
        that.session.ipLoggedIn = true;
        that.session.uid = iplogin.uid;
        that.session.user = that.user;
    }
});
```

### Custom Scoreboard Views

1. Resolver View (Tiny):
```typescript
scoreboard.addView('resolver-tiny', 'Resolver(Tiny)', { tdoc: 'tdoc' }, {
    async display({ tdoc }) {
        // Fetch contest data
        const teams = await ContestModel.getMultiStatus(tdoc.domainId, { docId: tdoc.docId });
        // Render resolver interface
        this.response.template = 'resolver.html';
    }
});
```

2. CDP Export View:
```typescript
scoreboard.addView('cdp', 'CDP', { tdoc: 'tdoc' }, {
    async display({ tdoc }) {
        // Generate CDP format data
        // Export as ZIP file
        this.binary(zip.toBuffer(), `contest-${tdoc._id}-cdp.zip`);
    }
});
```

## 3. UI Modifications

The plugin modifies the contest scoreboard UI by adding new views:

### Resolver View
Located in `/onsite-toolkit/templates/resolver.html`:
```html
{% extends "layout/html5.html" %}
{% block body %}
<header class="header">
    <span class="rank">#Rank</span>
    <span class="content">
        <span class="title">Contest</span>
        <span class="copyright">@Hydro/TinyResolver</span>
    </span>
    <span class="solved">Solved</span>
    <span class="penalty">Penalty</span>
</header>
<div class="rank-list" id="rank-list"></div>
{% endblock %}
```

Styled using `/onsite-toolkit/public/resolver.css`:
```css
.rank-list {
    background: repeating-linear-gradient(
      180deg,
      #3e3e3e 0px,
      #3e3e3e 80px,
      #121212 80px,
      #121212 160px
    );
}

.rank-list-item {
    padding: 10px 0;
    height: 80px;
    position: relative;
    background: transparent;
    display: flex;
    align-items: center;
}
```

The UI is rendered using React components in `/onsite-toolkit/frontend/resolver.page.tsx`:
```typescript
const MainList = (props: Props) => {
    // Render ranking list with animations
    return (
        <div className="rank-list">
            {teams.map((team, index) => (
                <animated.div
                    className="rank-list-item"
                    style={springs[index]}
                >
                    {/* Team info and problem status */}
                </animated.div>
            ))}
        </div>
    );
};
```

## Pages Modified

1. Contest Scoreboard Page (`/contest/{id}/scoreboard`)
   - Adds "Resolver(Tiny)" view option
   - Adds "CDP" export option

2. Contest Status Page (`/contest/{id}/status`) 
   - Enhanced with IP-based auto-login functionality

The plugin primarily focuses on enhancing the contest experience by providing:
- Automated login for onsite contests
- Visual resolver for dramatic award ceremonies
- CDP format export for integration with other systems
- Enhanced scoreboard visualization

All these modifications are achieved through HydroOJ's plugin system, utilizing the provided models and interfaces while maintaining compatibility with the core system.
