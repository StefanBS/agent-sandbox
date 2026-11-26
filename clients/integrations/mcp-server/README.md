# Kubernetes Agent Sandbox MCP Server

This MCP (Model Context Protocol) server provides tools to interact with Kubernetes Agent Sandbox, allowing language models to seamlessly provision, manage, and execute operations inside isolated sandbox environments.

## Tools

The MCP server exposes the following tools:

* **`create_sandbox`**: Creates a new Kubernetes agent sandbox from a warmpool.
  * **Arguments**:
    * `warmpool` (str)
    * `namespace` (str)
    * `sandbox_ready_timeout` (int, default: 180)
    * `labels` (dict)
    * `shutdown_after_seconds` (int, default: 300). Set as `None` to disable a timeout.
    * `pod_labels` (dict)
    * `pod_annotations` (dict)
  * **Returns**: The claim name of the newly created sandbox.

* **`delete_sandbox`**: Deletes a target sandbox claim.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)

* **`execute_command`**: Executes a shell command inside a sandbox.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `command` (str)
    * `timeout` (int, default: 60)
    * `max_output_chars` (int, default: 30000, min: 128, max: 300000) — Combined stdout/stderr character budget, including omission markers.
  * **Returns**: `exit_code`, `stdout`, and `stderr`, plus `stdout_total_chars`, `stderr_total_chars`, `stdout_truncated`, and `stderr_truncated`.
  * **Output budget**: stdout and stderr initially share the budget equally (stderr gets the odd character), and unused capacity is available to the other stream. Short output is unchanged. Each oversized stream keeps equal-sized beginning and end excerpts, with at most one extra character at the end, separated by `[... N characters omitted ...]`. The marker and its surrounding newlines count toward the budget; the minimum leaves room for a marker in both streams.
    Character counts use Unicode code points, not UTF-8 bytes, visual characters, or model tokens. The `*_total_chars` fields describe the SDK's output before this tool truncates it; the `*_truncated` fields indicate whether this tool omitted content. Truncation preserves the original exit code, including nonzero exits, and never reruns the command.
  * **Compatibility**: Output exceeding the default budget is now truncated, and the response includes four new metadata fields. Callers needing a larger excerpt can set `max_output_chars` up to 300000; clients with strict response schemas must accept the new fields. No output files are created. This limit applies to stdout/stderr text in completed results, not exception messages or the complete serialized MCP response. The SDK has already received the full result, so this does not reduce runtime capture memory.

* **`upload_file`**: Uploads a file to a target sandbox.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `path` (str)
    * `content` (str)
    * `binary` (bool, default: False) — Base64-encoded strings should be passed if `binary` is set to True.
    * `timeout` (int, default: 60)
  * **Returns**: The number of bytes written to the file.

* **`download_file`**: Downloads a file from a target sandbox.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `path` (str)
    * `binary` (bool, default: False) — If True, returns the contents as a base64-encoded string.
    * `timeout` (int, default: 60)
  * **Returns**: Content of the file and the number of bytes read.

* **`list_files`**: Lists the contents of a directory in a sandbox.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `path` (str)
    * `timeout` (int, default: 60)
    * `max_entries` (int, default: 1000, max: 10000) — Caps how many entries are returned.
  * **Returns**: The directory entries (each with `name`, `size`, `type` (`file` or `directory`) and `mod_time` as a POSIX timestamp), plus `total_entries` — the full count before truncation — and `truncated`.
  * **Note**: the response is capped at `max_entries` so that listing a directory with very many
    entries cannot exhaust an LLM's context window. When `truncated` is `True`, narrow the path or use
    `execute_command` for a targeted listing. The cap bounds the MCP response only; the SDK's
    `files.list()` has already materialized the full listing before this tool sees it.

* **`file_exists`**: Checks whether a file or directory exists in a sandbox.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `path` (str)
    * `timeout` (int, default: 60)
  * **Returns**: Whether the path exists. A missing path is a successful `exists: false` result, not an error.

* **`get_sandbox_status`**: Reports the readiness of a sandbox, derived from its Kubernetes conditions. 
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
  * **Returns**: `status` (one of `SandboxReady`, `SandboxNotReady`, or `SandboxNotFound`), `ready` (True only when the `Ready` condition is True), and `message` (the Kubernetes condition message, empty if none is set).

