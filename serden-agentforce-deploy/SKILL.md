---
name: serden-agentforce-deploy
description: Deploy a custom Agentforce chat widget to a React website using the Agent API (not Embedded Messaging). Covers the full path: Connected App setup, proxy server for token minting, useAgentforce hook, streaming React component, and Vite proxy wiring. Use when building a custom Agentforce chat UI, integrating the Agent API into a web app, or setting up client credentials flow for Agentforce.
---

# Deploying Agentforce to a React Website (Agent API)

End-to-end recipe for embedding Agentforce into a standalone React/Vite site using the Agent API directly — not Embedded Messaging.

## Architecture

```
Browser (React)
  └─ /api/agent/* (Vite dev proxy)
       └─ Express proxy server :3001
            └─ api.salesforce.com/einstein/ai-agent/v1
```

Secrets never reach the browser. The proxy mints tokens and forwards requests.

---

## Step 1 — Get the Agent ID

```bash
sf data query --query "SELECT Id, DeveloperName FROM BotDefinition WHERE DeveloperName = 'Your_Agent_Name'"
```

Note the `Id` — this is `AGENT_ID` used in all API calls.

---

## Step 2 — Create a Connected App

Deploy via metadata. The `ExternalClientApplication` type is **not yet in the sf CLI registry** — use `ConnectedApp` instead.

```xml
<!-- force-app/main/default/connectedApps/MyApp_Agent_API.connectedApp-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ConnectedApp xmlns="http://soap.sforce.com/2006/04/metadata">
    <contactEmail>admin@yourorg.com</contactEmail>
    <label>My App Agent API</label>
    <oauthConfig>
        <callbackUrl>https://login.salesforce.com/services/oauth2/success</callbackUrl>
        <isClientCredentialEnabled>true</isClientCredentialEnabled>
        <scopes>Api</scopes>
        <scopes>RefreshToken</scopes>
        <scopes>Chatbot</scopes>
    </oauthConfig>
    <oauthPolicy>
        <ipRelaxation>ENFORCE</ipRelaxation>
        <refreshTokenPolicy>infinite</refreshTokenPolicy>
    </oauthPolicy>
</ConnectedApp>
```

**Known invalid scope values** (will fail deployment):
- `Sfap` — not a valid enum; add `sfap_api` via the Setup UI instead
- `SfapApi` — also invalid
- `issueJwtBasedAccessToken` — XML element doesn't exist on `ConnectedAppOauthConfig`
- `requireSecretForRefreshToken` — also invalid in this context

```bash
sf project deploy start --source-dir force-app/main/default/connectedApps
```

---

## Step 3 — Configure the Connected App in Setup UI

Three things must be done in Setup that cannot be done via metadata:

1. **Add `sfap_api` scope** — Edit the Connected App → Selected OAuth Scopes → add "Access the Salesforce API Platform (sfap_api)"
2. **Enable JWT tokens** — Check "Issue JSON Web Token (JWT)-based access tokens for named users"
3. **Set Client Credentials Run As user** — Manage → Edit Policies → Client Credentials Flow → set "Run As" to the Einstein Agent User (e.g. `pasha_sales_agent@00DJ...ext`)

> Without all three, the Agent API returns 404 even with a valid token.

**Verify the token is correct** before wiring into code:

```bash
curl -s -X POST "https://YOUR_DOMAIN.my.salesforce.com/services/oauth2/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=CONSUMER_KEY&client_secret=CONSUMER_SECRET" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print('scope:', d.get('scope','')); print('token_format:', d.get('token_format',''))"
```

Expected output: `scope: sfap_api chatbot_api api` and `token_format: jwt`.
If `token_format` is empty, the JWT checkbox is not checked.
If the scope is missing `sfap_api`, the Agent API will return 404.

---

## Step 4 — Proxy Server

Create `web/proxy-server.mjs`. Key points:

- Use `await getAccessToken()` — it's async; forgetting `await` causes silent 404s
- Cache the token (JWT tokens are valid ~1 hour); refresh 5 min early
- The Agent API base is `https://api.salesforce.com/einstein/ai-agent/v1` — not the org URL
- `bypassUser: true` in session creation uses the agent's assigned user, not the token user

```js
// Token cache pattern
let cachedToken = null;
let tokenExpiresAt = 0;

async function getAccessToken() {
  if (cachedToken && Date.now() < tokenExpiresAt) return cachedToken;
  const res = await fetch(`${SF_INSTANCE_URL}/services/oauth2/token`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ grant_type: 'client_credentials', client_id: KEY, client_secret: SECRET }),
  });
  const data = await res.json();
  cachedToken = data.access_token;
  tokenExpiresAt = Date.now() + 55 * 60 * 1000;
  return cachedToken;
}
```

**Debugging proxy 404s:** Add a request logger middleware early. If the route logs but returns 404, the upstream API call is failing — log `upstream.status` and `upstream body` explicitly. The most common cause is missing `await` on `getAccessToken()`.

---

## Step 5 — Vite Proxy Config

```ts
// vite.config.ts
server: {
  proxy: {
    '/api/agent': {
      target: 'http://localhost:3001',
      rewrite: (path) => path.replace(/^\/api\/agent/, '/agent'),
      changeOrigin: true,
      secure: false,
    },
  },
}
```

Do **not** set `https: true` alongside `basicSsl()` plugin — they conflict and cause a TypeScript build error.

---

## Step 6 — React Hook Pattern

The hook (`useAgentforce`) manages session lifecycle:

- Call `startSession()` on first chat open — the session start response includes the welcome message
- Use the streaming endpoint (`/messages/stream`) for all messages — SSE with `TextChunk` → `Inform` → `EndOfTurn` events
- `mutable` agent variables are **not** auto-populated from conversation; the agent must use `@utils.setVariables` (`capture_contact` action) to store name/email into variables before calling Flows

```ts
// SSE parsing pattern
const processLine = (line: string) => {
  if (!line.startsWith('data: ')) return;
  const evt = JSON.parse(line.slice(6));
  if (evt.message.type === 'TextChunk') appendChunk(evt.message.message);
  if (evt.message.type === 'Inform')    setFinalText(evt.message.message);
  if (evt.message.type === 'EndOfTurn') markDone();
};
```

---

## Step 7 — Start Everything

```json
// package.json scripts
"dev:proxy": "node proxy-server.mjs",
"dev:all": "concurrently \"npm run dev:proxy\" \"npm run dev\""
```

```bash
npm install concurrently
npm run dev:all
```

---

## Key Learnings Summary

| Area | Learning |
|---|---|
| Connected App metadata | `ExternalClientApplication` not in sf CLI registry — use `ConnectedApp` |
| OAuth scopes | `sfap_api` and JWT checkbox must be added via Setup UI, not XML |
| Client credentials | "Run As" user must be set in Policies, not the app definition |
| Token validation | Always verify `token_format: jwt` and `sfap_api` in scope before wiring to code |
| Proxy async | `getAccessToken()` is async — missing `await` causes silent 404, not an error |
| Agent variables | `mutable` variables require explicit `@utils.setVariables` action — LLM does not auto-populate them |
| `capture_contact` pattern | Use `@utils.setVariables` with `with field = ...` syntax; reference as `{!@actions.capture_contact}` in instructions |
| Vite + basicSsl | Do not set `https: true` in server config alongside the `basicSsl()` plugin |
| Express 5 | All route handlers must `await` async calls; unhandled rejections are caught but may return unexpected status codes |
| Agent API 404 | Almost always means: wrong agent ID, missing `sfap_api` scope, non-JWT token, or wrong `instanceConfig.endpoint` |
