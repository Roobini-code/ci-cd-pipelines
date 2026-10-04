# Taskboard AWS infrastructure and GitHub Actions setup

This guide sets up the infrastructure used by the reusable Taskboard pipeline.
Follow the sections in order:

1. Create and prepare the AWS account and EC2 instance.
2. Configure EC2 Systems Manager (SSM) access.
3. Create the GitHub OIDC provider, deployment policy, and IAM role.
4. Configure Docker Hub and the `java-project` GitHub repository.
5. Run a pull request and then a deployment.

The deployment uses **GitHub OIDC + AWS Systems Manager**. It does not use
AWS access keys in GitHub, an EC2 private SSH key in GitHub, or inbound SSH
from a GitHub-hosted runner.

## Before setup: what these AWS terms mean

You do not need to know AWS terminology in advance. These are the main pieces
and why this pipeline uses each one:

| Term | In plain language | Purpose in this pipeline |
| --- | --- | --- |
| **AWS account** | The container for your AWS resources, permissions, and billing. | Owns the EC2 server and IAM configuration. |
| **EC2 instance** | A virtual computer running in AWS. | Hosts the Taskboard Docker container and its persistent data volume. |
| **IAM (Identity and Access Management)** | AWS's system for deciding who or what can access AWS services. | Controls what GitHub Actions and the EC2 server are each allowed to do. |
| **IAM role** | A set of permissions that a service or trusted identity can temporarily use. Unlike an IAM user, it does not have a permanent password or access key. | We create two separate roles: one for the EC2 machine and one for GitHub Actions. |
| **IAM policy** | A JSON list of allowed or denied AWS actions and resources. | `AmazonSSMManagedInstanceCore` lets EC2 use SSM. `TaskboardGitHubDeployPolicy` lets the GitHub role send a deployment command only to the configured instance and read its result. |
| **Trust policy / trust relationship** | The “who may use this role?” rule attached to an IAM role. It is not the same as a permissions policy. | The GitHub role trusts only the GitHub OIDC identity for `Roobini-code/java-project` on `main`; the EC2 role trusts the EC2 service. |
| **GitHub OIDC** | A way for GitHub Actions to prove its repository and branch identity to AWS using a short-lived signed token. | Lets the workflow assume its AWS role without storing permanent AWS access keys in GitHub. |
| **OIDC identity provider** | AWS's registration of an external token issuer it is willing to validate. | Registers `token.actions.githubusercontent.com` so AWS can validate GitHub's token. It does not grant AWS permissions on its own. |
| **AWS STS** | AWS Security Token Service; it issues temporary credentials after a role's trust rules are satisfied. | Exchanges the valid GitHub OIDC identity for short-lived credentials scoped to the GitHub deployment role. |
| **EC2 instance profile** | The attachment that makes an IAM role available to an EC2 instance. | Carries `TaskboardEC2SSMRole` to the EC2 machine so its SSM agent can authenticate to AWS. |
| **Systems Manager (SSM)** | An AWS service for securely managing and running commands on registered servers. | Relays the deployment command to EC2 and returns its output/status to GitHub Actions. |
| **SSM agent** | A background program installed and running on EC2. | Connects outbound to SSM over HTTPS, receives the deployment command, runs Docker commands locally, and reports the result. |
| **Security group** | A virtual firewall attached to network interfaces or an EC2 instance. | Allows intended browser access to the app and outbound HTTPS. GitHub does not need an inbound SSH rule because deployment uses SSM. |
| **Docker named volume** | Persistent disk storage managed by Docker, separate from a replaceable container. | `taskboard-data` stores the H2 database so replacing the container does not erase tasks. |

### The two roles, simply

- **`TaskboardEC2SSMRole` belongs to the EC2 machine.** Its attached
  `AmazonSSMManagedInstanceCore` policy lets the SSM agent register with AWS
  and receive/report commands. GitHub Actions never assumes this role.
- **`TaskboardGitHubActionsDeployRole` belongs to the deployment workflow.**
  GitHub proves its identity through OIDC, then AWS STS gives the runner
  temporary credentials for the role. Its `TaskboardGitHubDeployPolicy`
  permits deployment commands only for the configured EC2 instance.

Think of the **trust policy** as “who is allowed to borrow this role?” and
the **permissions policy** as “what can the role do after it is borrowed?”
Both checks must pass before the runner can deploy.