* **`get_sandbox_health`**: Reports whether `sandboxd` is serving inside a sandbox. Requires the sandboxd runtime.
  * **Arguments**:
    * `sandbox_claim_name` (str)
    * `namespace` (str)
    * `timeout` (int, default: 60)
  * **Returns**: `healthy` (True only when `sandboxd` is serving requests), `uptime_seconds` (how long `sandboxd` has been running, null when not healthy), and `message` (a short reason when not healthy, empty otherwise).
  * **Note**: this complements `get_sandbox_status`, which reports Kubernetes readiness. A sandbox whose
    Pod is Ready can still report `healthy` false here, because `sandboxd` stops serving while it shuts
    down. That case returns `healthy` false rather than failing, so an agent can decide whether to wait
    or create a new sandbox. On the legacy python-runtime the tool fails with an error naming the
    sandboxd requirement.

*(The server also provides a `get_sandboxes` resource to fetch a list of existing sandboxes.)*

## Configuration

The server is configured using environment variables via `pydantic-settings` and their names must start with the `K8S_SANDBOX` prefix. Nested configurations are denoted by a double underscore (`__`). You can define these variables in the environment or provide them via a `.env` file in the working directory.

### Connection Configuration

You must specify the connection method to the Kubernetes backend via the `K8S_SANDBOX_CONNECTION__TYPE` variable. It accepts four possible values: `"gateway"`, `"direct"`, `"in-cluster"`, or `"sandboxd-in-cluster"`. Depending on the chosen type, different environment variables apply:

#### 1. Gateway Connection (`K8S_SANDBOX_CONNECTION__TYPE="gateway"`)
Connects via a Kubernetes Gateway API resource.
* `K8S_SANDBOX_CONNECTION__GATEWAY_NAME`: Name of the Gateway resource. (Required)
* `K8S_SANDBOX_CONNECTION__GATEWAY_NAMESPACE`: Namespace where the Gateway resource resides. (Default: `"default"`)
* `K8S_SANDBOX_CONNECTION__GATEWAY_READY_TIMEOUT`: Timeout in seconds to wait for Gateway IP. (Default: `180`)
* `K8S_SANDBOX_CONNECTION__SERVER_PORT`: Port the sandbox container listens on. (Default: `8888`)

#### 2. Direct Connection (`K8S_SANDBOX_CONNECTION__TYPE="direct"`)
Connects directly to the Sandbox router API URL.
* `K8S_SANDBOX_CONNECTION__API_URL`: Direct URL to the router. (Required)
* `K8S_SANDBOX_CONNECTION__SERVER_PORT`: Port the sandbox container listens on. (Default: `8888`)

#### 3. In-Cluster Connection (`K8S_SANDBOX_CONNECTION__TYPE="in-cluster"`)
Connects directly to the sandbox pod from within the cluster (bypassing the router) using stable K8s DNS or the Pod IP.
* `K8S_SANDBOX_CONNECTION__USE_POD_IP`: If set to `true`, connect via the Pod IP fetched from the Sandbox status instead of cluster DNS. (Default: `false`)
* `K8S_SANDBOX_CONNECTION__SERVER_PORT`: Port the sandbox container listens on. (Default: `8888`)

#### 4. sandboxd In-Cluster Connection (`K8S_SANDBOX_CONNECTION__TYPE="sandboxd-in-cluster"`)
Connects directly to [sandboxd](../../../packages/sandboxd/USER_GUIDE.md) in the sandbox pod from within the cluster. It needs no sandbox-router and no `kubectl port-forward`. Commands run over sandboxd's gRPC port and file tools use its REST port.
* `K8S_SANDBOX_CONNECTION__MODE`: `in-cluster-service` (uses `Sandbox.status.serviceFQDN`, so the template needs `spec.service: true`) or `in-cluster-pod-ip` (uses `Sandbox.status.podIPs`). Neither falls back to the other. (Required)
* `K8S_SANDBOX_CONNECTION__REST_PORT`: sandboxd REST port. (Default: `8080`)
* `K8S_SANDBOX_CONNECTION__GRPC_PORT`: sandboxd gRPC port. (Default: `9090`)

