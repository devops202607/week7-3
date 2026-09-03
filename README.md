# week7-3

GKE + Flux GitOps demo with Nginx app.

## Repository structure

```
week7-3/
├── app/                        # Nginx app (Dockerfile + config)
├── charts/nginx-app/           # Helm chart
├── flux/                       # Flux GitRepository + HelmRelease
├── terraform/                  # GKE cluster + Flux bootstrap
├── Taskfile.yml                # Local version bump tool
└── .github/workflows/
    └── release.yml             # Build & push image on v* tag
```

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.5
- [Task](https://taskfile.dev/installation/) (Taskfile runner)
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) authenticated
- GitHub PAT with `repo` + `write:packages` scopes

## Deploy

### 1. Provision GKE + Flux via Terraform

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# edit terraform.tfvars — set project_id, github_owner, etc.

export TF_VAR_github_token=<your_github_pat>

terraform init
terraform apply
```

Terraform will:
- Create a 2-node GKE cluster in `us-central1`
- Generate SSH key pair
- Add public key as deploy key to GitHub repo
- Bootstrap Flux into the cluster pointing at `./flux`

### 2. Configure kubectl

```bash
gcloud container clusters get-credentials week7-3 --region us-central1-a --project devops202607
```

### 3. Create ghcr.io pull secret

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=devops202607 \
  --docker-password=<your_github_pat> \
  --namespace=default
```

### 4. Apply Flux manifests

After bootstrap, Flux will automatically pick up `flux/` directory.
Verify:

```bash
kubectl get gitrepository -n flux-system
kubectl get helmrelease -n default
```

## Release workflow

```bash
# bump version, commit, tag and push
task bump -- v1.0.1
```

This will:
1. Update `appVersion` in `charts/nginx-app/Chart.yaml`
2. `git commit` + `git tag v1.0.1`
3. `git push origin main --tags`

GitHub Actions triggers on `v*` tag:
- Builds Docker image `ghcr.io/devops202607/nginx-app:v1.0.1`
- Pushes to GitHub Container Registry

Flux detects chart change → updates HelmRelease → rolling update in GKE.

## App response

```
curl http://<EXTERNAL_IP>

Version:   v1.0.1
Container: nginx-app-7d9f8b-xkv2p
IP:        10.0.0.5
```
