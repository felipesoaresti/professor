---
tags:
  - exercicios
  - kubernetes
  - configmaps
  - configuracao
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/configmaps]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — ConfigMaps no Kubernetes

> Conteúdo: [[05-Kubernetes/configmaps]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `estudo` (criar para estes exercícios)
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Inventário — inspecionar ConfigMaps existentes no cluster

**Contexto:** Você chegou num cluster em produção e precisa entender quais ConfigMaps existem, o que cada um armazena (sem expor dados sensíveis), e quem os consome.

**Missão:** Faça um mapeamento completo dos ConfigMaps do cluster sem alterar nada.

**Requisitos:**
- [ ] Listar todos os ConfigMaps de todos os namespaces: `kubectl get cm -A`
- [ ] Contar quantas chaves tem cada ConfigMap no namespace `kube-system` (usar `-o yaml` ou `jsonpath`)
- [ ] Para o namespace `databases`: listar os ConfigMaps e ver quais chaves cada um tem (sem ver os valores)
- [ ] Identificar quais ConfigMaps do cluster têm `immutable: true` (usar `-o yaml` e grep, ou jsonpath)
- [ ] Para um ConfigMap qualquer, verificar qual Deployment o referencia (procurar em `describe deployment` ou `get deployment -o yaml`)
- [ ] Documentar: qual é o tamanho máximo de um ConfigMap? Onde essa informação pode ser verificada no cluster?

**Verificação:**
```bash
kubectl get cm -A --no-headers | wc -l
# Total de ConfigMaps no cluster

kubectl get cm -n databases -o jsonpath='{range .items[*]}{.metadata.name}: {range $k, $v := .data}{$k} {end}{"\n"}{end}'
# Lista nomes de chaves sem os valores

kubectl get cm -A -o json | jq '.items[] | select(.immutable == true) | .metadata.name'
# ConfigMaps imutáveis
```

---

## Exercício 2: Criar ConfigMap de três formas e comparar

**Contexto:** O time tem três desenvolvedores que preferem formas diferentes de criar ConfigMaps. Você vai dominar as três e entender o que cada uma gera de diferente.

**Missão:** Crie o mesmo ConfigMap usando literal, arquivo `.env` e YAML, e compare os resultados.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] **Forma 1 (literal):** `kubectl create configmap app-v1 --from-literal=APP_ENV=producao --from-literal=APP_PORT=8080 -n estudo`
- [ ] **Forma 2 (.env file):** criar arquivo `app.env` com as mesmas chaves e `kubectl create configmap app-v2 --from-env-file=app.env -n estudo`
- [ ] **Forma 3 (YAML com `data`):** criar YAML com as mesmas chaves e `kubectl apply`
- [ ] Comparar os três com `kubectl get cm <nome> -o yaml` — são idênticos?
- [ ] **Armadilha:** criar `app-v4` com `--from-file=app.env` (sem o `--env`) — o que é diferente?
  - Quantas chaves tem `app-v4`? Qual é o valor da chave?
- [ ] Documentar a diferença entre `--from-env-file` e `--from-file`

**Verificação:**
```bash
kubectl get cm app-v1 -n estudo -o jsonpath='{.data}'
kubectl get cm app-v2 -n estudo -o jsonpath='{.data}'
kubectl get cm app-v3 -n estudo -o jsonpath='{.data}'
# Os três devem ter as mesmas chaves e valores

kubectl get cm app-v4 -n estudo -o jsonpath='{.data}' | jq 'keys'
# Deve ter uma única chave: "app.env" (o nome do arquivo)
```

---

## Exercício 3: Consumir como variável de ambiente (`env` e `envFrom`)

**Contexto:** A equipe quer externalizar configurações do servidor para facilitar mudanças entre ambientes sem rebuildar imagens.

**Missão:** Injete ConfigMap em um Pod como variável de ambiente de duas formas e compare o comportamento.

**Requisitos:**
- [ ] Criar ConfigMap `webapp-config` com as chaves:
  - `APP_COLOR=blue`, `APP_MODE=production`, `APP_PORT=8080`, `log-level=debug`
- [ ] Criar Pod `env-selective` que consuma **apenas** `APP_COLOR` e `APP_MODE` via `configMapKeyRef`
- [ ] Criar Pod `env-all` que consuma **todas** as chaves via `envFrom`
- [ ] Dentro de `env-selective`: verificar que `APP_COLOR` e `APP_MODE` existem mas `APP_PORT` não
- [ ] Dentro de `env-all`: verificar que `APP_COLOR`, `APP_MODE` e `APP_PORT` existem — `log-level` aparece?
- [ ] Atualizar o ConfigMap adicionando `APP_REGION=br-south`
- [ ] Sem reiniciar os Pods, verificar se a nova variável aparece nos containers
- [ ] Reiniciar os Pods e confirmar que `APP_REGION` agora está disponível

