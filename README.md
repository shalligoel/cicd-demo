# CI/CD demo: GitHub Actions to AWS (Windows setup)

Run every command in PowerShell. Replace OWNER/REPO with your GitHub username and repository name.

## 1. Install tools (one time)
```powershell
winget install Git.Git
winget install Python.Python.3.12
winget install Docker.DockerDesktop
winget install Amazon.AWSCLI
winget install Hashicorp.Terraform
winget install Microsoft.VisualStudioCode
```
Close and reopen PowerShell, then restart Windows if Docker Desktop asks (it uses WSL2).

## 2. Run and test locally
```powershell
cd cicd-demo
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
ruff check .
pytest -q
docker build -t cicd-demo .
docker run -p 8080:8080 cicd-demo
```
Open http://localhost:8080/health. You should see {"status":"ok"}.

## 3. Create the GitHub repository and push
Create an empty repo on github.com, then:
```powershell
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/OWNER/REPO.git
git push -u origin main

if there is issue then create token or public key
Option A: Personal Access Token (quickest)
Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token.
Tick the repo scope (add workflow too, since this is a CI/CD project and you’ll likely push .github/workflows files).
Copy the token, then run git push -u origin main again. For the username, use your GitHub username (not your email), and paste the token as the password.
To avoid retyping it each time:
git config --global credential.helper store

Option B: create ssh keys
ssh-keygen -t ed25519 -C "shalligoel@gmail.com"
cat ~/.ssh/id_ed25519.pub
Add that public key at GitHub → Settings → SSH and GPG keys, then switch the remote:

	Create repo first - cicd-demo.git
git remote set-url origin git@github.com:shalligoel/cicd-demo.git
git remote -v
git push -u origin main


## 4. Connect your AWS account
Use an IAM user or IAM Identity Center (SSO) with permissions to create ECR and IAM resources, then:
```powershell
aws configure
aws sts get-caller-identity
```
Also set a billing alarm in the AWS console.

## 5. Create the AWS side with Terraform
```powershell
cd terraform
terraform init
terraform apply -var "github_repo=OWNER/REPO"
```
Note the role_arn output. If an OIDC provider for GitHub already exists in your account, import it or remove that resource.

## 6. Configure GitHub
- Settings > Secrets and variables > Actions > New repository secret: AWS_ROLE_ARN = the role_arn output.
- Settings > Environments > New environment named production, and add yourself as a required reviewer (private repos may need a paid plan for this).
- Settings > Branches > add a rule for main: require a pull request and require the test job to pass.

## 7. Watch it run
Push a change on a branch, open a pull request, and see the test job run. Merge it and see test, build-push, then deploy-prod (waiting for your approval) in the Actions tab. Confirm the image appears in ECR.

## 8. Clean up to avoid charges
```powershell
cd terraform
terraform destroy -var "github_repo=OWNER/REPO"
```

## Next steps
Replace the placeholder in the deploy-prod job with a real deploy (for example ECS Fargate), add dev and staging environments, and add Flyway migrations for your Oracle or PostgreSQL schema.
