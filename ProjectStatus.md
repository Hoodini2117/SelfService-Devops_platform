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

