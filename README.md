# Deploy a 3-Tier Application on Amazon EKS with GitHub Actions

Automate Kubernetes deployments to Amazon Elastic Kubernetes Service (Amazon EKS) using **GitHub Actions and OpenID Connect (OIDC)**. This project demonstrates how to authenticate to AWS without storing long-lived AWS access keys in GitHub.

## Overview

The repository contains Kubernetes manifests and a GitHub Actions workflow that applies those manifests to an existing EKS cluster whenever changes are pushed to the `main` branch.

### Highlights

- **Amazon EKS** for managed Kubernetes.
- **GitHub Actions** for CI/CD automation.
- **AWS IAM OIDC federation** for short-lived AWS credentials.
- **Kubernetes manifests** maintained in the `K8s/` directory.
- **Manual deployment support** using `kubectl`.
- No long-lived AWS access keys required in GitHub Actions.

> **Scope:** This workflow applies Kubernetes manifests that already exist in `K8s/`. It does not build container images or push them to Amazon ECR. If your application requires new images to be built and published, add build and push steps before deploying the manifests.

## Architecture

```text
Developer
   |
   | git push to main
   v
GitHub Repository
   |
   v
GitHub Actions Workflow
   |
   | OIDC token
   v
AWS IAM Role
   |
   | Short-lived AWS credentials
   v
Amazon EKS API
   |
   | kubectl apply -f K8s/
   v
Kubernetes Resources
```

The IAM role trust policy restricts which GitHub repository and branch can assume the role. EKS access is granted separately through an EKS access entry or the legacy `aws-auth` ConfigMap.

## Repository Structure

```text
.
├── K8s/                         # Kubernetes manifests
│   ├── ...                      # Deployments, Services, Ingress, etc.
└── .github/
    └── workflows/
        └── deploy.yml           # GitHub Actions deployment workflow
```

Adjust the workflow filename or manifest paths if your repository uses a different structure.

## Prerequisites

Before you begin, make sure you have:

- An AWS account and an existing Amazon EKS cluster.
- Permissions to configure IAM, EKS access, and (if needed) ECR.
- An EKS cluster reachable by the GitHub-hosted runner through its configured API endpoint and network access rules.
- `aws` CLI and `kubectl` installed locally for manual deployment and troubleshooting.
- Admin access to the GitHub repository.
- Kubernetes manifests in the `K8s/` directory.
- An IAM OIDC identity provider for GitHub Actions.
- An IAM role for the workflow.

## 1. Create the GitHub OIDC Identity Provider

In the AWS console:

1. Open **IAM → Identity providers**.
2. Choose **Add provider**.
3. Select **OpenID Connect**.
4. Set the provider URL to:

   ```text
   https://token.actions.githubusercontent.com
   ```

5. Set the audience to:

   ```text
   sts.amazonaws.com
   ```

6. Create the provider if one does not already exist in the AWS account.

The provider ARN has this format:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

Replace `<AWS_ACCOUNT_ID>` with your AWS account ID.

## 2. Create an IAM Role for GitHub Actions

Create an IAM role with **Web identity** as the trusted entity and select the GitHub OIDC provider. Use a trust policy similar to the following.

Replace the repository name and account ID if yours differ.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YogeshYSRathore/Deploy-a-3-Tier-App-on-Amazon-EKS-Cluster:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

This policy allows the workflow to assume the role only when the OIDC token has the expected audience and comes from the specified repository's `main` branch.

### Least-privilege IAM permissions

Attach a permissions policy that grants only the AWS actions needed by the workflow. At minimum, `aws eks update-kubeconfig` requires `eks:DescribeCluster` for the target cluster. Add other permissions only if the workflow performs those actions—for example, ECR permissions if it builds or pushes images.

**Important:** IAM permissions to call the EKS API do not automatically grant Kubernetes permissions inside the cluster. Configure EKS access separately in Step 5.

### Supporting other branches or events

