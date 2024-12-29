1. Plugin Overview:
The import-hoj plugin is designed to import problem sets from HOJ (presumably another OJ system) into HydroOJ. It handles ZIP files containing problem data and configurations.

2. Imported Packages Usage:

```typescript
import {
    AdmZip, buildContent, Context, fs, Handler, PERM,
    ProblemConfigFile, ProblemModel, ValidationError, yaml,
} from 'hydrooj';
```

- AdmZip: Used for ZIP file handling
```typescript
zip = new AdmZip(zipfile); // Creates ZIP instance
zip.extractAllToAsync(tmp, true, (err) => ...); // Extracts ZIP contents
```

- fs: File system operations
```typescript
fs.ensureDirSync(tmpdir); // Create directory
fs.readdir(tmp, { withFileTypes: true }); // Read directory contents
fs.readFile(path.join(tmp, `${folder}.json`)); // Read files
```

- Handler: Base class for request handling
```typescript
class ImportHojHandler extends Handler {
    async get() { ... }
    async post({ domainId }) { ... }
}
```

- PERM: Permission management
```typescript
ctx.Route('problem_import_hoj', '/problem/import/hoj', ImportHojHandler, PERM.PERM_CREATE_PROBLEM);
```

- ProblemModel: Database operations for problems
```typescript
await ProblemModel.add(
    domainId, pdoc.display_id, pdoc.title, buildContent(content, 'markdown'),
    this.user._id, doc.tags || [],
);
```

- ValidationError: Error handling
```typescript
throw new ValidationError('zip', null, e.message);
```

- yaml: YAML parsing/dumping
```typescript
yaml.dump(config) // Converts config object to YAML
```

3. Main Components:

ImportHojHandler Class:
```typescript
class ImportHojHandler extends Handler {
    // Handles file import
    async fromFile(domainId: string, zipfile: string) { ... }
    
    // GET request handler
    async get() { ... }
    
    // POST request handler 
    async post({ domainId }) { ... }
}
```

The fromFile method:
- Extracts ZIP file
- Processes problem JSON files
- Creates problem documents
- Handles test cases
- Manages special judge configurations
- Sets up extra files

4. UI Modifications:

The plugin modifies the UI through:

```typescript
export async function apply(ctx: Context) {
    // Adds route for import
    ctx.Route('problem_import_hoj', '/problem/import/hoj', ImportHojHandler, PERM.PERM_CREATE_PROBLEM);
    
    // Injects UI component
    ctx.injectUI('ProblemAdd', 'problem_import_hoj', { 
        icon: 'copy', 
        text: 'From HOJ Export' 
    });
    
    // Adds i18n support
    ctx.i18n.load('zh', {
        'From HOJ Export': 'ä» HOJ å¯¼å¥',
    });
}
```

The UI changes appear in:
- Problem management page
- Adds import option in problem creation interface
- Provides a dedicated import page at '/problem/import/hoj'

5. Problem Import Process:

```typescript
// Configuration handling
const config: ProblemConfigFile = {
    time: `${pdoc.timeLimit}ms`,
    memory: `${pdoc.memoryLimit}m`,
    subtasks: [],
};

// Test case processing
for (const tc of doc.samples) {
    tasks.push(ProblemModel.addTestdata(
        domainId, pid, tc.input,
        path.join(tmp, folder, tc.input),
    ));
    // ... output handling
}

// Special judge handling
if (pdoc.spjLanguage === 'C++') {
    tasks.push(ProblemModel.addTestdata(
        domainId, pid, 'checker.cc',
        Buffer.from(pdoc.spjCode),
    ));
    config.checker = 'checker.cc';
    config.checker_type = 'testlib';
}
```

6. File Structure:
```
/import-hoj/
  ├── index.ts      # Main plugin code
  └── package.json  # Plugin metadata
```

The plugin creates a temporary directory for processing:
```typescript
const tmpdir = path.join(os.tmpdir(), 'hydro', 'import-hoj');
```

The plugin supports importing:
- Problem descriptions
- Test cases
- Special judge code
- Extra files for both users and judges
- Problem configurations
- Tags and metadata

All imported problems are validated and properly configured before being added to the HydroOJ system.