**Verificação:**
```bash
kubectl exec -n estudo env-selective -- env | grep APP_
# APP_COLOR e APP_MODE presentes; APP_PORT ausente

kubectl exec -n estudo env-all -- env | grep -E 'APP_|log'
# APP_COLOR, APP_MODE, APP_PORT presentes
# log-level: ausente (hífen é inválido como env var — silenciosamente ignorado)
```

---

## Exercício 4: Consumir como volume e observar atualização automática

**Contexto:** O time de infra precisa injetar um `nginx.conf` customizado dentro do container sem incluí-lo na imagem — e precisa que mudanças sejam propagadas sem restart.

**Missão:** Monte um ConfigMap como volume e observe a atualização automática em tempo real.

**Requisitos:**
- [ ] Criar arquivo local `custom-nginx.conf`:
  ```nginx
  server {
      listen 80;
      server_name localhost;
      location /healthz {
          return 200 'ok\n';
          add_header Content-Type text/plain;
      }
  }
  ```
- [ ] Criar ConfigMap `nginx-config` a partir desse arquivo via comando imperativo
- [ ] Criar Pod `nginx-custom` que monte o ConfigMap em `/etc/nginx/conf.d/`
- [ ] Confirmar que o arquivo existe no container e testar `curl localhost/healthz`
- [ ] Atualizar o ConfigMap: adicionar um comentário `# versao 2` ao início do arquivo via `kubectl edit cm nginx-config -n estudo`
- [ ] Monitorar o arquivo dentro do container sem reiniciar o Pod (aguardar até 2 min):
  ```bash
  kubectl exec -n estudo nginx-custom -- watch -n 10 cat /etc/nginx/conf.d/custom-nginx.conf
  ```
- [ ] Documentar: o arquivo mudou? Em quanto tempo? O nginx recarregou automaticamente?
- [ ] Verificar a estrutura de symlinks que o kubelet criou:
  ```bash
  kubectl exec -n estudo nginx-custom -- ls -la /etc/nginx/conf.d/
  # Deve mostrar symlinks ..data/ e arquivos apontando para ele
  ```

**Verificação:**
```bash
kubectl exec -n estudo nginx-custom -- curl -s localhost/healthz
# ok

kubectl exec -n estudo nginx-custom -- ls -la /etc/nginx/conf.d/
# Deve mostrar: custom-nginx.conf -> ..data/custom-nginx.conf
#              ..data -> ..2025_04_18_xx_xx_xx.xxxxx/
```

---

## Exercício 5: O comportamento contraintuitivo do `subPath`

**Contexto:** Um desenvolvedor quer montar apenas o `nginx.conf` em `/etc/nginx/nginx.conf` (sobrescrevendo o arquivo padrão) sem montar o diretório inteiro. Ele usa `subPath`. Mas depois reporta que "o volume não atualiza mais".

**Missão:** Reproduza o comportamento do `subPath` e documente por que ele não atualiza automaticamente.

**Requisitos:**
- [ ] Criar ConfigMap `nginx-single` com uma chave `nginx.conf` contendo um server block simples (versão 1)
- [ ] Criar Pod `nginx-subpath` com `subPath` montando apenas a chave `nginx.conf` em `/etc/nginx/conf.d/default.conf`
- [ ] Confirmar que o arquivo existe e que o container funciona
- [ ] Atualizar o ConfigMap para versão 2 (adicionar comentário)
- [ ] Aguardar 2 minutos e verificar se o arquivo dentro do container atualizou
- [ ] Comparar: o comportamento é diferente do Exercício 4 (sem `subPath`)?
- [ ] Documentar: por que `subPath` não atualiza? Como resolver quando precisar de ambos (arquivo único E atualização automática)?

**Verificação:**
```bash
kubectl exec -n estudo nginx-subpath -- cat /etc/nginx/conf.d/default.conf
# Após 2 min: ainda mostra versão 1 (subPath não atualiza)

# Comparação: mesmo ConfigMap montado sem subPath em outro Pod
# O arquivo atualiza dentro de ~60s
```

---

## Exercício 6: ConfigMap imutável — performance e restrições

**Contexto:** Em um cluster com 500 ConfigMaps, o time de plataforma identificou que o kubelet está gerando carga excessiva no apiserver com watches. A solução é marcar ConfigMaps estáticos como `immutable: true`.

**Missão:** Crie um ConfigMap imutável, entenda suas restrições, e implemente o padrão de versionamento.

**Requisitos:**
- [ ] Criar via YAML o ConfigMap `app-config-v1` com `immutable: true` e chaves `VERSION=1.0` e `ENV=producao`
- [ ] Tentar atualizar `VERSION` para `1.1` — o que acontece exatamente?
- [ ] Tentar remover o campo `immutable: true` via `kubectl edit` — é possível?
- [ ] Para "atualizar" a configuração: criar `app-config-v2` com `VERSION=1.1` e `immutable: true`
- [ ] Criar Deployment `versioned-app` referenciando `app-config-v2`
- [ ] Deletar `app-config-v1` após confirmar que o Deployment usa a v2
- [ ] Documentar: quais são as implicações operacionais do `immutable: true` no ciclo de deploy?

