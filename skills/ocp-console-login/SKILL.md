---
name: ocp-console-login
description: >
  Log into the Red Hat OpenShift web console from the Playwright MCP browser on a
  cluster with a self-signed certificate: get past the cert interstitial, pick the
  right IdP, fill the login form, land on a specific page. Use when the user asks
  to open, log into or browse the OCP/OpenShift console ("apri/entra nella console
  OCP", a "console-openshift-console" URL, "show me <resource> in the UI") or
  whenever a console page needs authenticated browser access. Does not cover `oc`
  CLI or API login (kubeconfig) or non-browser automation.
metadata:
  author: gmeghnag
  version: "1.3"
---

# OCP web console login (browser)

Authenticated browser access to an OpenShift web console whose certificate is
**self-signed**, driven by the Playwright MCP tools. One idea: the default MCP
context does NOT ignore TLS errors, so the flow is always "get past every cert
interstitial, pick the IdP the credentials belong to, log in".

## Requirement: Playwright MCP

This skill needs the **Playwright MCP server** — its tools are named
`mcp__playwright__browser_*` (`browser_navigate`, `browser_run_code_unsafe`, …).
Check that they are among your tools BEFORE anything else. If they are not, do
not try another way (no `curl`, no other browser): stop, tell the user the skill
needs Playwright MCP, and suggest installing it by adding it to `mcpServers` in
the tentacle config (`~/.config/tentacle/config.jsonc`), then restarting tentacle:

```jsonc
"mcpServers": {
  "playwright": { "command": "npx", "args": ["@playwright/mcp@latest"] }
}
```

(It needs Node.js / `npx`; the first run downloads the package and a browser.)

## Credentials

You need three things: the console URL, the identity provider (IdP), and a
username + password valid for THAT IdP. Never guess them. If the user has not
given them in the conversation, **ask the user** for them before logging in. Never
write them into this skill, a file, a commit or your output.

## Workflow

1. `browser_navigate` to the console URL. Expect `net::ERR_CERT_AUTHORITY_INVALID` —
   the page stays on the privacy-error interstitial (`chrome-error://chromewebdata`).
   That is the normal state, not a failure.
2. Get past the interstitial:
   - **macOS only:** type the literal string `thisisunsafe` on the interstitial
     (e.g. via `browser_run_code_unsafe`: `await page.keyboard.type('thisisunsafe')`,
     or the individual keys). No click needed; the page proceeds to the real URL.
     It is valid ONLY on a Mac — do not rely on it on any other OS.
   - **Any other OS (or if it does not proceed):** use the `ignoreHTTPSErrors`
     context described under *Alternative* below.
3. The console redirects to `oauth-openshift.apps.../oauth/authorize?client_id=console...`.
   If the privacy-error interstitial reappears (every redirect to a new domain),
   repeat step 2.
4. On "Log in with", click the link of the IdP the credentials belong to
   (username `kubeadmin` → ALWAYS `kube:admin`, e.g.
   `getByRole('link', { name: 'kube:admin' })`). If there are several and you do not
   know which one, ask the user. The SPA may not have rendered the inputs at
   DOMContentLoaded — wait a couple of seconds or `waitForSelector` first.
5. Fill username + password and click "Log in".
6. Verify you are on `/dashboards`, then navigate to the target resource, e.g.
   `.../k8s/ns/<ns>/apps~v1~Deployment/<name>` (the group-version-resource is
   URL-escaped with `~`).

## Alternative (any OS)

A single `browser_run_code_unsafe` call that creates
`browser.newContext({ ignoreHTTPSErrors: true })` and does navigate+login in it.
`globalThis` does not persist across calls, so do everything in ONE call, or locate
the page later via `for (const c of browser.contexts()) for (const pg of c.pages())`.

## Gotchas

- **User `kubeadmin` → always pick `kube:admin`.** The kubeadmin password works
  only through the `kube:admin` IdP, never through any other "Log in with" link,
  even if one of them has a user named `admin`.
- **Wrong IdP**: credentials are valid only for their own IdP. Users of another IdP
  (e.g. an `htpasswd` one) get "Invalid login or password" with the kubeadmin
  password, and vice versa. On that error, re-check the IdP before the password.
- **`thisisunsafe` (macOS only) must be typed on the interstitial itself**, and
  re-typed after every redirect to a new domain (console → oauth-openshift). Typing
  it on an already-loaded page does nothing.
- **Never write the password into the skill, a file, a commit or chat output.**

## Examples

Input: "apri la console e guardami il deployment web nel ns devops"

Expected: no credentials in the conversation → ask the user for console URL, IdP,
username and password → navigate → past the interstitial (`thisisunsafe` on a Mac,
else the `ignoreHTTPSErrors` context) → again after the oauth redirect → click the
IdP → fill the form → land on `/k8s/ns/devops/apps~v1~Deployment/web` → snapshot or
screenshot as proof.
