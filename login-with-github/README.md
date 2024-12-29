# 1. Plugin Overview

The login-with-github plugin enables GitHub OAuth authentication in Hydro OJ. It allows users to log in using their GitHub accounts by implementing OAuth 2.0 flow with GitHub's authentication services.

# 2. Imported Packages Usage

```typescript
import {
    Context, ForbiddenError, Handler, superagent, SystemModel,
    TokenModel, UserFacingError, ValidationError,
} from 'hydrooj';
```

Let's examine how each imported package is utilized:

## 2.1 Context
The Context object is used to register the OAuth provider and load i18n translations:

```typescript
export function apply(ctx: Context) {
    ctx.provideModule('oauth', 'github', {
        text: 'Login with Github',
        callback,
        get,
    });
    ctx.i18n.load('zh', {
        'Login With Github': 'ä½¿ç¨ Github ç»å½',
    });
}
```

## 2.2 Error Handling Classes
The plugin uses various error classes for different scenarios:

```typescript
// ValidationError - Used for invalid tokens
if (!s) throw new ValidationError('token');

// ForbiddenError - Used when email verification is missing
if (!ret.email) throw new ForbiddenError("You don't have a verified email.");

// UserFacingError - Used for OAuth errors
if (res.body.error) {
    throw new UserFacingError(
        res.body.error, 
        res.body.error_description, 
        res.body.error_uri,
    );
}
```

## 2.3 Handler
The Handler class is extended to handle OAuth requests:

```typescript
async function get(this: Handler) {
    const [appid, [state]] = await Promise.all([
        SystemModel.get('login-with-github.id'),
        TokenModel.add(TokenModel.TYPE_OAUTH, 600, { redirect: this.request.referer }),
    ]);
    this.response.redirect = `https://github.com/login/oauth/authorize?client_id=${appid}&state=${state}&scope=read:user,user:email`;
}
```

## 2.4 SystemModel
Used to store and retrieve GitHub OAuth configuration:

```typescript
const [appid, secret, endpoint, url] = await SystemModel.getMany([
    'login-with-github.id',
    'login-with-github.secret',
    'login-with-github.endpoint',
    'server.url',
]);
```

## 2.5 TokenModel
Manages OAuth tokens and states:

```typescript
// Creating OAuth token
const [state] = await TokenModel.add(TokenModel.TYPE_OAUTH, 600, { redirect: this.request.referer });

// Retrieving and validating token
const s = await TokenModel.get(state, TokenModel.TYPE_OAUTH);

// Deleting used token
await TokenModel.del(s._id, TokenModel.TYPE_OAUTH);
```

# 3. Plugin Components

## 3.1 Configuration Interface
```typescript
declare module 'hydrooj' {
    interface SystemKeys {
        'login-with-github.id': string;
        'login-with-github.secret': string;
        'login-with-github.endpoint': string;
    }
}
```

## 3.2 OAuth Flow Functions

### Get Function
Initiates the OAuth flow:
```typescript
async function get(this: Handler) {
    const [appid, [state]] = await Promise.all([
        SystemModel.get('login-with-github.id'),
        TokenModel.add(TokenModel.TYPE_OAUTH, 600, { redirect: this.request.referer }),
    ]);
    this.response.redirect = `https://github.com/login/oauth/authorize?client_id=${appid}&state=${state}&scope=read:user,user:email`;
}
```

### Callback Function
Handles OAuth callback and user data:
```typescript
async function callback({ state, code }) {
    // Get OAuth tokens and exchange for access token
    const res = await superagent.post(`${endpoint || 'https://github.com'}/login/oauth/access_token`)
        .send({
            client_id: appid,
            client_secret: secret,
            code,
            redirect_uri: `${url}oauth/github/callback`,
            state,
        });

    // Fetch user information
    const userInfo = await superagent.get(`${endpoint ? `${endpoint}/api` : 'https://api.github.com'}/user`)
        .set('Authorization', `token ${t}`);

    // Create user object
    const ret = {
        _id: `${userInfo.body.id}@github.local`,
        email: userInfo.body.email,
        bio: userInfo.body.bio,
        uname: [userInfo.body.name, userInfo.body.login].filter((i) => i),
        avatar: `github:${userInfo.body.login}`,
    };
}
```

# 4. UI Modifications

The plugin modifies Hydro OJ's UI by adding a GitHub login option. This is accomplished through the `provideModule` function:

```typescript
ctx.provideModule('oauth', 'github', {
    text: 'Login with Github',
    callback,
    get,
});
```

## 4.1 Affected Pages
- Login page: Adds a "Login with Github" button
- OAuth callback page: Handles the GitHub OAuth response

## 4.2 UI Functionality
When users click the "Login with Github" button:
1. They are redirected to GitHub's authorization page
2. After authorizing, they're redirected back to Hydro OJ
3. The plugin creates/updates their account using GitHub information
4. Users are logged in automatically

# 5. Configuration

The plugin requires configuration in `setting.yaml`:
```yaml
id:
  type: text
  category: system
  description: GitHub OAuth Application ID
secret:
  type: text
  category: system
  description: GitHub OAuth Secret
endpoint:
  type: text
  category: system
  description: GitHub API Endpoint
```

This comprehensive integration allows Hydro OJ users to seamlessly authenticate using their GitHub accounts while maintaining security through proper token management and error handling.