The trust policy above is intentionally restricted to pushes from `main`. If you need to allow other branches or events, update the `sub` condition carefully. For example, a broader `StringLike` pattern is:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:YogeshYSRathore/Deploy-a-3-Tier-App-on-Amazon-EKS-Cluster:*"
}
```

Use the narrowest condition that supports your workflow. A wildcard can permit additional branches or event contexts, so do not broaden it unless necessary. Pull requests may use a different subject format than branch pushes; validate the expected OIDC subject for your workflow.

## 3. Configure GitHub Actions Variables

In your GitHub repository, open:

**Settings → Secrets and variables → Actions → Variables**

Create these repository variables:

| Variable | Description | Example |
|---|---|---|
| `AWS_REGION` | AWS region where the EKS cluster exists | `us-east-1` |
| `EKS_CLUSTER_NAME` | Exact name of your EKS cluster | `my-eks-cluster` |

The examples are placeholders; use your actual cluster details.

You will also need the IAM role ARN in the workflow. You can either put the non-secret role ARN directly in the YAML or create a repository variable such as `AWS_ROLE_ARN` and reference it as `${{ vars.AWS_ROLE_ARN }}`. The ARN itself is not a password, but using a variable can make the workflow easier to maintain.

Do not store long-lived AWS access keys for this OIDC workflow.

## 4. Configure the GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Amazon EKS

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Deploy Kubernetes manifests
    runs-on: ubuntu-latest

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<GITHUB_ACTIONS_ROLE_NAME>
          aws-region: ${{ vars.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name "${{ vars.EKS_CLUSTER_NAME }}" \
            --region "${{ vars.AWS_REGION }}"

      - name: Validate Kubernetes access
        run: kubectl get nodes

      - name: Preview manifest changes
        run: kubectl diff -f K8s/

      - name: Apply Kubernetes manifests
        run: kubectl apply -f K8s/
```

### Before committing the workflow

1. Replace `<AWS_ACCOUNT_ID>` and `<GITHUB_ACTIONS_ROLE_NAME>` with your actual AWS account ID and IAM role name, or use a repository variable for the role ARN.
2. Verify that `AWS_REGION` and `EKS_CLUSTER_NAME` are configured correctly.
3. Confirm that the IAM role trust policy matches the repository and branch.
4. Confirm that the role has EKS API permissions and is authorized inside the cluster.
5. Ensure the GitHub-hosted runner can reach the EKS API endpoint.

`kubectl diff` may return exit code `1` when differences are found. If that causes the job to stop before applying manifests, handle that behavior explicitly or remove the preview step. The apply step is the actual deployment.

> **Production note:** GitHub Actions examples use pinned major versions for readability. For production, consider pinning third-party actions to reviewed full commit SHAs and reviewing updates regularly.

## 5. Grant the IAM Role Access to EKS

The GitHub Actions role must be authorized by the EKS cluster. There are two common approaches:

- **EKS access entries** (recommended for clusters that support them).
- The legacy `aws-auth` ConfigMap, if that is how your cluster manages IAM principals.

For an EKS access entry, create an entry for the GitHub Actions IAM role and grant a suitable access policy or Kubernetes group permissions. Choose the minimum permissions needed to deploy the resources in `K8s/`.

For example, a workflow deploying Deployments, Services, ConfigMaps, and Ingress resources in a dedicated namespace needs permissions for those resource types in that namespace. It should not automatically receive cluster-admin access.

Check the cluster's authentication mode and existing access configuration before changing it. Use an administrator identity to configure access; the workflow role cannot grant itself permissions unless it already has permission to do so.

## 6. Deploy by Pushing to GitHub

Commit the workflow and manifests, then push to `main`:

```bash
git add K8s/ .github/workflows/deploy.yml
git commit -m "Add EKS deployment workflow"
git push origin main
```

Open the repository's **Actions** tab to watch the workflow.

