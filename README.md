# GitHub MCP Kong Gateway Configuration

This repository contains Kong Gateway configuration for proxying and securing GitHub's Model Context Protocol (MCP) API with OAuth 2.0 authentication and fine-grained access controls.

## Overview

This Kong configuration acts as a secure gateway between Claude Code (or other MCP clients) and GitHub's MCP API. It validates GitHub OAuth tokens, correlates users with Kong consumers, and enforces tool-level access controls.

## Architecture

```
MCP Client (Claude Code)
    ↓ (Bearer token)
Kong Gateway (this config)
    ↓ (token validation)
GitHub User API
    ↓ (team membership)
GitHub Teams API
    ↓ (team → group mapping)
Kong Consumer Groups
    ↓ (ACL enforcement)
GitHub MCP API
```

## Features

### Authentication & Authorization
- **OAuth 2.0 Bearer Token Validation**: Validates GitHub OAuth tokens against GitHub's User API
- **Team-Based Access Control**: Automatically maps GitHub team memberships to Kong consumer groups
- **Consumer Correlation**: Optionally maps GitHub users to Kong consumers based on username
- **Dynamic Group Assignment**: Users inherit permissions from their GitHub team memberships in real-time
- **Tool-Level ACLs**: Fine-grained access control for individual MCP tools

### Security
- Proper OAuth 2.0 WWW-Authenticate challenge responses
- Token validation with GitHub API before proxying requests
- Consumer group-based authorization
- Comprehensive logging for auditing and debugging

## Configuration Files

### `kong/deck/kong.yaml`
Main Kong declarative configuration defining:
- **Service**: `github-mcp-service` proxying to `https://api.githubcopilot.com/mcp/`
- **Routes**:
  - `/.well-known/oauth-protected-resource/github-mcp-metadata`: OAuth metadata endpoint
  - `/github-mcp`: Main MCP proxy route
- **Consumer Groups**: `mcp-users`, `mcp-developers`
- **Consumers**: Example consumer `pverge2112` mapped to GitHub user

### `kong/deck/_plugins_config.yaml`
Plugin configurations including:

#### `mcp-github-pre-function`
Validates that requests include an Authorization header and returns proper OAuth challenge if missing.

#### `mcp-github-post-function`
Core authentication logic:
1. Extracts Bearer token from Authorization header
2. Validates token with GitHub User API
3. Retrieves GitHub user information
4. Correlates GitHub username to Kong consumer (if consumer exists)
5. Fetches user's GitHub teams via `/user/teams` endpoint
6. Maps GitHub team slugs to Kong consumer groups (direct name match)
7. Authenticates with matched consumer groups or falls back to consumer-based groups
8. Adds consumer context and group information for downstream processing

#### `mcp-github-ai-mcp-proxy`
Configures the AI MCP proxy plugin:
- **Mode**: `passthrough-listener`
- **Default ACL**: Allows `mcp-users` and `mcp-developers` groups access to tools
- **Tool-Specific ACLs**:
  - `create_repository`: Denied for `mcp-developers` group
- **Logging**: Enables payload, statistics, and audit logging

#### `mcp-github-request-termination`
Returns OAuth metadata for discovery by MCP clients.

### `kong/deck/_info.yaml`
Deck configuration with tag selector `github-mcp-test` for deployment management.

## Access Control Model

### GitHub Teams to Consumer Groups

The gateway automatically maps GitHub team membership to Kong consumer groups for dynamic, team-based access control:

