# GitHub OAuth Authentication Upgrade Manual

* **Source**: [GitHub Copilot SDK - GitHub OAuth Setup](https://github.com/github/copilot-sdk/blob/main/docs/guides/setup/github-oauth.md)
* **Goal**: Upgrade n8n-nodes-copilot authentication to support standard GitHub OAuth flow for Copilot SDK.

## Overview

This guide outlines the steps to implement proper GitHub OAuth authentication. This replaces simpler personal access token methods with a robust flow suitable for multi-user applications and organization/enterprise environments.

## Step 1: GitHub OAuth App Setup

Before implementing code, a GitHub OAuth App must be configured.

1.  Navigate to **GitHub Settings** -> **Developer Settings** -> **OAuth Apps** -> **New OAuth App**.
    *   *For Organizations*: Organization Settings -> Developer Settings -> OAuth Apps.
2.  **Configuration**:
    *   **Application Name**: `n8n-nodes-copilot` (or specific integration name).
    *   **Homepage URL**: Your n8n instance URL or application homepage.
    *   **Authorization Callback URL**: The endpoint where GitHub will redirect back with the `code`.
        *   Example: `https://your-n8n-instance.com/rest/oauth2-credential/callback` (Verify n8n specific callback).
3.  **Credentials**: Note the **Client ID** and generate a **Client Secret**.

> **Note**: A GitHub App can also be used and offers finer-grained permissions, but the OAuth App is simpler for this specific flow.

## Step 2: Implement Token Exchange

The application must exchange the temporary authorization `code` for a user access token.

### Server-Side Exchange Logic

```typescript
// Example Implementation
async function handleOAuthCallback(code: string): Promise<string> {
    const response = await fetch("https://github.com/login/oauth/access_token", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        body: JSON.stringify({
            client_id: process.env.GITHUB_CLIENT_ID,
            client_secret: process.env.GITHUB_CLIENT_SECRET,
            code,
        }),
    });

    const data = await response.json();
    // Expected format: { access_token: "gho_...", scope: "...", token_type: "bearer" }
    return data.access_token;
}
```

## Step 3: SDK Integration

Pass the retrieved token to the Copilot SDK client.

### Client Initialization

```typescript
import { CopilotClient } from "@github/copilot-sdk";

function createClientForUser(userToken: string): CopilotClient {
    return new CopilotClient({
        githubToken: userToken,
        useLoggedInUser: false, // Critical: Disable CLI fallback for pure OAuth flow
    });
}

// Usage Example
const userToken = "gho_example_token"; // Retrieved from OAuth flow
const client = createClientForUser(userToken);

const session = await client.createSession({
    sessionId: `user-session-${userId}`,
    model: "gpt-4.1", // Ensure model is available to the user
});
```

## Supported Token Types

Ensure the system validates and accepts the following token prefixes:

| Prefix | Type | Status |
| :--- | :--- | :--- |
| `gho_` | OAuth user access token | ✅ Recommended |
| `ghu_` | GitHub App user access token | ✅ Supported |
| `github_pat_` | Fine-grained PAT | ✅ Supported |
| `ghp_` | Classic PAT | ❌ Deprecated / Discouraged |

## Enterprise & Organization Verification

For enterprise use cases, verify organization membership after authentication.

```typescript
async function verifyOrgMembership(token: string, requiredOrg: string): Promise<boolean> {
    const response = await fetch("https://api.github.com/user/orgs", {
        headers: { Authorization: `Bearer ${token}` },
    });
    const orgs = await response.json();
    return orgs.some((org: any) => org.login === requiredOrg);
}
```

## Token Lifecycle Management

**Important**: The SDK does **not** manage token storage, refresh, or expiration. The n8n node/application is responsible for:

1.  **Storage**: Securely storing the `access_token` and `refresh_token` (if applicable).
2.  **Refresh**: Using the `refresh_token` to get a new `access_token` when it expires.
3.  **Expiration Handling**: Detecting 401 errors and prompting re-authentication or auto-refreshing.

## Next Steps for Implementation

1.  Review existing `GitHubCopilotApi.credentials.ts` to see how it currently handles tokens.
2.  Plan the migration from simple input fields to n8n's OAuth2 credential type if possible, or implement a custom OAuth flow helper.
3.  Update the `GitHubCopilotChatAPI` node to initialize the SDK with the new `createClientForUser` pattern.
