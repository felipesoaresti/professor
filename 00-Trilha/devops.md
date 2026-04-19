---
tags:
  - trilha
  - devops
area: devops
tipo: trilha
---

# Trilha DevOps

> Canvas geral: [[trilha-devops-sre]]

## Assuntos Cobertos

| Assunto | Conteúdo | Exercícios | Status |
|---|---|---|---|
| Git | [[03-DevOps/git]] | [[11-Exercicios/git]] | Exercícios gerados — pendente execução |

## Sequência Recomendada

### Fundamentos de Versionamento e Automação
1. [[03-DevOps/git]] — Object store, rebase, hooks, workflows de time
2. *GitHub Actions* — Workflows, matrix, reusable workflows, OIDC *(a fazer)*
3. *Docker avançado* — Multi-stage builds, layer cache, scan de imagem *(a fazer)*

### CI/CD e Deploy
4. *Deploy strategies* — Blue-green, canary, rolling com GitHub Actions *(a fazer)*
5. *GitOps com ArgoCD* — Pull model, reconciliation, sync policies *(a fazer)*

### Infrastructure as Code
6. *Terraform fundamentals* — State, plan/apply, backend remoto *(a fazer)*
7. *Terraform avançado* — Módulos, workspaces, drift detection, import *(a fazer)*

### Segurança e Qualidade
8. *DevSecOps* — Secret scanning, SAST, SBOM, least privilege *(a fazer)*
9. *Ansible* — Idempotência, roles, inventory dinâmico *(a fazer)*

## Próximo Sugerido

[[03-DevOps/github-actions]] — natural continuidade depois de Git: pipelines que disparam em push/PR e deployam no cluster.
