# ocp-console-login — a tentacle package

A [tentacle](https://github.com/gmeghnag/tentacle) package with one skill.

| Content | What it does |
|---|---|
| skill `ocp-console-login` | Logs into a Red Hat OpenShift web console with a self-signed certificate from the Playwright MCP browser: gets past the certificate interstitial, picks the right identity provider (`kubeadmin` → `kube:admin`), fills the login form and lands on the requested page (e.g. a Deployment) |

It needs a Playwright MCP server configured in tentacle. The skill holds no
credentials: when the console URL, IdP, username or password are not in the
conversation, the agent asks you for them, and never writes them anywhere.

## Install

```sh
tentacle install https://github.com/gmeghnag/tentacle-ocp-console-login.git#v0.1.0
tentacle package list
```

## Try it

Ask: "open the OpenShift console and show me the deployment `web` in namespace `devops`".

Remove it with `tentacle package remove ocp-console-login`.
