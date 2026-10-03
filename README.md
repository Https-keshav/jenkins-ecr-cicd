# Dockerized CI/CD Pipeline with Jenkins, SonarQube and AWS ECR

A declarative Jenkins pipeline that takes a Java web application from source code to a Docker image in AWS Elastic Container Registry (ECR). On every run it fetches the code, builds, tests, runs static analysis, enforces a quality gate, builds a multistage Docker image and pushes it to ECR.

The application used is the open-source [vprofile project](https://github.com/hkhcoder/vprofile-project) (`docker` branch).

---

## Table of Contents

- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
- [The Jenkinsfile](#the-jenkinsfile)
- [Running the Pipeline](#running-the-pipeline)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Architecture

```
GitHub ──► Jenkins ──► Maven build ──► Unit tests ──► Checkstyle
                                                          │
                                                          ▼
AWS ECR ◄── docker push ◄── docker build ◄── Quality Gate ◄── SonarQube
```

## Pipeline Stages

| # | Stage | What it does |
|---|-------|--------------|
| 1 | Fetch code | Clones the `docker` branch of the vprofile repo |
| 2 | Build | `mvn install -DskipTests`, archives the `.war` artifact |
| 3 | Unit Test | `mvn test` |
| 4 | Checkstyle Analysis | `mvn checkstyle:checkstyle` for code style report |
| 5 | Sonar Code Analysis | Runs `sonar-scanner` against SonarQube |
| 6 | Quality Gate | Waits for SonarQube's verdict and aborts on failure |
| 7 | Build App Image | Builds the multistage Docker image, tagged with the build number |
| 8 | Upload App Image | Pushes `:<BUILD_NUMBER>` and `:latest` to ECR |

## Tech Stack

- **CI server:** Jenkins (Pipeline as Code)
- **Build:** Maven 3.9, JDK 17
- **Code quality:** Checkstyle, SonarQube
- **Containers:** Docker (multistage build)
- **Registry:** AWS ECR
- **Cloud and access:** AWS IAM, AWS CLI

## Prerequisites

- A Jenkins server (Linux VM or EC2) with SSH access
- A SonarQube server connected to Jenkins (server name `sonarserver`)
- An AWS account
- Jenkins tools configured under **Manage Jenkins → Tools**:
  - Maven named `MAVEN3.9`
  - JDK named `JDK17`
  - SonarQube Scanner named `sonar8.0`

---

## Setup Guide

### 1. Prepare the Jenkins server

SSH into the Jenkins server.

**Install the AWS CLI**
```bash
sudo snap install aws-cli --classic
```

**Install Docker Engine**
Follow the official instructions: https://docs.docker.com/engine/install/

**Allow Jenkins to use Docker**
```bash
sudo usermod -aG docker jenkins
sudo reboot
```

Verify after the reboot:
```bash
sudo -u jenkins docker ps
```

### 2. Create an IAM user

1. In AWS, go to **IAM → Users → Create user**.
2. Attach a policy that allows pushing to ECR. `AmazonEC2ContainerRegistryPowerUser` is enough and is more restrictive than full access.
3. Open the user, go to **Security credentials → Create access key**, choose **CLI** and save the Access Key ID and Secret Access Key.

### 3. Create the ECR repository

1. Go to **ECR → Repositories → Create repository**.
2. Name it `vprofileimage` (private).
3. Note the region and the repository URI, for example `<account-id>.dkr.ecr.us-east-1.amazonaws.com/vprofileimage`.

### 4. Install Jenkins plugins

Go to **Manage Jenkins → Plugins → Available plugins** and install:

- Amazon Web Services SDK :: All (or the ECR SDK and credentials plugins)
- Amazon ECR
- Docker Pipeline
- CloudBees Docker Build and Publish

Restart Jenkins afterwards.

### 5. Store AWS credentials in Jenkins

1. **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Kind: **AWS Credentials**
3. ID: `awscred`
4. Paste the Access Key ID and Secret Access Key from step 2.

### 6. Configure the pipeline

Create a new **Pipeline** job. Either paste the contents of [`Jenkinsfile`](./Jenkinsfile) into the script box, or choose **Pipeline script from SCM** and point it at this repo. Update the `environment` block in the Jenkinsfile:

| Variable | Value |
|----------|-------|
| `registryCredential` | `ecr:<region>:awscred` |
| `appRegistry` | `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>` |
| `vprofileRegistry` | `https://<account-id>.dkr.ecr.<region>.amazonaws.com` |

---

## The Jenkinsfile

The complete pipeline lives in [`Jenkinsfile`](./Jenkinsfile) in this repo. The stage-by-stage breakdown is in [Pipeline Stages](#pipeline-stages).

Before using it, edit the `environment` block at the top with your own values (see [step 6](#6-configure-the-pipeline)). Everything else works as is.

---

## Running the Pipeline

1. Click **Build Now** on the Jenkins job.
2. Watch the stage view. All eight stages should turn green.
3. Open **ECR → your repository** and confirm two image tags: the build number and `latest`.
4. Pull and run it to test:
   ```bash
   aws ecr get-login-password --region us-east-1 | \
     docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
   docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/vprofileimage:latest
   ```

---

## Troubleshooting

Issues I hit while building this, and their fixes.

| Symptom | Cause | Fix |
|---------|-------|-----|
| `permission denied ... docker.sock` | Jenkins user not in the docker group | `sudo usermod -aG docker jenkins`, then restart or reboot |
| `name unknown: repository ... does not exist` | Typo in `appRegistry` | The name must match the ECR repository exactly |
| `The request signature we calculated does not match` | Wrong or badly pasted AWS secret key | Recreate the access key and update the `awscred` credential, no trailing spaces |
| `Could not find credentials entry with ID 'awscred'` | Wrong credential ID or kind | Use kind **AWS Credentials** with ID `awscred` |
| `AccessDeniedException ... ecr:` | IAM user lacks ECR permissions | Attach an ECR push policy |
| Quality Gate status `ERROR` | Project fails the default Sonar way gate (coverage, hotspots) | Review issues in SonarQube, create a custom gate, or set `abortPipeline: false` temporarily |
| Docker build killed, exit code 137 | Not enough memory on the instance | Add swap or use a larger instance |

---

## Security Notes

- Never commit AWS keys or put them in the Jenkinsfile. Keep them in the Jenkins credential store.
- Use the least-privilege IAM policy that still allows pushing to ECR.
- Rotate access keys if they were ever exposed in logs, screenshots or chats.
- On AWS-hosted Jenkins, prefer an IAM role attached to the instance over long-lived keys.

## Future Improvements

- Deploy the image to ECS or EKS as a final stage
- Add JaCoCo XML coverage reporting (`sonar.coverage.jacoco.xmlReportPaths`) so the quality gate reflects real coverage
- Add a Trivy or ECR image scan stage
- Move to a webhook trigger on push instead of manual builds
- Slack or email notifications on pipeline result
- Provision the Jenkins server with Terraform

## Author

**Keshav Toshniwal**
- GitHub: [Https-keshav](https://github.com/Https-keshav)

Based on the vprofile project by [hkhcoder](https://github.com/hkhcoder/vprofile-project).