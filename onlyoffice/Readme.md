# 1. Overview of OnlyOffice Plugin

The OnlyOffice plugin integrates document viewing capabilities into HydroOJ, allowing users to view documents (like PDFs) directly within the platform. The plugin primarily:

- Handles document viewing through OnlyOffice Document Server
- Generates JWT tokens for secure document access
- Provides PDF viewing capabilities
- Supports multiple languages

# 2. Imported Packages and Their Usage

## Context
From `hydrooj/src/context.ts`, Context provides the application framework:

```typescript
export interface Context extends cordis.Context {
    loader: Loader;
    addScript: typeof addScript;
    provideModule: typeof provideModule;
    injectUI: typeof inject;
}
```

The plugin uses Context to:
- Register routes (`ctx.Route`)
- Inject i18n translations (`ctx.inject(['i18n'])`)
- Handle plugin lifecycle (`ctx.on('dispose')`)

## Handler
From the Hydro framework, Handler is used to create HTTP endpoint handlers:

```typescript
class OnlyofficeJWTHandler extends Handler {
    noCheckPermView = true;
    notUsage = true;

    async get({ url }) {
        // Handle JWT token generation
    }
}
```

## SystemModel 
From `hydrooj/src/model/system.ts`, SystemModel manages system settings:

```typescript
// Usage in plugin
SystemModel.get('onlyoffice.api')
SystemModel.get('onlyoffice.jwtsecret')
SystemModel.get('onlyoffice.allowDownload')
```

## UiContextBase
From `hydrooj/src/service/layers/base.ts`, UiContextBase provides UI context:

```typescript
declare module 'hydrooj' {
    interface UiContextBase {
        onlyofficeApi?: string;
    }
}
```

# 3. Plugin Components

## OnlyofficeJWTHandler
Main handler for generating JWT tokens:

```typescript
class OnlyofficeJWTHandler extends Handler {
    async get({ url }) {
        // Handle external signing
        if (SystemModel.get('onlyoffice.externalSign')) {
            const res = await superagent.get(SystemModel.get('onlyoffice.externalSign')).query({ url });
            this.response.body = res.body;
            return;
        }

        // Generate JWT token
        const payload = {
            document: {
                fileType: path.split('.').pop(),
                key: Math.random().toString(36).substring(2),
                title: decodeURIComponent(path.split('/').pop()),
                url,
                permissions: {
                    comment: false,
                    copy: allowDownload,
                    download: allowDownload,
                    edit: false,
                    print: allowDownload,
                }
            },
            editorConfig: {
                lang: this.user.viewLang?.includes('_') 
                    ? this.user.viewLang.split('_')[0] 
                    : this.user.viewLang || 'zh',
                mode: 'view',
                user: {
                    group: 'Hydro',
                    id: this.user._id.toString(),
                    name: this.user.uname,
                }
            }
        };
        
        const token = sign(payload, SystemModel.get('onlyoffice.jwtsecret'));
        this.response.body = { ...payload, token };
    }
}
```

## UI Modifications

The plugin modifies the UI through:

1. Adding OnlyOffice API endpoint to UiContextBase:
```typescript
Object.defineProperty(UiContextBase, 'onlyofficeApi', {
    configurable: true,
    enumerable: true,
    get() {
        return SystemModel.get('onlyoffice.api');
    },
});
```

2. Providing PDF viewer module when enabled:
```typescript
if (SystemModel.get('onlyoffice.pdf')) {
    ctx.provideModule('richmedia', 'pdf', {
        get(service, src, md) {
            return `<div data-${service}>${md.utils.escapeHtml(src)}</div>`;
        },
    });
}
```

## Internationalization Support

The plugin provides translations for multiple languages:

```typescript
ctx.inject(['i18n'], (c) => {
    c.i18n.load('en', {
        'onlyoffice.not_configured': 'Onlyoffice API not configured.',
        'onlyoffice.initialize_fail': 'Failed to initialize onlyoffice: {0}',
    });
    c.i18n.load('zh', {
        'onlyoffice.not_configured': 'Onlyoffice 未配置。自行安装 onlyoffice 软件或者使用外部提供的 API。',
        'onlyoffice.initialize_fail': '初始化 onlyoffice 失败: {0}',
    });
});
```

# 4. UI Changes and Affected Pages

The plugin affects pages that display PDF documents or other supported document types. When enabled:

1. PDF files are rendered using OnlyOffice viewer instead of browser's default PDF viewer
2. The viewer provides features based on configuration:
   - Document viewing
   - Download (if allowed)
   - Printing (if allowed)
   - Copy text (if allowed)

The UI changes are implemented through the `richmedia` module when PDF support is enabled:

```typescript
if (SystemModel.get('onlyoffice.pdf')) {
    ctx.provideModule('richmedia', 'pdf', {
        get(service, src, md) {
            return `<div data-${service}>${md.utils.escapeHtml(src)}</div>`;
        },
    });
}
```

This creates a container div that will be populated with the OnlyOffice viewer when the page loads.

# 5. Configuration Options

The plugin uses several system settings:

```typescript
// Example configuration
{
    'onlyoffice.api': 'https://document-server/api',
    'onlyoffice.jwtsecret': 'your-secret-key',
    'onlyoffice.allowDownload': true,
    'onlyoffice.pdf': true,
    'onlyoffice.externalSign': 'https://external-signing-service'
}
```

These settings control the plugin's behavior and features, including document server connection, security, and user permissions.

The plugin integrates seamlessly with HydroOJ's existing infrastructure while providing enhanced document viewing capabilities through OnlyOffice's document server.
