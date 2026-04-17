---
tags:
  - exercicios
  - kubernetes
  - teoria
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/kubernetes-teoria-inicial]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Kubernetes Teoria Inicial

> Infraestrutura: k8s-master (192.168.3.30), k8s-worker1 (192.168.3.31), k8s-worker2 (192.168.3.32)
> kubectl requer `sudo` no k8s-master
> Referência: DK8S Day1–Day2 (LINUXtips) | Conteúdo: [[05-Kubernetes/kubernetes-teoria-inicial]] | Trilha: [[00-Trilha/kubernetes]]

---

## Exercício 1: Reconhecimento do Cluster

**Contexto:** Você acabou de entrar em uma empresa e recebeu acesso ao cluster Kubernetes de produção. Antes de tocar em qualquer coisa, você precisa entender o que está rodando, em quais nodes e com quais recursos.

**Missão:** Mapear o estado atual do cluster usando apenas comandos de leitura.

**Requisitos:**
- [ ] Listar todos os nodes com seus IPs, roles e status
- [ ] Listar todos os namespaces existentes
- [ ] Listar todos os Pods em execução no cluster (todos os namespaces)
- [ ] Identificar em qual node cada Pod do namespace `giropops` está rodando
- [ ] Ver os detalhes completos de um dos nodes workers (taints, recursos alocados, pods agendados)
- [ ] Listar todos os Deployments e Services do namespace `giropops`

**Verificação:**
```bash
sudo kubectl get nodes -o wide
sudo kubectl get pods -A -o wide
sudo kubectl describe node k8s-worker1
```

---

## Exercício 2: Seu Primeiro Pod e Namespace

**Contexto:** O time de estudos definiu que cada trainee deve ter seu próprio namespace isolado para experimentos. Você precisa criar o seu e subir sua primeira aplicação.

**Missão:** Criar um namespace de estudo e executar um Pod nginx dentro dele.

**Requisitos:**
- [ ] Criar um namespace chamado `estudo-<seu-nome>` (ex: `estudo-felipe`)
- [ ] Criar um Pod chamado `meu-nginx` com a imagem `nginx:1.27` nesse namespace
- [ ] O Pod deve ter a label `app=meu-nginx`
- [ ] Verificar que o Pod entrou em estado `Running`
- [ ] Visualizar os logs do Pod
- [ ] Acessar um shell interativo dentro do Pod e executar `curl localhost`
- [ ] Deletar o Pod e confirmar que ele sumiu

**Verificação:**
```bash
sudo kubectl get pods -n estudo-<seu-nome> -o wide
sudo kubectl logs meu-nginx -n estudo-<seu-nome>
sudo kubectl get pods -n estudo-<seu-nome>  # após deletar — deve estar vazio
```

---

## Exercício 3: Deployment com Réplicas e Service

**Contexto:** Um Pod simples não é suficiente para produção — se ele cair, a aplicação para. Você precisa garantir alta disponibilidade com múltiplas réplicas e expor o serviço internamente no cluster.

**Missão:** Criar um Deployment com 3 réplicas e um Service ClusterIP para acessá-lo.

**Requisitos:**
- [ ] Criar um Deployment chamado `webapp` no namespace do exercício anterior com 3 réplicas usando a imagem `nginx:1.27`
- [ ] Cada Pod do Deployment deve ter as labels `app=webapp` e `env=estudo`
- [ ] Definir `requests` de CPU (100m) e memória (64Mi) e `limits` de CPU (200m) e memória (128Mi)
- [ ] Criar um Service do tipo `ClusterIP` chamado `webapp-svc` na porta 80 apontando para os Pods do Deployment
- [ ] Confirmar que o Service enxerga os 3 Pods como endpoints
- [ ] Testar o acesso via `port-forward` acessando `http://localhost:8080` na sua máquina

**Verificação:**
```bash
sudo kubectl get deployment webapp -n estudo-<seu-nome>
sudo kubectl get endpoints webapp-svc -n estudo-<seu-nome>
sudo kubectl port-forward svc/webapp-svc 8080:80 -n estudo-<seu-nome>
```

---

## Exercício 4: Comportamento do ReplicaSet sob Falha

**Contexto:** Um desenvolvedor júnior perguntou: "Se um Pod cair em produção, o que acontece?" Você vai demonstrar na prática o comportamento do self-healing do Kubernetes.

