# ShopVerse - Full-Stack E-Commerce Application

A production-ready 3-tier e-commerce web application built with React, Go (Fiber), and MySQL, deployed on AWS EKS using Helm charts.



<img width="1280" height="720" alt="Shopverse project - Thumbnail" src="https://github.com/user-attachments/assets/fd0a3efe-aeb3-48dc-8912-714517580010" />


## YouTube Video Link:

```
https://youtu.be/XQrJrf6pUvk?si=XGsLPOm6YC1AEJFk
```

## Architecture:

```
                    Internet (HTTPS)
                          |
              +-----------v------------+
              | Gateway (Gateway API)  |   managed by the infra repo
              +-----------+------------+
                          |  HTTPRoute (this repo's Helm chart)
            +-------------+--------------+
            |                            |
     /api/*, /health                    /*
            |                            |
   +--------v---------+       +----------v-----------+
   | Backend Service  |       | Frontend Service     |
   | Go + Fiber :8080 |       | React + nginx :80    |
   | ClusterIP        |       | ClusterIP            |
   +--------+---------+       +----------------------+
            |
            |  DB credentials + JWT secret synced from
            |  AWS Secrets Manager by External Secrets Operator
   +--------v---------+
   | Amazon RDS MySQL |   managed by the infra repo
   +------------------+
```

| Environment | Hostname              | EKS cluster | ECR repositories                                              | Deploys                  |
|-------------|-----------------------|-------------|---------------------------------------------------------------|--------------------------|
| dev         | `dev.kmanavar.online` | dev         | `app-dev-shopverse-frontend`, `app-dev-shopverse-backend`     | Automatically on `main`  |
| prod        | `kmanavar.online`     | prod        | `app-prod-shopverse-frontend`, `app-prod-shopverse-backend`   | After manual approval    |

## Tech Stack

| Layer    | Technology                                           |
|----------|------------------------------------------------------|
| Frontend | React 18, TailwindCSS, Vite, nginx                   |
| Backend  | Go 1.27, Fiber, GORM, JWT                            |
| Database | Amazon RDS for MySQL                                 |
| Platform | AWS EKS, ECR, Gateway API, External Secrets Operator |
| CI/CD    | GitHub Actions (OIDC), Helm 4, Trivy                 |
| IaC      | Terraform, in the separate infra repo                |

## API Endpoints

| Method | Endpoint            | Auth     | Description             |
|--------|---------------------|----------|-------------------------|
| POST   | /api/auth/register  | No       | Register new user       |
| POST   | /api/auth/login     | No       | Login, returns JWT      |
| GET    | /api/products       | No       | List products           |
| GET    | /api/products/:id   | No       | Get single product      |
| POST   | /api/products       | JWT      | Create product (admin)  |
| GET    | /api/cart           | JWT      | Get user's cart         |
| POST   | /api/cart           | JWT      | Add item to cart        |
| PUT    | /api/cart/:id       | JWT      | Update cart item qty    |
| DELETE | /api/cart/:id       | JWT      | Remove cart item        |
| GET    | /api/orders         | JWT      | Get user's orders       |
| POST   | /api/orders         | JWT      | Place order from cart   |
| GET    | /health             | No       | Health check            |

---

## Local Development

### Prerequisites
- Docker & Docker Compose
- Node.js 22+ (for frontend dev)
- Go 1.26+ (for backend dev)

### Quick Start with Docker Compose

```bash
# Clone the repo
git clone <repo-url> && cd shopverse

# Start all services
docker compose up --build

# Access the app
# Frontend: http://localhost:3000
# Backend:  http://localhost:8080/health  (API only, "/" returns 404)
```

### Run Frontend Individually (Hot Reload)

```bash
cd frontend
npm ci
npm run dev
# Runs on http://localhost:3000 with proxy to backend
```

### Run Backend Individually

```bash
cd backend
# Start only the database from docker-compose
docker compose up -d mysql
DB_HOST=localhost DB_USER=shopverse DB_PASSWORD=shopverse123 DB_NAME=shopverse go run ./cmd/main.go
```

---

## Deployment to AWS EKS (GitHub Actions)

The infrastructure (VPC, EKS clusters, RDS, ECR, OIDC roles, Gateway) is created by the separate infra repo. This repo only builds the application and deploys it with Helm.

```
pull request  ->  CI: Go vet/test, frontend lint/build, Helm lint, image build + Trivy scan
push to main  ->  CI  ->  dev (automatic)  ->  prod (waits for approval)
```

- Images are built and scanned **once**. Dev receives the scanned images. Prod receives a copy of the same dev images, tagged with the same git SHA.
- GitHub Actions signs in to AWS with **OIDC**. There are no AWS access keys in GitHub.
- A failed Helm upgrade rolls back automatically, and each deploy ends with a smoke test against the public hostname.

