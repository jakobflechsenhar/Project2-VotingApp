# ------------------------------------------------------ #
# README: Voting App - Kubernetes Deployment with CI/CD
# ------------------------------------------------------ #

A microservices-based voting application deployed on Amazon EKS with automated CI/CD using GitHub Actions.


## Architecture Overview
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Browser   │────▶│   NGINX     │────▶│  Vote App   │
└─────────────┘     │  Ingress    │     │  (Python)   │
                    │             │     └──────┬──────┘
┌─────────────┐     │             │            │
│   Browser   │────▶│  /vote      │            ▼
└─────────────┘     │  /result    │     ┌─────────────┐
                    │             │     │    Redis    │
┌─────────────┐     │             │     │   (Cache)   │
│   Browser   │────▶│             │     └──────┬──────┘
└─────────────┘     └──────┬──────┘            │
                           │                    ▼
                           │            ┌─────────────┐
                           │            │   Worker    │
                           │            │   (.NET)    │
                           │            └──────┬──────┘
                           │                   │
                    ┌──────▼──────┐            ▼
                    │ Result App  │     ┌─────────────┐
                    │  (Node.js)  │◀────│ PostgreSQL  │
                    └─────────────┘     │     (DB)    │
                                        └─────────────┘
```

## Directory Structure
```
.
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yaml    # GitHub Actions CI/CD pipeline
├── existing-code-repo/            # Original voting app source code (Diogo's code/images)
│   ├── ...
│   ├── ...
├── k8s/                           # Kubernetes manifests
│   ├── ingress.yaml               # NGINX routing rules
│   ├── postgres-deployment.yaml   # PostgreSQL database + persistent storage
│   ├── redis-deployment.yaml      # Redis cache for temporary vote storage
│   ├── result-deployment.yaml     # Result frontend deployment
│   ├── vote-deployment.yaml       # Vote frontend deployment (2 replicas)
│   └── worker-deployment.yaml     # Background worker for vote processing
├── result/
│   └── Dockerfile                # Container image for result service (created using the existing code in existing-code-repo)
├── vote/
│   └── Dockerfile                # Container image for vote service (created using the existing code in existing-code-repo)
├── worker/
│   └── Dockerfile                # Container image for worker service (created using the existing code in existing-code-repo)
├── .gitmodules                   # specifies that the existing-code-repo folder is itself a cloned repo from my Github account
```


## Components

#### 1. Vote App (Python Flask)
- Purpose: Frontend for users to cast votes between two options
- Port: 80
- Replicas: 2 (load balanced)
- Flow: Receives votes → Stores in Redis

#### 2. Redis (In-Memory Cache)
- Purpose: Temporary storage for votes before processing
- Port: 6379
- Why Redis?: Fast, handles high-frequency writes efficiently

#### 3. Worker (.NET)
- Purpose: Background processor that moves votes from Redis to PostgreSQL
- Flow: Reads from Redis → Processes → Writes to PostgreSQL
- Note: No service needed (doesn't accept incoming traffic)

#### 4. PostgreSQL (Database)
- Purpose: Permanent storage for processed votes
- Port: 5432
- Storage: 5GB persistent volume (survives pod restarts)
- Credentials: Managed via Kubernetes Secrets

#### 5. Result App (Node.js)
- Purpose: Frontend displaying real-time voting results
- Port: 80
- Flow: Reads from PostgreSQL → Displays results

#### 6. NGINX Ingress
- Purpose: Routes external traffic to internal services
- Routes:
  - `/vote` → Vote service
  - `/result` → Result service


## CI/CD Pipeline
The GitHub Actions pipeline automates the entire deployment process:
Pipeline Steps:
1. Trigger: Activates on push to main branch or manual dispatch
2. Build: Creates Docker images for vote, result, and worker services
3. Push: Tags and pushes images to Docker Hub with:
   - `latest` tag for current version
4. Deploy: Updates Kubernetes deployments with new images
5. Verify: Checks all deployments are healthy


## Access points:
- Vote: http://<INGRESS-HOSTNAME>/vote
- Result: http://<INGRESS-HOSTNAME>/result
