1. Overview:
The login-with-google plugin enables Google OAuth authentication in Hydro OJ, allowing users to sign in using their Google accounts. It adds a "Login with Google" button to the login interface.

2. Imported Packages Usage:

```typescript
import {
    Context, Handler, superagent, SystemModel, TokenModel, UserFacingError,
} from 'hydrooj';
```

- Context: Provides plugin infrastructure and lifecycle management
- Handler: Handles HTTP requests/responses for OAuth endpoints 
- superagent: Makes HTTP requests to Google's OAuth APIs
- SystemModel: Stores Google OAuth credentials (client ID/secret)
- TokenModel: Manages OAuth state tokens for security
- UserFacingError: Handles user-facing error messages

3. Core Components:

a) OAuth Handler Functions:

```typescript
// Initiates Google OAuth flow
async function get(this: Handler) {
    const [appid, url, [state]] = await Promise.all([
        SystemModel.get('login-with-google.id'),
        SystemModel.get('server.url'), 
        TokenModel.add(TokenModel.TYPE_OAUTH, 600, { redirect: this.request.referer }),
    ]);
    
    // Redirects to Google OAuth consent screen
    this.response.redirect = `https://accounts.google.com/o/oauth2/v2/auth?client_id=${appid}&response_type=code...`;
}

// Handles OAuth callback
async function callback(this: Handler, { state, code, error }) {
    // Validates state token
    const [[appid, secret, url], s] = await Promise.all([
        SystemModel.getMany(['login-with-google.id', 'login-with-google.secret', 'server.url']),
        TokenModel.get(state, TokenModel.TYPE_OAUTH),
    ]);

    // Exchanges code for tokens
    const res = await superagent.post('https://oauth2.googleapis.com/token')
        .send({
            client_id: appid,
            client_secret: secret,
            code,
            grant_type: 'authorization_code',
            redirect_uri: `${url}oauth/google/callback`,
        });

    // Processes user info
    const payload = decodeJWT(res.body.id_token).payload;
    return {
        _id: payload.email,
        email: payload.email,
        uname: [payload.given_name, payload.name, payload.family_name],
        viewLang: payload.locale.replace('-', '_'),
    };
}
```

b) Plugin Registration:

```typescript
export function apply(ctx: Context) {
    // Registers OAuth provider
    ctx.provideModule('oauth', 'google', {
        text: 'Login with Google',
        callback,
        get,
    });
    
    // Adds translations
    ctx.i18n.load('zh', {
        'Login With Google': 'ä½¿ç¨ Google ç»å½',
    });
}
```

4. UI Modifications:

The plugin modifies the login page UI by utilizing Hydro's module system:

```typescript
// In apply() function
ctx.provideModule('oauth', 'google', {
    text: 'Login with Google', // Button text
    callback,
    get,
});
```

This adds a "Login with Google" button to the login page at `/login`. When clicked:

1. Redirects to Google consent screen
2. User authorizes access
3. Returns to callback URL
4. Creates/updates user account with Google info

5. Token Management:

The plugin uses TokenModel for secure OAuth state management:

```typescript
// Creates temporary state token
TokenModel.add(TokenModel.TYPE_OAUTH, 600, { redirect: this.request.referer })

// Validates token in callback
TokenModel.get(state, TokenModel.TYPE_OAUTH)
```

From TokenModel (hydro repository):
```typescript
class TokenModel {
    static TYPE_OAUTH = 4;
    
    static async add(tokenType: number, expireSeconds: number, data: any) {
        const now = new Date();
        const payload = {
            ...data,
            tokenType,
            createAt: now,
            expireAt: new Date(now.getTime() + expireSeconds * 1000),
        };
        await TokenModel.coll.insertOne(payload);
        return [id, payload];
    }
}
```

6. System Configuration:

The plugin extends SystemModel to store OAuth credentials:

```typescript
declare module 'hydrooj' {
    interface SystemKeys {
        'login-with-google.id': string,
        'login-with-google.secret': string,
    }
}
```

These credentials are managed through Hydro's system settings interface.

7. Error Handling:

Uses UserFacingError for user-friendly error messages:

```typescript
if (error) throw new UserFacingError(error);
```

The plugin integrates deeply with Hydro's infrastructure while maintaining clean separation through the module system. It primarily affects the login page UI but uses core Hydro services for user management, session handling, and configuration.

All code examples are referenced from:
- `/login-with-google/index.ts`
- `/packages/hydrooj/src/model/token.ts`
- `/packages/hydrooj/src/context.ts`
- `/packages/hydrooj/src/model/system.ts`