**Verificação:**
```bash
kubectl edit cm app-config-v1 -n estudo
# Deve falhar com: ConfigMap is immutable and cannot be updated

kubectl get cm -n estudo -o jsonpath='{range .items[*]}{.metadata.name}: immutable={.immutable}{"\n"}{end}'
# app-config-v1: immutable=true
# app-config-v2: immutable=true
```

---

## Exercício 7: Troubleshooting — Deployment com múltiplos problemas de ConfigMap

**Contexto:** Você recebeu um alerta: o Deployment `api-server` no namespace `estudo` está com Pods em `CreateContainerConfigError`. Não há documentação. Não há acesso ao desenvolvedor. Diagnostique e corrija sozinho.

**Missão:** Aplique o manifest abaixo e corrija **todos** os problemas sem olhar a resposta.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: estudo
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
        envFrom:
        - configMapRef:
            name: feature-flags
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

Há pelo menos **4 problemas** nesse manifest. Encontre todos usando apenas `kubectl describe`, `kubectl get events` e `kubectl get cm`.

**Requisitos:**
- [ ] Aplicar o manifest e observar o estado dos Pods
- [ ] Identificar cada problema com o método de diagnóstico correto (não adivinhar)
- [ ] Criar os recursos ausentes com as chaves corretas
- [ ] Corrigir referências erradas sem alterar o Deployment (criar ConfigMaps com nomes/chaves que o Deployment espera)
- [ ] Confirmar que os 2 Pods entram em `Running`
- [ ] Executar `env | grep -E 'DB_|feature'` dentro de um Pod para confirmar os valores

**Verificação:**
```bash
kubectl describe pod -l app=api-server -n estudo
# Events vão revelar cada problema

kubectl get events -n estudo --sort-by='.lastTimestamp' | tail -20

kubectl get pods -n estudo -l app=api-server
# READY: 2/2 após todas as correções
```

---

## Exercício 8: Projected volume — ConfigMap + Secret no mesmo diretório (avançado)

**Contexto:** A aplicação espera que sua configuração e suas credenciais estejam no mesmo diretório `/etc/app/`. Hoje o time monta dois volumes separados. Você vai consolidar com `projected`.

**Missão:** Use um Projected Volume para montar ConfigMap e Secret em um único diretório.

**Requisitos:**
- [ ] Criar ConfigMap `app-settings` com chaves `LOG_LEVEL=info` e `PORT=8080`
- [ ] Criar Secret `app-creds` com chaves `DB_PASSWORD=s3cr3t` e `API_KEY=abc123`
- [ ] Criar Pod `app-projected` com um único volume `projected` que combine os dois:
  ```yaml
  volumes:
  - name: app-config
    projected:
      sources:
      - configMap:
          name: app-settings
      - secret:
          name: app-creds
  ```
- [ ] Montar o volume em `/etc/app/` no container
- [ ] Verificar que todos os 4 arquivos existem em `/etc/app/` (`LOG_LEVEL`, `PORT`, `DB_PASSWORD`, `API_KEY`)
- [ ] Verificar as permissões: ConfigMap e Secret têm permissões diferentes?
- [ ] Atualizar o ConfigMap e confirmar que o arquivo correspondente atualiza automaticamente dentro do volume projetado
- [ ] Limpeza final: `kubectl delete namespace estudo`

**Verificação:**
```bash
kubectl exec -n estudo app-projected -- ls -la /etc/app/
# LOG_LEVEL, PORT, DB_PASSWORD, API_KEY

kubectl exec -n estudo app-projected -- stat /etc/app/DB_PASSWORD
# Permissão padrão: 0644 (Secret em projected volume mantém 0644 — não 0400)
# Compare com um Secret em volume normal
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura pura — sem risco, sempre primeiro.
> Exercícios 2 e 3 cobrem criação e consumo — base obrigatória, fazer em sequência.
> Exercício 4 é central — o comportamento de atualização de volume é o mais importante para produção.
> Exercício 5 é complementar ao 4 — o `subPath` é o edge case que mais surpreende; faça logo após o 4.
> Exercício 6 é independente — pode ser feito a qualquer momento após o 2.
> Exercício 7 é o mais próximo do dia a dia — troubleshooting real, faça após dominar os básicos.
> Exercício 8 é avançado — requer ter feito Secrets antes.

> [!warning] Limpeza
> Ao final de todos os exercícios: `kubectl delete namespace estudo`
> Isso remove Pods, ConfigMaps, Secrets e todos os outros recursos criados no namespace.
