# Demo Portal
```jsonc
  https://www.wingmanhardware.com
```
# Source Code
```jsonc
  https://github.com/chhsBusinessSolution/pos_sys_admin_portal
```

# AWS ECS + ECR + GitHub Actions (Public‑Safe Guide)

This guide shows how to deploy a containerized **frontend (static)** or **Node.js API** to **Amazon ECS (Fargate)** using **Amazon ECR** and **GitHub Actions** with **OIDC** — without storing long‑lived AWS keys. It uses **placeholders** you should replace for your own account.

> ✅ Safe for public repos: no secrets, only example ARNs and policy docs with placeholders.

---

## 0) Replace These Placeholders

| Placeholder | Replace with |
|---|---|
| `<AWS_REGION>` | e.g., `ap-southeast-1` |
| `<ACCOUNT_ID>` | Your 12‑digit AWS account id, e.g., `111122223333` |
| `<ECR_REPO_NAME>` | e.g., `pos-sys-admin-portal` |
| `<CLUSTER_NAME>` | e.g., `pos-admin-cluster` |
| `<SERVICE_NAME>` | e.g., `pos-admin-service` |
| `<IAM_ROLE_NAME>` | e.g., `GitHubOIDCDeployRole` |
| `<REPO_OWNER>` / `<REPO_NAME>` | Your GitHub org/user and repository |

---

## 1) Create an ECR Repository

Console ► **ECR** ► *Private* ► **Create repository**  
- **Name**: `<ECR_REPO_NAME>`  
- **Tag immutability**: optional (recommended)  
- **Scan on push**: optional (recommended)  

Copy the repo URI: `\<ACCOUNT_ID\>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPO_NAME>`

---

## 2) Create an ECS Cluster (Fargate)

Console ► **ECS** ► **Clusters** ► **Create cluster**  
- **Cluster name**: `<CLUSTER_NAME>`  
- **Infrastructure**: **Fargate**  
- Create.

---

## 3) Task Definition (Fargate)

Console ► **ECS** ► **Task definitions** ► **Create**  
- **Launch type**: Fargate  
- **Task role**: *none* (or your app role)  
- **Task execution role**: *auto* or `ecsTaskExecutionRole`  
- **CPU/Memory**: e.g., `0.25 vCPU / 0.5 GB`  
- **Container**:  
  - Name: `web`  
  - Image: `\<ACCOUNT_ID\>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPO_NAME>:latest`  
  - Port mappings: `80/tcp` (for Nginx/static) or your app port  
- Create.

> For HTTPS via ALB, the container still listens on **80**; the ALB terminates TLS on **443**.

---

## 4) Service + Networking

Console ► **ECS** ► **Clusters** ► `<CLUSTER_NAME>` ► **Create service**  
- **Compute**: Fargate  
- **Task definition**: the one from step 3  
- **Service name**: `<SERVICE_NAME>`  
- **Desired tasks**: `1`  
- **Networking**: choose your **VPC** and **public subnets**, enable **public IP**  

### (Optional) Load Balancer (Recommended)
- **Load balancer**: **Application Load Balancer (ALB)**  
- Create **Target Group** (type **IP**, port **80**, protocol **HTTP**)  
- Listener rules:  
  - **HTTPS : 443** → forward to target group (requires ACM cert; see step 6)  
  - **HTTP : 80** → **redirect** to HTTPS 443  

> Without a domain/cert, you can run **HTTP only** (port 80). Keep security group scoped (e.g., your IP) when testing.

---

## 5) Security Groups

- **Service/Tasks SG**: allow **inbound 80/tcp** *from ALB security group* (or from your IP if no ALB).  
- **ALB SG** (if used): allow **80/tcp** and **443/tcp** from the internet (`0.0.0.0/0`) or restricted IP ranges.

---

## 6) HTTPS Certificate (Optional but Recommended)

Console ► **Certificate Manager (ACM)** ► **Request public certificate**  
- `example.com`, `*.example.com` (or just your domain)  
- Complete DNS validation on your registrar.  
- Back in **ALB Listener 443**, select this certificate.

> No domain? You can skip this and serve **HTTP** until you have one.

---

## 7) GitHub OIDC: Short‑Lived AWS Auth (No Static Keys)