| Workflow | File | Purpose |
|----------|------|---------|
| CI | [.github/workflows/ci.yaml](.github/workflows/ci.yaml) | Runs on PRs. CD also calls it. |
| CD | [.github/workflows/cd.yaml](.github/workflows/cd.yaml) | `main` -> dev -> prod |
| Deploy environment | [.github/workflows/deploy-env.yaml](.github/workflows/deploy-env.yaml) | Reusable: push or promote images, `helm upgrade`, smoke test |

**One-time setup** (GitHub Environments, variables, EKS access entries, RBAC, Secrets Manager, IAM): see [docs/cicd.md](docs/cicd.md).

### Check a deployment

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>

kubectl -n shopverse get deploy,pods,svc,httproute
kubectl -n shopverse get externalsecret,secretstore   # STATUS should be SecretSynced / Valid
helm -n shopverse history shopverse
```

### Roll back

```bash
helm -n shopverse history shopverse
helm -n shopverse rollback shopverse <REVISION> --wait
```

Or re-run the CD workflow for an older commit (Actions > CD > Run workflow), which redeploys that commit's image.

### Scale

Replica counts are in `helm/shopverse/values-dev.yaml` and `values-prod.yaml`. Change them there and merge to `main`. A `kubectl scale` change is overwritten by the next deploy.

---

## Querying the Database

The database runs on Amazon RDS, and the app reads its credentials from the `shopverse-secret` Kubernetes Secret, which External Secrets Operator syncs from Secrets Manager. To open a MySQL shell, start a temporary client pod inside the cluster. The cluster can reach RDS; your laptop usually can't.

```bash
kubectl -n shopverse run mysql-client --rm -it --restart=Never --image=mysql:8.0 \
  --overrides='{"spec":{"containers":[{"name":"mysql-client","image":"mysql:8.0","stdin":true,"tty":true,
    "command":["sh","-c","MYSQL_PWD=\"$DB_PASSWORD\" exec mysql -h \"$DB_HOST\" -P \"$DB_PORT\" -u \"$DB_USER\" \"$DB_NAME\""],
    "envFrom":[{"configMapRef":{"name":"shopverse-config"}},{"secretRef":{"name":"shopverse-secret"}}]}]}}'
```

The pod reads the host from the `shopverse-config` ConfigMap and the credentials from `shopverse-secret`, so the password never shows up in your terminal or shell history. Running this needs permission to create and attach to pods. The CI deploy role doesn't have that permission, so use an admin or operator role.

### Useful queries

Once inside the MySQL shell (you'll see `mysql>` prompt):

#### View all tables
```sql
SHOW TABLES;
```
This shows all 5 tables: `users`, `products`, `orders`, `order_items`, `cart_items`.

#### View registered users
```sql
SELECT id, name, email, created_at FROM users;
```
Shows all users who registered through the app. Passwords are hashed with bcrypt so they are not shown here.

#### View all products
```sql
SELECT id, name, category, price, original_price, rating, badge FROM products;
```
Lists all 28 seeded products with their category, pricing, rating, and badge info.

#### View products grouped by category
```sql
SELECT category, COUNT(*) AS total_products,
       ROUND(AVG(price), 2) AS avg_price,
       ROUND(MIN(price), 2) AS min_price,
       ROUND(MAX(price), 2) AS max_price
FROM products
GROUP BY category
ORDER BY total_products DESC;
```
Shows product count and price stats per category (Electronics, Clothing, Accessories, Food & Drinks, Sports, Home & Living).

#### View all orders with customer info
```sql
SELECT
    o.id AS order_id,
    u.name AS customer_name,
    u.email AS customer_email,
    o.total_amount,
    o.status,
    o.created_at AS order_date
FROM orders o
JOIN users u ON o.user_id = u.id
ORDER BY o.created_at DESC;
```
**How this works:**
- `JOIN users u ON o.user_id = u.id` links each order to the user who placed it
- `ORDER BY o.created_at DESC` shows newest orders first
- `o.status` shows the order status (e.g., pending, completed)

#### View order items with product details
```sql
SELECT
    oi.order_id,
    p.name AS product_name,
    p.category,
    oi.quantity,
    oi.price AS unit_price,
    (oi.quantity * oi.price) AS subtotal
