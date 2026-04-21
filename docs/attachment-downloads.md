# Gmail Attachment Downloads

This document describes every delivery mode for `get_gmail_attachment_content` and the environment variables that control each one.

---

## How the tool works

When called, the tool:

1. Fetches the raw attachment bytes from the Gmail API (base64-encoded).
2. Re-fetches the parent message to resolve the original filename and MIME type (with fallbacks by size and position).
3. Delivers the attachment via one of three modes depending on configuration.

---

## Delivery modes

### 1. stdio — local file path

**When:** transport is `stdio` (default) and `WORKSPACE_MCP_STATELESS_MODE` is not `true`.

The file is decoded and written to disk. The tool returns the absolute path to that file.

```
📎 Saved to: /home/user/.workspace-mcp/attachments/report_3f2a1b4c.pdf
```

The file persists until it expires (default 1 hour) or the process restarts.

**Required config:**

| Variable | Purpose | Default |
|---|---|---|
| `WORKSPACE_ATTACHMENT_DIR` | Directory where files are saved | `~/.workspace-mcp/attachments/` |

---

### 2. HTTP — temporary download URL

**When:** transport is `streamable-http` and `WORKSPACE_MCP_STATELESS_MODE` is not `true`.

The file is saved to disk. The tool returns a one-time download URL served by the workspace-mcp HTTP server at `GET /attachments/{file_id}`.

```
📎 Download URL: https://your-domain.com/attachments/550e8400-e29b-41d4-a716-446655440000
```

The URL is valid for 1 hour. After that the file is deleted and the endpoint returns 404.

**HTTP endpoint details:**

- Route: `GET /attachments/{file_id}`
- No authentication required — the UUID file ID is the access token.
- Returns the file with correct `Content-Type` and `Content-Disposition` headers.
- Returns `404 JSON` if the ID is unknown or expired.

**Required config:**

| Variable | Purpose | Default |
|---|---|---|
| `WORKSPACE_EXTERNAL_URL` | Base URL prepended to `/attachments/{file_id}` in the returned link. Must be the publicly reachable URL of this service (no trailing slash). | Falls back to `WORKSPACE_MCP_BASE_URI:WORKSPACE_MCP_PORT` |
| `WORKSPACE_MCP_BASE_URI` | Used as fallback base URL when `WORKSPACE_EXTERNAL_URL` is not set. | `http://localhost` |
| `WORKSPACE_MCP_PORT` / `PORT` | Port the server listens on. Used in the fallback URL. | `8000` |
| `WORKSPACE_ATTACHMENT_DIR` | Directory where files are saved. | `~/.workspace-mcp/attachments/` |

**Set `WORKSPACE_EXTERNAL_URL` when:**
- The service is behind a reverse proxy, load balancer, or API gateway.
- The service is deployed on Render, Kubernetes, or any platform where the public-facing URL differs from `localhost:8000`.

Example:
```
WORKSPACE_EXTERNAL_URL=https://workspace.example.com
```

The returned URL will then be:
```
https://workspace.example.com/attachments/550e8400-e29b-41d4-a716-446655440000
```

---

### 3. Stateless mode — full base64 in response

**When:** `WORKSPACE_MCP_STATELESS_MODE=true`.

No file is written to disk. The complete base64-encoded attachment data is returned inline in the tool response. The calling LLM or client must decode it.

```
⚠️ Stateless mode: File storage disabled.

Base64-encoded content:
JVBERi0xLjQKJeLjz9MKNCAwIG9iago8PAov...
```

This mode exists for truly stateless deployments (e.g. serverless platforms) where ephemeral disk is not available or not desirable.

**Constraint:** `WORKSPACE_MCP_STATELESS_MODE=true` requires `MCP_ENABLE_OAUTH21=true`.

**Required config:**

| Variable | Required value |
|---|---|
| `WORKSPACE_MCP_STATELESS_MODE` | `true` |
| `MCP_ENABLE_OAUTH21` | `true` |

---

## Configuration reference

| Variable | Description | Default |
|---|---|---|
| `WORKSPACE_MCP_STATELESS_MODE` | Disables all file I/O. Returns full base64 inline. Requires `MCP_ENABLE_OAUTH21=true`. | `false` |
| `WORKSPACE_EXTERNAL_URL` | Public base URL used to build attachment download links in HTTP mode. No trailing slash. | Unset (falls back to `base_uri:port`) |
| `WORKSPACE_MCP_BASE_URI` | Internal base URI. Used as fallback when `WORKSPACE_EXTERNAL_URL` is not set. | `http://localhost` |
| `WORKSPACE_MCP_PORT` / `PORT` | Port the HTTP server listens on. | `8000` |
| `WORKSPACE_ATTACHMENT_DIR` | Directory for temporary attachment storage. Created automatically with mode `0700`. | `~/.workspace-mcp/attachments/` |

---

## Decision tree

```
Transport = stdio?
  └─ yes → Save to disk → return file path
  └─ no (streamable-http)
       └─ WORKSPACE_MCP_STATELESS_MODE=true?
            └─ yes → Return full base64 inline
            └─ no  → Save to disk → return download URL
                      (uses WORKSPACE_EXTERNAL_URL if set,
                       otherwise base_uri:port)
```

---

## Expiry and cleanup

- All saved files expire after **1 hour** (hardcoded default in `core/attachment_storage.py`).
- Expired files are cleaned up lazily on the next access attempt for that file ID.
- Files are saved with permissions `0600` (owner read/write only).
- File names follow the pattern `{original_stem}_{first8charsOfUUID}{ext}` to avoid collisions.

---

## Reverse proxy / gateway setup

When workspace-mcp sits behind a gateway (e.g. an API gateway or Cloudflare Worker), two things must be true for download URLs to work:

1. **`WORKSPACE_EXTERNAL_URL`** must be set to the public URL that clients will use to reach the service — not the internal service URL.

2. **The gateway must proxy `GET /attachments/{file_id}`** through to workspace-mcp without requiring authentication, since end users open this URL directly in a browser. The UUID file ID is the only access control.

Example for a gateway proxying to an internal Render service:

```
Client browser
  → GET https://public.example.com/attachments/{file_id}
    → gateway (no auth check on /attachments/*)
      → GET https://internal-service.onrender.com/attachments/{file_id}
        → workspace-mcp serves FileResponse
```

workspace-mcp env config for this setup:
```
WORKSPACE_EXTERNAL_URL=https://public.example.com
```