## 1. What is being connected?

There are two IAM roles. They are separate and must not be confused:

| IAM role | Attached to | Purpose |
| --- | --- | --- |
| `TaskboardEC2SSMRole` | The EC2 instance profile | Lets the SSM agent on EC2 register with Systems Manager and receive commands. Attach AWS-managed policy `AmazonSSMManagedInstanceCore`. |
| `TaskboardGitHubActionsDeployRole` | Assumed temporarily by the GitHub Actions runner through OIDC | Lets the `java-project` `main` workflow send a deployment command to the one configured EC2 instance and read the command result. Attach `TaskboardGitHubDeployPolicy`. |

The GitHub OIDC provider is an IAM account-level identity provider, not an
instance role. It allows AWS STS to validate signed identity tokens issued by
GitHub Actions. The role trust relationship restricts assumption to
`Roobini-code/java-project` on branch `main`.

## 2. Detailed architecture

### 2.1 Who trusts whom, and which role has which permissions?

Read this diagram from top to bottom. Solid arrows show a request or credential
flow. Dotted arrows show a policy or trust relationship, not a network
connection.

```mermaid
flowchart TB
  APP["1. App workflow<br/>Roobini-code/java-project<br/>main branch"]
  REUSABLE["2. Reusable workflow<br/>ci-cd-pipelines"]
  RUNNER["3. GitHub-hosted runner"]
  ISSUER["4. GitHub OIDC issuer<br/>token.actions.githubusercontent.com"]
  TOKEN["5. Signed OIDC token<br/>sub = repo:Roobini-code/java-project:ref:refs/heads/main<br/>aud = sts.amazonaws.com"]
  PROVIDER["6. AWS IAM OIDC provider<br/>registers GitHub as token issuer"]
  TRUST["7. Role trust policy<br/>allows only this repo and main"]
  ROLE["8. TaskboardGitHubActionsDeployRole"]
  POLICY["9. TaskboardGitHubDeployPolicy<br/>SSM command access limited to target EC2"]
  STS["10. AWS STS<br/>validates token and issues temporary credentials"]
  SSM["11. AWS Systems Manager"]
  EC2ROLE["12. TaskboardEC2SSMRole<br/>AmazonSSMManagedInstanceCore"]
  INSTANCE["13. EC2 instance profile + taskboard-ec2"]
  AGENT["14. SSM agent on EC2"]
  APP_CONTAINER["15. Taskboard container<br/>taskboard-data Docker volume"]

  APP -->|"calls"| REUSABLE
  REUSABLE -->|"starts job on"| RUNNER
  RUNNER -->|"requests OIDC token"| ISSUER
  ISSUER -->|"issues"| TOKEN
  RUNNER -->|"presents token to"| STS
  PROVIDER -. "token issuer configured in AWS" .-> STS
  STS -. "validates issuer/audience using" .-> PROVIDER
  TRUST -. "trust rules attached to" .-> ROLE
  POLICY -. "permissions attached to" .-> ROLE
  STS -. "checks token subject against" .-> TRUST
  STS -->|"returns short-lived credentials to"| RUNNER
  RUNNER -->|"calls SSM API using temporary credentials"| SSM
  RUNNER -. "configured with this role ARN" .-> ROLE
  EC2ROLE -. "attached to EC2 instance profile" .-> INSTANCE
  INSTANCE -->|"runs"| AGENT
  AGENT -->|"authenticates with instance role"| EC2ROLE
  AGENT -->|"outbound HTTPS 443: polls for commands"| SSM
  SSM -->|"returns queued deployment command"| AGENT
  AGENT -->|"runs Docker deployment on"| APP_CONTAINER
```

Important distinction: the GitHub role is assumed by the runner; the EC2 role
is attached to the instance and used by its SSM agent. The two roles are not
interchangeable.

<a id="22-event-and-network-flow"></a>

### 2.2 Numbered deployment flow: what runs, and where?

Start at **1**. Follow the arrows. The PR path ends after CI; the `main` path
continues through image publishing and deployment.

