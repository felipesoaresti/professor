---
tags:
  - kubernetes
  - configmaps
  - configuracao
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

> Pré-requisito: [[kubernetes-teoria-inicial]] | Exercícios: [[11-Exercicios/configmaps]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

Um **ConfigMap** é um objeto Kubernetes para armazenar configuração não-sensível em pares chave/valor. O objetivo é desacoplar configuração de imagem: a mesma imagem roda em dev, staging e produção com ConfigMaps diferentes — sem rebuildar nada.

> [!warning] ConfigMap não é Secret
> ConfigMap armazena dados em **texto puro** no etcd, sem criptografia em repouso por padrão. Para senhas, tokens e certificados, use [[secrets]].

Limite de tamanho: **1 MiB por ConfigMap**. Para configurações maiores, use volumes externos (NFS, S3) ou refatore para arquivos menores.

---

## Como funciona internamente

### Armazenamento no etcd

O ConfigMap é armazenado como um objeto no etcd com seus dados em `data` (texto puro) ou `binaryData` (base64, para binários). Cada chave é um campo separado dentro do objeto.

```
etcd key: /registry/configmaps/<namespace>/<nome>
etcd value: objeto ConfigMap serializado em protobuf
```

### Propagação para volumes — o mecanismo de symlinks

Quando um Pod monta um ConfigMap como volume, o kubelet:

1. Cria um diretório temporário com os arquivos das chaves
2. Cria um symlink atômico apontando para esse diretório
3. Quando o ConfigMap é atualizado, cria um **novo** diretório temporário e troca o symlink atomicamente

```
/etc/app/config/         ← mountPath no container
  └── ..data/            ← symlink → ..2025_04_18_12_00_00.123456789/
       └── app.conf      ← arquivo real
  └── app.conf           ← symlink → ..data/app.conf
```

A troca do symlink é **atômica** — não há estado intermediário corrompido. O delay de propagação é controlado pelo `--sync-frequency` do kubelet (padrão: **1 minuto**).

### Propagação para variáveis de ambiente — sem propagação

Variáveis de ambiente (`env`/`envFrom`) são **copiadas** para o processo no momento em que o container é criado. Não há mecanismo de atualização posterior — o processo precisa ser reiniciado para ler novos valores.

### Watch no apiserver

O kubelet faz **watch** em todos os ConfigMaps referenciados pelos Pods no node. Em clusters grandes com muitos ConfigMaps, isso pode gerar carga no apiserver. ConfigMaps `immutable: true` eliminam esse watch (ver seção avançada).

---

## Na prática — comandos e exemplos reais

### Criação imperativa

```bash
# De literais
kubectl create configmap app-config \
  --from-literal=APP_ENV=producao \
  --from-literal=APP_PORT=8080 \
  -n estudo

# De arquivo (nome do arquivo = chave)
kubectl create configmap nginx-conf \
  --from-file=nginx.conf=/etc/nginx/nginx.conf \
  -n estudo

# De diretório (cada arquivo vira uma chave)
kubectl create configmap app-conf \
  --from-file=./configs/ \
  -n estudo

# De arquivo .env (cada linha KEY=VALUE vira uma chave separada)
kubectl create configmap app-env \
  --from-env-file=app.env \
  -n estudo
```

> [!info] `--from-file` vs `--from-env-file`
> `--from-file=app.env` cria **uma chave** chamada `app.env` cujo valor é o conteúdo inteiro do arquivo.
> `--from-env-file=app.env` cria **uma chave por linha** do arquivo (formato `KEY=VALUE`).

### Dry-run para gerar YAML

```bash
kubectl create configmap meu-config \
  --from-literal=APP_ENV=producao \
  --dry-run=client -o yaml > meu-config.yaml
```

### YAML declarativo

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: estudo
data:
  # Chaves simples (boas para variáveis de ambiente)
  APP_ENV: "producao"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
  # Chave multiline (boa para arquivos de configuração)
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
        }
    }
