# Ghost Helm Chart

Helm chart and supporting files for deploying Ghost to Kubernetes. The defaults match the AtarNet `blog` namespace, but the repo now also includes a local override for quick tests on Docker Desktop.

## What's Inside
- `ghost-chart/` - Helm chart templates and default values.
- `ghost-chart/values.local.yaml` - Overrides tuned for Docker Desktop (no ingress, ephemeral storage).
- `ci/kubernetes/trivy.yaml` - Job manifest used by CI to scan the Ghost image.
- `Jenkinsfile` - Pipeline that lints, scans, and deploys the chart in shared environments.

## Test Locally with Docker Desktop
Docker Desktop ships with a single-node Kubernetes cluster. These steps assume you already enabled it in Docker settings and can run `kubectl` commands.

1. **Select the context**
   ```bash
   kubectl config use-context docker-desktop
   ```

2. **Lint and render (optional but recommended)**
   ```bash
   helm lint ghost-chart
   helm template ghost ghost-chart -f ghost-chart/values.local.yaml > /tmp/ghost-local.yaml
   ```

3. **Install/upgrade the release**
   ```bash
   helm upgrade --install ghost ghost-chart \
     --namespace blog \
     --create-namespace \
     -f ghost-chart/values.local.yaml
   ```
   The local values file disables ingress and persistence, so the pod comes up without requiring a storage class. Adjust it if you want to keep content between restarts.

4. **Wait for the pod**
   ```bash
   kubectl rollout status deploy/ghost -n blog
   kubectl get pods -n blog
   ```

5. **Access Ghost locally**
   ```bash
   kubectl port-forward svc/ghost -n blog 2368:80
   ```
   Then browse to [http://localhost:2368](http://localhost:2368). The Ghost admin setup wizard appears on first launch.
   The local overrides set `mail__transport=Direct` so the initial admin signup works without configuring SMTP.

6. **Clean up when done**
   ```bash
   helm uninstall ghost -n blog
   kubectl delete namespace blog --wait=false
   ```

## Customise the Local Run
- Want persistence? Set `persistence.enabled=true` and pick a storage class available in Docker Desktop (often `hostpath` or `nfs-client` if you installed one).
- Need ingress? Flip `ingress.enabled=true`, set a host, and make sure an ingress controller is running.
- Target a different Ghost image: `--set image.repository=<repo> --set image.tag=<tag>` or edit `ghost-chart/values.local.yaml`.
- Verify tags before pinning: `docker pull ghost:5.92` (or whichever tag you plan to set) should succeed locally.

## Optional: Run the Trivy Job By Hand
The same image scan used in CI can run against your cluster if you want quick CVE feedback:
```bash
IMAGE=ghost:5.92.0
sed "s|__IMAGE_DEST__|$IMAGE|g" ci/kubernetes/trivy.yaml | kubectl apply -n blog -f -
kubectl logs job/trivy-scan -n blog
kubectl delete job trivy-scan -n blog
```