```mermaid
flowchart TD
  START["1. Trigger<br/>PR to main OR push/merge to main"]
  CHECKOUT["2. Runner checks out app<br/>GITHUB_TOKEN"]
  TEST["3. Runner installs Java 21<br/>mvn clean verify"]
  BUILD_CI["4. Runner builds Docker image locally"]
  EVENT{"5. Is this a push to main?"}
  PR_DONE["PR ends here<br/>Checks reported to GitHub<br/>No publish, AWS access, or deployment"]
  VERSION["6. Runner derives unique image tag<br/>Maven version + run number/attempt"]
  DOCKER_LOGIN["7. Runner logs in to Docker Hub<br/>using repository secrets"]
  DOCKER_PUSH["8. Runner pushes versioned image + latest"]
  OIDC["9. Runner requests GitHub OIDC token<br/>for this repository's main branch"]
  STS["10. AWS STS validates OIDC claims<br/>and returns temporary role credentials"]
  SEND["11. Runner calls SSM SendCommand<br/>for configured EC2 instance ID"]
  AGENT_POLL["12. EC2 SSM agent receives command<br/>over its outbound HTTPS connection"]
  PULL["13. EC2 pulls public versioned image<br/>from Docker Hub"]
  REPLACE["14. EC2 replaces container<br/>reuses taskboard-data volume"]
  HEALTH{"15. EC2 local HTTP health check passes?"}
  ROLLBACK["Attempt to restore previous container<br/>Workflow fails; no Git tag"]
  TAG["16. Runner creates and pushes<br/>matching Git tag"]
  DONE["17. Deployment complete"]

  START --> CHECKOUT --> TEST --> BUILD_CI --> EVENT
  EVENT -->|"No: pull request"| PR_DONE
  EVENT -->|"Yes: push/merge to main"| VERSION
  VERSION --> DOCKER_LOGIN --> DOCKER_PUSH --> OIDC --> STS --> SEND
  SEND --> AGENT_POLL --> PULL --> REPLACE --> HEALTH
  HEALTH -->|"No"| ROLLBACK
  HEALTH -->|"Yes"| TAG --> DONE
```

#### Which machine starts each connection?

| Number | Connection/request | Initiated by | Direction and port |
| ---: | --- | --- | --- |
| 2 | App checkout | GitHub Actions runner | Runner → GitHub over HTTPS 443 using `GITHUB_TOKEN`. |
| 7–8 | Docker login and image push | GitHub Actions runner | Runner → Docker Hub over HTTPS 443. |
| 9–10 | OIDC token exchange / STS role assumption | GitHub Actions runner | Runner → GitHub OIDC and AWS STS over HTTPS 443. |
| 11 | Send deployment command | GitHub Actions runner | Runner → AWS Systems Manager API over HTTPS 443 using temporary role credentials. |
| 12 | SSM agent check-in / command retrieval | EC2 SSM agent | EC2 → AWS Systems Manager over outbound HTTPS 443. |
| 13 | Pull deployment image | EC2 Docker Engine | EC2 → public Docker Hub over outbound HTTPS 443. |
| 15 | Local app health check | EC2 deployment command | EC2 → `127.0.0.1:80`, inside the instance. |

The runner does **not** create a network connection to EC2. Do not open EC2 SSH
to GitHub or add GitHub runner IP ranges. The SSM agent on EC2 initiates its
outbound HTTPS connection, and the Systems Manager service relays the command.
The user's browser reaches the app separately on EC2 port 80, subject to the
HTTP security-group rule.

### 2.3 AWS-service map: what is created in each service?

This view groups resources under the AWS service where you configure or see
them. Text inside each box explains what the item is for. Arrows show runtime
calls; dotted lines show permissions or attachments, not network connections.

