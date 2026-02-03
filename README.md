# Enterprise GitOps Workflow & Architecture

This repository follows a strict **Multi-Tenant GitOps** architecture designed for enterprise scalability. This document explains the relationship between files and the standard workflow for developers and operators.

## 1. High-Level Architecture

The setup separates **Definition** (What app?) from **Instantiation** (Where does it run?) and **Permission** (Who owns it?).

### File Interaction Diagram

```mermaid
graph TD
    User[Operator] -->|kubectl apply| Bootstrap(Bootstrap App)
    
    subgraph "Phase 1: Security & Governance"
    Bootstrap -->|Installs| Projects[projects/*.yaml]
    Projects -->|Creates| TeamProject(AppProject: dev-team-a)
    end
    
    subgraph "Phase 2: Automation"
    Bootstrap -->|Installs| AppSets[clusters/dev-cluster/*.yaml]
    AppSets -->|Generates| ArgoApp(Application: demo-app-dev)
    end
    
    subgraph "Phase 3: Deployment"
    ArgoApp -->|Reads Manifests| GitRepo[apps/dev-team-a/demo-app]
    ArgoApp -->|Deploys to| K8s(Kubernetes Cluster)
    end
    
    TeamProject -.->|Enforces RBAC| ArgoApp