**Missão:** Simular a morte de Pods e observar o comportamento do ReplicaSet.

**Requisitos:**
- [ ] Com o Deployment `webapp` do exercício anterior rodando com 3 réplicas, deletar 1 Pod manualmente
- [ ] Observar (em tempo real) o K8s recriar o Pod automaticamente
- [ ] Anotar quantos segundos levou para o novo Pod ficar `Running`
- [ ] Deletar 2 Pods simultaneamente e observar o comportamento
- [ ] Escalar o Deployment para 5 réplicas e confirmar a distribuição entre os workers
- [ ] Escalar de volta para 2 réplicas e verificar quais Pods foram terminados

**Verificação:**
```bash
# Em um terminal, watch nos pods:
sudo kubectl get pods -n estudo-<seu-nome> -w

# Em outro terminal, deletar um pod:
sudo kubectl delete pod <nome-do-pod> -n estudo-<seu-nome>

# Ver distribuição nos nodes:
sudo kubectl get pods -n estudo-<seu-nome> -o wide
```

---

## Exercício 5: Rollout e Rollback

**Contexto:** O time de desenvolvimento liberou uma nova versão da imagem. Você precisa atualizar o Deployment em produção com zero downtime e, caso algo dê errado, reverter rapidamente.

**Missão:** Realizar um rollout controlado e praticar o rollback.

**Requisitos:**
- [ ] Atualizar a imagem do Deployment `webapp` de `nginx:1.27` para `nginx:1.27-alpine`
- [ ] Acompanhar o rollout em tempo real até a conclusão
- [ ] Verificar o histórico de revisões do Deployment (deve ter pelo menos 2 revisões)
- [ ] Forçar um rollout com uma imagem inválida (`nginx:versao-que-nao-existe`) e observar o que acontece
- [ ] Identificar o problema usando os eventos do namespace
- [ ] Fazer o rollback para a última versão estável
- [ ] Confirmar que os Pods voltaram ao estado `Running`

**Verificação:**
```bash
sudo kubectl rollout status deployment/webapp -n estudo-<seu-nome>
sudo kubectl rollout history deployment/webapp -n estudo-<seu-nome>
sudo kubectl get events -n estudo-<seu-nome> --sort-by='.lastTimestamp'
sudo kubectl rollout undo deployment/webapp -n estudo-<seu-nome>
```

---

## Exercício 6 (CKA-Level): Troubleshooting — Pod com Problema

**Contexto:** Você recebeu um alerta às 3h da manhã: um Pod está em `CrashLoopBackOff` no cluster. Não há documentação. Você precisa identificar e corrigir o problema sem derrubar o serviço.

**Missão:** Aplicar o manifest abaixo, diagnosticar o problema e corrigi-lo.

**Manifest para aplicar** (salve como `broken-pod.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  namespace: estudo-<seu-nome>
  labels:
    app: broken-app
spec:
  containers:
  - name: app
    image: nginx:1.27
    command: ["/bin/sh", "-c", "cat /config/app.conf && nginx -g 'daemon off;'"]
    volumeMounts:
    - name: config-vol
      mountPath: /config
  volumes:
  - name: config-vol
    configMap:
      name: app-config-inexistente
```

**Requisitos:**
- [ ] Aplicar o manifest e observar o status do Pod
- [ ] Identificar o motivo exato pelo qual o Pod não consegue iniciar (sem adivinhar — usar os comandos certos)
- [ ] Corrigir o problema criando o recurso ausente com qualquer conteúdo válido
- [ ] Confirmar que o Pod entra em `Running` após a correção
- [ ] Documentar (no próprio terminal, via `kubectl describe`) o ciclo de eventos que levou ao problema

**Verificação:**
```bash
sudo kubectl describe pod broken-app -n estudo-<seu-nome>
sudo kubectl get events -n estudo-<seu-nome> --sort-by='.lastTimestamp'
sudo kubectl get pod broken-app -n estudo-<seu-nome>  # deve estar Running após correção
```

---

## Limpeza Final

Após concluir todos os exercícios:

```bash
sudo kubectl delete namespace estudo-<seu-nome>
```

> Isso remove todos os recursos criados dentro do namespace de uma vez.

---

## Quando terminar

Volte aqui com:
1. O que funcionou de primeira
2. O que travou e como resolveu
3. Qual exercício achou mais difícil e por quê

O feedback direciona o próximo assunto.