```mermaid
flowchart TB
  subgraph GITHUB["GitHub"]
    APP["java-project workflow<br/>Starts on pull request or push to main"]
    RUNNER["GitHub-hosted runner<br/>Runs tests; on main, builds and publishes image"]
    OIDC["GitHub OIDC token<br/>Identifies repository and branch to AWS"]
    APP -->|"calls reusable workflow"| RUNNER
    RUNNER -->|"requests token"| OIDC
  end

  subgraph HUB["Docker Hub"]
    IMAGE["roobinidevops/taskboard-java<br/>Stores versioned and latest images"]
  end

  subgraph AWS["AWS account"]
    subgraph IAM["IAM - identity, trust, and permissions"]
      PROVIDER["OIDC identity provider<br/>Registers GitHub as a trusted token issuer"]
      GHROLE["TaskboardGitHubActionsDeployRole<br/>Temporary role the runner assumes"]
      TRUST["Role trust policy<br/>Only java-project on main may assume this role"]
      DEPLOYPOLICY["TaskboardGitHubDeployPolicy<br/>Allows SSM deployment commands for the target instance"]
      EC2ROLE["TaskboardEC2SSMRole<br/>Role attached to EC2; has AmazonSSMManagedInstanceCore attached"]
      PROFILE["EC2 instance profile<br/>Makes the EC2 role available to the instance"]
      TRUST -. "controls who may assume" .-> GHROLE
      DEPLOYPOLICY -. "grants actions to" .-> GHROLE
      EC2ROLE -. "is carried by" .-> PROFILE
    end

    subgraph STS["AWS STS - Security Token Service"]
      CREDENTIALS["Temporary AWS credentials<br/>Issued after OIDC and trust checks pass"]
    end

    subgraph SSM["Systems Manager (SSM)"]
      COMMAND["Run Command<br/>AWS-RunShellScript runs the deployment script"]
      NODE["Managed node entry<br/>Appears when the SSM agent registers and checks in"]
    end

    subgraph EC2["EC2"]
      INSTANCE["taskboard-ec2<br/>Amazon Linux 2023 virtual server"]
      SOFTWARE["Installed on the host<br/>Docker Engine and curl<br/>SSM agent enabled and running"]
      CONTAINER["Taskboard Docker container<br/>Listens on host port 80; container port 8080"]
      VOLUME["taskboard-data Docker volume<br/>Keeps H2 database data when container is replaced"]
      FIREWALL["Security group<br/>Inbound HTTP 80 for intended users<br/>Outbound HTTPS 443 for AWS and Docker Hub"]
      INSTANCE --> SOFTWARE
      SOFTWARE --> CONTAINER
      CONTAINER --> VOLUME
      FIREWALL -. "filters instance network traffic" .-> INSTANCE
      PROFILE -. "provides EC2 role credentials" .-> INSTANCE
    end
  end

  OIDC -->|"token over HTTPS"| CREDENTIALS
  PROVIDER -. "issuer AWS validates" .-> CREDENTIALS
  GHROLE -. "trust checked before assumption" .-> CREDENTIALS
  CREDENTIALS -->|"runner calls SSM API over HTTPS 443"| COMMAND
  DEPLOYPOLICY -. "authorizes SendCommand and status reads" .-> COMMAND
  SOFTWARE -->|"agent registers and polls outbound over HTTPS 443"| NODE
  NODE -->|"SSM relays command to the agent"| SOFTWARE
  RUNNER -->|"pushes images over HTTPS 443"| IMAGE
  SOFTWARE -->|"Docker pulls versioned image over HTTPS 443"| IMAGE
```

**Where the SSM managed node comes from:** you do not create a managed node
manually. Amazon Linux 2023 normally includes the SSM agent. Once
`TaskboardEC2SSMRole` is attached to the instance and the agent is running, the
agent uses that role to register with Systems Manager over outbound HTTPS.
The instance then appears under **Systems Manager → Fleet Manager** or
**Managed nodes** in the same Region, usually as **Online**. If the agent is
not running, the role is missing, or outbound access is blocked, the instance
will not appear online.

**What is installed on EC2:** Docker Engine runs the Taskboard container, and
`curl` is used for the local HTTP health check after deployment. The SSM agent
receives and reports deployment commands. Java is packaged in the application
image; it does not need to be installed separately on the EC2 host. The
workflow uses the host's `taskboard-data` Docker volume to preserve the H2
database when it replaces the container.

**What the runner calls:** after tests pass on a push to `main`, the runner
pushes the Docker image to Docker Hub, requests a GitHub OIDC token, and
exchanges it with AWS STS for temporary credentials. It then calls the SSM
`SendCommand` API and polls for the command result. It does not call EC2 over
SSH. The SSM agent initiates its own outbound connection to Systems Manager,
receives the command, pulls the image from Docker Hub, and runs the deployment
script on the EC2 host.

## 3. AWS prerequisites

### 3.1 Account, Region, and cost

1. Use an AWS account you are authorized to administer. Enable MFA and avoid
   using the root user for routine work.
2. Select one AWS Region for the EC2 instance, Systems Manager view, and
   GitHub `AWS_REGION` variable. The IAM OIDC provider is an AWS-account-level
   provider. The setup so far has shown `us-east-2`; confirm the instance's
   actual Region before using that value.
