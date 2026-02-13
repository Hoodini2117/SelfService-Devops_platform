# Phase 1 — Containerizing a Real App 

** Goal: 
Get a real production-grade web app running locally and inside Docker as the base for cloud deployment. **

### What I did:

Chose a real Next.js e-commerce app (Stripe + Sanity)

Fixed broken dependencies and peer conflicts

Pinned stable versions:

React 18.2

React-DOM 18.2

Next.js 13

Got the app running locally

Wrote a production-grade multi-stage Dockerfile

Built and ran the app successfully inside Docker using Node 20

### What broke (and why):

npm install failed due to peer dependency conflicts

App crashed because modern React/Next versions didn’t match the old codebase

Docker build failed because Node 18 didn’t meet Next.js requirements

npm run dev worked locally but failed in containers

### How I fixed it:

Used --legacy-peer-deps to resolve npm conflicts

Forced compatible React + Next versions

Switched Docker base image to Node 20

Used proper production flow: next build + next start

### Key learnings:

Modern JS apps break fast if versions aren’t pinned

Containers freeze runtime versions and prevent CI/CD surprises

Dev mode ≠ production mode in real deployments

Multi-stage Docker builds matter

### State after Phase 1:

App runs locally ✅

App runs in Docker ✅

Not yet in cloud ❌

Not yet on Kubernetes ❌

# Phase 2 — Cloud, Kubernetes & Public Access

** Goal:
Turn the Dockerized app into a real, internet-accessible system. **

### Infrastructure (Terraform):

Rebuilt everything cleanly using code:

Resource Group

Azure Container Registry (ACR)

Azure Kubernetes Service (AKS)

### Fixed:

Empty Terraform state issues

Missing subscription binding

Provider authentication

Azure VM quota limitations

### Result:
Infrastructure is fully reproducible from Terraform.

Containers → Cloud

Rebuilt the Docker image

Tagged and pushed it to ACR

Verified the image exists in the registry

This created a versioned, cloud-hosted runtime artifact.

Kubernetes Setup

### Created:

Namespace

Deployment (replicas)

Service

NGINX Ingress Controller

Ingress rules

Wired the full request path:

Azure Load Balancer → NGINX → Service → Pods → App

Live on the Internet

Azure assigned a public IP

Opened the site from my home network

The e-commerce app loaded successfully

### This confirmed:
The platform works end-to-end on the public internet.

What broke (and what it taught me)

Terraform showing 0 resources → state vs actual infra matters

AKS not visible → subscription context is critical

VM size rejected → regional quota constraints are real

Empty ACR → images must be explicitly pushed

Pods stuck creating → registry permissions matter

Ingress failing on campus Wi-Fi → real networks filter traffic

Works on home network → platform setup is correct

# Phase 3 — Blue–Green Deployment Strategy
Overview

In Phase 3, the project implements a Blue–Green deployment strategy on Kubernetes to achieve zero-downtime releases, safe rollouts, and instant rollback capability.

This phase focuses on decoupling deployment from traffic routing, which is a core requirement for building a reliable, self-service DevOps platform.

### What is Blue–Green Deployment?
Blue–Green deployment runs two identical environments in parallel:

Blue — the stable production version

Green — the new version being deployed

Only one environment receives live traffic at a time, eliminating downtime and reducing release risk.
How It Works in This Project
Parallel Deployments

Two Kubernetes Deployments run simultaneously:

ecommerce-app-blue
ecommerce-app-green

They are identical except for a version label:
```bash
app: ecommerce
version: blue | green 
``` 
### Traffic Control via Service

A single Kubernetes Service routes traffic based on labels.
Example: routing traffic to Green
```bash
selector:
  app: ecommerce
  version: green
```
Updating the Service selector switches traffic instantly without restarting Pods.

### Instant Rollback

If an issue occurs, traffic can be switched back to Blue by updating the Service selector. Rollback is immediate and does not require redeployment.

### Why This Matters

This phase demonstrates:

Zero-downtime deployments

Safe CI/CD-driven releases

Instant rollback capability

Production-grade Kubernetes traffic management

Blue–Green deployment forms the foundation for automating safe releases in the self-service DevOps platform.
### Installed NGINX Ingress Controller

AKS does not expose applications by default, so I installed an NGINX Ingress Controller using Helm to provide a public IP and route traffic to the service.

This gave a stable entry point:

Public IP → Ingress → Service → Blue/Green Pods

