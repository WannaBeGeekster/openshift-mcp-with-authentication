# OpenShift MCP Server with OAuth passthrough

Standalone manifests (no AutoShift needed). They deploy the Red Hat OpenShift MCP Server,
tech preview 0.4. Each caller authenticates with their own OpenShift token, and every tool runs
with that user's own RBAC. This is the configuration running on the lab hub. It was verified on
2026-11-06: a request without a token gets 401, and a request with a token lists the tools.

## How the OAuth part works

The MCP server runs no login flow and touches no cluster OAuth or IdP configuration. It relies
on the cluster's normal login:

1. The user logs in to OpenShift through any identity provider on the built-in OAuth server
   (Keycloak, LDAP, htpasswd, and so on).
2. The user takes their token (`oc whoami -t`) and gives it to the MCP client as
   `Authorization: Bearer <token>`.
3. The server settings in `config.toml`:
   - `require_oauth = true`: no token means 401.
   - `skip_jwt_verification = true`: OpenShift tokens are opaque, not JWTs.
   - `cluster_auth_mode = "passthrough"`: the server calls the API with the caller's token, so
     the API server authenticates it and applies that user's RBAC.

   The ServiceAccount needs no cluster permissions.

## Prerequisites

- The OpenShift 4.x cluster pull secret can pull from `registry.redhat.io`.
- Users can log in through an identity provider.
- Users have RBAC on what they should see. At least `view` on the relevant namespaces, or
  `cluster-reader` for cluster-wide read.
- Check that the client accepts Red Hat tech preview support terms for this image.

## Deploy

```bash
HOST=mcp.apps.$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
sed "s/MCP_HOST/$HOST/g" openshift-mcp-server.yaml | oc apply -f -
oc rollout status deploy/openshift-mcp-server -n openshift-mcp-server
```

The trusted CA bundle is injected into `openshift-mcp-server-trusted-ca` by the cluster within a
few seconds. If the pod starts before that, restart it once:
`oc rollout restart deploy/openshift-mcp-server -n openshift-mcp-server`.

## Verify

```bash
URL=https://$HOST/mcp
# 1. No token: expect 401
curl -s -o /dev/null -w '%{http_code}\n' -X POST $URL -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}'
# 2. With a token: expect 200 and an mcp-session-id header
TOKEN=$(oc whoami -t)
curl -s -D - -o /dev/null -X POST $URL -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}' | grep -i -E '^HTTP|mcp-session-id'
```

MCP client settings: URL `https://<HOST>/mcp`, header `Authorization: Bearer <token>`.

## Tuning

- **Toolsets:** edit `toolsets` in the ConfigMap. The lab hub runs: cluster-diagnostics,
  cni-diagnostics, config, core, helm, kcp, kubevirt, netedge, netobserv, oadp,
  observability/logs, observability/metrics, observability/otelcol, observability/traces,
  openshift, openshift/mustgather, ossm, ovn-kubernetes, tekton. Change the ConfigMap, then
  restart the deployment.
- **Write access:** `read_only = false` and `disable_destructive = false` enable mutating tools,
  still limited by each user's RBAC.
- **Image:** pinned by digest. To update, run
  `skopeo inspect docker://registry.redhat.io/openshift-mcp-tech-preview/openshift-mcp-server-rhel9:0.4 --format '{{.Digest}}'`
  and replace the digest.

## Limits

- **OAuth tokens expire** (24 hours by default). MCP clients do not refresh them, so users paste
  a fresh `oc whoami -t`.
- **Not configured: an MCP-native OAuth login flow.** This is where the client itself opens a
  browser login. It needs an `OAuthClient` and `authorization_url`, which this bundle does not
  include and which was not tested.
- **Do not combine `require_oauth` with `cluster_auth_mode = "kubeconfig"`.** Image 0.4 refuses
  to start with that combination.
# openshift-mcp-with-authentication
