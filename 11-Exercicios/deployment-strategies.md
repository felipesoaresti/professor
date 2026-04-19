---
tags:
  - exercicios
  - kubernetes
  - deployment
  - strategies
  - canary
  - blue-green
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/deployment-strategies]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Deployment Strategies

> Conteúdo: [[05-Kubernetes/deployment-strategies]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30) | Ingress: 192.168.3.100 → `*.staypuff.info`
**Namespace de trabalho:** `default` ou `estudo-felipe`
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Comparar RollingUpdate vs Recreate na prática

**Contexto:** O time de produto questiona se vale a pena configurar a strategy ou deixar o padrão. Você vai demonstrar concretamente a diferença de comportamento entre as duas strategies nativas.

**Missão:** Crie dois Deployments idênticos — um com `RollingUpdate` e outro com `Recreate` — e observe o comportamento durante a atualização de imagem.

**Requisitos:**
- [ ] Criar Deployment `webapp-rolling` com `strategy: RollingUpdate` e 4 réplicas usando `nginx:1.26`
- [ ] Criar Deployment `webapp-recreate` com `strategy: Recreate` e 4 réplicas usando `nginx:1.26`
- [ ] Atualizar a imagem de ambos para `nginx:1.27` simultaneamente em dois terminais
- [ ] No terminal 1: `kubectl get pods -l app=webapp-rolling -w` para observar o rolling
- [ ] No terminal 2: `kubectl get pods -l app=webapp-recreate -w` para observar o recreate
- [ ] Registrar: em qual strategy todos os Pods ficaram com status `Terminating` ao mesmo tempo?
- [ ] Registrar: em qual strategy sempre havia Pods `Running` durante a transição?
- [ ] Deletar os dois Deployments ao final

**Verificação:**
```bash
kubectl rollout status deployment/webapp-rolling
kubectl rollout status deployment/webapp-recreate
# Rolling → avança gradualmente; Recreate → derruba tudo, depois cria
```

---

## Exercício 2: Tuning de maxSurge e maxUnavailable

**Contexto:** Você tem um Deployment crítico com SLA de 99.9% e precisa garantir zero-downtime durante deploys. O time também quer que o rollout seja rápido quando há urgência.

**Missão:** Configure e teste dois perfis de RollingUpdate: um conservador (zero-downtime) e um agressivo (rápido).

**Requisitos:**
- [ ] Criar Deployment `webapp-safe` com 4 réplicas, `maxUnavailable: 0` e `maxSurge: 1`
- [ ] Criar Deployment `webapp-fast` com 4 réplicas, `maxUnavailable: 2` e `maxSurge: 2`
- [ ] Para cada um, atualizar a imagem e contar o tempo até o rollout completar
- [ ] Observar quantos Pods existem ao mesmo tempo durante o rollout de cada um
- [ ] Verificar: em `webapp-safe`, o total de Pods nunca vai abaixo de 4?
- [ ] Verificar: em `webapp-fast`, o rollout completa mais rápido?
- [ ] Explicar o trade-off de cada configuração
- [ ] Deletar os Deployments ao final

**Verificação:**
```bash
time kubectl rollout status deployment/webapp-safe
time kubectl rollout status deployment/webapp-fast
# webapp-fast deve completar em menos tempo
# webapp-safe nunca deve ter < 4 pods Ready durante o rollout
```

---

## Exercício 3: Rollout controlado com pause e resume

**Contexto:** O time de QA quer validar manualmente um subset de Pods com a nova versão antes de continuar o rollout. Você vai implementar um processo de rollout controlado por humano.

**Missão:** Faça um rollout que pause no meio, permita inspeção manual da nova versão, e só continue após validação.

**Requisitos:**
- [ ] Criar Deployment `webapp` com 6 réplicas usando `nginx:1.26`
- [ ] Iniciar atualização para `nginx:1.27` e pausar o rollout **imediatamente**
- [ ] Confirmar que o rollout está em estado `paused` via `kubectl rollout status`
- [ ] Identificar quais Pods são da versão nova e quais são da versão antiga
- [ ] Testar a nova versão fazendo `kubectl exec` em um Pod v2 e confirmando a versão do nginx
- [ ] Retomar o rollout com `kubectl rollout resume`
- [ ] Aguardar completion e verificar que todos os Pods são v2
- [ ] Deletar o Deployment ao final

