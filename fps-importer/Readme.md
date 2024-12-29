# FPS-Importer Plugin Overview

The FPS-Importer plugin allows importing problem sets in FPS (Free Problem Set) format into HydroOJ. It supports both single XML files and ZIP archives containing multiple FPS files.

## Key Components and Functionality

### 1. Core Imports and Their Usage

```typescript
import {
    _, AdmZip, BadRequestError, buildContent, Context, FileTooLargeError, fs, Handler,
    PERM, ProblemConfigFile, ProblemModel, ProblemType, SettingModel, SolutionModel, SystemModel, ValidationError, yaml,
} from 'hydrooj';
```

Key imports and their roles:

- `ProblemModel`: Handles database operations for problems
- `Context`: Manages application context and routing
- `SystemModel`: Handles system-wide settings
- `SolutionModel`: Manages problem solutions
- `Handler`: Base class for request handling

### 2. FpsProblemImportHandler Class

This is the main handler class that processes FPS imports:

```typescript
class FpsProblemImportHandler extends Handler {
    async get() {
        this.response.template = 'problem_import_fps.html';
    }

    async post({ domainId }) {
        if (!this.request.files.file) throw new ValidationError('file');
        // File processing logic...
    }

    async run(domainId: string, result: any) {
        if (!result?.fps) throw new BadRequestError('Selected file is not a valid FPS problemset.');
        // Problem import logic...
    }
}
```

Key methods:

1. `get()`: Renders the import page
2. `post()`: Handles file uploads
3. `run()`: Processes the FPS data

### 3. Problem Import Process

The plugin processes problems with the following steps:

```typescript
async run(domainId: string, result: any) {
    for (const p of result.fps.item) {
        // 1. Build problem content
        let content = buildContent({
            description: p.description?.[0],
            input: p.input?.[0],
            output: p.output?.[0],
            samples: p.sample_input?.map((input, i) => [input, p.sample_output[i]]),
            hint: p.hint?.[0],
            source: p.source?.join(' '),
        });

        // 2. Create problem configuration
        const config: ProblemConfigFile = {
            time: p.time_limit[0]._ + p.time_limit[0].$.unit,
            memory: p.memory_limit[0]._ + p.memory_limit[0].$.unit,
        };

        // 3. Add problem to database
        const pid = await ProblemModel.add(domainId, null, title, content, this.user._id, tags);

        // 4. Process testdata and additional files
        // ... testdata processing logic ...
    }
}
```

### 4. UI Modifications

The plugin modifies HydroOJ's UI through:

1. Template Addition:
```html
<!-- problem_import_fps.html -->
{% extends "layout/basic.html" %}
{% block content %}
<div class="row">
  <div class="medium-9 columns">
    <div class="section">
      <div class="section__body">
        <form method="post" enctype="multipart/form-data">
          <!-- File upload form -->
        </form>
      </div>
    </div>
  </div>
</div>
{% endblock %}
```

2. UI Integration:
```typescript
export async function apply(ctx: Context) {
    // Add route
    ctx.Route('problem_import_fps', '/problem/import/fps', FpsProblemImportHandler, PERM.PERM_CREATE_PROBLEM);
    
    // Add UI button
    ctx.injectUI('ProblemAdd', 'problem_import_fps', { 
        icon: 'copy', 
        text: 'From FPS File' 
    });
}
```

### 5. Settings and Configuration

The plugin adds system settings for file size limits:

```typescript
ctx.inject(['setting'], (c) => {
    c.setting.SystemSetting(
        SettingModel.Setting(
            'setting_limits', 
            'import-fps.limit', 
            64 * 1024 * 1024, 
            'text', 
            'Maximum file size for FPS problemset import'
        ),
    );
});
```

## File Structure and Dependencies

```json
{
    "name": "@hydrooj/fps-importer",
    "version": "1.5.13",
    "dependencies": {
        "decode-html": "^2.0.0",
        "xml2js": "^0.6.2"
    }
}
```

## Key Features

1. **File Format Support**:
   - Single XML files
   - ZIP archives containing multiple FPS files
   - HTML and Markdown content formats

2. **Content Processing**:
   - Problem description
   - Input/Output specifications
   - Sample cases
   - Test data
   - Additional files (images)
   - Solutions

3. **Configuration Management**:
   - Time limits
   - Memory limits
   - Remote judge settings

4. **UI Integration**:
   - Adds import button to problem management page
   - Provides dedicated import interface
   - Displays import status and errors

The plugin integrates seamlessly with HydroOJ's existing problem management system while providing specialized functionality for FPS format imports. It handles various aspects of problem importing, from file parsing to database storage, while maintaining compatibility with HydroOJ's permission system and UI conventions.
