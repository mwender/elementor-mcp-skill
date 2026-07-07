# Onboarding a new Elementor MCP site

Use this reference when starting work on a WordPress site that doesn't yet have the MCP plumbing in place. If EMCP tools (`mcp__*__emcp-tools-*`, or legacy `mcp__*__elementor-mcp-*`) are already available in the session, the site is wired up — skip this entire document and return to the main skill's [Workflow](../SKILL.md#workflow).

## Detect framework

WordPress sites come in two flavors. The install paths and `wp --path` arguments differ; getting this wrong is the most common stumble.

| Signal | Standard WP | Bedrock |
|---|---|---|
| Webroot | `public/`, `public_html/`, or repo root | `web/` |
| WP core | `wp-admin/`, `wp-includes/` at webroot | `web/wp/` |
| Plugins | `wp-content/plugins/` | `web/app/plugins/` |
| `composer.json` at project root | usually none | yes, with `roots/wordpress` |
| `wp-cli.yml` at project root | usually none | yes, sets `path: web/wp` |
| `wp-config.php` | at webroot | `web/wp-config.php` + `config/application.php` |

One-shot check:

```bash
test -f composer.json && grep -q '"roots/wordpress"' composer.json && echo BEDROCK || echo STANDARD
```

## Install the EMCP Tools plugin (single plugin — adapter is bundled)

The plugin **`emcp-tools`** (repo: [msrbuilds/elementor-mcp](https://github.com/msrbuilds/elementor-mcp), renamed from "Elementor MCP" at v2.0) has bundled the WordPress MCP Adapter since v1.7.4 — **no separate `mcp-adapter` plugin install is needed.** One plugin registers both MCP servers and the `wp mcp-adapter` CLI command.

### Standard WP and Bedrock — wp-cli + GitHub Releases zip

```bash
cd <project-root or webroot>

# Get the latest release asset URL (Composer/Packagist install is not supported):
LATEST=$(curl -s "https://api.github.com/repos/msrbuilds/elementor-mcp/releases/latest" \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['assets'][0]['browser_download_url'])")
wp plugin install --activate "$LATEST"
```

Alternative: download the `emcp-tools-X.Y.Z.zip` release asset and upload via Plugins → Add New → Upload Plugin in WP admin. Once installed, free users get subsequent updates through the plugin's in-dashboard updater (since v3.1).

Verify both servers registered:

```bash
wp mcp-adapter list
# Expected: mcp-adapter-default-server (3 tools) + emcp-tools-server (90+ tools)
```

> Benign noise: every WP load may emit PHP notices on **stderr** like `Ability "emcp-tools/create-theme-template" is already registered` (known plugin bug as of 3.1.0). They go to stderr, not stdout, so they don't corrupt the stdio JSON-RPC stream — ignore them.

**Optional — expose credentials as WP constants (Bedrock):** If the Application Password is stored in `.env` (e.g., `WP_APP_USER` / `WP_APP_PASSWORD`), wire them up in `config/application.php` using the same `Config::define()` pattern used for DB credentials:

```php
Config::define('WP_APP_USER', env('WP_APP_USER'));
Config::define('WP_APP_PASSWORD', env('WP_APP_PASSWORD'));
```

This makes them available as PHP constants inside WordPress, consistent with Bedrock's config conventions. It does NOT make them available in the shell environment — see the [HTTP transport note](#http-cloud-friendly-no-local-cli-needed) about why `${ENV_VAR}` interpolation doesn't work in `.mcp.json` on Bedrock.

Check the [EMCP Tools README](https://github.com/msrbuilds/elementor-mcp) for current install guidance — the plugin ships frequent releases and install/connection options may have expanded since this reference was written (e.g. v3.0 added a one-click `.mcpb` bundle for Claude Desktop).

> **WP 6.8 note:** the MCP Adapter requires the [Abilities API](https://github.com/WordPress/abilities-api), which is in core from 6.9 onward. On 6.8 you also need to install it separately (`wp plugin install abilities-api --activate` on standard WP, `composer require wordpress/abilities-api` on Bedrock).

## Verify the servers registered

```bash
wp plugin list --status=active | grep -E "emcp-tools|elementor"
wp mcp-adapter list
```

Expected output from `wp mcp-adapter list` (verbatim from a v3.1.0 site):

```
ID                              Name                            Version    Tools
mcp-adapter-default-server      MCP Adapter Default Server      v1.0.0     3
emcp-tools-server               MCP Tools for Elementor Server  v3.1.0     94
```

If `emcp-tools-server` is missing, check that the plugin is active. Note: since v3.0 the WordPress-management tools register without Elementor, but the Elementor page-building family needs Elementor active and ≥ 3.20.

## Write `.mcp.json` at the project root

Two transports, with trade-offs:

### STDIO (preferred for local sites with filesystem access)

```json
{
  "mcpServers": {
    "<site>-wp": {
      "type": "stdio",
      "command": "wp",
      "args": ["mcp-adapter", "serve",
               "--server=mcp-adapter-default-server",
               "--user=<admin-username>",
               "--path=<absolute-webroot>"]
    },
    "<site>-elementor": {
      "type": "stdio",
      "command": "wp",
      "args": ["mcp-adapter", "serve",
               "--server=emcp-tools-server",
               "--user=<admin-username>",
               "--path=<absolute-webroot>"]
    }
  }
}
```

Webroot: standard WP = `<project>/public/` (or wherever the site lives); Bedrock = `<project>/web/wp/`.

Pros: no Application Password, no SSL/cert friction for local self-signed certs, no `Mcp-Session-Id` header juggling.

Cons: only works where Claude has shell + filesystem access to the WordPress install.

### HTTP (cloud-friendly, no local CLI needed)

```json
{
  "mcpServers": {
    "<site>-wp": {
      "type": "http",
      "url": "https://<site>/wp-json/mcp/mcp-adapter-default-server",
      "headers": { "Authorization": "Basic <base64(user:app-password)>" }
    },
    "<site>-elementor": {
      "type": "http",
      "url": "https://<site>/wp-json/mcp/emcp-tools-server",
      "headers": { "Authorization": "Basic <base64(user:app-password)>" }
    }
  }
}
```

The user generates the Application Password themselves (Users → Profile → Application Passwords). **Never generate credentials on the user's behalf** — they should be the only one to create one and hand it over.

Compute the base64 value from the username and Application Password:

```bash
python3 -c "import base64; print(base64.b64encode(b'<user>:<app-password>').decode())"
```

**Hardcode the base64 value directly in `.mcp.json` — do not use `${ENV_VAR}` interpolation.** Claude Code expands `${...}` references from the shell environment. On Bedrock sites, `.env` is loaded by PHP/Dotenv when WordPress boots — it never reaches the shell Claude Code runs in. On standard WP sites the same applies: the shell environment won't have those vars unless they're explicitly exported. `${VAR}` will silently expand to an empty string, the Authorization header fails, and the MCP server won't load. Since `.mcp.json` will contain a credential, add it to `.gitignore`:

```bash
echo ".mcp.json" >> .gitignore
```

If credentials expire, recompute the base64 and update the header value in `.mcp.json`.

> Default to STDIO for any local Valet / DDEV / Local-by-Flywheel site. Reach for HTTP when the site is remote, or when running in a CI/cloud agent without WP-CLI on PATH.

After writing `.mcp.json`, the user needs to restart Claude Code so the new servers are loaded.

## First-session checklist

Once MCP tools are available in the session, verify the site is ready before any page-building:

```bash
# 0. Permalinks configured? (virgin sites often miss this — REST API returns HTML if not set)
wp option get permalink_structure   # should be non-empty, e.g. /%postname%/
# If empty: wp rewrite structure '/%postname%/' && wp rewrite flush

# 1. Plugin active? (single plugin — adapter is bundled)
wp plugin list --status=active | grep -E "emcp-tools|elementor"

# 2. Tool gate — are the skill-critical tools enabled? v3.1 disables 39 abilities by default,
#    including add-custom-css, set-dynamic-tag, list-dynamic-tags, add-pro-widget,
#    create-theme-template, set-template-conditions. See SKILL.md "The v3 tool gate".
wp option get emcp_tools_disabled_tools --format=json

# 3. safe-svg installed AND role-gate open?
wp plugin list --status=active | grep -i safe-svg
wp option get safe_svg_upload_roles    # should be ["administrator"] or similar; if empty, fix it
# wp option update safe_svg_upload_roles '["administrator"]' --format=json

# 4. Active kit ID — needed for any kit-level CSS changes
wp option get elementor_active_kit

# 5. agent-browser viewport sized for desktop visual review
agent-browser set viewport 1280 800
```

Plus the [Kit-level CSS conventions](../SKILL.md#kit-level-css-conventions) baseline check (workflow step 3 in SKILL.md).

A passing checklist means you can start the [Workflow](../SKILL.md#workflow) without surprises.

## Diagnostic: one-shot JSON-RPC probe (no MCP client needed)

If the stdio MCP servers disconnect mid-session and won't reload, or you need to verify the server side independently of any client, pipe a one-shot JSON-RPC sequence straight into `wp mcp-adapter serve`. This exercises the exact same 90+ tools the live session sees — only the transport differs:

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | wp mcp-adapter serve --server=emcp-tools-server --user=<admin-username> --path=<absolute-webroot>
```

Swap the `tools/list` line for a `tools/call` (`{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"<tool>","arguments":{...}}}`) to invoke any tool directly. Useful for: distinguishing a client-side connection problem from a server-side registration problem, testing after enabling gated tools, and scripted checks in CI. (PHP notices on stderr are expected and harmless — see the install section.)

## Memory templates

After the site is wired up and the checklist passes, drop a small set of memory files at `~/.claude/projects/<sanitized-cwd>/memory/`. These give every future Claude session on this site a fast cold-start without re-deriving the onboarding facts.

The user's memory dir uses an `MEMORY.md` index file that lists the others. Loaded at session start.

### `MEMORY.md` (index)

```markdown
- [User role](user_role.md) — who the user is, their tech comfort level
- [<site> site context](project_<site>_site.md) — local + remote URLs, stack, paths, MCP wiring
- [MCP transport for <site>](feedback_mcp_transport_<site>.md) — stdio vs http rationale for this site
```

(Add more lines as additional memory files accumulate. The user may already have site-agnostic feedback memories like `feedback_wp_mcp_setup_workflow.md` from prior projects — keep those in their existing location.)

### `project_<site>_site.md` (site context)

Captures the framework + MCP decisions from onboarding so they don't get re-derived every session.

```markdown
---
name: <site> site context
description: <site> stack, paths, and MCP wiring
type: project
---

- Local URL: https://<site>.test (or wherever)
- Remote/production URL: https://<site>
- Framework: Standard WP / Bedrock
- Webroot: <absolute path>
- WP-CLI invocation: `wp` (from <webroot>) or globally via `--path=<webroot>`
- Theme: <theme name> (child theme: yes/no)
- Key plugins + versions:
  - Elementor X.X.X
  - Elementor Pro X.X.X
  - emcp-tools X.X.X (bundles the MCP Adapter)
  - safe-svg X.X.X
- EMCP tools enabled beyond defaults: <list, e.g. add-custom-css, set-dynamic-tag> (v3 tool gate)
- MCP server IDs in .mcp.json: `<site>-wp`, `<site>-elementor`
- Transport: stdio (or http with App Password)
- Active kit ID: <N>
- WP admin user for MCP: <username>
```

### `feedback_mcp_transport_<site>.md` (transport rationale)

One paragraph on why this site uses the transport it does. Saves future sessions from second-guessing.

```markdown
---
name: MCP transport preference for <site>
description: Default to <stdio/http> on this site; rationale below
type: feedback
---

<site> uses <stdio/http> because <reason — e.g., "local Valet site, Claude has filesystem access, avoids App Password churn">.

If the situation changes (e.g., need to access from a cloud agent), switch to the alternate transport — full templates in `references/onboarding.md` of the elementor-mcp skill.
```

### Project-level vs user-level memory — what goes where

The above three are **project-level** memories (live in `~/.claude/projects/<sanitized-cwd>/memory/`). They apply only to one site.

For preferences that span **all** Elementor projects — e.g., *"always use viewport 1280x800 for desktop screenshots,"* *"I prefer the Elementor Canvas template for landing pages,"* *"prefer stdio MCP transport for any local site"* — use **user-level** memory (`~/.claude/CLAUDE.md` or an equivalent user-scoped file). Don't duplicate user-level preferences in every project's memory dir.

Quick test for placement: *would a different developer working on the same project want this memory?* If yes → project-level. If it's only your personal pattern → user-level.

### Optional starter files

If the user has additional patterns worth capturing on day one (e.g., per-project agency preferences, branding conventions, content style notes), suggest them but don't write them without confirmation. Memory files are persistent and surface across every future session — they should reflect genuine recurring needs, not speculative ones.