3. Review EC2, EBS, public IPv4, and data-transfer charges. Configure a budget
   alert; an alert is not a spending cap.

### 3.2 EC2 instance requirements

The `taskboard-ec2` instance should use Amazon Linux 2023, be in the same
Region configured for the pipeline, and have:

- A public image repository on Docker Hub, or an explicitly configured
  alternative image-pull authentication mechanism.
- Outbound access to the internet/AWS service endpoints over HTTPS (TCP 443).
- A root EBS volume with enough space for the OS, Docker, and image layers.
  The current Taskboard persistent data is in Docker volume `taskboard-data`
  on the EC2 disk; this workflow does not use EFS.
- An attached instance profile role with `AmazonSSMManagedInstanceCore`
  (created in Step 4).
- Docker Engine and `curl` installed.
- The SSM agent installed, enabled, running, and online as a Systems Manager
  managed node.

The instance needs no inbound rule for GitHub Actions. For your own optional
manual SSH access, retain SSH TCP 22 restricted to your current public IP
(`/32`). For the app, allow HTTP TCP 80 only from intended users. Taskboard
does not provide authentication. Do not open SSH to `0.0.0.0/0`.

### 3.3 Docker Hub requirements

The existing workflow pushes to
`roobinidevops/taskboard-java`. Confirm the repository exists and the account
can push to it. Create a Docker Hub access token with **Read & Write**
permission.

The repository must be **Public** with the current deployment code. The GitHub
runner authenticates to push, but the EC2 deployment command performs
`docker pull` without logging in to Docker Hub. If the repository is private,
modify the deployment command to authenticate on EC2 before relying on it.

## 4. Configure the EC2 Systems Manager role

This is the instance role, not the GitHub deployment role.

1. In the AWS Console, open **IAM → Roles → Create role**.
2. Choose **AWS service** and the **EC2** use case.
3. Attach the AWS-managed policy **`AmazonSSMManagedInstanceCore`**.
4. Name the role `TaskboardEC2SSMRole` and create it.
5. Open **EC2 → Instances**, select `taskboard-ec2`, then choose **Actions →
   Security → Modify IAM role**. Attach `TaskboardEC2SSMRole` and save.
6. Connect to EC2 for setup/verification (SSH limited to your IP is fine) and
   run:

   ```bash
   sudo dnf update -y
   sudo dnf install -y docker curl
   sudo systemctl enable --now docker
   sudo docker --version
   sudo systemctl is-active docker
   sudo systemctl enable --now amazon-ssm-agent
   sudo systemctl is-active amazon-ssm-agent
   ```

   Docker and the SSM agent checks should print a version and `active`.
   Amazon Linux 2023 normally includes the SSM agent. If its service is
   missing, install the Amazon Linux package with
   `sudo dnf install -y amazon-ssm-agent`, then enable and start it.
7. In **Systems Manager → Fleet Manager** or **Managed nodes**, select the
   same Region and wait for this EC2 instance to appear **Online**.

If it remains offline, verify that the EC2 instance profile is attached, the
agent is active, the instance has DNS resolution and outbound TCP 443, and
the VPC route/NAT path can reach regional Systems Manager endpoints. A
security-group outbound rule allowing all traffic (the current setup) already
permits HTTPS; do not add an inbound 443 rule. With restrictive network ACLs,
allow outbound HTTPS and the corresponding return traffic.

## 5. Create the GitHub OIDC provider in IAM

Create this once per AWS account:

1. Open **IAM → Identity providers → Add provider**.
2. Provider type: **OpenID Connect**.
3. Provider URL: `https://token.actions.githubusercontent.com`.
4. Audience: `sts.amazonaws.com`.
5. Add the provider. If it already exists, reuse it; do not create a duplicate.

This provider establishes GitHub as an OIDC token issuer. It does not itself
grant permissions; the role trust relationship and attached permissions policy
control access.

## 6. Create the GitHub deployment permissions policy

This is the **permissions policy**, which controls what a successfully
authenticated GitHub role can do. It is not the role trust policy.

1. In the AWS Console, open **IAM → Policies → Create policy → JSON**.
2. Replace `<REGION>`, `<ACCOUNT_ID>`, and `<INSTANCE_ID>` in the JSON below
   with the EC2 Region, 12-digit AWS account ID, and exact EC2 instance ID.