When the workflow succeeds, it has applied the Kubernetes manifests to the configured EKS cluster. Verify application readiness and external access separately.

## Deploying Manually

You can deploy from a workstation that has AWS CLI, `kubectl`, and access to the cluster.

### 1. Verify the current Kubernetes context

```bash
kubectl config current-context
```

Make sure the context points to the intended cluster before applying any changes.

### 2. Configure AWS credentials

Use your organization's approved AWS authentication method. Then update kubeconfig:

```bash
aws eks update-kubeconfig \
  --name <EKS_CLUSTER_NAME> \
  --region <AWS_REGION>
```

### 3. Preview and apply the manifests

```bash
kubectl diff -f K8s/
kubectl apply -f K8s/
```

### 4. Verify the deployment

```bash
kubectl get deployments -A
kubectl get pods -A
kubectl get services -A
kubectl get ingress -A
```

For a particular namespace, replace the `-A` option with `-n <NAMESPACE>`.

## Troubleshooting

| Error | Likely cause | What to check |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | OIDC provider, trust policy, or workflow permissions are incorrect | Verify provider URL and audience, the exact `sub` condition, role ARN, and `id-token: write`. |
| `AccessDenied` for `eks:DescribeCluster` | IAM role lacks the permission needed to discover cluster details | Grant narrowly scoped `eks:DescribeCluster` access for the target cluster. |
| `Unauthorized` or `Forbidden` from `kubectl` | The IAM role is not authorized for Kubernetes operations | Configure an EKS access entry or `aws-auth` mapping and grant the required Kubernetes permissions. |
| `kubectl` cannot reach the cluster | EKS endpoint is private or network access is restricted | Check endpoint access settings, security groups, routing, and whether the runner can reach the API. |
| `AWS_REGION` or `EKS_CLUSTER_NAME` is empty | GitHub variable is missing or named incorrectly | Check repository variables under **Settings → Secrets and variables → Actions**. |
| `Username and password required` during Docker login | The workflow is using a login action without credentials | For Amazon ECR, use `aws-actions/amazon-ecr-login` after AWS OIDC authentication. |
| `kubectl diff` exits with code `1` | Differences were found between the live resources and manifests | Review the diff. Configure the workflow to handle this exit code if it should continue to apply changes. |
| `! [rejected] main -> main (non-fast-forward)` | The remote branch contains commits not present locally | Run `git pull --rebase origin main`, resolve conflicts, then push. Avoid force-pushing shared branches. |
| Manifests apply, but the app is not ready | A Kubernetes resource may be unhealthy or misconfigured | Inspect pods, events, logs, Services, Ingress resources, image references, and application configuration. |

Useful diagnostics:

```bash
aws sts get-caller-identity
kubectl config current-context
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

## Security Best Practices

- Use OIDC federation instead of storing long-lived AWS access keys in GitHub.
- Restrict the IAM trust policy to the intended repository, branch, and workflow context.
- Apply least-privilege IAM permissions and Kubernetes RBAC.
- Keep production deployments protected with GitHub environment approvals where appropriate.
- Never commit AWS credentials, Kubernetes secrets, or other sensitive values.
- Review third-party GitHub Actions and pin them to full commit SHAs for stronger supply-chain security.
- Protect the `main` branch and require reviews for workflow or infrastructure changes.
- Separate development and production roles and clusters where possible.

## Cleanup

Only remove resources when you are sure they are no longer needed. Review the manifests and target namespace before running deletion commands. For example, to delete resources defined in the same manifest directory:

```bash
kubectl delete -f K8s/
```

This command deletes resources described by those files; it does not delete the EKS cluster itself. Review the manifests first, and do not run it against a production cluster without an approved cleanup plan.

## Contributing

1. Create a feature branch.
2. Make and test your changes.
3. Open a pull request for review.
4. Merge into `main` after checks pass.

## License

Add a `LICENSE` file and update this section with the license you choose for the project.
