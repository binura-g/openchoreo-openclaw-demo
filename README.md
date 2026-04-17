# OpenClaw On OpenChoreo with Agent Sandbox

This is a sample config for running [OpenClaw](https://openclaw.ai/) agents on [OpenChoreo](https://openchoreo.dev) with [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/).

# Pre-requisites

1. Install OpenChoreo https://openchoreo.dev/docs/getting-started/try-it-out/on-k3d-locally/. 
    - You do not need to clone this repository to follow along. If you're running a different OpenChoreo setup, you will need to clone and modify the OpenChoreo resources in this repository to match your tooling.

2. An API key from https://platform.openai.com. 
    - OpenClaw supports other model providers, but you'll need to update the OpenChoreo resources in this repository to match the provider's configuration.

## 1. Install Agent-Sandbox CRDs and controllers on the data plane cluster(s)

```shell
# Reference: https://agent-sandbox.sigs.k8s.io/docs/overview/#installation
# https://github.com/kubernetes-sigs/agent-sandbox/releases
export VERSION="v0.3.10"
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/$VERSION/manifest.yaml
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/$VERSION/extensions.yaml
```

Give the OpenChoreo cluster-agent access to modify agent-sandbox CRs:

```sh
kubectl patch clusterrole cluster-agent-dataplane-openchoreo-data-plane \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/rules/-",
      "value": {
        "apiGroups": ["agents.x-k8s.io"],
        "resources": ["sandboxes"],
        "verbs": ["get","list","watch","create","update","patch","delete"]
      }
    },
    {
      "op": "add",
      "path": "/rules/-",
      "value": {
        "apiGroups": ["agents.x-k8s.io"],
        "resources": ["sandboxes/status","sandboxes/finalizers"],
        "verbs": ["get","update","patch"]
      }
    }
  ]'
```

## 2. Add the secrets to the secret store (OpenBao) and create the OpenChoreo SecretReferences

```shell
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"
echo "OpenClaw Gateway Token: ${OPENCLAW_GATEWAY_TOKEN}" # You'll need this to sign in later

kubectl exec -n openbao openbao-0 -- sh -c "
  export BAO_ADDR=http://127.0.0.1:8200 BAO_TOKEN=root
  bao kv put secret/openai-api-key value='<YOUR_OPENAI_API_KEY>'
  bao kv put secret/openclaw-gateway-token value='${OPENCLAW_GATEWAY_TOKEN}'
"
```

```sh
cat <<'EOF' | kubectl apply -f -
apiVersion: openchoreo.dev/v1alpha1
kind: SecretReference
metadata:
  name: openai-api-key
  namespace: default
spec:
  refreshInterval: 1h
  template:
    type: Opaque
  data:
    - secretKey: value
      remoteRef:
        key: openai-api-key
        property: value
---
apiVersion: openchoreo.dev/v1alpha1
kind: SecretReference
metadata:
  name: openclaw-gateway-token
  namespace: default
spec:
  refreshInterval: 1h
  template:
    type: Opaque
  data:
    - secretKey: value
      remoteRef:
        key: openclaw-gateway-token
        property: value
EOF
```

## 2. Apply OpenChoreo Resources

This will create a "openclaw-agents" Project, a custom "openclaw-sandbox-agent" ClusterComponentType and a Component to try it out.

> Alternatively, you can use the OpenChoreo CLI to apply these instead of kubectl with `occ apply -f <filename/url>`


```sh
kubectl create -f https://raw.githubusercontent.com/binura-g/openchoreo-openclaw-demo/refs/heads/main/project.yaml
kubectl create -f https://raw.githubusercontent.com/binura-g/openchoreo-openclaw-demo/refs/heads/main/openclaw-sandbox-cct/component-type.yaml
kubectl create -f https://raw.githubusercontent.com/binura-g/openchoreo-openclaw-demo/refs/heads/main/openclaw-sandbox-cct/component.yaml
kubectl create -f https://raw.githubusercontent.com/binura-g/openchoreo-openclaw-demo/refs/heads/main/openclaw-sandbox-cct/workload.yaml
```

Or if you have the repository cloned locally:

```sh
kubectl create -f project.yaml
kubectl create -f openclaw-sandbox-cct/component-type.yaml
kubectl create -f openclaw-sandbox-cct/component.yaml
kubectl create -f openclaw-sandbox-cct/workload.yaml
```

## 4. Log in to create a pairing request to OpenClaw

Open the external gateway URL shown on the OpenChoreo Portal (via the Component > Deploy tab > View Endpoint URLs) and log in using the `OPENCLAW_GATEWAY_TOKEN` we created earlier.

```sh
echo $OPENCLAW_GATEWAY_TOKEN
```

You'll get an error saying "Pairing required" when you click "Connect" - this is expected.

## 5. Approve Browser pairing request for OpenClaw

Get the pairing request ID from the OpenClaw pod created by the agent-sandbox by running:

```sh
NS=$(kubectl get pod -A -l openchoreo.dev/component=openclaw-sandbox-agent -o jsonpath='{.items[0].metadata.namespace}')
POD=$(kubectl get pod -n "$NS" -l openchoreo.dev/component=openclaw-sandbox-agent -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n "$NS" "$POD" -- node /app/dist/index.js devices list
```

You'll see a table with pairing requests - copy the request ID (this is a UUID) and run the following command:

```sh
kubectl exec -n "$NS" "$POD" -- node /app/dist/index.js devices approve <requestId>
```

## 6. Log into the OpenClaw Control UI

Once the pairing is approved, you should be able to log into OpenClaw Control UI and start clawing away :)

---

> **Note:** There's another directory in this repository called `/openclaw-cct` which provides a custom component type for OpenClaw using default K8s primitives such as a apps/v1 deployment and PVC, without the abstractions provided by agent-sandbox. You technically do not need a custom component type (CCT) for this, even the default `deployment/web-app` CCT should suffice.

**If you enjoyed using OpenChoreo, consider giving the project a star on GitHub: https://github.com/openchoreo/openchoreo**
