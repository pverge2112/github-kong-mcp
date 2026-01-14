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
    ↓ (consumer correlation)
GitHub MCP API
```

## Features

### Authentication & Authorization
- **OAuth 2.0 Bearer Token Validation**: Validates GitHub OAuth tokens against GitHub's User API
- **Consumer Correlation**: Automatically maps GitHub users to Kong consumers based on username
- **Consumer Groups**: Organizes users into `mcp-users` and `mcp-developers` groups
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
4. Correlates GitHub username to Kong consumer
5. Authenticates consumer and sets consumer groups
6. Adds consumer context for downstream processing

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

### Consumer Groups
- **mcp-users**: Standard users with default tool access
- **mcp-developers**: Developer users with restricted access to certain tools

### Tool-Level Permissions
Individual MCP tools can have custom ACLs. Example:
- `create_repository`: Denied to `mcp-developers`, allowed to `mcp-users`

This allows fine-grained control over which users can perform specific GitHub operations.

## Adding New Consumers

To add a new user, add an entry to the `consumers` section in kong/deck/kong.yaml:

```yaml
consumers:
  - username: github_username
    custom_id: github:github_username
    groups:
      - name: mcp-users
```

The `username` should match the GitHub user's login name for automatic correlation.

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
   - Looks up Kong consumer by GitHub username
   - Authenticates consumer and sets consumer groups
   - Adds consumer context to request

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

## Logging

The configuration includes comprehensive logging:
- Request/response payloads
- Statistics for performance monitoring
- Audit logs for compliance
- Error logs for troubleshooting

## Requirements

- Kong Gateway with AI MCP proxy plugin
- GitHub OAuth application for token generation
- Network access to GitHub API (`api.github.com`)

## Future Enhancements

Commented sections in the configuration suggest potential future features:
- Additional GitHub user metadata headers
- File logging to `/tmp/mcp.json`
- Consumer group enumeration logging
