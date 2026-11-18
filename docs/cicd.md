# CI/CD setup: dev and gated prod on EKS

These are one-time steps to connect this repo's pipeline to the dev and prod infrastructure created by the infra repo.

```
PR            -> ci.yaml : Go vet/test, frontend lint/build, Helm lint, image build + Trivy scan
push to main  -> cd.yaml : ci.yaml -> dev (automatic) -> prod (manual approval)
```

| | dev | prod |
|---|---|---|
| GitHub Environment | `dev` | `prod` (required reviewers) |
| Hostname | `dev.kmanavar.online` | `kmanavar.online` |
| ECR repos | `app-dev-shopverse-frontend`, `app-dev-shopverse-backend` | `app-prod-shopverse-frontend`, `app-prod-shopverse-backend` |
| Images | Pushed from the CI artifact (the scanned build) | Copied from the dev repos, same git SHA tag |

---

## 0. What the infra repo must provide

Check these before the first run:

- [ ] Two EKS clusters (dev, prod), each with an **IAM OIDC provider** (needed for IRSA).
- [ ] **Gateway API** CRDs (standard channel, `gateway.networking.k8s.io/v1`), a Gateway controller, and a **Gateway** on each cluster that:
  - has an HTTPS listener with a TLS certificate for the environment's hostname,
  - allows routes from the `shopverse` namespace (`allowedRoutes.namespaces.from: Selector` with label `shopverse.io/gateway-access: "true"`, or `All`),
  - redirects HTTP to HTTPS. That redirect belongs on the Gateway, not in this chart.
- [ ] **DNS** records for `dev.kmanavar.online` and `kmanavar.online` pointing to each Gateway's load balancer.
- [ ] **External Secrets Operator** installed on both clusters (provides `external-secrets.io/v1`).
- [ ] **RDS for MySQL** for each environment, with a database named `shopverse` and the master credentials stored in Secrets Manager (RDS-managed secret, JSON keys `username` and `password`).
- [ ] The RDS security group allows port 3306 from the EKS nodes or pods.
- [ ] The four ECR repositories listed above. Turn on **tag immutability** and **scan on push**.
- [ ] One GitHub OIDC IAM role per environment (see step 1).

---

## 1. IAM: GitHub deploy roles (one per environment)

### Trust policy

The role must trust **this** repository and **its GitHub Environment**. The trust policy written for the infra repo does not match this repo, so `configure-aws-credentials` will fail until you update it.

Repositories created on or after 15 July 2026 put the org and repo IDs in the `sub` claim ([source](https://github.com/aws-actions/configure-aws-credentials)). Because this repo is new, its claim will look like this:

```
repo:<OWNER>@<ORG_ID>/<REPO>@<REPO_ID>:environment:dev
```

Older repositories leave out the `@<ID>` parts: `repo:<OWNER>/<REPO>:environment:dev`.

To get the IDs, run `gh api repos/<OWNER>/<REPO> --jq '.id, .owner.id'`.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<OWNER>@<ORG_ID>/<REPO>@<REPO_ID>:environment:prod"
      }
    }
  }]
}
```

Scope the dev role to `:environment:dev` and the prod role to `:environment:prod`. Then a run that hasn't passed the prod approval gate can't assume the prod role.

> **Don't reuse the infra repo's Terraform role** if it has admin-level permissions. An app pipeline only needs the permissions below. Create a separate `shopverse-app-deploy-<env>` role in the infra repo.

### Permissions

**Dev role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "ecr:GetAuthorizationToken", "Resource": "*" },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability", "ecr:InitiateLayerUpload", "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload", "ecr:PutImage", "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer", "ecr:DescribeImages"
      ],
      "Resource": [
        "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/app-dev-shopverse-frontend",
        "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/app-dev-shopverse-backend"
      ]
    },
    { "Effect": "Allow", "Action": "eks:DescribeCluster", "Resource": "arn:aws:eks:<REGION>:<ACCOUNT_ID>:cluster/<DEV_CLUSTER>" }
  ]
}
```