3. Create the policy named `TaskboardGitHubDeployPolicy`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SendDeploymentOnlyToTaskboardInstance",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": [
        "arn:aws:ssm:<REGION>::document/AWS-RunShellScript",
        "arn:aws:ec2:<REGION>:<ACCOUNT_ID>:instance/<INSTANCE_ID>"
      ]
    },
    {
      "Sid": "ReadAndCancelDeploymentCommand",
      "Effect": "Allow",
      "Action": [
        "ssm:GetCommandInvocation",
        "ssm:CancelCommand"
      ],
      "Resource": "*"
    }
  ]
}
```

Example account, Region, and instance details previously shown in the AWS
screenshots were `142643434331`, `us-east-2`, and
`i-09f9a21c034a3947a`. Treat these only as examples: verify all three against
the current AWS console before creating the policy. If the EC2 instance is
replaced, update the policy resource and the GitHub `EC2_INSTANCE_ID` variable.

## 7. Create the GitHub Actions IAM role

This is the separate role assumed by GitHub Actions. The GitHub owner/org field
in the IAM wizard is `Roobini-code`, the repository is `java-project`, and the
branch is `main`.

1. Open **IAM → Roles → Create role → Web identity**.
2. Select the provider `token.actions.githubusercontent.com` and audience
   `sts.amazonaws.com`.
3. If the wizard asks for **GitHub organization**, enter `Roobini-code`. Enter
   repository `java-project` and branch `main` if those fields are offered.
   Do not leave the repository/branch as wildcard `*`.
4. Attach the existing `TaskboardGitHubDeployPolicy`. Do not paste the trust
   policy into the permissions-policy editor.
5. Name the role `TaskboardGitHubActionsDeployRole` and create it.
6. Open the role's **Trust relationships** and ensure it restricts assumption
   to this repository's `main` branch. The trust relationship should be:

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
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
             "token.actions.githubusercontent.com:sub": "repo:Roobini-code/java-project:ref:refs/heads/main"
           }
         }
       }
     ]
   }
   ```

   Replace `<ACCOUNT_ID>` with your AWS account ID. If the wizard generated a
   broader trust policy, edit the role's trust relationship to match the
   restricted policy above.
7. In the role summary, copy the ARN. It has the form
   `arn:aws:iam::<ACCOUNT_ID>:role/TaskboardGitHubActionsDeployRole`; this
   becomes the GitHub repository variable `AWS_ROLE_ARN`.

The workflow requests `id-token: write` only to obtain a short-lived OIDC
token. It does not create or store AWS access keys. The AWS credentials
returned by STS are temporary and scoped by `TaskboardGitHubDeployPolicy`.

## 8. Configure GitHub Actions

### 8.1 Allow the reusable workflow

In `Roobini-code/ci-cd-pipelines`, ensure
`.github/workflows/taskboard-java.yml` is on the referenced branch (`main` in
the current app workflow).

In `Roobini-code/java-project`, ensure Actions are enabled and the repository
is allowed to call the reusable workflow. If `ci-cd-pipelines` is private,
configure its **Settings → Actions → General → Access** to allow `java-project`
to use its workflows.

For production, change the app workflow's `uses:` reference from `@main` to a
reviewed immutable commit SHA or release tag.

### 8.2 Add repository variables

In `java-project`, open **Settings → Secrets and variables → Actions →
Variables → New repository variable**. Add:

| Variable | Value |
| --- | --- |
| `AWS_REGION` | The exact Region containing EC2, for example `us-east-2`. |
| `EC2_INSTANCE_ID` | The exact EC2 instance ID, for example `i-09f9a21c034a3947a` after verifying it. |
| `AWS_ROLE_ARN` | Full role ARN copied in Step 7. |

These are configuration values, not secret credentials. Add them to the
**application repository** `java-project`, not to `ci-cd-pipelines`.

### 8.3 Add Docker Hub repository secrets

In `java-project`, open **Settings → Secrets and variables → Actions →
Secrets → New repository secret**. Add:

| Secret | Value |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub username with push permission. |
| `DOCKERHUB_TOKEN` | Docker Hub access token with Read & Write permission. |

Do not add a GitHub PAT, AWS access key, EC2 private SSH key, EC2 hostname, or
host-key entry as a workflow secret for this SSM design. If old
`EC2_HOST`, `EC2_SSH_PRIVATE_KEY`, or `EC2_KNOWN_HOSTS` secrets exist from the
previous SSH design, they are unused and can be removed.