**Verificação:**
```bash
kubectl rollout status deployment/webapp
# Deve mostrar: deployment "webapp" successfully rolled out

kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
# Todos devem mostrar nginx:1.27
```

---

## Exercício 4: Blue-Green deployment manual

**Contexto:** O time vai lançar uma versão com mudanças visuais significativas e quer poder reverter instantaneamente se houver problema. Você vai implementar Blue-Green no cluster.

**Missão:** Configure um ambiente Blue-Green completo e execute uma troca de versão com rollback posterior.

**Requisitos:**
- [ ] Criar Deployment `webapp-blue` (3 réplicas, `nginx:1.26`) com label `slot: blue`
- [ ] Criar Service `webapp` com selector `app: webapp, slot: blue`
- [ ] Confirmar que o Service está roteando para os Pods blue via `kubectl describe service`
- [ ] Criar Deployment `webapp-green` (3 réplicas, `nginx:1.27`) com label `slot: green`
- [ ] Aguardar que todos os Pods green estejam `Running` e `Ready`
- [ ] Executar a troca: fazer `kubectl patch service` para apontar para `slot: green`
- [ ] Confirmar que os endpoints do Service mudaram para os Pods green
- [ ] Simular um problema: executar o rollback para blue com outro `patch`
- [ ] Confirmar que o Service voltou a apontar para Pods blue
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
kubectl describe service webapp | grep -E "Selector|Endpoints"
# Após o patch green: Selector deve conter slot=green, Endpoints devem ser IPs dos Pods green
# Após o rollback: Selector deve conter slot=blue, Endpoints devem ser IPs dos Pods blue
```

---

## Exercício 5: Canary por proporção de réplicas

**Contexto:** O time quer expor uma nova feature para aproximadamente 20% dos usuários antes de um rollout completo. Você vai implementar um Canary simples com dois Deployments e um Service compartilhado.

**Missão:** Configure um Canary deployment onde ~20% do tráfego vai para a nova versão e ~80% para a versão estável.

**Requisitos:**
- [ ] Criar Deployment `webapp-stable` com 8 réplicas usando `nginx:1.26`
- [ ] Criar Deployment `webapp-canary` com 2 réplicas usando `nginx:1.27`
- [ ] Criar Service `webapp-svc` com selector apenas `app: webapp` (sem distinção de versão)
- [ ] Confirmar que o Service tem endpoints de ambos os Deployments (10 endpoints no total)
- [ ] Fazer 20 requests para o Service via `kubectl run` e contar quantos respondem com versão canary
- [ ] Aumentar o canary para 5 réplicas (50/50) e observar a mudança na proporção
- [ ] Decidir "promover": escalar stable para a nova imagem e deletar o canary
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
kubectl describe service webapp-svc | grep Endpoints
# Deve listar 10 IPs: 8 dos Pods stable + 2 dos Pods canary

# Teste de proporção (executar de dentro do cluster):
kubectl run test --image=nginx --restart=Never --rm -it -- sh -c \
  'for i in $(seq 1 20); do curl -s http://webapp-svc | grep -o "nginx/[0-9.]*"; done'
# Aproximadamente 2/10 das respostas devem ser nginx/1.27.x
```

---

## Exercício 6: Canary com Ingress weight annotation (avançado)

**Contexto:** O time quer controle preciso de tráfego — exatamente 10% para o canary, independente do número de réplicas. Você vai usar as annotations do nginx Ingress Controller.

**Missão:** Configure um Canary com Ingress weight, ajuste o peso gradualmente, e promova a nova versão.

**Requisitos:**
- [ ] Criar Deployment `webapp-stable` (2 réplicas, `nginx:1.26`) e Service `svc-stable`
- [ ] Criar Deployment `webapp-canary` (2 réplicas, `nginx:1.27`) e Service `svc-canary`
- [ ] Criar Ingress principal para `webapp.staypuff.info` apontando para `svc-stable`
- [ ] Criar Ingress canary com annotations `nginx.ingress.kubernetes.io/canary: "true"` e `canary-weight: "10"` apontando para `svc-canary`
- [ ] Confirmar que o Ingress canary está ativo via `kubectl describe ingress webapp-canary`
- [ ] Aumentar o peso para 50% via `kubectl annotate --overwrite`
- [ ] Aumentar o peso para 100% (promoção completa)
- [ ] Executar a promoção final: deletar o Ingress canary e atualizar o Deployment stable para nginx:1.27
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
kubectl get ingress
# Deve mostrar dois Ingress: webapp-stable e webapp-canary

