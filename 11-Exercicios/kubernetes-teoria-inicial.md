---
tags:
  - exercicios
  - kubernetes
  - teoria
  - arquitetura
  - fundamentos
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/kubernetes-teoria-inicial]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Kubernetes Teoria Inicial

> Conteúdo: [[05-Kubernetes/kubernetes-teoria-inicial]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**kubectl:** requer `sudo` no k8s-master
**Namespace de trabalho:** `estudo`
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Reconhecimento do cluster (leitura apenas)

**Contexto:** Você acabou de entrar numa empresa e recebeu acesso ao cluster de produção. Antes de tocar em qualquer coisa, você precisa mapear o que existe, onde está rodando, e quais são os recursos disponíveis.

**Missão:** Produza um inventário completo do cluster usando apenas comandos de leitura.

**Requisitos:**
- [ ] Listar todos os nodes com IPs, roles e status: `kubectl get nodes -o wide`
- [ ] Listar todos os namespaces existentes
- [ ] Listar todos os Pods em todos os namespaces: `kubectl get pods -A -o wide`
- [ ] Identificar em qual node cada Pod do namespace `databases` está rodando
- [ ] Ver os detalhes completos de `k8s-worker1`: taints, recursos alocados, Pods agendados
- [ ] Listar todos os Deployments e Services do namespace `databases`
- [ ] Verificar os componentes do control plane: `kubectl get pods -n kube-system`
- [ ] Verificar o uso de recursos dos nodes: `kubectl top nodes`

**Verificação:**
```bash
sudo kubectl get nodes -o wide
# Deve mostrar k8s-master (.30), worker1 (.31), worker2 (.32)

sudo kubectl get pods -A -o wide
# Todos os pods com IPs e nodes

sudo kubectl describe node k8s-worker1
# Seção "Allocated resources": ver CPU e memória usados vs capacidade
```

---

## Exercício 2: Primeiro Pod e namespace de estudo

**Contexto:** O time de estudos criou um namespace isolado para experimentos. Você precisa criar sua primeira aplicação manualmente e entender o ciclo de vida de um Pod.

**Missão:** Criar um Pod nginx no namespace `estudo` e explorar seu ciclo de vida completo.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Criar um Pod chamado `meu-nginx` com imagem `nginx:1.27`, label `app=meu-nginx`, no namespace `estudo`
- [ ] Verificar que o Pod entrou em `Running` (qual node ele foi para?)
- [ ] Ver os logs do Pod
- [ ] Entrar no container e executar `curl localhost` — qual o retorno?
- [ ] Verificar o IP do Pod: `kubectl get pod meu-nginx -n estudo -o wide`
- [ ] Deletar o Pod e confirmar que ele sumiu
- [ ] Tentar recriar o Pod com o mesmo nome imediatamente — o IP é o mesmo?

**Verificação:**
```bash
sudo kubectl get pod meu-nginx -n estudo -o wide
# Deve mostrar node, IP, status Running

sudo kubectl logs meu-nginx -n estudo
# Logs de acesso do nginx

sudo kubectl get pods -n estudo
# Após deletar: lista vazia
```

---

## Exercício 3: Deployment com réplicas e self-healing

**Contexto:** Um Pod simples não é suficiente para produção. Se cair, para. Você precisa demonstrar o self-healing do Kubernetes na prática.

**Missão:** Criar um Deployment com 3 réplicas e provar que o K8s mantém o estado desejado sob falhas.

**Requisitos:**
- [ ] Criar Deployment `webapp` no namespace `estudo` com 3 réplicas usando `nginx:1.27`
- [ ] Labels dos Pods: `app=webapp`
- [ ] Definir `requests`: 100m CPU, 64Mi memória; `limits`: 200m CPU, 128Mi memória
- [ ] Verificar a distribuição dos Pods entre os workers: `kubectl get pods -n estudo -o wide`
- [ ] Abrir terminal com `kubectl get pods -n estudo -w` e em outro terminal deletar 1 Pod manualmente
- [ ] Quanto tempo levou para o Pod ser recriado?
- [ ] Deletar 2 Pods simultaneamente e observar o comportamento
- [ ] Escalar para 5 réplicas e verificar a nova distribuição
- [ ] Escalar de volta para 2 e observar quais Pods foram terminados (os mais novos ou os mais velhos?)

**Verificação:**
```bash
sudo kubectl get deployment webapp -n estudo
# READY: 3/3

sudo kubectl get pods -n estudo -o wide
# Distribuição entre worker1 e worker2

# Após delete manual de um Pod:
sudo kubectl get pods -n estudo -w
# Novo Pod aparece em segundos
```

---

## Exercício 4: Service e acesso interno

**Contexto:** Os Pods têm IPs efêmeros — quando o Pod morre e é recriado, o IP muda. O Service fornece um IP e DNS estáveis para o grupo de Pods.

**Missão:** Criar um Service ClusterIP, confirmar que ele encontra os Pods via labels, e testar o acesso.

**Requisitos:**
- [ ] Criar Service `webapp-svc` tipo ClusterIP na porta 80 para o Deployment `webapp` (namespace `estudo`)
- [ ] Verificar o ClusterIP atribuído: `kubectl get svc webapp-svc -n estudo`
- [ ] Verificar que os 3 Pods aparecem como Endpoints: `kubectl get endpoints webapp-svc -n estudo`
- [ ] Criar Pod temporário de debug: `kubectl run debug -n estudo --image=busybox:1.36 --restart=Never -- sleep 3600`
- [ ] De dentro do Pod de debug, acessar o webapp via DNS: `wget -qO- webapp-svc.estudo.svc.cluster.local`
- [ ] Deletar 1 Pod do webapp — o Endpoint some e volta automaticamente?
- [ ] Testar o acesso de fora do cluster via port-forward: `kubectl port-forward svc/webapp-svc 8080:80 -n estudo`
- [ ] Deletar o Pod de debug e o Service ao final

**Verificação:**
```bash
sudo kubectl get endpoints webapp-svc -n estudo
# 3 IPs listados (um por Pod)

sudo kubectl exec -n estudo debug -- wget -qO- webapp-svc.estudo.svc.cluster.local
# Retorna a página HTML do nginx
```

---

## Exercício 5: Rollout e rollback

**Contexto:** O time de dev liberou uma nova versão. Você precisa atualizar sem downtime e, quando algo der errado, reverter rapidamente.

**Missão:** Executar um rollout controlado, forçar uma falha e praticar o rollback.

**Requisitos:**
- [ ] Atualizar a imagem do Deployment `webapp` de `nginx:1.27` para `nginx:1.27-alpine`
- [ ] Acompanhar o rollout em tempo real até a conclusão
- [ ] Ver o histórico de revisões (deve ter pelo menos 2)
- [ ] Forçar um rollout com imagem inválida `nginx:versao-que-nao-existe`
- [ ] Observar o estado do Deployment — os Pods antigos continuam rodando? (estratégia RollingUpdate)
- [ ] Identificar o problema usando eventos do namespace
- [ ] Fazer rollback para a última versão estável
- [ ] Confirmar que todos os Pods voltaram ao estado `Running`
- [ ] **Desafio:** reverter para uma revisão específica (`--to-revision=N`)

**Verificação:**
```bash
sudo kubectl rollout status deployment/webapp -n estudo
# deployment "webapp" successfully rolled out

sudo kubectl rollout history deployment/webapp -n estudo
# Lista de revisões

sudo kubectl get events -n estudo --sort-by='.lastTimestamp' | tail -10
# Eventos de ErrImagePull / ImagePullBackOff

sudo kubectl rollout undo deployment/webapp -n estudo
# Reverte para revisão anterior
```

---

## Exercício 6: Labels e selectors — a cola do K8s

**Contexto:** Um desenvolvedor criou Pods manualmente com labels errados. O Service não encontra os Pods e a aplicação está inacessível. Você precisa entender e corrigir o mismatch de labels.

**Missão:** Demonstre como labels e selectors funcionam e corrija um mismatch propositalmente criado.

**Requisitos:**
- [ ] Criar um Service `mismatch-svc` com selector `app=correto` no namespace `estudo`
- [ ] Criar 2 Pods com label `app=errado` (nome: `pod-errado-1` e `pod-errado-2`)
- [ ] Verificar que os Endpoints estão vazios: `kubectl get endpoints mismatch-svc -n estudo`
- [ ] Corrigir **sem deletar os Pods** — adicionar o label correto aos Pods com `kubectl label`
- [ ] Confirmar que os Endpoints aparecem após a correção
- [ ] Adicionar o label `version=1.0` a apenas um dos Pods
- [ ] Criar um segundo Service `version-svc` com selector `app=correto,version=1.0`
- [ ] Verificar que `version-svc` enxerga apenas 1 Pod nos Endpoints

**Verificação:**
```bash
sudo kubectl get endpoints mismatch-svc -n estudo
# Antes da correção: <none>
# Após a correção: 10.x.x.x:80, 10.y.y.y:80

sudo kubectl get pods -n estudo --show-labels
# Mostra todos os labels dos Pods

sudo kubectl get endpoints version-svc -n estudo
# Apenas 1 IP (o Pod com version=1.0)
```

---

## Exercício 7: kubectl explain — referência offline (CKA-level)

**Contexto:** No exame CKA não há acesso à internet — apenas `kubectl explain` e a documentação local. Você precisa dominar essa ferramenta para encontrar qualquer campo de qualquer objeto sem depender de memória.

**Missão:** Use `kubectl explain` para construir um Pod YAML completo sem consultar anotações.

**Requisitos:**
- [ ] Descobrir todos os campos de primeiro nível de um Pod: `kubectl explain pod`
- [ ] Ver a estrutura de `pod.spec`: `kubectl explain pod.spec`
- [ ] Encontrar os campos de `resources` (requests e limits): `kubectl explain pod.spec.containers.resources`
- [ ] Encontrar como configurar uma `readinessProbe` via explain (sem abrir o conteúdo de probes)
- [ ] Usando apenas `kubectl explain` como referência, construir um Pod YAML com:
  - namespace: `estudo`
  - image: `nginx:1.27`
  - `requests`: 50m CPU, 32Mi memória
  - `limits`: 100m CPU, 64Mi memória
  - `readinessProbe` com httpGet na porta 80, path `/`
  - `restartPolicy: Never`
- [ ] Aplicar o Pod e confirmar que fica `Running` e `READY 1/1`

**Verificação:**
```bash
sudo kubectl explain pod.spec.containers --recursive | grep -A2 readinessProbe
# Mostra os subcampos disponíveis

sudo kubectl get pod <nome> -n estudo
# READY: 1/1, STATUS: Running
```

---

## Exercício 8: Troubleshooting sistemático — diagnose sem dicas (CKA/Staff-level)

**Contexto:** São 3h da manhã. Você recebeu 4 alertas simultâneos. Cada um descreve um problema diferente num namespace. Sem documentação, sem o desenvolvedor, apenas você e o `kubectl`.

**Missão:** Aplique os 4 manifests abaixo e corrija todos os problemas usando o método sistemático de diagnóstico.

**Manifest 1 — Pod com imagem inexistente:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-sem-imagem
  namespace: estudo
spec:
  containers:
  - name: app
    image: nginx:versao-que-nao-existe-2099
```

**Manifest 2 — Pod sem recursos no namespace (criar ResourceQuota primeiro):**
```yaml
# Aplicar ANTES do Pod:
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-apertada
  namespace: estudo
spec:
  hard:
    limits.memory: "1Mi"   # intencionalmente pequeno
---
apiVersion: v1
kind: Pod
metadata:
  name: app-sem-recursos
  namespace: estudo
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      limits:
        memory: "128Mi"
```

**Manifest 3 — Service com selector errado:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-sem-endpoints
  namespace: estudo
spec:
  selector:
    app: nao-existe-no-cluster
  ports:
  - port: 80
```

**Manifest 4 — Pod crashando:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-crash
  namespace: estudo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c", "exit 1"]
```

**Requisitos:**
- [ ] Aplicar todos os 4 manifests
- [ ] Para cada problema: identificar o sintoma exato, o motivo exato, e a correção
- [ ] Usar o método: `get pods` → `describe pod` → `logs` → `get events`
- [ ] Corrigir o Manifest 1: usar uma imagem válida
- [ ] Corrigir o Manifest 2: deletar a ResourceQuota e recriar o Pod
- [ ] Corrigir o Manifest 3: criar um Pod com o label que o Service espera
- [ ] Corrigir o Manifest 4: mudar o command para algo que não saia com erro
- [ ] Documentar: qual foi o sintoma (`kubectl get pods`) e qual foi a causa raiz (`describe`) de cada um?

**Verificação:**
```bash
# Após correções:
sudo kubectl get pods -n estudo
# Todos os Pods: Running

sudo kubectl get endpoints svc-sem-endpoints -n estudo
# 1 IP listado (após criar Pod com label correto)
```

---

## Exercício 9: Multi-container Pods — Sidecar, Init e Debug

**Contexto:** O time precisa de três padrões de multi-container: (1) um sidecar que compartilha logs com o container principal via volume, (2) um init container que aguarda um serviço ficar disponível antes de subir a app, e (3) um ephemeral container para debugar um Pod sem shell.

**Missão:** Implemente os três padrões e demonstre como cada um funciona na prática.

**Requisitos:**

**Parte A — Sidecar:**
- [ ] Criar Pod `app-sidecar` com dois containers:
  - Container `app`: `nginx:1.27`, escreve logs em `/var/log/nginx/`, monta volume `logs`
  - Container `log-reader`: `busybox:1.36`, executa `tail -f /var/log/nginx/access.log`, monta o mesmo volume `logs`
  - Volume `logs`: tipo `emptyDir`
- [ ] Confirmar que o Pod ficou `READY 2/2`
- [ ] Fazer uma requisição para gerar log: `kubectl exec -n estudo app-sidecar -c app -- curl -s localhost`
- [ ] Verificar o log **pelo container sidecar**: `kubectl logs app-sidecar -n estudo -c log-reader`
- [ ] O sidecar viu a requisição sem ter acesso à porta do nginx? Por quê?

**Parte B — Init Container:**
- [ ] Criar Pod `app-init` com:
  - Init container `aguarda-servico`: `busybox:1.36`, tenta `nc -z webapp-svc 80` em loop até conseguir
  - Container principal `app`: `nginx:1.27`
- [ ] Observar o estado do Pod antes de criar o Service: `kubectl get pod app-init -n estudo -w`
- [ ] O Pod fica em que estado? (`Init:0/1`?)
- [ ] Criar Service `webapp-svc` apontando para qualquer Pod existente no namespace `estudo`
- [ ] Observar o Pod `app-init` progredir para `Running` após o Service existir
- [ ] Ver os logs do init container durante a espera: `kubectl logs app-init -n estudo -c aguarda-servico`

**Parte C — Ephemeral Container (debug):**
- [ ] Criar Pod `app-sem-shell` com imagem `gcr.io/distroless/static:nonroot` (sem shell, sem ferramentas):
  - command: `["/bin/true"]` com `restartPolicy: Never`
- [ ] Tentar `kubectl exec -it app-sem-shell -n estudo -- /bin/sh` — o que acontece?
- [ ] Adicionar ephemeral container de debug:
  ```bash
  kubectl debug -it app-sem-shell -n estudo \
    --image=busybox:1.36 \
    --target=app-sem-shell
  ```
- [ ] Dentro do ephemeral container: executar `ps aux` — você vê o processo do container principal?
- [ ] Verificar que o ephemeral container aparece na spec: `kubectl get pod app-sem-shell -n estudo -o jsonpath='{.spec.ephemeralContainers}'`

**Verificação:**
```bash
kubectl get pod app-sidecar -n estudo
# READY: 2/2, STATUS: Running

kubectl logs app-sidecar -n estudo -c log-reader
# Logs do nginx lidos pelo sidecar via volume compartilhado

kubectl get pod app-init -n estudo
# STATUS: Init:0/1 → Running (após criar o Service)

kubectl logs app-init -n estudo -c aguarda-servico
# "aguardando..." repetido até o Service existir

kubectl get pod app-sem-shell -n estudo -o jsonpath='{.spec.ephemeralContainers[0].name}'
# Nome do ephemeral container adicionado
```

---

> [!tip] Ordem recomendada
> Exercícios 1 e 2 são de baixo risco — leitura e criação básica.
> Exercícios 3, 4 e 5 são sequenciais — cada um usa recursos do anterior.
> Exercícios 6 e 7 são independentes — podem ser feitos em qualquer ordem após o 3.
> Exercício 8 é o mais desafiador — faça apenas após dominar os anteriores.
> Exercício 9 cobre multi-containers — pode ser feito após o exercício 2.

> [!warning] Limpeza
> Ao final de todos os exercícios: `kubectl delete namespace estudo`
> Isso remove todos os recursos criados de uma vez, incluindo ResourceQuotas, Pods, Services e Deployments.

> [!info] Próximo passo
> Após completar estes exercícios: [[11-Exercicios/configmaps]] — externalizar configuração com ConfigMaps, formas de criação e consumo.
