# Reusable CI/CD pipelines

This repository contains GitHub Actions reusable workflows that application
repositories can call instead of duplicating pipeline code.

For a step-by-step AWS infrastructure, IAM/OIDC, Systems Manager, GitHub
repository configuration, and deployment walkthrough, see
[`doc/java-project/TASKBOARD-AWS-GITHUB-ACTIONS-INFRASTRUCTURE.md`](doc/java-project/TASKBOARD-AWS-GITHUB-ACTIONS-INFRASTRUCTURE.md).

## React + Kubernetes

- `/.github/workflows/k8s-app-deployment-ci.yml` runs the React tests,
  production build, and a non-publishing container build for pull requests.
- `/.github/workflows/k8s-app-deployment-cd.yml` tests and builds again after
  merge, publishes an immutable commit-SHA-tagged image to Docker Hub, and
  commits the image update for the inactive Deployment and the Service slot
  switch to the application repository. Argo CD detects the Git change and
  syncs it to K3s, waiting for the new Deployment to become healthy before
  switching traffic.
- The caller repository stores its small trigger workflows in
  `k8s-app-deployment/.github/workflows/ci.yml` and `cd.yml`.
- The example caller repository is
  [`k8s-app-deployment`](https://github.com/OWNER/k8s-app-deployment). Replace
  `OWNER` with the GitHub account or organization that owns both repositories.

For the full React, Docker Hub, K3s-on-EC2, Argo CD, workflow setup, cost
controls, and cleanup walkthrough, see
[`k8s-app-deployment/doc/COMPLETE-HANDS-ON-GUIDE.md`](https://github.com/OWNER/k8s-app-deployment/blob/main/doc/COMPLETE-HANDS-ON-GUIDE.md).

Detailed guides are also maintained with the reusable workflows:

- [CI/CD pipeline setup for `k8s-app-deployment`](doc/k8s-app-deployment/CI-CD-PIPELINE-SETUP.md)
- [K3s, Argo CD, and blue/green deployment](doc/k8s-app-deployment/K3S-ARGOCD-BLUE-GREEN-GUIDE.md)

## Taskboard Java

`/.github/workflows/taskboard-java.yml` is called by
[`Roobini-code/java-project`](https://github.com/Roobini-code/java-project):

- Pull requests targeting `main` run `mvn clean verify` and build the Docker
  image locally. They do not receive deployment secrets, publish an image, or
  deploy to EC2.
- A push to `main` (including a merged pull request) publishes the Docker image
  to Docker Hub, assumes an AWS role through GitHub OIDC, deploys through AWS
  Systems Manager, verifies the HTTP endpoint, then creates a Git tag.
- The image tags are `1.0.0-<run-number>.<run-attempt>` and `latest`. The Git
  tag is the same version prefixed with `v`, for example
  `v1.0.0-42.1`. The Maven project version is the base version.

### Configure `java-project`

Create these repository variables in `java-project` under **Settings → Secrets
and variables → Actions → Variables**:

| Variable | Value |
| --- | --- |
| `AWS_REGION` | Region containing the EC2 instance |
| `EC2_INSTANCE_ID` | Target instance ID (`i-...`) |
| `AWS_ROLE_ARN` | ARN of the GitHub OIDC deployment role |

Create these repository secrets in the same repository under **Secrets**:

| Secret | Value |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub username with push access to `roobinidevops/taskboard-java` |
| `DOCKERHUB_TOKEN` | Docker Hub access token with Read & Write permission |

Do not add the EC2 SSH key, EC2 host key, AWS access keys, or AWS console
password. The workflow deploys by assuming `AWS_ROLE_ARN` with OIDC and sending
an SSM command to `EC2_INSTANCE_ID`.

### AWS requirements

- The EC2 instance has Docker and curl installed, the SSM agent running, and
  an instance profile with `AmazonSSMManagedInstanceCore`.
- EC2 has outbound HTTPS access to Systems Manager and Docker Hub. No inbound
  SSH access from GitHub-hosted runners is required.
- The IAM OIDC provider is `token.actions.githubusercontent.com` with audience
  `sts.amazonaws.com`.
- The IAM role trust policy is restricted to
  `repo:Roobini-code/java-project:ref:refs/heads/main`.
- The role permissions allow `ssm:SendCommand` only for
  `AWS-RunShellScript` and the configured instance, plus command status reads.
- The Docker Hub repository is public so the EC2 instance can pull the image.

The caller workflow needs `contents: write` to create tags and `id-token:
write` to request an OIDC token. Protect `main` and require pull requests if
deployment should only follow reviewed merges; every push to `main` deploys.

The reusable workflow repository must be public or otherwise configured to
allow `java-project` to use its workflows. For production, pin the caller's
`uses:` reference to an approved commit SHA or version tag rather than `@main`.
