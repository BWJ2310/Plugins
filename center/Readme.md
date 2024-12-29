# 1. Center Plugin Overview

The center plugin is designed to handle data reporting and monitoring for HydroOJ installations. It collects system information, usage statistics, and version details from HydroOJ instances.

## Imported Package Usage

```typescript
import {
    db, definePlugin, ForbiddenError, Handler, post, Types, yaml,
} from 'hydrooj';
```

Let's examine how each package is used:

### 1. db
```typescript
const coll = db.collection('dataReport');
// Used for database operations
await coll.updateOne({ _id: installId }, {
    $addToSet: { ips: this.request.ip },
    $set: setPayload,
    $setOnInsert: { init: new Date() },
}, { upsert: true });
```

### 2. definePlugin
```typescript
export default definePlugin({
    apply(ctx) {
        ctx.Route('data_report', '/center/report', DataReportHandler);
    },
});
```

### 3. Handler & post decorator
```typescript
class DataReportHandler extends Handler {
    noCheckPermView = true;
    category = '#center';

    @post('installId', Types.String)
    @post('payload', Types.String)
    async post(domainId: string, installId: string, _payload: string) {
        // Handler implementation
    }
}
```

# 2. Models and Functions

## DataReportHandler
The main handler for processing installation reports:

```typescript
class DataReportHandler extends Handler {
    noCheckPermView = true;
    category = '#center';

    @post('installId', Types.String)
    @post('payload', Types.String)
    async post(domainId: string, installId: string, _payload: string) {
        // Decrypt and validate payload
        let payload: any;
        try {
            payload = yaml.load(decrypt(_payload));
        } catch (e) {
            payload = yaml.load(_payload);
        }

        // Validate required fields
        try {
            assert(typeof payload.url === 'string');
        } catch (e) {
            throw new ForbiddenError();
        }

        // Store report data
        const setPayload = {
            version: payload.version,
            name: payload.name,
            url: payload.url,
            addons: payload.addons,
            mem: payload.memory,
            osinfo: payload.osinfo,
            cpu: payload.cpu,
            ip: this.request.ip,
            flags: payload.flags,
            update: new Date(),
            domainCount: payload.domainCount,
            userCount: payload.userCount,
            problemCount: payload.problemCount,
            discussionCount: payload.discussionCount,
            recordCount: payload.recordCount,
            sandbox: payload.sandbox,
            dbVersion: payload.dbVersion,
        };

        // Update database
        await coll.updateOne(
            { _id: installId },
            {
                $addToSet: { ips: this.request.ip },
                $set: setPayload,
                $setOnInsert: { init: new Date() },
            },
            { upsert: true }
        );
    }
}
```

## Encryption/Decryption
```typescript
function decrypt(encrypted: string) {
    return crypto.DES.decrypt(
        { ciphertext: crypto.enc.Hex.parse(encrypted) },
        crypto.enc.Utf8.parse('hydro-oj'),
        { mode: crypto.mode.ECB },
    ).toString(crypto.enc.Utf8);
}
```

# 3. Database Schema

The plugin uses a `dataReport` collection with the following structure:

```typescript
interface DataReport {
    _id: string;           // Installation ID
    version: string;       // HydroOJ version
    name: string;         // Installation name
    url: string;          // Installation URL
    addons: string[];     // Installed addons
    mem: number;          // Memory usage
    osinfo: {             // OS information
        platform: string;
        release: string;
        arch: string;
    };
    cpu: {                // CPU information
        model: string;
        cores: number;
    };
    ip: string;           // Reporter IP
    flags: string[];      // System flags
    update: Date;         // Last update time
    init: Date;           // Initial report time
    ips: string[];        // All reporter IPs
    domainCount: number;  // Number of domains
    userCount: number;    // Number of users
    problemCount: number; // Number of problems
    discussionCount: number; // Number of discussions
    recordCount: number;  // Number of records
    sandbox: string;      // Sandbox information
    dbVersion: string;    // Database version
}
```

# 4. UI Integration

The center plugin primarily operates in the background and doesn't add significant UI elements, but it does provide:

1. A reporting endpoint at `/center/report`
2. Potential notification delivery system (TODO in code)

```typescript
// Response format
this.response.body = { 
    code: 0,
    notification: old?.notification // Optional notification
};
```

The plugin extends HydroOJ's functionality by:

1. Collecting system metrics
2. Monitoring installation health
3. Tracking version information
4. Managing installation notifications

# 5. Event System

The plugin integrates with HydroOJ's event system:

```typescript
declare module 'hydrooj' {
    interface EventMap {
        'center/report': (
            thisArg: DataReportHandler, 
            installId: string, 
            old: any, 
            payload: any
        ) => void;
    }
}

// Event emission
this.ctx.emit('center/report', this, installId, old, payload);
```

This allows other parts of the system to react to new installation reports.

The center plugin is primarily a backend service that collects and manages installation data rather than providing user-facing UI elements. Its main purpose is to gather system information and usage statistics from HydroOJ installations for monitoring and management purposes.
