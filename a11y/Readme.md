
# **a11y Plugin Overview**

The `a11y` plugin in HydroOJ primarily focuses on **performance testing** and **user administration** rather than traditional accessibility (a11y) features. It does not include UI templates or frontend modifications.

---

## **Plugin Structure**

The plugin leverages core functionalities of HydroOJ and interacts with its models and event handlers.

### **Imported Packages from HydroOJ**

The plugin imports the following packages:

```typescript
import { Context, Schema, UserModel } from 'hydrooj';
import { ProblemModel, RecordModel, sleep, STATUS, yaml } from 'hydrooj';
```

---

## **Core Components**

### **1. Context**
Used for plugin initialization and event handling.

```typescript
export async function apply(ctx: Context) {
    ctx.on('handler/after/UserRegisterWithCode#post', async (that) => {
        if (that.session.uid === 2) await UserModel.setSuperAdmin(2);
    });
}
```

### **2. Schema**
Defines configuration structures for scripts and settings.

```typescript
ctx.addScript('performance-test', 'test', 
    Schema.object({ 
        enable5: Schema.boolean().default(false) 
    }), 
    startPerformanceTest
);
```

### **3. UserModel**
Handles user management operations, such as assigning roles.

```typescript
await UserModel.setSuperAdmin(2); // Sets user with ID 2 as super admin
```

### **4. ProblemModel**
Manages problems, including adding test problems and configuring test data.

```typescript
const docId = await ProblemModel.add('system', 'PTEST', 'Performance Test', 'test only', 1, [], 
    { hidden: true });
await ProblemModel.addTestdata('system', docId, 'config.yaml', /* ... */);
```

### **5. RecordModel**
Handles execution records of performance tests.

```typescript
const id = await RecordModel.add('system', docId, 1, 'cc.cc14o2', 
    `// TEST ${key}\n${head}${TESTS[key]}\nreturn 0;}`, true);
```

---

## **Main Functionalities**

### **1. Automatic Super Admin Assignment**

The plugin assigns **super admin privileges** to the user with ID `2` after they register.

```typescript
ctx.on('handler/after/UserRegisterWithCode#post', async (that) => {
    if (that.session.uid === 2) await UserModel.setSuperAdmin(2);
});
```

---

### **2. Performance Testing System**

#### **Creating a Hidden Test Problem**
The plugin creates a hidden test problem for performance testing:

```typescript
const docId = await ProblemModel.add('system', 'PTEST', 'Performance Test', 'test only', 1, [], { hidden: true });
await ProblemModel.addTestdata('system', docId, 'config.yaml', Buffer.from(yaml.dump({
    time: '3s',
    memory: args.enable5 ? '2g' : '512m',
    cases: new Array(20).fill({
        input: '1.in',
        output: '1.out',
    }),
})));
```

#### **Running Tests and Processing Results**

Test cases are executed, and results are analyzed for errors:

```typescript
const results = {};
await Promise.all(Object.keys(TESTS).map(async (key) => {
    // ... test execution ...
    const result = await RecordModel.get('system', id);
    if (result.status !== STATUS.STATUS_ACCEPTED && 
        result.status !== STATUS.STATUS_WRONG_ANSWER) {
        report({ message: `Test ${key} failed (${id}) ${result.status}` });
    } else {
        results[key] = result.testCases.map((i) => i.time);
    }
}));
```

#### **Performance Metrics**
The plugin calculates and reports performance metrics such as average time, standard deviation, and min/max values:

```typescript
const avgTime = Math.sum(...timeArr) / 20;
await report({
    message: [
        `-------Test ${key}-------`,
        ` Avg: ${formatL(avgTime)}  D:  ${formatR(Math.sum(...timeArr.map((i) => (i - avgTime) ** 2)) / 20, 5)} `,
        ` Max: ${formatL(Math.max(...timeArr))}  Min: ${formatR(Math.min(...timeArr))} `,
    ].join('\n'),
});
```

---

## **Key Takeaways**

The `a11y` plugin primarily focuses on:
1. **System Administration**:
   - Automatically assigns super admin privileges to specific users.
2. **Performance Testing**:
   - Creates and runs performance tests on the system.
   - Reports detailed statistics and performance metrics.
3. **Test Result Analysis**:
   - Provides detailed logs and reports for each test.

**Note**: Despite its name, the plugin does not provide traditional accessibility (a11y) features or modify the UI.