```

### Consumo 1 — Variável de ambiente (chave específica)

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    env:
    - name: APP_ENV             # nome da variável no container
      valueFrom:
        configMapKeyRef:
          name: app-config      # nome do ConfigMap
          key: APP_ENV          # chave dentro do ConfigMap
          optional: false       # Pod falha se a chave não existir (padrão)
```

### Consumo 2 — Todas as chaves como variáveis (`envFrom`)

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    envFrom:
    - configMapRef:
        name: app-config        # todas as chaves viram variáveis de ambiente
```

> [!warning] Chaves com hífen e `envFrom`
> Chaves com `-` (ex: `log-level`) são inválidas como nomes de variável de ambiente e são **silenciosamente ignoradas** pelo `envFrom`. Use `_` ao invés de `-` em chaves destinadas a env vars.

### Consumo 3 — Volume completo

```yaml
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: config-vol
      mountPath: /etc/app/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config          # cada chave vira um arquivo no mountPath
```

Resultado: `/etc/app/config/APP_ENV`, `/etc/app/config/APP_PORT`, `/etc/app/config/nginx.conf`.

### Consumo 4 — Volume com chaves específicas

```yaml
volumes:
- name: config-vol
  configMap:
    name: app-config
    items:
    - key: nginx.conf           # apenas esta chave
      path: nginx.conf          # nome do arquivo no container
      mode: 0440                # permissão opcional (padrão: 0644)
```

### Consumo 5 — `subPath` (montagem de arquivo único)

```yaml
volumeMounts:
- name: config-vol
  mountPath: /etc/nginx/nginx.conf   # arquivo, não diretório
  subPath: nginx.conf                # chave específica do ConfigMap
```

> [!warning] `subPath` não atualiza automaticamente
> Diferente da montagem de volume completa, um volume montado com `subPath` **não recebe atualizações automáticas** quando o ConfigMap muda. O container precisa ser reiniciado. Isso é um edge case que pega muita gente de surpresa em produção.

---

## Casos de uso e boas práticas

### Quando usar volume vs variável de ambiente

| Cenário | Recomendação |
|---|---|
| Configuração que muda em runtime (sem restart) | Volume |
| Arquivo de configuração estruturado (nginx.conf, etc.) | Volume |
| Variáveis simples que a aplicação lê uma vez | env var |
| Muitas variáveis para não listar uma a uma | `envFrom` |
| Arquivo montado em path exato sobrescrevendo outro | `subPath` (sem auto-update) |

### Naming e organização

Um ConfigMap por domínio de configuração, não um ConfigMap gigante por aplicação. Exemplos:
- `app-server-config` — configuração do servidor web
- `app-db-config` — configuração de banco de dados
- `app-feature-flags` — feature flags separadas para rotação independente

### Versionamento de ConfigMaps imutáveis

Para configurações estáticas que não precisam mudar em runtime, use o padrão de versão no nome:

```bash
kubectl create configmap app-config-v3 --from-file=config.yaml
# Atualizar o Deployment para referenciar app-config-v3
# Deletar app-config-v2 após rollout completo
```

Isso permite rollback do ConfigMap junto com rollback do Deployment — eles ficam acoplados por versão.

---

## Troubleshooting — cenários reais de produção

### Pod em `Pending` com `FailedMount`

```bash
kubectl describe pod <pod> -n <ns>
```

```
Warning  FailedMount  MountVolume.SetUp failed: configmap "app-config" not found
```

**Causa:** ConfigMap referenciado no volume não existe.
**Fix:** Criar o ConfigMap antes de criar o Pod — ou usar `optional: true` se a ausência é aceitável.

```yaml
volumes:
- name: config-vol
  configMap:
    name: app-config
    optional: true    # Pod sobe mesmo sem o ConfigMap
```

### Pod em `CreateContainerConfigError`

```bash
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

```
Error: couldn't find key "DB_HOST" in ConfigMap default/database-config
```

