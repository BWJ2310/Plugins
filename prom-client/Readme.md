1. Overview of prom-client Plugin:
The prom-client plugin integrates Prometheus metrics monitoring into HydroOJ. It collects various system and application metrics and exposes them in Prometheus format.

2. Package Imports and Their Usage:

```typescript
import { hostname } from 'os';
import { AggregatorRegistry, Metric } from 'prom-client';
import { Context, Handler, superagent, SystemModel } from 'hydrooj';
```

- `os`: Used to get system hostname
- `prom-client`: Core Prometheus client library
- `hydrooj` imports:
  - `Context`: Application context management
  - `Handler`: Base class for HTTP request handling
  - `superagent`: HTTP client for pushing metrics
  - `SystemModel`: System configuration management

3. Core Components:

a) MetricsHandler:
```typescript
class MetricsHandler extends Handler {
    noCheckPermView = true;
    notUsage = true;

    async get() {
        // Basic auth handling
        if (!this.request.headers.authorization) {
            this.response.status = 401;
            this.response.body = {};
            this.response.addHeader('WWW-Authenticate', 'Basic');
            return;
        }
        // Verify credentials
        const [name, password] = SystemModel.getMany(['prom-client.name', 'prom-client.password']);
        const key = this.request.headers.authorization.split('Basic ')?.[1];
        if (!key || key !== Buffer.from(`${name}:${password}`).toString('base64')) {
            this.response.status = 403;
            this.response.body = {};
            return;
        }
        // Return metrics
        this.response.body = await AggregatorRegistry.aggregate(Object.values(instances)).metrics();
        this.response.type = 'text/plain';
    }
}
```

b) Metrics Collection (metrics.ts):
```typescript
export function createRegistry(ctx: Context) {
    const registry = new Registry();
    
    // Request counter
    const reqCounter = createMetric(Counter, 'hydro_reqcount', 'reqcount', {
        labelNames: ['domainId'],
    });
    
    // Judge counter
    const judgeCounter = createMetric(Counter, 'hydro_judgecount', 'judgecount');
    
    // User registration gauge
    createMetric(Gauge, 'hydro_regcount', 'regcount', {
        async collect() {
            this.set({}, await db.collection('user').countDocuments());
        },
    });
    
    // More metrics...
}
```

4. Key Functionalities:

a) Metric Types:
- Counters: Track cumulative values (requests, judgments)
- Gauges: Track current values (user count, connections)

b) Automatic Collection:
```typescript
ctx.setInterval(async () => {
    try {
        const [gateway, name, pass] = SystemModel.getMany([
            'prom-client.gateway', 
            'prom-client.name', 
            'prom-client.password'
        ]);
        if (gateway) {
            // Push to Prometheus gateway
            const endpoint = `${prefix}metrics/job/hydro-web/instance/${hostname()}:${process.env.NODE_APP_INSTANCE}`;
            await superagent.post(endpoint)
                .auth(name, pass, { type: 'basic' })
                .send(await registry.metrics());
        }
    } catch (e) {
        pushError = e.message;
    }
}, 5000 * (+SystemModel.get('prom-client.collect_rate') || 1));
```

5. Metrics Collected:

```typescript
// Request metrics
reqCounter.inc({ domainId: h.args.domainId });

// Judge metrics
judgeCounter.inc();

// User registration metrics
createMetric(Gauge, 'hydro_regcount', 'regcount');

// Submission metrics
submissionCounter.inc({ lang: that.args.lang, domainId: that.args.domainId });

// Connection metrics
connectionGauge.inc({ domainId: h.args.domainId });
```

6. UI Integration:
The plugin doesn't directly modify HydroOJ's UI. Instead, it exposes metrics at the `/metrics` endpoint which can be scraped by Prometheus and visualized using tools like Grafana.

7. System Integration:

```typescript
export function apply(ctx: Context) {
    if (process.env.HYDRO_CLI) return;
    
    // Create metrics registry
    const registry = createRegistry(ctx);
    
    // Setup metrics broadcast
    ctx.on('metrics', (id, metrics) => { instances[id] = metrics; });
    
    // Add system checker
    ctx.inject(['check'], (c) => {
        c.check.addChecker('prom-client', async (_, log, warn, error) => {
            if (pushError) error(`Prometheus push error: ${pushError}`);
            else if (gateway) log(`Prometheus pushed to gateway: ${gateway}`);
            else log('Prometheus metrics server running.');
        });
    });
    
    // Register metrics endpoint
    ctx.Route('metrics', '/metrics', MetricsHandler);
}
```

8. Configuration:
The plugin uses system settings:
```typescript
interface SystemKeys {
    'prom-client.name': string;
    'prom-client.password': string;
    'prom-client.gateway': string;
    'prom-client.collect_rate': number;
}
```

9. Integration with HydroOJ Events:
```typescript
// Track submissions
ctx.on('handler/after/ProblemSubmit#post', (that) => {
    submissionCounter.inc({ 
        lang: that.args.lang, 
        domainId: that.args.domainId 
    });
});

// Track connections
ctx.on('connection/active', (h) => {
    connectionGauge.inc({ 
        domainId: h.args.domainId 
    });
});
```

The plugin provides comprehensive monitoring capabilities without modifying the UI directly. Instead, it integrates with Prometheus's monitoring ecosystem, allowing administrators to:
1. Collect metrics about system performance
2. Monitor user activity
3. Track submission patterns
4. Watch system resource usage

The metrics can be visualized using external tools like Grafana by connecting to the `/metrics` endpoint, which requires basic authentication configured through the system settings.