**Prod role:** the same statements with the `app-prod-*` repositories and the prod cluster, plus **read-only access to the dev repositories** so it can copy the images:

```json
{
  "Effect": "Allow",
  "Action": ["ecr:BatchGetImage", "ecr:GetDownloadUrlForLayer", "ecr:BatchCheckLayerAvailability"],
  "Resource": [
    "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/app-dev-shopverse-frontend",
    "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/app-dev-shopverse-backend"
  ]
}
```

---

## 2. IAM: secrets reader role for External Secrets (one per environment)

The chart creates the ServiceAccount `shopverse/shopverse-secrets` with an IRSA annotation. ESO uses that ServiceAccount to read Secrets Manager. The application pods never get AWS credentials.

Trust policy (use each cluster's OIDC provider):

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<CLUSTER_OIDC_ID>" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.<REGION>.amazonaws.com/id/<CLUSTER_OIDC_ID>:sub": "system:serviceaccount:shopverse:shopverse-secrets",
      "oidc.eks.<REGION>.amazonaws.com/id/<CLUSTER_OIDC_ID>:aud": "sts.amazonaws.com"
    }
  }
}
```

Permissions:

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
  "Resource": [
    "<RDS_SECRET_ARN>",
    "arn:aws:secretsmanager:<REGION>:<ACCOUNT_ID>:secret:shopverse/<ENV>/jwt-secret-*"
  ]
}
```

If the RDS secret is encrypted with a customer-managed KMS key, also add `kms:Decrypt` on that key.

### JWT signing secret

Create one per environment. The backend falls back to a hard-coded development key if `JWT_SECRET` is empty, so this secret must exist and must not be empty.

```bash
aws secretsmanager create-secret --name shopverse/dev/jwt-secret \
  --secret-string "$(openssl rand -base64 48)"
aws secretsmanager create-secret --name shopverse/prod/jwt-secret \
  --secret-string "$(openssl rand -base64 48)"
```

To use different names, change `secrets.jwt.secretName` in `helm/shopverse/values-<env>.yaml`.

---

## 3. EKS access entries (one per cluster)

This step maps the GitHub deploy role to the Kubernetes group `shopverse-deployers`. No AWS access policy is attached. RBAC in step 4 grants the actual permissions.

```bash
aws eks create-access-entry \
  --cluster-name <DEV_CLUSTER> \
  --principal-arn <DEV_DEPLOY_ROLE_ARN> \
  --kubernetes-groups shopverse-deployers \
  --type STANDARD

aws eks create-access-entry \
  --cluster-name <PROD_CLUSTER> \
  --principal-arn <PROD_DEPLOY_ROLE_ARN> \
  --kubernetes-groups shopverse-deployers \
  --type STANDARD
```

The cluster's authentication mode must include `API`: either `API` or `API_AND_CONFIG_MAP`.

---

## 4. Namespace and RBAC (one per cluster, as cluster admin)

```bash
aws eks update-kubeconfig --name <CLUSTER> --region <REGION>
kubectl apply -f k8s/bootstrap/deployer-rbac.yaml
```

This creates the `shopverse` namespace, labelled so the Gateway accepts routes from it. It also creates a Role that lets the deployer manage only what the chart deploys, and only in that namespace: Deployments, Services, ConfigMaps, Secrets (Helm stores release state there), ServiceAccounts, PDBs, HTTPRoutes, ExternalSecrets, and SecretStores. It can also read Pods, Events, and logs for diagnostics.

To check it, impersonate the group:

```bash
kubectl auth can-i create httproutes -n shopverse --as=ci --as-group=shopverse-deployers   # yes
kubectl auth can-i create deployments -n default   --as=ci --as-group=shopverse-deployers   # no
```

---

## 5. GitHub settings

### Environments (Settings > Environments)

