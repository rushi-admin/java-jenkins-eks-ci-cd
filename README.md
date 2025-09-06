# Jenkins → Maven → DockerHub → EKS (Spring Boot Demo)

**Flow**: GitHub → Jenkins (agent) → Maven build & tests → Docker build & push → `kubectl apply` to EKS.

## Quick start
1. Create a new GitHub repository and push this project.
2. In Jenkins:
   - Create a **Pipeline** job → **Pipeline script from SCM** → point to your repo.
   - Ensure your job runs on the agent that has Docker, AWS CLI, kubectl, Maven.
   - Create credentials **dockerhub-creds** (username/password) globally.
3. First build:
   - Click **Build with Parameters** → set:
     - `DOCKERHUB_REPO`: `yourdockerhubuser/java-eks-demo`
     - `EKS_CLUSTER`: your cluster name
     - `AWS_REGION`: cluster region (e.g. `ap-south-1`)
     - `K8S_NAMESPACE`: `demo`
4. After deploy, Jenkins prints the **Load Balancer hostname**. Open it in a browser.

## What gets deployed?
A small Spring Boot app exposing `/` that returns JSON. Health probes are via Actuator:
- `/actuator/health/liveness`
- `/actuator/health/readiness`

## Notes
- The pipeline tags images with both `:latest` and `:<short-git-sha>` and updates k8s manifest to the commit-tag for traceability.
- Ensure your Jenkins agent user belongs to the `docker` group.
- Prefer attaching an IAM Role to the Jenkins agent EC2 with EKS access. Otherwise, configure AWS keys for the agent environment.
