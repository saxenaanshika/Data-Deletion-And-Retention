# Instructions

- Following Playwright test failed.
- Explain why, be concise, respect Playwright best practices.
- Provide a snippet of code with the fix, if possible.

# Test info

- Name: data-privacy/do-not-sell.spec.ts >> Do Not Sell (DSAR_DONOTSELL) API Tests >> POST DNS-032: submitting the same dsarRequestId twice is idempotent — returns same orchestratorRequestId
- Location: tests/data-privacy/do-not-sell.spec.ts:785:7

# Error details

```
Error: Missing OAuth2 config. Set OAUTH_TOKEN_URL, OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET in .env
```

# Test source

```ts
  1   | import * as dotenv from 'dotenv';
  2   | import * as path from 'path';
  3   | 
  4   | dotenv.config();
  5   | // Load pre-fetched tokens written by globalSetup (no-op if file doesn't exist)
  6   | dotenv.config({ path: path.resolve(process.cwd(), '.env.tokens'), override: false });
  7   | 
  8   | interface TokenResponse {
  9   |   access_token: string;
  10  |   token_type: string;
  11  |   expires_in: number;
  12  | }
  13  | 
  14  | interface CachedToken {
  15  |   token: string;
  16  |   expiresAt: number;
  17  | }
  18  | 
  19  | const tokenCache: Map<string, CachedToken> = new Map();
  20  | 
  21  | /**
  22  |  * Fetch OAuth2 token using Client Credentials flow.
  23  |  * Caches the token until it expires.
  24  |  *
  25  |  * Usage: const token = await getOAuthToken();
  26  |  *
  27  |  * Required .env variables:
  28  |  *   OAUTH_TOKEN_URL     - Token endpoint URL
  29  |  *   OAUTH_CLIENT_ID     - Client ID
  30  |  *   OAUTH_CLIENT_SECRET - Client secret
  31  |  *   OAUTH_SCOPE         - (optional) Space-separated scopes
  32  |  */
  33  | export async function getOAuthToken(
  34  |   tokenUrl?: string,
  35  |   clientId?: string,
  36  |   clientSecret?: string,
  37  |   scope?: string
  38  | ): Promise<string> {
  39  |   const url = tokenUrl || process.env.OAUTH_TOKEN_URL || '';
  40  |   const id = clientId || process.env.OAUTH_CLIENT_ID || '';
  41  |   const secret = clientSecret || process.env.OAUTH_CLIENT_SECRET || '';
  42  |   const oauthScope = scope || process.env.OAUTH_SCOPE || '';
  43  | 
  44  |   if (!url || !id || !secret) {
> 45  |     throw new Error(
      |           ^ Error: Missing OAuth2 config. Set OAUTH_TOKEN_URL, OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET in .env
  46  |       'Missing OAuth2 config. Set OAUTH_TOKEN_URL, OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET in .env'
  47  |     );
  48  |   }
  49  | 
  50  |   // Check if globalSetup pre-fetched this token — use it directly if available
  51  |   const prefix = tokenUrl ? '' : 'OAUTH_';
  52  |   const preFetchedKey = tokenUrl
  53  |     ? null
  54  |     : clientId === process.env.BATCHES_OAUTH_CLIENT_ID ? 'BATCHES_OAUTH_TOKEN'
  55  |     : clientId === process.env.UAP_OAUTH_CLIENT_ID    ? 'UAP_OAUTH_TOKEN'
  56  |     : 'OAUTH_TOKEN';
  57  |   if (preFetchedKey && process.env[preFetchedKey]) {
  58  |     return process.env[preFetchedKey]!;
  59  |   }
  60  | 
  61  |   const cacheKey = `${url}::${id}`;
  62  |   const cached = tokenCache.get(cacheKey);
  63  | 
  64  |   // Return cached token if still valid (with 30s buffer)
  65  |   if (cached && Date.now() < cached.expiresAt - 30_000) {
  66  |     return cached.token;
  67  |   }
  68  | 
  69  |   // Build form body — Cognito requires credentials in Basic Auth header
  70  |   const params = new URLSearchParams({ grant_type: 'client_credentials' });
  71  | 
  72  |   if (oauthScope) {
  73  |     params.append('scope', oauthScope);
  74  |   }
  75  | 
  76  |   const credentials = Buffer.from(`${id}:${secret}`).toString('base64');
  77  | 
  78  |   const response = await fetch(url, {
  79  |     method: 'POST',
  80  |     headers: {
  81  |       'Content-Type': 'application/x-www-form-urlencoded',
  82  |       'Authorization': `Basic ${credentials}`,
  83  |     },
  84  |     body: params.toString(),
  85  |   });
  86  | 
  87  |   if (!response.ok) {
  88  |     const error = await response.text();
  89  |     throw new Error(`OAuth2 token request failed [${response.status}]: ${error}`);
  90  |   }
  91  | 
  92  |   const data = (await response.json()) as TokenResponse;
  93  | 
  94  |   // Cache the token
  95  |   tokenCache.set(cacheKey, {
  96  |     token: data.access_token,
  97  |     expiresAt: Date.now() + data.expires_in * 1000,
  98  |   });
  99  | 
  100 |   return data.access_token;
  101 | }
  102 | 
  103 | /**
  104 |  * Returns Authorization header value: "Bearer <token>"
  105 |  * Use this directly in restClient or RequestBuilder
  106 |  */
  107 | export async function getBearerHeader(
  108 |   tokenUrl?: string,
  109 |   clientId?: string,
  110 |   clientSecret?: string,
  111 |   scope?: string
  112 | ): Promise<string> {
  113 |   const token = await getOAuthToken(tokenUrl, clientId, clientSecret, scope);
  114 |   return `Bearer ${token}`;
  115 | }
  116 | 
  117 | /**
  118 |  * Clears the token cache (useful between test suites)
  119 |  */
  120 | export function clearTokenCache(): void {
  121 |   tokenCache.clear();
  122 | }
  123 | 
  124 | /**
  125 |  * Shorthand for CDPR Service API token (uses OAUTH_* env vars)
  126 |  * Returns "Bearer <token>"
  127 |  */
  128 | export async function getToken(): Promise<string> {
  129 |   return getBearerHeader();
  130 | }
  131 | 
  132 | /**
  133 |  * Token for the Batches service (uses BATCHES_OAUTH_* env vars — separate Cognito pool)
  134 |  * Returns "Bearer <token>"
  135 |  */
  136 | export async function getBatchesToken(): Promise<string> {
  137 |   return getBearerHeader(
  138 |     process.env.BATCHES_OAUTH_TOKEN_URL,
  139 |     process.env.BATCHES_OAUTH_CLIENT_ID,
  140 |     process.env.BATCHES_OAUTH_CLIENT_SECRET,
  141 |   );
  142 | }
  143 | 
```