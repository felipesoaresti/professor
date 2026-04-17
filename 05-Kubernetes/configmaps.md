---
tags:
  - kubernetes
  - configmaps
aliases:
  - ConfigMap
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[kubernetes-teoria-inicial]]"
next:
  - "[[11-Exercicios/configmaps]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# ConfigMaps no Kubernetes

> Referência: [Docs Oficiais](https://kubernetes.io/docs/concepts/configuration/configmap/) | DK8S Day3 (LINUXtips)
> Pré-requisito: [[kubernetes-teoria-inicial]] | Exercícios: [[11-Exercicios/configmaps]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

Um **ConfigMap** é um objeto do Kubernetes para armazenar configuração não-sensível em pares chave/valor. Ele desacopla a configuração da imagem de container — a mesma imagem pode rodar diferente em dev, staging e produção simplesmente trocando o ConfigMap, sem rebuildar nada.

> Para dados sensíveis (senhas, tokens, certificados) use **Secret**, não ConfigMap.

---

## Como criar — Formas imperativas (comandos)

### 1. De literais (`--from-literal`)

```bash
kubectl create configmap meu-config \
  --from-literal=APP_ENV=producao \
  --from-literal=APP_PORT=8080 \
  -n estudo-felipe
```

Cada `--from-literal` vira uma chave no ConfigMap.

### 2. De arquivo (`--from-file`)

```bash
# Uma chave por arquivo (nome do arquivo = chave)
kubectl create configmap nginx-conf \
  --from-file=nginx.conf=/etc/nginx/nginx.conf \
  -n estudo-felipe

# Todos os arquivos de um diretório
kubectl create configmap app-conf \
  --from-file=./configs/ \
  -n estudo-felipe
```

### 3. De arquivo `.env` (`--from-env-file`)

```bash
# Dado um arquivo app.env com conteúdo:
# APP_ENV=producao
# APP_PORT=8080

kubectl create configmap app-env \
  --from-env-file=app.env \
  -n estudo-felipe
```

Diferença do `--from-file`: cada linha do arquivo vira uma chave separada (não uma chave única com o conteúdo do arquivo inteiro).

### 4. Dry-run para gerar YAML

```bash
kubectl create configmap meu-config \
  --from-literal=APP_ENV=producao \
  --dry-run=client -o yaml > meu-config.yaml
```

---

## Como criar — YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: estudo-felipe
data:
  # Chaves simples (variáveis de ambiente)
  APP_ENV: "producao"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
  # Chave com conteúdo de arquivo (multiline)
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
        }
    }
```

---

## Como consumir

### Forma 1 — Variável de ambiente (chave específica)

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    env:
    - name: APP_ENV             # nome da variável dentro do container
      valueFrom:
        configMapKeyRef:
          name: app-config      # nome do ConfigMap
          key: APP_ENV          # chave dentro do ConfigMap
```

### Forma 2 — Todas as chaves como variáveis de ambiente (`envFrom`)

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    envFrom:
    - configMapRef:
        name: app-config        # todas as chaves viram variáveis de ambiente
```

**Cuidado:** se o ConfigMap tiver chaves com `-` (hífen), elas serão ignoradas pois não são variáveis de ambiente válidas.

### Forma 3 — Volume (montado como arquivos)

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: config-vol
      mountPath: /etc/app/config   # diretório criado no container
  volumes:
  - name: config-vol
    configMap:
      name: app-config             # cada chave vira um arquivo nesse diretório
```

O resultado: `/etc/app/config/APP_ENV`, `/etc/app/config/APP_PORT`, etc.

### Forma 4 — Volume montando apenas uma chave específica

```yaml
  volumes:
  - name: config-vol
    configMap:
      name: app-config
      items:
      - key: nginx.conf            # apenas essa chave
        path: nginx.conf           # nome do arquivo no container
```

---

## Troubleshooting

### Pod fica em `Pending` ou falha com `FailedMount`

```bash
kubectl describe pod <pod> -n <ns>
```

Sintoma típico nos eventos:
```
Warning  FailedMount  MountVolume.SetUp failed for volume "config-vol" :
  configmap "app-config-inexistente" not found
```

**Causa:** O Pod referencia um ConfigMap que não existe.

**Fix:**
```bash
# Criar o ConfigMap que estava faltando
kubectl create configmap app-config-inexistente \
  --from-literal=placeholder=valor \
  -n estudo-felipe
```

### Variável de ambiente não aparece no container

```bash
kubectl exec -it <pod> -n <ns> -- env | grep APP_
```

Se não aparece, verifica:
1. Nome do ConfigMap está correto?
2. Chave existe no ConfigMap? `kubectl get configmap app-config -o yaml`
3. A chave tem caractere inválido (hífen com `envFrom`)?

### Atualizei o ConfigMap — a aplicação não atualizou

Variáveis de ambiente (`env`/`envFrom`) são **copiadas** no momento em que o Pod inicia. Para pegar o novo valor, o Pod precisa reiniciar:

```bash
kubectl rollout restart deployment/<nome> -n <ns>
```

Volumes montados **atualizam automaticamente** (delay de ~1 min via kubelet), mas a aplicação precisa ler o arquivo novamente — nem todas releem em runtime.

---

## Nível Avançado

### ConfigMap Imutável (Kubernetes 1.21+)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
immutable: true
data:
  APP_ENV: producao
```

Vantagens:
- O kubelet para de fazer watch no ConfigMap → menos carga no apiserver em clusters grandes
- Evita alterações acidentais

Para mudar: criar um novo ConfigMap com nome diferente e atualizar o Deployment.

### Ver todos os ConfigMaps do cluster

```bash
kubectl get configmap -A
kubectl get cm -A          # cm é o shortname
```

### Inspecionar conteúdo

```bash
kubectl get cm app-config -n estudo-felipe -o yaml
kubectl describe cm app-config -n estudo-felipe
```

### ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| Dados sensíveis | Não | Sim |
| Encoding | Texto puro | Base64 |
| Montagem | Igual | Igual |
| RBAC | Recomendado | Essencial |

---

## Padrões CKA

No exame CKA você precisa saber:
1. Criar ConfigMap via comando (imperativo) **sem YAML**
2. Referenciar ConfigMap em Pod tanto como `env` quanto como `volume`
3. Identificar um Pod que falha por ConfigMap ausente via `describe`
4. Corrigir o problema (criar o ConfigMap ou corrigir o nome)

```bash
# Criação rápida (praticada para o exame)
kubectl create cm db-config --from-literal=DB_HOST=postgres-svc.databases.svc.cluster.local --from-literal=DB_PORT=5432 -n estudo-felipe

# Verificação rápida
kubectl get cm db-config -n estudo-felipe -o jsonpath='{.data}'
```
