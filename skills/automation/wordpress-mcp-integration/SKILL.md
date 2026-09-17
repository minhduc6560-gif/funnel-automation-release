---
name: wordpress-mcp-integration
description: Connect WordPress to Hermes through official MCP tools.
version: 1.0.0
author: Minh Duc (minhduc6560-gif), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [wordpress, mcp, cms, publishing]
    related_skills: []
---

# WordPress MCP Integration

Connect Hermes to a WordPress site through the official WordPress MCP Adapter path. Start read-only, keep credentials in the active profile’s secret scope, and require explicit approval before creating, editing, publishing, or deleting content.

## When to Use

Use when the user wants Hermes to inspect or manage a WordPress site through MCP and can install the official WordPress MCP Adapter.

Do not use this skill for an unreviewed third-party REST wrapper, direct database access, shared administrator passwords, or unsupported page-builder internals.

## Prerequisites

Require:

- a WordPress version supported by the selected MCP Adapter release;
- HTTPS and a reachable WordPress REST API;
- permission to install and activate the official adapter;
- a dedicated least-privilege WordPress user;
- a WordPress Application Password for that user;
- the site URL and approved capabilities.

Verify the current official WordPress documentation and package ownership before installation. Do not infer compatibility from a similarly named package.

## Secret Handling

Resolve `$HERMES_HOME` and store secrets in `$HERMES_HOME/.env`. Use separate variables such as:

```text
WORDPRESS_SITE_URL
WORDPRESS_USERNAME
WORDPRESS_APP_PASSWORD
```

Reference them as `${WORDPRESS_SITE_URL}`, `${WORDPRESS_USERNAME}`, and `${WORDPRESS_APP_PASSWORD}` in MCP configuration. Never paste secret values into `config.yaml`, a repository, or chat.

## Procedure

1. **Confirm the site and scope.** Record the exact site URL, WordPress user, required read/write operations, and whether draft creation or publication is allowed.
2. **Install the official adapter.** Install a reviewed compatible release of `WordPress/mcp-adapter`, activate it, and confirm the documented MCP endpoint for that release.
3. **Create least-privilege credentials.** Use a dedicated WordPress user and Application Password; do not reuse the normal login password.
4. **Prepare MCP configuration.** Start from `templates/hermes-wordpress-mcp.json`. Keep the npm connector pinned to the reviewed version in that template.
5. **Constrain capabilities.** Include only the tools or WordPress Abilities needed for the approved task.
6. **Test read-only first.** Read a permission-protected resource such as a draft with the smallest practical page size. Listing a public post does not prove authentication.
7. **Inspect available Abilities.** A successful MCP connection does not guarantee content or media management; only exposed and permitted Abilities are callable.
8. **Mutate only on request.** Default new content to `draft`. Require explicit user intent before publishing, deleting, or changing existing content.
9. **Read back every write.** Verify the returned ID, resource type, status, title, and content before reporting success.
10. **Verify the rendered result.** For HTML content, inspect `content.rendered` and the public or preview URL. WordPress filters and themes may mutate or surround submitted markup.

Completion criterion: the exact site is connected with least-privilege credentials, a protected read succeeds, exposed capabilities are documented, and every approved mutation is read back and rendered correctly.

## Media Safety

- Prefer direct WordPress media upload capabilities.
- Never upload local assets to an unrelated third-party host without explicit disclosure and approval.
- Do not publish content while any asset still points to a local or temporary URL.
- Upload large media serially or at low concurrency; a timed-out request may still have completed, so check the exact target before retrying.

## Pitfalls

- Use the WordPress username, not an email address, when Application Password authentication expects the login name.
- Preserve spaces in an Application Password unless the reviewed connector explicitly requires normalization.
- Do not assume installing the adapter exposes post, page, or media operations; capability discovery is authoritative.
- Pin executable npm connectors to a reviewed version. Do not use mutable `latest` or an unpinned branch.
- Keep the connection read-only until authenticated identity and protected-resource reads succeed.
- Do not promise Elementor, ACF, Yoast, Rank Math, or custom-field operations unless the site exposes them through approved Abilities.

## Verification Checklist

- [ ] Official package and compatible versions were verified.
- [ ] Credentials live only in `$HERMES_HOME/.env` or another approved secret source.
- [ ] MCP JSON contains placeholders, not secret values.
- [ ] A protected read proves authentication.
- [ ] Available Abilities and tool scope are recorded.
- [ ] Writes occurred only after explicit user approval.
- [ ] Every write was read back by exact returned ID.
- [ ] Draft/published status and rendered output match the request.