### Modified Azure DevOps Pipeline to Support Blue-Green Workflow

I redesigned the CI/CD pipeline to follow a controlled promotion model:

-Deploy updates only to Green
-Wait for validation
-Use kubectl patch service to switch traffic instantly
-Keep Blue untouched for rollback
### a lot of challenges were to be faced this time
- Main problem was my azure account which i had to reconfigure as due to some circumstances old account was unavailable.

- reconfigured all the aks - acr - service connections - agent pools to work with the new azurea ccount.

-VM SKU and vCPU Quota Errors During AKS Creation Caused by switching accounts fixed by requesting 4 bs2_v2 cpu's.

-git repo was not tracked correctly so had to reconfigure it whole.

-ManualValidation Task Error in Pipeline

Azure DevOps requires approval tasks to run in an agentless job.
I split the stage into:

Server job → waits for approval

Agent job → performs the traffic switch

-Ingress Controller Was Not Installed After Rebuild

Terraform does not install ingress automatically, so routing initially failed.
I installed it manually using Helm and reapplied ingress rules.

### What I Achieved by the End of This Phase

✔ Implemented production-style Blue-Green deployment
✔ Achieved zero-downtime releases
✔ Enabled instant rollback capability
✔ Built a controlled promotion workflow in Azure DevOps
✔ Validated separation of infrastructure (Terraform) and delivery (CI/CD)
✔ Created a safer release model aligned with real-world DevOps practices

to complete the fully automated blue-green deployment..we needed to make some changes and add some features as well. In them ,there were :-
- a unique image tag for each build so that we can build green using this tag and blue will remain untouched with the previous tag, this ensures versions are traceable.
- automated validation - currently validation is being done manually. To be called truly self - service we need to automate this process, 
- rollback - currently rollback is manual. To counter this we will add an automated rollback if and ever validation fails. This will be fairly  simple as blue is active and we just need to switch the selector.

Errors & Problems I Faced

This is where i learned most of the stuff...as learning from errors is better than having everything work on one go which is imppossible.

### 1. Service Selector Corruption (version="")

At one point, my Promote stage ran when targetEnv was empty.

This patched the service like this:

version: ""


Kubernetes accepted it.

Pipeline didn’t fail.

But detection completely broke.

 Symptom:
Live environment :


(blank)

Fix:

Added safety check before patch:

if [ -z "$(targetEnv)" ]; then exit 1


Manually repaired service selector.

This taught me:

Kubernetes accepts empty strings silently — pipelines must guard against bad state.

### 2. Variable Not Propagating Between Stages

Azure DevOps output variables didn’t automatically flow across multiple stages.

Deploy stage worked.
Promote stage didn’t receive targetEnv.

 Root Cause:

Output variables are stage-scoped.

Fix:

Forwarded targetEnv stage-by-stage using:

##vso[task.setvariable variable=targetEnv;isOutput=true]


This was a major Azure DevOps learning moment.

### 3. Bash Syntax Error

I accidentally wrote:

TARGET = "blue"


Instead of:

TARGET="blue"


That small space corrupted the variable and even produced:


###  4. Service YAML vs Cluster State Confusion

I fixed the service in the cluster manually, but forgot to update service.yaml in Git.

Later, when applying YAML again, it removed version from selector.

This caused version detection to break again.

 Fix:

Updated service.yaml permanently to include:
```bash
selector:
  app: ecommerce
  version: blue
```

This stabilized the architecture.

Major lesson:

Git desired state must match cluster structure.

### 5. Old Deployment Interference

Initially I still had:

ecommerce-app


Running without version label.

Service was routing to all deployments.

This made Blue–Green meaningless.

### Fix:

Deleted old deployment.
Kept only blue and green.

Lesson:

Blue–Green must be structurally isolated.

### Final Stable Flow

After all fixes, my pipeline now does:

Detect live environment

Deploy to idle environment

Validate rollout

Automatically promote (no manual gate)

Patch service selector safely

Verify switch

Now the flow is:
```bash
Git push → zero-downtime production release
```

And alternation works correctly:

Blue → Green → Blue → Green

### What I Really Learned

This wasn’t just about Blue–Green.
I learned:

Kubernetes service selector behavior
How fragile label-based routing can be
Azure DevOps output variable scoping
Why safety guards are mandatory
How silent failures happen in CI/CD
Why infra YAML and runtime state must align
How small Bash mistakes can break production logic