kubectl describe ingress webapp-canary | grep -A5 Annotations
# nginx.ingress.kubernetes.io/canary: true
# nginx.ingress.kubernetes.io/canary-weight: 10
```

---

## Exercício 7: Rollout travado — diagnóstico e recuperação (avançado)

**Contexto:** Você recebe um alerta: "deploy da webapp está travado há 10 minutos". O `kubectl rollout status` está pendurado. Você precisa diagnosticar e resolver.

**Missão:** Simule um rollout travado e execute o diagnóstico completo até a resolução.

**Requisitos:**
- [ ] Criar Deployment `webapp` com 4 réplicas e `nginx:1.26`
- [ ] Atualizar a imagem para `nginx:versao-que-nao-existe` (simula imagem inválida)
- [ ] Observar o rollout travando via `kubectl rollout status`
- [ ] Diagnosticar: identificar o Pod com problema via `kubectl get pods`
- [ ] Ler os eventos do Pod problemático via `kubectl describe pod`
- [ ] Identificar a causa raiz (ImagePullBackOff)
- [ ] Executar o rollback: `kubectl rollout undo deployment/webapp`
- [ ] Confirmar que todos os Pods voltaram para `nginx:1.26` e estão `Running`
- [ ] Ver o histórico de revisões com `kubectl rollout history`
- [ ] Deletar o Deployment ao final

**Verificação:**
```bash
kubectl rollout status deployment/webapp
# Após o undo: deployment "webapp" successfully rolled out

kubectl get pods -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# Todos devem mostrar nginx:1.26

kubectl rollout history deployment/webapp
# Deve mostrar pelo menos 3 revisões: 1.26 → versao-invalida (falhou) → rollback para 1.26
```

---

## Exercício 8: Deploy com DB migration — zero-downtime real (Staff-level)

**Contexto:** A equipe vai fazer um deploy que inclui uma migration de banco de dados. A migration adiciona uma nova coluna nullable — compatível com a versão antiga. Você precisa garantir zero-downtime com a seguinte sequência: migrate → deploy v2 → validar → limpar.

**Missão:** Simule o processo completo de um deploy com migration usando Init Container para rodar a migration antes da aplicação subir.

**Requisitos:**
- [ ] Criar um namespace `estudo-db` para este exercício
- [ ] Criar um ConfigMap simulando os scripts de migration:
  ```
  migrate.sh: echo "Running migration v2: ALTER TABLE users ADD COLUMN preferences TEXT NULL"
  ```
- [ ] Criar Deployment `webapp` com:
  - Init Container que lê e executa o ConfigMap (simula o migrate)
  - Container principal com `nginx:1.27`
  - `maxUnavailable: 0` e `maxSurge: 1` (zero-downtime)
  - `minReadySeconds: 15`
- [ ] Confirmar via `kubectl logs <pod> -c init-migrate` que a migration rodou
- [ ] Confirmar que o Init Container terminou antes do container principal iniciar
- [ ] Fazer update do ConfigMap para "migration v3" e do Deployment para uma nova versão
- [ ] Observar que cada novo Pod executa a migration antes de entrar em serviço
- [ ] Deletar o namespace `estudo-db` ao final

**Verificação:**
```bash
kubectl logs <pod-name> -c init-migrate -n estudo-db
# Deve mostrar: "Running migration v2: ALTER TABLE..."

kubectl describe pod <pod-name> -n estudo-db | grep -A5 "Init Containers"
# Init Container deve mostrar State: Terminated, Reason: Completed

kubectl get pods -n estudo-db
# Todos os Pods devem estar Running (Init Container concluído com sucesso)
```

---

> [!tip] Ordem recomendada
> Exercícios 1 e 2 são fundamentais — faça nessa ordem, são a base para tudo.
> Exercícios 3, 4 e 5 podem ser feitos em qualquer ordem após 1 e 2.
> Exercício 6 requer que o Ingress nginx esteja funcional no cluster (MetalLB + nginx-ingress em 192.168.3.100).
> Exercícios 7 e 8 são independentes e podem ser feitos a qualquer momento.