### 7.1 Create an IAM Role for GitHub
Console ► **IAM** ► **Roles** ► **Create**  
- **Trusted entity**: **Web identity** ► Provider **GitHub** (or create new OIDC provider `https://token.actions.githubusercontent.com`)  
- **Condition** (repository bound): set `sub` to `repo:<REPO_OWNER>/<REPO_NAME>:ref:refs/heads/master` (adjust as needed).  
- **Role name**: `<IAM_ROLE_NAME>`

Attach this **minimal policy** (replace placeholders):

```jsonc
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECSDeploy",
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeClusters"
      ],
      "Resource": [
        "arn:aws:ecs:<AWS_REGION>:<ACCOUNT_ID>:service/<CLUSTER_NAME>/<SERVICE_NAME>",
        "arn:aws:ecs:<AWS_REGION>:<ACCOUNT_ID>:cluster/<CLUSTER_NAME>"
      ]
    }
  ]
}
```

> If you also tag images with semantic versions in the workflow, this policy is sufficient.

### 7.2 GitHub Repo Secrets
Add these in **GitHub ► Settings ► Secrets and variables ► Actions**:
- `AWS_REGION` = `<AWS_REGION>`  
- `AWS_ACCOUNT_ID` = `<ACCOUNT_ID>`  
- `ECR_REPOSITORY` = `<ECR_REPO_NAME>`  
- `CLUSTER_NAME` = `<CLUSTER_NAME>`  
- `SERVICE_NAME` = `<SERVICE_NAME>`  
- `AWS_ROLE_TO_ASSUME` = `arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME>`

> `GITHUB_TOKEN` is provided automatically by GitHub — **do not create it manually**.

---

## 8) GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Build & Deploy to ECS (latest)

on:
  push:
    branches: [ master ]
    tags: ['v*']           # optional: run on version tags too

permissions:
  id-token: write          # for OIDC
  contents: read

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPO:   ${{ secrets.ECR_REPOSITORY }}
  CLUSTER:    ${{ secrets.CLUSTER_NAME }}
  SERVICE:    ${{ secrets.SERVICE_NAME }}

jobs:
  deploy:
    if: "!contains(github.event.head_commit.message, 'skip-deploy')"  # opt-out flag
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build & push image (latest + sha)
        env:
          IMAGE: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPO }}
          SHA:   ${{ github.sha }}
        run: |
          docker build -t "$IMAGE:latest" -t "$IMAGE:$SHA" .
          docker push "$IMAGE:latest"
          docker push "$IMAGE:$SHA"

      - name: Force new deployment
        run: |
          aws ecs update-service             --cluster "$CLUSTER"             --service "$SERVICE"             --force-new-deployment

  release:
    needs: deploy
    if: startsWith(github.ref, 'refs/tags/v')    # only when tagging vX.Y.Z
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ github.ref_name }}
          name: Release ${{ github.ref_name }}
          body: |
            - image: ${{ env.ECR_REPO }}
            - sha:   ${{ github.sha }}
```

**Usage tips**
- Auto-release: push a tag like `v1.2.3` to trigger the `release` job.  
- Skip deploy: include `skip-deploy` in your commit message.

---

## 9) Dockerfile (example: Vite + Nginx)

```dockerfile
# Step 1: build
FROM node:23-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Step 2: serve
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Minimal `nginx.conf` for SPA:
```nginx
server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;
  location / {
    try_files $uri /index.html;
  }
}
```

---

## 10) Verify

- GitHub ► **Actions** shows build logs.  
- ECR ► **Images** contains `latest` + image by sha.  
- ECS ► **Service** shows **1 Running** task.  
- If using ALB, open the **ALB DNS name**. Otherwise, use the task **Public IP**.

---

## 11) Security Checklist (Public Repos)

- Do **not** commit any `.env`, keys, or real account numbers.  
- Lock down the IAM role **trust policy** to your repo/branch/tag.  
- Prefer ALB + ACM when exposing to the internet.  
- Keep SGs tight; only open what you need.

---

## 12) Troubleshooting

- **task starts then stops** → container port mismatch; ensure container exposes **80** and target group health checks point to `/` on **80**.  
- **ALB 502/504** → security groups not linked correctly (ALB SG must be allowed into service SG).  
- **No image found** → check the ECR repo name/URI and workflow push permissions.  
- **OIDC fails** → trust relationship conditions (repo, branch) don’t match your push.

---

**License**: MIT — copy and adapt freely.
