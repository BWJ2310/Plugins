# Overview of import-qduoj Plugin

The import-qduoj plugin is designed to import problem sets from QDUOJ into Hydro OJ. It handles ZIP file extraction, problem data validation, and integration with Hydro's problem management system.

## Imported Packages Usage

1. **AdmZip**
Used for handling ZIP file operations:
```typescript
let zip: AdmZip;
try {
    zip = new AdmZip(zipfile);
} catch (e) {
    throw new ValidationError('zip', null, e.message);
}
```

2. **buildContent**
Constructs formatted problem content from imported data:
```typescript
const content = buildContent({
    description: pdoc.description?.value,
    input: pdoc.input_description?.value,
    output: pdoc.output_description?.value,
    samples: pdoc.samples.map((sample) => [sample.input, sample.output]),
    hint: pdoc.hint?.value,
    source: typeof pdoc.source === 'string' ? pdoc.source : pdoc.source?.value || '',
}, 'html');
```

3. **Context**
Provides application context for plugin integration:
```typescript
export async function apply(ctx: Context) {
    ctx.Route('problem_import_qduoj', '/problem/import/qduoj', ImportQduojHandler, PERM.PERM_CREATE_PROBLEM);
    ctx.injectUI('ProblemAdd', 'problem_import_qduoj', { icon: 'copy', text: 'From QDUOJ Export' });
}
```

4. **FileTooLargeError & ValidationError**
Handle file size and validation errors:
```typescript
if (stat.size > 256 * 1024 * 1024) throw new FileTooLargeError('256m');
if (!cnt) throw new ValidationError('zip', 'No problemset imported');
```

5. **fs**
Handles file system operations:
```typescript
fs.ensureDirSync(tmpdir);
const buf = await fs.readFile(path.join(tmp, folder, 'problem.json'));
await fs.remove(tmp);
```

6. **Handler**
Base class for request handling:
```typescript
class ImportQduojHandler extends Handler {
    async get() {
        this.response.body = { type: 'QDUOJ' };
        this.response.template = 'problem_import.html';
    }
}
```

7. **PERM**
Defines permission requirements:
```typescript
ctx.Route('problem_import_qduoj', '/problem/import/qduoj', ImportQduojHandler, PERM.PERM_CREATE_PROBLEM);
```

8. **ProblemModel**
Manages problem data in the database:
```typescript
const pid = await ProblemModel.add(
    domainId, pdoc.display_id, pdoc.title, content,
    this.user._id, pdoc.tags || [],
);
```

9. **Schema**
Validates imported problem data:
```typescript
const ProblemSchema = Schema.object({
    tags: Schema.array(Schema.string()),
    title: Schema.string().required(),
    description: StringValue,
    // ... other fields
});
```

## Models and Functions

### StringValue Schema
```typescript
const StringValue = Schema.object({
    format: Schema.union(['html', 'markdown']).default('html'),
    value: Schema.string(),
});
```
Validates string content with format specification.

### ProblemSchema
```typescript
const ProblemSchema = Schema.object({
    tags: Schema.array(Schema.string()),
    title: Schema.string().required(),
    // ... other fields
});
```
Defines the structure and validation rules for imported problems.

### ImportQduojHandler Class
Main handler for import operations:

1. **fromFile Method**
```typescript
async fromFile(domainId: string, zipfile: string) {
    // Extracts ZIP file
    // Processes problem data
    // Adds problems to database
}
```

2. **get Method**
```typescript
async get() {
    this.response.body = { type: 'QDUOJ' };
    this.response.template = 'problem_import.html';
}
```

3. **post Method**
```typescript
async post({ domainId }) {
    // Handles file upload
    // Validates file size
    // Calls fromFile to process import
}
```

## UI Modifications

The plugin modifies Hydro's UI in the following ways:

1. **Adds Import Option**
```typescript
ctx.injectUI('ProblemAdd', 'problem_import_qduoj', { 
    icon: 'copy', 
    text: 'From QDUOJ Export' 
});
```
- Adds a new import option in the problem creation interface
- Location: Problem creation/management page
- Functionality: Provides interface for uploading QDUOJ export files

2. **i18n Support**
```typescript
ctx.i18n.load('zh', {
    'From QDUOJ Export': 'ä» QDUOJ å¯¼å¥',
});
```
Adds translations for UI elements.

3. **Template Usage**
```typescript
this.response.template = 'problem_import.html';
```
Uses a dedicated template for the import interface.

## Integration Flow

1. User clicks "From QDUOJ Export" in problem management
2. System displays upload interface
3. User uploads QDUOJ export ZIP file
4. Handler processes file:
   - Validates size and format
   - Extracts contents
   - Processes problem data
   - Creates problems in Hydro
5. Redirects to problem list on success

The plugin seamlessly integrates with Hydro's existing infrastructure while providing new functionality for importing QDUOJ problems. It maintains consistency with Hydro's permission system and UI patterns while adding specific features for QDUOJ import handling.