**Causa:** A chave referenciada em `configMapKeyRef.key` não existe no ConfigMap.
**Fix:** Verificar as chaves reais do ConfigMap:

```bash
kubectl get cm database-config -n <ns> -o jsonpath='{.data}' | jq 'keys'
```

### Variável de ambiente não atualizou após mudar o ConfigMap

Variáveis de ambiente são copiadas no start do container. Não há propagação posterior.

```bash
kubectl rollout restart deployment/<nome> -n <ns>
```

### Volume atualizado mas aplicação não recarregou

O arquivo foi atualizado (symlink trocado), mas a aplicação só lê o arquivo no boot:

```bash
# Opção 1: reiniciar o Deployment
kubectl rollout restart deployment/<nome> -n <ns>

# Opção 2: enviar sinal SIGHUP ao processo (se a aplicação suportar reload)
kubectl exec <pod> -- kill -HUP 1
```

### `subPath` não atualizou — comportamento esperado

Se o volume usa `subPath`, o arquivo não atualiza automaticamente. Esse é o comportamento correto do Kubernetes — não é um bug. Requer restart do Pod.

### ConfigMap com chave de hífen ignorada no `envFrom`

```bash
kubectl exec <pod> -- env | grep LOG
# LOG-LEVEL não aparece — silenciosamente ignorado

kubectl get cm app-config -o jsonpath='{.data}' | jq 'keys'
# ["LOG-LEVEL"]  ← hífen é inválido como nome de env var
```

**Fix:** Renomear a chave para `LOG_LEVEL` (underscore).

---

## Nível avançado — edge cases e cenários CKA/Staff

### ConfigMap imutável

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v3
immutable: true
data:
  APP_ENV: producao
```

Vantagens:
- O kubelet para o watch → menos carga no apiserver em clusters com centenas de ConfigMaps
- Proteção contra alteração acidental

Para mudar: criar novo ConfigMap com nome diferente e atualizar os Pods. Não é possível reverter `immutable: true` sem deletar e recriar o ConfigMap.

### Projected volumes — combinar ConfigMap + Secret + token

```yaml
volumes:
- name: combined
  projected:
    sources:
    - configMap:
        name: app-config
    - secret:
        name: app-secret
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
```

Útil quando a aplicação espera todos os arquivos de configuração em um único diretório.

### Inspecionar o delay de propagação

O kubelet propaga mudanças de ConfigMap para volumes dentro do `--sync-frequency` (padrão 1 min). Para confirmar o delay real:

```bash
# Atualizar o ConfigMap
kubectl patch configmap app-config -n estudo \
  --type merge -p '{"data":{"VERSION":"2"}}'

# No outro terminal, watch no arquivo dentro do container
kubectl exec -it <pod> -- watch -n 5 cat /etc/app/config/VERSION
# O arquivo deve mudar dentro de ~60s
```

### ResourceVersion e conflitos de escrita

```bash
kubectl get cm app-config -o jsonpath='{.metadata.resourceVersion}'
# Ex: 12345

# Duas atualizações concorrentes → a segunda falha com Conflict
# Use kubectl apply (server-side merge) ou adicione retry logic no CI
```

### Padrões CKA — criação rápida

```bash
# Criar ConfigMap (imperativo, sem YAML)
kubectl create cm db-config \
  --from-literal=DB_HOST=postgres.databases.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  -n estudo

# Verificar conteúdo rapidamente
kubectl get cm db-config -n estudo -o jsonpath='{.data}'

# Criar Pod com ConfigMap como env var (via dry-run + edição)
kubectl run app --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml
# Editar pod.yaml para adicionar envFrom/env, depois aplicar
kubectl apply -f pod.yaml
```

---

## Referências

- [ConfigMaps — Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- DK8S Day3 — LINUXtips (Descomplicando Kubernetes)
- [[secrets]] — para dados sensíveis
- [[volumes-storage]] — para volumes persistentes
- [[deployment-strategies]] — como coordenar mudanças de ConfigMap com rollouts
