# Reusable CI/CD pipelines

This repository contains GitHub Actions reusable workflows that application
repositories can call instead of duplicating pipeline code.

## Taskboard Java

`/.github/workflows/taskboard-java.yml` is called by
[`Roobini-code/java-project`](https://github.com/Roobini-code/java-project):

- Pull requests targeting `main` run `mvn clean verify` and build the Docker
  image locally. They do not receive deployment secrets, publish an image, or
  deploy to EC2.
- A push to `main` (including a merged pull request) publishes the Docker image
  to Docker Hub, deploys it to EC2, verifies the HTTP endpoint, then creates a
  Git tag in the application repository.
- The image tags are `1.0.0-<run-number>.<run-attempt>` and `latest`. The Git
  tag is the same version prefixed with `v`, for example
  `v1.0.0-42.1`. The Maven project version is used as the base version.

Configure these Actions repository secrets in `java-project` before merging to
`main`:

| Secret | Value |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub username with publish access to `roobinidevops/taskboard-java` |
| `DOCKERHUB_TOKEN` | Docker Hub access token with Read & Write permission |
| `EC2_HOST` | EC2 public DNS name or IP address, without a scheme |
| `EC2_SSH_PRIVATE_KEY` | Private key for the EC2 key pair |
| `EC2_KNOWN_HOSTS` | Verified SSH host-key entry for `EC2_HOST` |

The caller workflow needs `contents: write` permission to push Git tags. Protect
`main` and require pull requests if deployment should only follow reviewed
merges; each push to `main` triggers deployment.

The EC2 instance must have Docker installed, permit SSH from GitHub-hosted
runners, and allow the configured SSH user (`ec2-user`) to run Docker with
`sudo`. The app persists H2 data in the `taskboard-data` Docker volume. Since
GitHub-hosted runner IP ranges change, restrict SSH using a self-hosted runner
or an AWS Systems Manager deployment if a static source IP is required.

The reusable workflow repository must be public or otherwise configured to
allow `java-project` to use its workflows. For production use, pin the caller's
`uses:` reference to an approved commit SHA or version tag rather than `@main`.