The warm pool's template must run sandboxd, as in [examples/sandboxd-sandbox](../../../examples/sandboxd-sandbox/sandbox-template.yaml). The default `Managed` network policy admits only sandbox-router, so add a `spec.networkPolicy.ingress` rule to the template that admits the MCP server on TCP ports `8080` and `9090`. Supplying `spec.networkPolicy` replaces all secure defaults, so also add the `spec.networkPolicy.egress` rules the sandbox needs: an omitted egress list denies all outbound traffic. The server's service account needs `create`, `get`, `list`, `watch` and `delete` on `sandboxclaims`, and `get`, `list` and `watch` on `sandboxes`. It needs no `pods/portforward`. See the [sandboxd guide](../../../packages/sandboxd/USER_GUIDE.md#agent-sandbox-sdk-access) for details.


### General Settings

* `K8S_SANDBOX_SESSION_ID_LABEL_KEY`: The Kubernetes label key to apply for tracking session IDs on the sandboxes. (Default: `"mcp.k8s-agent-sandbox/session-id"`)
* `K8S_SANDBOX_PROBE_NAMESPACE`: Namespace the `/readyz` probe lists SandboxClaims in to confirm the Kubernetes API is reachable. Must be a namespace the server's service account can list claims in. (Default: `"default"`)

## Health checks

The server exposes two endpoints for Kubernetes probes:

| Endpoint | Meaning |
|---|---|
| `GET /healthz` | **Liveness.** The process is up and routing HTTP. Never contacts the Kubernetes API. |
| `GET /readyz` | **Readiness.** Lists SandboxClaims in `K8S_SANDBOX_PROBE_NAMESPACE` to confirm the Kubernetes API is reachable. Returns `200` with `{"ready": true}`, or `503` with `{"ready": false, "reason": "..."}`. |

`/readyz` failure reasons are `"starting"` (the server has not finished initializing) and
`"kubernetes API unavailable"`. The reason is deliberately generic — the endpoint is
unauthenticated, so the underlying exception is logged rather than returned.

`/healthz` is deliberately independent of the Kubernetes API: wiring a liveness probe to an
external dependency turns a transient control-plane blip into a rolling restart of every replica.
Use `/readyz` — which does check — to pull a pod out of the Service instead.

Because `/readyz` makes a real API call, keep `periodSeconds` modest rather than polling it
aggressively:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8000
  initialDelaySeconds: 5
readinessProbe:
  httpGet:
    path: /readyz
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 15
```

## Using with n8n

This walkthrough connects an [n8n](https://n8n.io/) workflow to sandboxes on a local kind cluster. The MCP server and n8n run on your machine. The MCP server reaches sandboxes through sandbox-router, using the `direct` connection type.

### Set up the cluster

Create a kind cluster:

```bash
kind create cluster --name agent-sandbox
```

Install the agent-sandbox core and extensions by following the [installation guide](https://github.com/kubernetes-sigs/agent-sandbox#installation). Then deploy sandbox-router with its [Helm chart](../../../sandbox-router/helm/README.md) from the repository root. Set `VERSION` to the agent-sandbox release you installed, such as `v1.0.12`:

```bash
helm install sandbox-router ./sandbox-router/helm/ \
  --namespace agent-sandbox-system \
  --set image.tag=${VERSION}
```

The sandboxes in this walkthrough run the [Python runtime](../../../examples/python-runtime-sandbox/README.md), which serves the HTTP API the `direct` connection type calls on port `8888`. Build the image and load it into the cluster from the repository root:

```bash
docker build -t sandbox-runtime:latest examples/python-runtime-sandbox
kind load docker-image sandbox-runtime:latest --name agent-sandbox
```

Create a `SandboxTemplate` that runs the image and a `SandboxWarmPool` that keeps one sandbox ready:

```bash
kubectl apply -f - <<EOF
apiVersion: extensions.agents.x-k8s.io/v1beta1
kind: SandboxTemplate
metadata:
  name: python-runtime-template
  namespace: default
spec:
  podTemplate:
    spec:
      containers:
      - name: python-runtime
        image: sandbox-runtime:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8888
        readinessProbe:
          httpGet:
            path: "/"
            port: 8888
          periodSeconds: 1
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "250m"
            memory: "512Mi"
---
apiVersion: extensions.agents.x-k8s.io/v1beta1
kind: SandboxWarmPool
metadata:
  name: python-runtime-warmpool
  namespace: default
spec:
  replicas: 1
  sandboxTemplateRef:
    name: python-runtime-template
EOF
```

### Run the MCP server

In a separate terminal, forward a local port to sandbox-router:

```bash
kubectl port-forward svc/sandbox-router-svc 8080:8080 -n agent-sandbox-system
```

In the `clients/integrations/mcp-server` directory, create a `.env` file that points the server at the forwarded port:

```.env
K8S_SANDBOX_CONNECTION__TYPE=direct
K8S_SANDBOX_CONNECTION__API_URL=http://127.0.0.1:8080
```

Install the server in a virtual environment and start it:

```bash
python -m venv .venv
. .venv/bin/activate
pip install .
uvicorn k8s_agent_sandbox_mcp_server.app:app --host 0.0.0.0 --port 8000
```

The server has no authentication, and `--host 0.0.0.0` exposes it on every network interface so that the n8n container can reach it. Run this only on a trusted network. See [Security](#security).

### Connect n8n

In a third terminal, start n8n in a container:

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  --add-host=host.docker.internal:host-gateway \
  -e GENERIC_TIMEZONE="Etc/UTC" \
  -e TZ="Etc/UTC" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Open `http://127.0.0.1:5678` and create an owner account.

The server lets only the MCP session that created a SandboxClaim use it. n8n opens a new MCP session for each MCP Client step and for each workflow execution. This decides which n8n setup can use a sandbox after creating it:

- An **MCP Client** step in a plain workflow can create a sandbox, but later steps can't use it.
- An **AI Agent** with an **MCP Client Tool** can create a sandbox and use it within one execution.

These steps were tested with n8n 2.49.7.

#### Create a sandbox from a workflow step

1. Create a workflow, select **Add first step**, and choose **Trigger manually**.
2. After the trigger, add an **MCP Client** node. Don't use **MCP Client Tool**, which is for AI Agent nodes.
3. Set **MCP Endpoint URL** to `http://host.docker.internal:8000/mcp`. n8n reads the tool list from the server and offers each tool and its arguments in drop-down menus. Set:
   - **Tool**: `create_sandbox`
   - **warmpool**: `python-runtime-warmpool`
   - **namespace**: `default`
4. Select **Execute step**, and then confirm that n8n created a SandboxClaim:

   ```bash
   kubectl get sandboxclaim
   ```

   The output lists the new claim:

   ```log
   NAME                     AGE
   sandbox-claim-106ba612   3s
   ```

A second **MCP Client** step that calls `execute_command` with this claim name fails with `Sandbox claim '...' is not found`, because it runs in a new MCP session. Delete the claim with `kubectl delete sandboxclaim <name>`.

#### Use a sandbox from an AI Agent

1. Add a credential for a chat model that supports tool calling.
2. Create a workflow with a **Chat Trigger** node, which n8n names **When chat message received**. After it, add an **AI Agent** node, and attach a chat model that uses your credential.
3. Under the agent's **Tool** connector, add an **MCP Client Tool** node. Set **Endpoint** to `http://host.docker.internal:8000/mcp`, and keep **Server Transport** as `HTTP Streamable` and **Authentication** as `None`.
4. Select **Open chat** and send a message that asks for all the work at once. For example:

   ```text
   Create a sandbox from the warm pool python-runtime-warmpool in namespace default, run "echo hi" in it, tell me the output, and then delete the sandbox.
   ```

The agent calls `create_sandbox`, `execute_command`, and `delete_sandbox` in one execution, and the MCP Client Tool uses one MCP session for all of them. Each chat message starts a new execution with a new MCP session, so a later message can't use a sandbox that an earlier message created. If the agent doesn't delete the sandbox, delete its claim with `kubectl delete sandboxclaim <name>`.

### Clean up

Stop n8n, the MCP server, and the port-forward. Then delete the cluster and the n8n data:

```bash
kind delete cluster --name agent-sandbox
docker volume rm n8n_data
```

## Security

The server ships with **no authentication**: the Dockerfile serves streamable HTTP on `0.0.0.0:8000`, and any client that can reach that port gets a fresh session and full access to every tool (create sandboxes, execute arbitrary commands, read/write files) across any namespace its Kubernetes service account can touch.

Do not expose this server directly to untrusted networks. Deploy it behind an authenticating proxy or wire up [fastmcp's auth support](https://gofastmcp.com/servers/auth/authentication/) directly.
