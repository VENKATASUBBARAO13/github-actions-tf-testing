# Terraform CI/CD with GitHub Actions and AWS

A hands-on project that uses **GitHub Actions** to automate Terraform validation, planning, and deployment of AWS infrastructure. The workflow is manually triggered, stores AWS credentials as GitHub Actions secrets, and uses a protected `production` environment to require approval before Terraform applies changes.

## Project Overview

This repository demonstrates a Terraform CI/CD workflow that provisions a basic AWS network:

- An Amazon VPC
- A subnet inside that VPC
- Terraform-managed infrastructure using AWS provider credentials
- A GitHub Actions workflow with separate **Plan** and **Apply** jobs
- A manual approval gate before the production deployment

**Repository:** [github-actions-tf-testing](https://github.com/VENKATASUBBARAO13/github-actions-tf-testing)

## Architecture and Workflow

The pipeline follows this sequence:

```text
Manual workflow_dispatch
        |
        v
  Terraform deploy job
        |
        +--> Checkout repository
        +--> Setup Terraform
        +--> terraform init
        +--> terraform validate
        +--> terraform plan -out=tfplan
        +--> Upload tfplan artifact
                    |
                    v
          production environment
           (required reviewer)
                    |
             Manual approval
                    |
                    v
           Terraform Apply job
                    |
                    +--> Checkout repository
                    +--> Setup Terraform
                    +--> terraform init
                    +--> Download tfplan artifact
                    +--> terraform apply -auto-approve tfplan
                    |
                    v
              AWS resources
             VPC + Subnet
```

The plan and apply jobs are separate. The apply job depends on the plan job and references the `production` environment, so GitHub pauses it until the configured reviewer approves the deployment.

## Technologies Used

- **Cloud:** Amazon Web Services (AWS)
- **Infrastructure as Code:** Terraform
- **CI/CD:** GitHub Actions
- **Version control:** Git and GitHub
- **Workflow runner:** GitHub-hosted Ubuntu runner
- **Terraform provider:** AWS
- **AWS region configured in workflow:** `us-east-1` (US East – N. Virginia)

## Repository Structure

```text
github-actions-tf-testing/
├── .github/
│   └── workflows/
│       └── terraform-cicd.yml
├── main.tf
├── provider.tf
└── variables.tf
```

| File | Purpose |
|---|---|
| `.github/workflows/terraform-cicd.yml` | GitHub Actions workflow for Terraform plan and approval-gated apply |
| `main.tf` | Defines the AWS VPC and subnet resources |
| `provider.tf` | Configures the AWS Terraform provider |
| `variables.tf` | Defines configurable VPC and subnet CIDR blocks and Name tags |

### Infrastructure Configuration

The current Terraform configuration uses these defaults:

| Resource / setting | Default |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| VPC Name tag | `dev-vpc` |
| Subnet CIDR | `10.0.1.0/24` |
| Subnet Name tag | `dev-subnet` |
| AWS region | `us-east-1` |

The values are declared in `variables.tf`. Review and change them to suit your environment before deployment.

## Setup Guide

### Step 1: Create AWS credentials

1. Sign in to GitHub and open this repository.
2. Navigate to **Settings**.
3. Under **Secrets and variables**, select **Actions**.
4. In **Repository secrets**, click **New repository secret**.
5. Create the following two secrets:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your AWS IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | Your AWS IAM secret access key |

The workflow reads these values from GitHub's encrypted secrets store:

```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: us-east-1
```

**Security:** Never put AWS credentials directly in Terraform files or commit them to Git. Use an IAM identity with only the permissions required for this project. Prefer short-lived credentials through GitHub OIDC for production-grade deployments.

### Step 2: Create the production environment

1. Open the repository **Settings**.
2. Select **Environments** from the left navigation.
3. Click **New environment**.
4. Enter the environment name exactly as `production`.
5. Open the environment configuration.
6. Enable **Required reviewers** and add the reviewer account(s).
7. Save the protection rules.

The environment name must match the workflow declaration:

```yaml
environment: production
```

This protection rule makes the apply job wait for a reviewer before it can access the protected environment and continue.

> In this project, the AWS credentials shown in the setup  are configured as repository-level Actions secrets. They are not environment secrets.

### Step 3: Open GitHub Actions and select the workflow

1. Return to the repository.
2. Open the **Actions** tab.
3. Select the **Terraform Deploy** workflow from the workflow list.
4. Click **Run workflow**.
5. Select the `main` branch and start the workflow.

The workflow uses `workflow_dispatch`, so it is manually triggered. The push trigger is currently commented out in the YAML file.

### Step 4: Review the Terraform plan

The first job runs on `ubuntu-latest` and performs these steps:

1. Checks out the repository using `actions/checkout@v4`.
2. Installs Terraform using `hashicorp/setup-terraform@v3`.
3. Runs `terraform init`.
4. Runs `terraform validate`.
5. Generates a saved plan using `terraform plan -out=tfplan`.
6. Uploads the plan as an artifact named `tfplan`.

Review the plan output before approving the deployment. The saved plan artifact is then passed to the apply job.

### Step 5: Approve the production deployment

After the plan job succeeds, the apply job reaches the `production` environment.

1. GitHub displays a pending deployment / **Review deployments** prompt.
2. Click **Review deployments**.
3. Select the `production` environment.
4. Review the deployment and add an optional comment.
5. Click **Approve and deploy**.

The apply job remains paused until the required approval is provided. Configure **Prevent self-review** if you want to require approval from someone other than the person who triggered the workflow.

### Step 6: Terraform applies the saved plan

After approval, the apply job:

1. Checks out the repository.
2. Sets up Terraform.
3. Runs `terraform init`.
4. Downloads the `tfplan` artifact.
5. Executes `terraform apply -auto-approve tfplan`.

When both jobs complete successfully, check the workflow run in **Actions** and verify the resulting VPC and subnet in the AWS VPC console in the configured region.

## GitHub Actions Workflow

The workflow file is located at:

[`.github/workflows/terraform-cicd.yml`](.github/workflows/terraform-cicd.yml)

Important workflow configuration:

- `workflow_dispatch`: manual execution from the Actions tab.
- `plan` job: initializes, validates, and creates the Terraform plan.
- `actions/upload-artifact@v4`: stores the generated plan for the next job.
- `needs: plan`: ensures apply runs only after the plan job succeeds.
- `environment: production`: applies the configured deployment protection rules.
- `actions/download-artifact@v4`: retrieves the exact plan generated by the plan job.
- `terraform apply -auto-approve tfplan`: applies the reviewed saved plan after approval.

## Run Terraform Locally (Optional)

If you want to test the Terraform configuration from your machine:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
```

To remove resources created by this configuration:

```bash
terraform destroy
```

Run these commands from the repository root. Make sure your AWS CLI credentials or environment are configured before running Terraform locally.

## Important Security and Cost Notes

- Terraform apply creates or changes real AWS resources, which may incur charges.
- Check the Terraform plan and AWS region before approving a deployment.
- Use least-privilege IAM permissions. Do not use root-account access keys.
- Terraform state and plan files can contain sensitive data. Do not commit `terraform.tfstate`, `.tfstate.backup`, `tfplan`, or the local `.terraform/` directory.
- Run `terraform destroy` when the resources are no longer needed, and verify cleanup in AWS.
- A saved Terraform plan can contain sensitive values. Protect the artifact and restrict who can access workflow runs.

## What I Learned

- Automating Terraform workflows using GitHub Actions.
- Managing AWS credentials securely with GitHub Actions secrets.
- Separating Terraform plan and apply stages.
- Passing a saved Terraform plan between workflow jobs using artifacts.
- Configuring GitHub Environments and required deployment reviewers.
- Implementing a manual approval gate before infrastructure changes.
- Deploying and verifying AWS networking resources with Terraform.

## Author

**Venkata Subbarao**

- GitHub: [@VENKATASUBBARAO13](https://github.com/VENKATASUBBARAO13)
- LinkedIn: [Venkata Subbarao](https://www.linkedin.com/in/ventrapragada-venkata-subbarao/)
- Portfolio: [venkata-subbarao-portfolio.vercel.app](https://venkata-subbarao-portfolio.vercel.app)

---

If you find this project useful, feel free to ⭐ the repository!
