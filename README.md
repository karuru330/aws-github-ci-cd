# aws-github-ci-cd

Initial Setup:

1. Setup AWS IAM Role for GitHub OIDC

Step 1.1: Sign in to AWS Console

Go to: https://console.aws.amazon.com/iam/

Log in with an account that can create IAM roles.

🔹 Step 1.2: Go to IAM Identity Providers
Open: https://console.aws.amazon.com/iamv2/home#/identity_providers

Click Add provider

🔹 Step 1.3: Fill in Provider Details

Provider type: Select OIDC

Provider URL:
https://token.actions.githubusercontent.com

Audience (default):
sts.amazonaws.com

Click Add provider

You’ve now registered GitHub as an OIDC provider.


Step 1.4: Create IAM Role with OIDC Trust

Go to: IAM → Roles → Create Role

Select trusted entity type:

Choose: Web Identity

Identity provider: GitHub

OIDC Provider URL: https://token.actions.githubusercontent.com

Audience: sts.amazonaws.com

Click Next

Step 1.5: Attach Permissions

Attach a policy for CloudFormation. For now, you can use a managed one:

Search and select: AdministratorAccess (for testing only — replace with least-privilege later)

Or use a custom policy like this:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "s3:*",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}

Click Next.


Step 1.6: Name and Create the Role

Role name: GitHubActionsOIDCDeployer

Description: (optional)

Click Create role

Step 1.7: Edit Trust Policy

Now configure the trust policy to allow your GitHub repo to assume the role:

Go to IAM → Roles → Click on the role you just created

Click Trust relationships → Edit trust policy

Replace it with this (edit accordingly):

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG_OR_USER/YOUR_REPO_NAME:*"
        }
      }
    }
  ]
}
Replace:

YOUR_AWS_ACCOUNT_ID → your 12-digit AWS account ID

YOUR_ORG_OR_USER/YOUR_REPO_NAME → your GitHub org/user and repo name

Click Update policy


Step 2: Configure GitHub Actions Workflow
Now create or update the GitHub Actions workflow in your repo.

Step 2.1: Add Workflow YAML

Create a file at: .github/workflows/deploy.yml

name: Deploy CloudFormation using OIDC

on:
  push:
    branches: [main]  # or your deployment branch

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      id-token: write    # Needed for OIDC
      contents: read     # Needed to checkout code

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials from OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/GitHubActionsOIDCDeployer
        aws-region: us-east-1  # or your preferred region

    - name: Deploy CloudFormation Stack
      run: |
        aws cloudformation deploy \
          --template-file template.yaml \
          --stack-name my-stack-name \
          --capabilities CAPABILITY_NAMED_IAM \
          --no-fail-on-empty-changeset
Replace:

YOUR_AWS_ACCOUNT_ID with your AWS account ID

template.yaml with your actual CloudFormation template file name

Step 2.2: Commit and Push

Commit the .github/workflows/deploy.yml file to your repo

Push to the branch (e.g., main) to trigger the workflow