**How it works:**
1. After validating the GitHub token, the gateway fetches the user's team memberships via the GitHub `/user/teams` API
2. For each team the user belongs to, it looks for a Kong consumer group with a matching name (using the team's `slug` field)
3. All matching consumer groups are automatically assigned to the request
4. If no teams match or the teams API is unavailable, it falls back to consumer-based group assignment

**Example Mapping:**
```
GitHub Team Slug          →  Kong Consumer Group
──────────────────────────────────────────────────
engineering               →  engineering
mcp-users                 →  mcp-users
platform-team             →  platform-team
```

**Token Requirements:**
- The GitHub OAuth token must have the `read:org` scope to access team memberships
- If SSO is enabled on the GitHub organization, the token may need SSO authorization
- Without proper scopes, the gateway logs a warning and falls back to consumer-based groups

**Benefits:**
- **No manual group assignment**: Users automatically inherit permissions from their GitHub team memberships
- **Centralized management**: Team changes in GitHub immediately reflect in Kong access controls
- **Multi-team support**: Users in multiple teams receive all corresponding consumer group permissions

### Consumer Groups
Kong consumer groups control access to MCP tools. Example groups:
- **mcp-users**: Standard users with default tool access
- **mcp-developers**: Developer users with restricted access to certain tools
- **engineering**: Custom team-based group for engineering team members

### Tool-Level Permissions
Individual MCP tools can have custom ACLs. Example:
- `create_repository`: Denied to `mcp-developers`, allowed to `mcp-users`

This allows fine-grained control over which users can perform specific GitHub operations.

## Adding New Consumers

### Team-Based Access (Recommended)

With GitHub team-based authentication, **users do not need to be manually added as consumers**. Simply:

1. Create a Kong consumer group matching your GitHub team slug:
   ```yaml
   consumer_groups:
     - name: engineering  # matches GitHub team slug
   ```

2. Configure tool ACLs for that group in the MCP proxy plugin

3. Add users to the GitHub team - they'll automatically get the corresponding permissions

### Manual Consumer Addition (Optional)

To add specific consumers or use consumer-based groups as fallback:

```yaml
consumers:
  - username: github_username
    custom_id: github:github_username
    groups:
      - name: mcp-users
```

**Note**: The `username` should match the GitHub user's login name. Consumers are optional when using team-based authentication but provide a fallback if team lookup fails.

## Deployment

This configuration uses [deck](https://docs.konghq.com/deck/) for declarative configuration management:

```bash
# Validate configuration
deck validate --state kong/deck

# Sync to Kong Gateway
deck sync --state kong/deck

# Diff against running Kong
deck diff --state kong/deck
```

## How It Works

### Request Flow

1. **Client Request**: MCP client sends request to `/github-mcp` with `Authorization: Bearer <github_token>` header

2. **Pre-Function Check**: Validates Authorization header exists, returns 401 with OAuth challenge if missing

3. **Post-Function Authentication**:
   - Extracts token from Bearer header
   - Calls GitHub User API to validate token and get user info
   - Looks up Kong consumer by GitHub username (optional)
   - Fetches user's GitHub teams via `/user/teams` endpoint
   - Maps team slugs to Kong consumer groups
   - Authenticates with matched consumer groups or falls back to consumer-based groups
   - Adds consumer and group context to request headers (`X-Kong-Consumer-Groups`)

4. **MCP Proxy**:
   - Checks consumer group memberships against tool ACLs
   - Proxies allowed requests to GitHub MCP API
   - Logs payloads, statistics, and audit events

5. **Response**: Returns GitHub MCP API response to client

### OAuth Metadata Endpoint

The `/.well-known/oauth-protected-resource/github-mcp-metadata` endpoint provides resource metadata for OAuth discovery:

```json
{
  "resource": "http://localhost:8000/github-mcp",
  "authorization_servers": ["https://github.com/login/oauth"]
}
```

## Configuration Notes

- **Timeout**: GitHub API calls have a 10-second timeout
- **User-Agent**: Identifies as `Kong-Gateway-MCP` to GitHub
- **API Version**: Uses GitHub API version `2022-11-28`
- **Consumer Lookup**: Tries direct username match first, falls back to `github_` prefix for backward compatibility
- **Token Scopes**: OAuth token requires `read:org` scope for team-based authentication; without it, falls back to consumer-based groups
- **Team Correlation**: GitHub team slug must exactly match Kong consumer group name (case-sensitive)
- **Fallback Behavior**: If teams API fails (403, timeout, decode error), automatically falls back to consumer-based group assignment

## Logging

The configuration includes comprehensive logging:
- Request/response payloads
- Statistics for performance monitoring
- Audit logs for compliance
- Error logs for troubleshooting
- GitHub team matching and consumer group assignments
- Token validation results

### Headers Added to Requests

The authentication plugin adds the following headers to upstream requests:
- `X-Kong-Consumer-ID`: Kong consumer UUID (if consumer exists)
- `X-Kong-Consumer-Username`: Kong consumer username (if consumer exists)
- `X-Kong-Consumer-Groups`: Comma-separated list of matched consumer group names

## Requirements

- Kong Gateway with AI MCP proxy plugin
- GitHub OAuth application for token generation
- Network access to GitHub API (`api.github.com`)

## Future Enhancements

Commented sections in the configuration suggest potential future features:
- Additional GitHub user metadata headers
- File logging to `/tmp/mcp.json`
- Consumer group enumeration logging