FROM order_items oi
JOIN products p ON oi.product_id = p.id
ORDER BY oi.order_id, p.name;
```
**How this works:**
- `order_items` stores what was purchased in each order
- `JOIN products p ON oi.product_id = p.id` links item to its product details
- `(oi.quantity * oi.price)` calculates the subtotal for each line item

#### View complete order breakdown (orders + items together)
```sql
SELECT
    o.id AS order_id,
    u.name AS customer,
    p.name AS product,
    oi.quantity,
    oi.price AS unit_price,
    (oi.quantity * oi.price) AS subtotal,
    o.total_amount AS order_total,
    o.status,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
ORDER BY o.id, p.name;
```
This is the most complete view - joins 4 tables to show who ordered what, quantities, prices, and order status.

#### View current cart items
```sql
SELECT
    ci.id AS cart_item_id,
    u.name AS customer,
    p.name AS product,
    p.category,
    ci.quantity,
    p.price AS unit_price,
    (ci.quantity * p.price) AS subtotal
FROM cart_items ci
JOIN users u ON ci.user_id = u.id
JOIN products p ON ci.product_id = p.id
ORDER BY u.name;
```
Shows items currently in users' shopping carts (items that haven't been ordered yet).

#### Dashboard summary
```sql
SELECT
    (SELECT COUNT(*) FROM users) AS total_users,
    (SELECT COUNT(*) FROM products) AS total_products,
    (SELECT COUNT(*) FROM orders) AS total_orders,
    (SELECT COALESCE(SUM(total_amount), 0) FROM orders) AS total_revenue,
    (SELECT COUNT(*) FROM cart_items) AS items_in_carts;
```
A quick overview of the entire application's data - total users, products, orders, revenue, and active cart items.

#### Exit MySQL shell
```sql
EXIT;
```

---

## Project Structure

```
shopverse/
├── frontend/                    # React + TailwindCSS (Vite), served by nginx
│   ├── Dockerfile               # Node 22 build -> nginx
│   └── nginx.conf               # SPA routing + /api proxy to shopverse-backend-svc
├── backend/                     # Go + Fiber REST API
│   ├── cmd/main.go              # Entry point, routes
│   ├── internal/                # handlers, models, database (migrations + seed), middleware (JWT)
│   └── Dockerfile               # Go build -> distroless (non-root)
├── helm/shopverse/              # Helm chart
│   ├── values.yaml              # Shared defaults
│   ├── values-dev.yaml          # Dev overrides (dev.kmanavar.online)
│   ├── values-prod.yaml         # Prod overrides (kmanavar.online)
│   └── templates/
│       ├── configmap.yaml           # DB host/port/name, CORS origin
│       ├── external-secret.yaml     # ServiceAccount (IRSA) + SecretStore + ExternalSecret
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml     # ClusterIP
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml    # ClusterIP
│       ├── httproute.yaml           # Gateway API routing
│       └── pdb.yaml                 # PodDisruptionBudgets (when replicas > 1)
├── k8s/bootstrap/
│   └── deployer-rbac.yaml       # One-time: namespace + least-privilege RBAC for CI
├── .github/workflows/
│   ├── ci.yaml                  # PR checks; called by CD
│   ├── cd.yaml                  # main -> dev -> prod (approval)
│   └── deploy-env.yaml          # Reusable per-environment deploy
├── docs/cicd.md                 # One-time CI/CD + cluster setup
├── docker-compose.yml           # Local development (includes a local MySQL)
└── README.md
```

## Troubleshooting

The deploy job prints pods, events, ExternalSecret status, and backend logs when it fails. The common causes:

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| `configure-aws-credentials` fails: *Not authorized to perform sts:AssumeRoleWithWebIdentity* | The role's trust policy doesn't match this repo or environment | Trust policy `sub` must be `repo:<owner>/<repo>:environment:dev` (or `:prod`) |
| `helm upgrade` fails with *forbidden* | Missing EKS access entry or RBAC | `docs/cicd.md` steps 3–4 |
| Backend pods stuck in `CreateContainerConfigError` | `shopverse-secret` hasn't been synced yet | `kubectl -n shopverse describe externalsecret shopverse-secret` |
| ExternalSecret `SecretSyncedError` | IRSA role can't read the secret, or the secret name is wrong | `vars.SECRETS_ROLE_ARN`, `vars.RDS_SECRET_NAME`, IAM policy |
| Backend logs `Database connection attempt ... failed` | RDS security group doesn't allow the EKS nodes/pods on 3306 | Infra repo security groups |
| Smoke test fails but pods are Ready | HTTPRoute not accepted by the Gateway, or DNS/TLS not set up | `kubectl -n shopverse describe httproute shopverse` (look at `status.parents`) |
| Backend errors after a few days, `Access denied for user` | RDS rotated the password; running pods still have the old one | Restart backend, or install Reloader (see `docs/cicd.md`) |

---

## License

MIT
