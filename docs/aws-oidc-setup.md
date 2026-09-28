# AWS OIDC Setup for GitHub Actions → ECR

This document explains the one-time IAM setup required so that GitHub Actions
can push images to Amazon ECR **without** storing long-lived AWS access keys as
GitHub secrets.

## How it works

```
GitHub Actions runner
        │
        │  (OIDC token from GitHub)
        ▼
AWS STS AssumeRoleWithWebIdentity
        │
        │  (short-lived credentials, ~1 hour)
        ▼
   IAM Role  →  ECR permissions
```

GitHub acts as an OIDC IdP. AWS trusts GitHub's OIDC tokens and exchanges them
for temporary credentials. No long-lived secret ever leaves GitHub.

---

## 1. Add GitHub as an OIDC Identity Provider in each AWS account

Do this **once per account** (both prod and non-prod).

> **Do not run these commands** — they modify AWS accounts. Run them yourself
> when ready. Shown here for reference only.

```bash
# Check if the provider already exists first
aws iam list-open-id-connect-providers --profile <PROFILE>

# If it doesn't exist, create it
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --profile <PROFILE>
```

---

## 2. Create the ECR push policy

Create a file `ecr-push-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "ECRPush",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/kubernetes-events-exporter"
    }
  ]
}
```

---

## 3. Create the IAM role (do this in each account)

### Trust policy

Create `trust-policy.json` — replace `YOUR_GITHUB_ORG` with your org/username:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/kubernetes-events-exporter:*"
        }
      }
    }
  ]
}
```

The `StringLike` condition scopes trust to **only** your fork's repo, so no
other GitHub repo can assume this role.

### Create the role

```bash
# Create role with trust policy
aws iam create-role \
  --role-name github-actions-ecr-push \
  --assume-role-policy-document file://trust-policy.json \
  --profile <PROFILE>

# Attach the inline ECR push policy
aws iam put-role-policy \
  --role-name github-actions-ecr-push \
  --policy-name ecr-push \
  --policy-document file://ecr-push-policy.json \
  --profile <PROFILE>
```

After creation you'll get an ARN like:
```
arn:aws:iam::<ACCOUNT_ID>:role/github-actions-ecr-push
```

Put these ARNs into the workflow files:

| File | Variable | Account |
|------|----------|---------|
| `.github/workflows/release.yml` | `role_arn` (prod matrix entry) | `664404405793` |
| `.github/workflows/release.yml` | `role_arn` (nonprod matrix entry) | `381491828628` |
| `.github/workflows/ci.yml` | `role-to-assume` in `push-nonprod` job | `381491828628` |

---

## 4. Create the ECR repositories (once per account)

```bash
# Non-prod
aws ecr create-repository \
  --repository-name kubernetes-events-exporter \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true \
  --profile <NONPROD_PROFILE>

# Prod
aws ecr create-repository \
  --repository-name kubernetes-events-exporter \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true \
  --profile <PROD_PROFILE>
```

The resulting URIs will be:
- **Non-prod**: `381491828628.dkr.ecr.ap-south-1.amazonaws.com/kubernetes-events-exporter`
- **Prod**: `664404405793.dkr.ecr.ap-south-1.amazonaws.com/kubernetes-events-exporter`

---

## 5. EKS node IAM permissions (so pods can pull the image)

Attach the AWS-managed policy to your EKS node role — no `imagePullSecret`
needed:

```
AmazonEC2ContainerRegistryPullOnly
```

Or if using IRSA / Pod Identity for the exporter's service account, add the
same ECR read permissions scoped to that role instead.

---

## 6. GitHub secret for the Helm chart version bump (optional)

The `release.yml` workflow commits back to `main` after bumping `Chart.yaml`.
By default it uses `GITHUB_TOKEN`, which works but won't trigger other
`push` workflows.

If you want downstream workflows to fire on that commit, create a PAT with
`repo` scope and store it as:

```
GitHub → Settings → Secrets and variables → Actions → New repository secret
Name: RELEASE_BOT_TOKEN
Value: <your PAT>
```

---

## Summary: what to replace after reading this doc

| Location | Placeholder | Replace with |
|----------|-------------|--------------|
| `charts/kubernetes-events-exporter/Chart.yaml` | `YOUR_GITHUB_ORG` | your GitHub org/username |
| `charts/kubernetes-events-exporter/values.yaml` | ECR URI placeholder | actual ECR URI |
| `.github/workflows/ci.yml` | `AWS_REGION: ap-south-1` | your actual region (if different) |
| `.github/workflows/ci.yml` | `role-to-assume` ARN | non-prod IAM role ARN |
| `.github/workflows/release.yml` | `AWS_REGION: ap-south-1` | your actual region (if different) |
| `.github/workflows/release.yml` | both `role_arn` values | actual IAM role ARNs |
| `docs/aws-oidc-setup.md` | `YOUR_GITHUB_ORG` | your GitHub org/username |
| `docs/aws-oidc-setup.md` | `<REGION>` / `<ACCOUNT_ID>` | actual values |