| Setting | `dev` | `prod` |
|---|---|---|
| Deployment branches | `main` only | `main` only |
| Required reviewers | none | 1+ people or a team |
| Prevent self-review | – | on |

### Variables

**Repository variable** (Settings > Secrets and variables > Actions > Variables):

| Name | Example |
|---|---|
| `AWS_REGION` | `us-east-1` |

**Environment variables**, set separately on `dev` and on `prod`:

| Name | Description | Example |
|---|---|---|
| `AWS_ROLE_ARN` | Deploy role from step 1 | `arn:aws:iam::123456789012:role/shopverse-app-deploy-dev` |
| `EKS_CLUSTER_NAME` | Cluster name | `app-dev-eks` |
| `DB_HOST` | RDS endpoint | `app-dev-shopverse.xxxx.us-east-1.rds.amazonaws.com` |
| `RDS_SECRET_NAME` | Name or ARN of the RDS-managed secret | `rds!db-1a2b3c4d-...` |
| `SECRETS_ROLE_ARN` | ESO reader role from step 2 | `arn:aws:iam::123456789012:role/shopverse-secrets-dev` |
| `GATEWAY_NAME` | Gateway the HTTPRoute attaches to | `main-gateway` |
| `GATEWAY_NAMESPACE` | Namespace of that Gateway | `gateway-system` |
| `GATEWAY_SECTION` | *(optional)* listener name to attach to | `https` |

None of these are secret. GitHub needs **no secrets** at all for this pipeline. The database password and JWT key stay in Secrets Manager and never pass through GitHub.

### Branch protection for `main`

- Require a pull request before merging.
- Require these status checks: `Backend (Go)`, `Frontend (React)`, `Helm chart`, `Image (frontend)`, `Image (backend)`.

---

## 6. First run

1. Merge to `main`, or start **Actions > CD > Run workflow**.
2. CI builds and scans the images. The **Dev** job pushes them to the dev ECR repos, deploys, and runs the smoke test against `https://dev.kmanavar.online`.
3. The **Prod** job waits for approval. After a reviewer approves, it copies the same images into the prod repos and deploys to `https://kmanavar.online`.

If a Helm upgrade fails, it rolls back to the previous release (`--rollback-on-failure`), and the job prints pods, events, ExternalSecret status, and backend logs.

If several commits land while prod is waiting, GitHub keeps only the newest pending prod deploy. The older pending runs are cancelled, so a reviewer always approves the latest commit that passed dev.

---

## Known limitations and follow-ups

- **RDS password rotation:** RDS-managed secrets rotate every 7 days by default. ESO refreshes `shopverse-secret` every hour, but running pods keep the old value in their environment until they restart, so new DB connections fail after a rotation. Fix it in one of these ways:
  - Install [Stakater Reloader](https://github.com/stakater/Reloader) and set this in `values-<env>.yaml`:
    ```yaml
    backend:
      deploymentAnnotations:
        secret.reloader.stakater.com/reload: shopverse-secret
    ```
  - Or turn off automatic rotation for the RDS secret.
- **The app uses the RDS master user.** A dedicated MySQL user with privileges only on the `shopverse` schema is safer. Create it, store it in its own secret, and point `RDS_SECRET_NAME` at that secret.
- **No TLS to RDS:** the backend DSN doesn't set `tls=true`. It works while the RDS parameter `require_secure_transport` is `OFF`. Turning that parameter on requires a small backend change.
- **Migrations and seeding run at startup** in every backend replica (`AutoMigrate` and `seedProducts` in `internal/database/db.go`). Concurrent first starts can race, and prod gets the demo product catalogue. Move these into a Helm pre-upgrade Job and turn seeding off in prod.
- **The backend has no tests yet**, so `go test` passes trivially.
- **The frontend container runs nginx as root** (`nginx:alpine`). Switching to `nginxinc/nginx-unprivileged` on port 8080 would let the chart enforce `runAsNonRoot`.
