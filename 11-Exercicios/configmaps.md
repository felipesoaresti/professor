---
tags:
  - exercicios
  - kubernetes
  - configmaps
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/configmaps]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — ConfigMaps + Comandos Imperativos

> Infraestrutura: k8s-master (192.168.3.30), k8s-worker1 (192.168.3.31), k8s-worker2 (192.168.3.32)
> Namespace de trabalho: `estudo-felipe`
> Referência: DK8S Day3 (LINUXtips) | Conteúdo: [[05-Kubernetes/configmaps]] | Trilha: [[00-Trilha/kubernetes]]

**Foco desta série:** criação de recursos via comandos (`kubectl create`, `kubectl run`, `kubectl expose`) e domínio de ConfigMaps em todas as formas.

---

## Exercício 1: Fechar o Exercício 6 Anterior — Pod com ConfigMap Ausente

**Contexto:** Você tinha um Pod em `broken-app` que não subia por causa de um ConfigMap que não existia. Agora você sabe o que é um ConfigMap. Hora de fechar esse débito.

**Missão:** Criar o recurso ausente via comando imperativo e confirmar que o Pod sobe.

**Requisitos:**
- [ ] Aplicar o manifest abaixo (salvar como `broken-pod.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  namespace: estudo-felipe
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

- [ ] Observar o erro exato nos eventos (não adivinhar — `kubectl describe`)
- [ ] Criar o ConfigMap `app-config-inexistente` via **comando imperativo** (sem YAML) com pelo menos uma chave `app.conf` com qualquer conteúdo
- [ ] Confirmar que o Pod entra em `Running`
- [ ] Entrar no container e confirmar que `/config/app.conf` existe

**Verificação:**
```bash
kubectl describe pod broken-app -n estudo-felipe
kubectl get cm app-config-inexistente -n estudo-felipe -o yaml
kubectl get pod broken-app -n estudo-felipe
kubectl exec -it broken-app -n estudo-felipe -- ls /config/
```

---

## Exercício 2: ConfigMap como Variável de Ambiente

**Contexto:** A equipe de dev pediu para externalizar as configurações do servidor de aplicação para facilitar mudanças sem rebuild de imagem.

**Missão:** Criar um ConfigMap com variáveis de configuração e injetar no Pod.

**Requisitos:**
- [ ] Criar um ConfigMap chamado `webapp-env` no namespace `estudo-felipe` via **comando imperativo** com as chaves:
  - `APP_COLOR=blue`
  - `APP_MODE=production`
  - `APP_PORT=8080`
- [ ] Criar um Pod chamado `env-test` com a imagem `nginx:1.27-alpine` que consuma **apenas** `APP_COLOR` e `APP_MODE` como variáveis de ambiente (não `APP_PORT`)
- [ ] Entrar no container e confirmar que `APP_COLOR` e `APP_MODE` existem como variáveis mas `APP_PORT` não
- [ ] Atualizar o ConfigMap adicionando uma nova chave `APP_REGION=br-south`
- [ ] Verificar se a variável aparece no Pod em execução sem reiniciar (observação esperada: não aparece)
- [ ] Reiniciar o Pod e confirmar que a variável agora está disponível

**Verificação:**
```bash
kubectl get cm webapp-env -n estudo-felipe -o yaml
kubectl exec -it env-test -n estudo-felipe -- env | grep APP_
```

---

## Exercício 3: ConfigMap como Volume

**Contexto:** O time de infra precisa injetar um arquivo de configuração do nginx dentro do container sem incluí-lo na imagem.

**Missão:** Montar um ConfigMap como volume e verificar o comportamento.

**Requisitos:**
- [ ] Criar um arquivo local `custom-nginx.conf` com o seguinte conteúdo:
  ```nginx
  server {
      listen 80;
      server_name localhost;
      location /healthz {
          return 200 'ok';
          add_header Content-Type text/plain;
      }
  }
  ```
- [ ] Criar um ConfigMap chamado `nginx-config` a partir desse arquivo via **comando imperativo**
- [ ] Criar um Pod chamado `nginx-custom` que monte o ConfigMap no caminho `/etc/nginx/conf.d/`
- [ ] Entrar no container e confirmar que o arquivo `custom-nginx.conf` existe em `/etc/nginx/conf.d/`
- [ ] Testar que o endpoint `/healthz` responde com `ok`

**Verificação:**
```bash
kubectl get cm nginx-config -n estudo-felipe -o yaml
kubectl exec -it nginx-custom -n estudo-felipe -- ls /etc/nginx/conf.d/
kubectl exec -it nginx-custom -n estudo-felipe -- curl localhost/healthz
```

---

## Exercício 4: envFrom — Todas as Chaves de Uma Vez

**Contexto:** A aplicação `cotacoes` usa dezenas de variáveis de ambiente. Definir cada uma individualmente seria impraticável. Você vai usar `envFrom` para injetar todas de uma vez.

**Missão:** Usar `envFrom` para injetar um ConfigMap inteiro como variáveis de ambiente.

**Requisitos:**
- [ ] Criar um ConfigMap chamado `app-full-config` com as chaves:
  - `DB_HOST=postgres-svc.databases.svc.cluster.local`
  - `DB_PORT=5432`
  - `REDIS_HOST=redis-service.estudo-felipe.svc.cluster.local`
  - `LOG_LEVEL=debug`
- [ ] Criar um Pod chamado `full-env-test` usando `nginx:1.27-alpine` que consuma **todas** as chaves do ConfigMap via `envFrom`
- [ ] Confirmar via `exec` que todas as 4 variáveis estão disponíveis no container
- [ ] **Desafio:** Criar o YAML do Pod usando apenas `kubectl run --dry-run=client -o yaml` como ponto de partida, editar para adicionar o `envFrom`, e aplicar

**Verificação:**
```bash
kubectl exec -it full-env-test -n estudo-felipe -- env | grep -E 'DB_|REDIS_|LOG_'
```

---

## Exercício 5: Atualização de ConfigMap e Comportamento de Volume

**Contexto:** Em produção, você precisa entender exatamente quando uma mudança de configuração é propagada para os containers — para não ser pego de surpresa num incidente.

**Missão:** Observar o comportamento de atualização de ConfigMaps em volumes vs variáveis de ambiente.

**Requisitos:**
- [ ] Usar o ConfigMap `nginx-config` do Exercício 3 (ou recriar)
- [ ] Verificar o conteúdo do arquivo no container com `exec`
- [ ] Atualizar o ConfigMap adicionando um novo campo ao `custom-nginx.conf` (ex: um comentário `# versao 2`)
- [ ] Observar, sem reiniciar o Pod, se o arquivo dentro do container é atualizado (aguardar até 2 minutos)
- [ ] Documentar: o que você observou? O arquivo mudou? Em quanto tempo?
- [ ] Agora repita o mesmo experimento com variáveis de ambiente (`env-test` do Exercício 2): a variável mudou sem reiniciar?
- [ ] Concluir: qual forma de consumo (volume vs env var) permite reload sem restart?

**Verificação:**
```bash
kubectl exec -it nginx-custom -n estudo-felipe -- cat /etc/nginx/conf.d/custom-nginx.conf
# aguardar ~1 minuto e executar novamente
```

---

## Exercício 6: ConfigMap Imutável

**Contexto:** Em produção com centenas de ConfigMaps e Pods, o kubelet de cada node faz watch em todos eles. Isso gera carga desnecessária no apiserver para configs que raramente mudam.

**Missão:** Criar um ConfigMap imutável e entender as implicações.

**Requisitos:**
- [ ] Criar via YAML um ConfigMap chamado `app-v1-config` com `immutable: true` e as chaves `VERSION=1.0` e `ENV=producao`
- [ ] Aplicar e confirmar que foi criado
- [ ] Tentar atualizar o valor de `VERSION` para `1.1` (via `kubectl edit` ou `kubectl apply`) — o que acontece?
- [ ] Para "atualizar" a configuração, criar um novo ConfigMap `app-v2-config` com `VERSION=1.1`
- [ ] Criar um Pod que use `app-v2-config` e confirmar que pega a versão correta
- [ ] Deletar o ConfigMap imutável — é possível?

**Verificação:**
```bash
kubectl get cm app-v1-config -n estudo-felipe -o yaml
kubectl get cm app-v2-config -n estudo-felipe -o yaml
```

---

## Exercício 7 (CKA-Level): Troubleshooting — Deployment com Configuração Quebrada

**Contexto:** Você recebeu um alerta: o Deployment `api-server` no namespace `estudo-felipe` está com Pods em `CreateContainerConfigError`. Não há documentação. Descubra e corrija.

**Missão:** Aplicar o manifest abaixo e corrigir todos os problemas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: estudo-felipe
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: nginx:1.27-alpine
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: database-config
              key: host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: database-config
              key: porta
        volumeMounts:
        - name: app-conf
          mountPath: /etc/app
      volumes:
      - name: app-conf
        configMap:
          name: application-config
          items:
          - key: settings.yaml
            path: settings.yaml
```

**Há pelo menos 3 problemas nesse manifest. Encontre todos sem olhar a resposta.**

**Requisitos:**
- [ ] Aplicar o manifest e observar o estado dos Pods
- [ ] Identificar cada problema usando `describe` e `get events`
- [ ] Corrigir cada problema (criar recursos ausentes ou corrigir referências)
- [ ] Confirmar que os 2 Pods entram em `Running`
- [ ] Listar todos os ConfigMaps do namespace ao final

**Verificação:**
```bash
kubectl describe pod -l app=api-server -n estudo-felipe
kubectl get events -n estudo-felipe --sort-by='.lastTimestamp' | tail -20
kubectl get cm -n estudo-felipe
kubectl get pods -n estudo-felipe -l app=api-server
```

---

## Limpeza

```bash
kubectl delete pod broken-app env-test nginx-custom full-env-test -n estudo-felipe
kubectl delete cm app-config-inexistente webapp-env nginx-config app-full-config app-v1-config app-v2-config database-config application-config -n estudo-felipe
kubectl delete deployment api-server -n estudo-felipe
```

---

## Quando terminar

Volte com:
1. O que criou via comando imperativo vs YAML — qual pareceu mais natural?
2. O que observou no exercício de atualização de volume (exercício 5)?
3. Quais problemas encontrou no exercício 7 e como diagnosticou?