### 8.4 Set workflow permissions and branch protection

1. In `java-project`, open **Settings → Actions → General** and permit the
   workflow to write repository contents. The reusable workflow uses this to
   create and push the deployment Git tag.
2. The app caller requests `contents: write` for tagging and `id-token: write`
   for OIDC. Keep these permissions scoped to the workflow/job; do not grant
   broad organization-wide permissions.
3. Protect `main` and require pull requests/reviews. Every push to `main`
   publishes and deploys.

## 9. Pipeline execution details

### Pull request targeting `main`

The reusable workflow checks out the app using GitHub's automatic
`GITHUB_TOKEN`, installs Java 21, runs `mvn -B clean verify`, and builds a
local Docker image. It does not log in to Docker Hub, request AWS credentials,
send an SSM command, or deploy.

### Push/merge to `main`

After verification succeeds, the workflow:

1. Derives an image tag from the Maven project version and Actions run
   number/attempt, for example `1.0.0-42.1`.
2. Logs in to Docker Hub with the two repository secrets.
3. Builds and pushes both `roobinidevops/taskboard-java:<versioned-tag>` and
   `roobinidevops/taskboard-java:latest`.
4. Requests an OIDC token and assumes `TaskboardGitHubActionsDeployRole`.
5. Sends an `AWS-RunShellScript` command with `ssm:SendCommand` to the
   configured instance ID.
6. EC2's online SSM agent receives and runs the command. It pulls the
   versioned public image, replaces the container, and reuses the Docker
   volume `taskboard-data`.
7. The command polls the local app HTTP endpoint. If the new container fails
   to start or pass health checking, it attempts to restore the previous
   container and reports failure.
8. The GitHub runner reads the SSM command result. Only on success does it
   create and push the corresponding Git tag, for example `v1.0.0-42.1`.

The EC2 host has no Docker Hub login in this design, so its image repository
must remain public. The EC2 deployment command and instance must be in the
same AWS Region configured in `AWS_REGION`.

## 10. First-run checklist

- [ ] EC2 is running in the intended Region and has Docker installed.
- [ ] `TaskboardEC2SSMRole` with `AmazonSSMManagedInstanceCore` is attached to
  the instance.
- [ ] The SSM agent is active and the instance is Online in Systems Manager.
- [ ] EC2 outbound HTTPS works; no GitHub inbound SSH rule was added.
- [ ] Docker Hub repository exists, is Public, and the token can push.
- [ ] IAM OIDC provider exists with audience `sts.amazonaws.com`.
- [ ] `TaskboardGitHubDeployPolicy` uses the correct Region/account/instance.
- [ ] The GitHub role trust restricts subject to
  `repo:Roobini-code/java-project:ref:refs/heads/main`.
- [ ] The GitHub role has `TaskboardGitHubDeployPolicy` attached.
- [ ] The three GitHub repository variables and two Docker Hub secrets are
  configured in `java-project`.
- [ ] Cross-repository reusable-workflow access is enabled.
- [ ] `main` is protected and Actions can create tags.

Run a pull request first and confirm tests and the Docker build pass. Once
merged, inspect the Actions job logs, Systems Manager command result, Docker
Hub tags, running EC2 container, app endpoint, and new Git tag.

## Troubleshooting

| Symptom | Checks |
| --- | --- |
| EC2 is missing from Systems Manager | Confirm the instance profile role and `AmazonSSMManagedInstanceCore`, active SSM agent, same Region in the console, DNS, route, and outbound HTTPS 443. |
| OIDC `AssumeRoleWithWebIdentity` is denied | Confirm provider URL/audience, role ARN, caller `id-token: write`, and exact `sub` repository/branch in the role trust relationship. |
| GitHub gets `ssm:SendCommand` access denied | Confirm the permissions policy contains the AWS-RunShellScript document ARN and exact EC2 instance ARN for the configured Region/account. |
| SSM accepts command but execution fails | Inspect invocation stdout/stderr in the Actions logs; verify Docker is active, the Docker Hub image is public, outbound HTTPS works, and port 80 is available. |
| Docker push fails | Verify the Docker Hub username and Read & Write token are repository secrets in `java-project`, and that the account can push to the named repository. |
| Tag push fails | Verify Actions `contents: write` and confirm repository rules permit the GitHub Actions bot to create tags. |
