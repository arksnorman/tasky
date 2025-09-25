# Exercise Doc

## Overview

Here, I focused on building a multi-tier architecture in AWS with
deliberately weak configurations to demonstrate both functionality and
security risks. The infrastructure consisted of:

-   A Kubernetes cluster (EKS) running the **Tasky** web app.
-   An EC2 VM running MongoDB on an outdated Ubuntu version.
-   An S3 bucket for automated MongoDB backups with public-read enabled.
-   CI/CD pipelines using GitHub Actions for IaC (CloudFormation) and
    Docker image publishing.
-  Ansible for servers task automation

------------------------------------------------------------------------

## Steps and Commands

### 1. EC2 + MongoDB Setup

-   Launched Ubuntu 22.04 EC2 instance (deliberately outdated).
-   Used the default AWS-created `ubuntu` user.
-   Installed MongoDB:

``` bash
wget https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/6.0/multiverse/binary-amd64/mongodb-org-server_6.0.26_amd64.deb
sudo dpkg -i mongodb-org-server_6.0.26_amd64.deb
sudo systemctl enable mongod
sudo systemctl start mongod
tail -f /var/log/mongodb/mongod.log
```

-   Encountered **WiredTiger permission denied** error:

```
    wiredtiger_open: /var/lib/mongodb/WiredTiger.turtle: handle-open: open: Permission denied
```
-   Fixed by updating ownership/permissions on `/var/lib/mongodb`.

-   Installed MongoDB database tools:

``` bash
wget https://fastdl.mongodb.org/tools/db/mongodb-database-tools-ubuntu2204-arm64-100.13.0.deb
sudo dpkg -i mongodb-database-tools-ubuntu2204-arm64-100.13.0.deb
```

------------------------------------------------------------------------

### 2. S3 Bucket for Backups

-   Created bucket with public-read access.
-   Faced error:


```
    Bucket cannot have ACLs set with ObjectOwnership's BucketOwnerEnforced setting
```
-   Solved by removing ACLs since ObjectOwnership already enforced.

------------------------------------------------------------------------

### 3. CI/CD with GitHub Actions

-   Added AWS credentials + PEM key to GitHub Secrets.
-   Used CloudFormation deploy action:
    -   [aws-actions/aws-cloudformation-github-deploy](https://github.com/aws-actions/aws-cloudformation-github-deploy)
-   Pushed Docker images with GitHub Actions:
    -   [Publish Docker
        images](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
-   Encountered Docker build error:


```
    warning: FromAsCasing mismatch (line 10)
```
-   Fixed by ensuring `AS` casing matched `FROM`.

------------------------------------------------------------------------

### 4. EKS Cluster & App Deployment

-   Deployed via CloudFormation.
-   Errors encountered:
    -   **No cluster found** → solved with `dependsOn`.
    -   **AMI Type only supported for k8s ≤1.32** → solved by removing
        `amitype` so AWS chooses compatible type.
-   Installed `kubectl`:
    -   [Install
        kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
-   Updated kubeconfig:

``` bash
aws eks update-kubeconfig --name <eks-cluster-name>
```

-   Verified MongoDB connectivity:

``` bash
sudo ss -lnpt | grep mongod
```

-   Fixed pod to EC2 connectivity by:
    -   Allowing EKS Cluster VPC CIdr block  on port `27017`.
    -   Updating MongoDB bind IP from `127.0.0.1` → private EC2 IP.

------------------------------------------------------------------------

### 5. Database Backups (Cron)

-   Automated with `mongodump` cron job.
-   Faced syntax error in `/etc/cron.d`.
-   Solved by correcting cron file format ([StackOverflow
    reference](https://stackoverflow.com/questions/53151124/jobs-in-etc-cron-d-are-not-working-on-ubuntu)).

------------------------------------------------------------------------

## Challenges & Debugging

-   CloudFormation errors (EKS dependencies, S3 ACLs).
-   MongoDB WiredTiger storage permission issue.
-   Networking/security group misconfigurations.
-   GitHub Actions pipeline failures (Dockerfile casing, secret
    management).

Debugging tools/commands used: - `tail -f /var/log/mongodb/mongod.log` -
`ss -lnpt | grep mongod` - Security group rule checks in AWS Console. -
`kubectl get pods` / `kubectl logs`.

------------------------------------------------------------------------

## Weak Configurations & Consequences

-   Outdated Ubuntu OS + MongoDB → vulnerable to CVEs.
-   Public S3 bucket → sensitive data leakage risk.
-   Over-permissive IAM roles → privilege escalation.
-   Web app with cluster-admin privileges → full cluster compromise
    risk.
-   Public SSH access to DB VM → brute force attack surface.

------------------------------------------------------------------------

## References

-   [Ansible Installation
    Guide](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)
-   [MongoDB Downloads](https://www.mongodb.com/try/download/community)
-   [Install
    kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
-   [CloudFormation GitHub
    Action](https://github.com/aws-actions/aws-cloudformation-github-deploy)
-   [Publish Docker Images with GitHub
    Actions](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
-   [EKS CloudFormation
    Guide](https://medium.com/@priyanshigola8/create-a-aws-eks-cluster-using-cloudformation-f45e689f1d42)
-   [StackOverflow  cron jobs
    issue](https://stackoverflow.com/questions/53151124/jobs-in-etc-cron-d-are-not-working-on-ubuntu)
    