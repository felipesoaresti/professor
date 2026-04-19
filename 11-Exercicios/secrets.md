---
tags:
  - exercicios
  - kubernetes
  - secrets
  - segurança
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/secrets]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Secrets no Kubernetes

> Conteúdo: [[05-Kubernetes/secrets]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `default` ou `estudo-felipe`
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Inspecionar Secrets existentes no cluster

**Contexto:** Você é novo no time e precisa entender quais Secrets existem no cluster, quais tipos são usados, e como estão sendo consumidos — sem expor valores.

**Missão:** Faça um inventário dos Secrets do cluster, identificando tipos, namespaces, e quem os usa.

**Requisitos:**
- [ ] Listar todos os Secrets de todos os namespaces: `kubectl get secret -A`
- [ ] Identificar quantos Secrets existem por tipo (`Opaque`, `kubernetes.io/tls`, etc.)
- [ ] Para o namespace `databases`: listar os Secrets sem expor os valores (apenas nomes das chaves)
- [ ] Para um Secret TLS gerado pelo cert-manager: identificar qual Ingress o usa (`kubectl describe ingress`)
- [ ] Verificar quais Pods em `controle-gastos` ou `promobot` referenciam Secrets como env var ou volume (apenas `kubectl describe pod` — sem alterar nada)
- [ ] Documentar: qual é a diferença entre um Secret do tipo `Opaque` e `kubernetes.io/tls` que você observou?

**Verificação:**
```bash
kubectl get secret -A --no-headers | awk '{print $3}' | sort | uniq -c | sort -rn
# Conta Secrets por tipo

kubectl get secret -n databases -o jsonpath='{range .items[*]}{.metadata.name}: {.type}{"\n"}{end}'
# Lista nomes e tipos sem expor valores
```

---

## Exercício 2: Criar Secret de múltiplas formas e comparar

**Contexto:** O time tem três preferências diferentes para criar Secrets: literal, arquivo, e YAML com stringData. Você vai dominar as três formas e entender o que cada uma gera.

**Missão:** Crie o mesmo Secret de três formas diferentes e compare os resultados.

**Requisitos:**
- [ ] **Forma 1 (imperativa):** `kubectl create secret generic db-v1 --from-literal=username=admin --from-literal=password='Str0ng!Pass'`
- [ ] **Forma 2 (arquivo):** criar arquivo `creds.env` com `username=admin` e `password=Str0ng!Pass`, depois `kubectl create secret generic db-v2 --from-env-file=creds.env`
- [ ] **Forma 3 (YAML com stringData):** criar YAML usando `stringData` com os mesmos valores, aplicar com `kubectl apply`
- [ ] Para cada Secret: verificar que o `kubectl get secret <nome> -o yaml` mostra `data` (não `stringData`) com os valores em base64
- [ ] Decodificar a senha de cada Secret e confirmar que são idênticas
- [ ] Identificar o erro clássico: criar uma 4ª versão usando `echo "Str0ng!Pass" | base64` (com newline) e demonstrar a diferença com `xxd`
- [ ] Deletar todos os 4 Secrets ao final

**Verificação:**
```bash
kubectl get secret db-v1 -o jsonpath='{.data.password}' | base64 -d
kubectl get secret db-v2 -o jsonpath='{.data.password}' | base64 -d
kubectl get secret db-v3 -o jsonpath='{.data.password}' | base64 -d
# Todos devem retornar: Str0ng!Pass (sem newline)

# Versão com erro (newline):
kubectl get secret db-v4 -o jsonpath='{.data.password}' | base64 -d | xxd | head
# Deve mostrar 0a (newline) no final
```

---

## Exercício 3: Consumir Secret como variável de ambiente e como volume

**Contexto:** A equipe discute qual forma de consumir Secrets é mais segura. Você vai implementar as duas e demonstrar a diferença de segurança na prática.

**Missão:** Crie um Pod que consuma o mesmo Secret das duas formas e compare como os valores ficam expostos.

**Requisitos:**
- [ ] Criar Secret `app-secret` com `username=felipe` e `password=s3cr3t123`
- [ ] Criar Pod `app-envvar` que consuma o Secret como variável de ambiente via `secretKeyRef`
- [ ] Criar Pod `app-volume` que consuma o Secret como volume montado em `/run/secrets/app`
- [ ] **Teste de exposição via env:** dentro do `app-envvar`, executar `cat /proc/1/environ | tr '\0' '\n' | grep -i pass` — o valor aparece?
- [ ] **Teste de exposição via volume:** dentro do `app-volume`, executar `cat /run/secrets/app/password`
- [ ] Verificar que o volume é `tmpfs`: `df -h /run/secrets/app`
- [ ] Verificar as permissões dos arquivos no volume: `ls -la /run/secrets/app/`
- [ ] Documentar: qual método expõe menos informação e por quê?
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
# Dentro do pod app-envvar:
kubectl exec app-envvar -- cat /proc/1/environ | tr '\0' '\n' | grep PASSWORD
# Deve mostrar PASSWORD=s3cr3t123 ← valor exposto em /proc

# Dentro do pod app-volume:
kubectl exec app-volume -- df -h /run/secrets/app
# Deve mostrar: tmpfs (filesystem em memória)

kubectl exec app-volume -- ls -la /run/secrets/app/
# Deve mostrar permissão 0400 ou similar
```

---

## Exercício 4: imagePullSecret para registry privado

**Contexto:** O time criou um registry privado para imagens internas. Os Pods precisam de credenciais para fazer pull. Você vai configurar o imagePullSecret.

**Missão:** Crie um imagePullSecret e configure um Pod para usá-lo (mesmo que o registry não exista — o objetivo é a configuração correta).

**Requisitos:**
- [ ] Criar Secret do tipo `docker-registry` chamado `registry-creds` com credenciais fictícias para `registry.staypuff.info`
- [ ] Verificar o tipo do Secret: deve ser `kubernetes.io/dockerconfigjson`
- [ ] Inspecionar o conteúdo do Secret: `kubectl get secret registry-creds -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .`
- [ ] Verificar a estrutura: deve ter `auths → registry.staypuff.info → auth` com as credenciais em base64
- [ ] Criar um Pod que usa o imagePullSecret (pode usar uma imagem pública como `nginx:1.27` para não falhar no pull)
- [ ] Confirmar que o Pod referencia o imagePullSecret: `kubectl get pod <nome> -o jsonpath='{.spec.imagePullSecrets}'`
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
kubectl get secret registry-creds -o jsonpath='{.type}'
# kubernetes.io/dockerconfigjson

kubectl get secret registry-creds \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys'
# ["registry.staypuff.info"]
```

---

## Exercício 5: Rotação de Secret e comportamento de Pod

**Contexto:** A senha do banco de dados foi comprometida. O time precisa rotacioná-la imediatamente. Você vai executar a rotação e verificar que a aplicação absorve o novo valor.

**Missão:** Rotacione um Secret e compare o comportamento entre Pods com volume (atualização automática) e Pods com env var (requer restart).

**Requisitos:**
- [ ] Criar Secret `db-secret` com `password=senha-original`
- [ ] Criar dois Deployments: `app-vol` (volume) e `app-env` (env var), ambos lendo `db-secret`
- [ ] Ambos os containers devem imprimir a senha a cada 10s no log
- [ ] Atualizar o Secret: `kubectl patch secret db-secret --type='json' -p='[{"op":"replace","path":"/data/password","value":"'"$(echo -n nova-senha | base64)"'"}]'`
- [ ] Monitorar os logs de `app-vol`: o valor muda automaticamente em até 60s?
- [ ] Monitorar os logs de `app-env`: o valor permanece o antigo?
- [ ] Reiniciar `app-env` com `kubectl rollout restart` e confirmar que agora usa o novo valor
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
kubectl logs -l app=app-vol -f
# Após ~60s da atualização do Secret, deve mostrar "nova-senha"

kubectl logs -l app=app-env -f
# Continua mostrando "senha-original" até o rollout restart

kubectl rollout restart deployment/app-env
kubectl logs -l app=app-env --since=30s
# Agora deve mostrar "nova-senha"
```

---

## Exercício 6: RBAC granular para Secrets (avançado)

**Contexto:** O time de aplicação precisa ler o Secret de banco de dados, mas não deve conseguir ler outros Secrets do namespace. O princípio de mínimo privilégio exige restrição por `resourceNames`.

**Missão:** Configure RBAC que permite leitura de apenas um Secret específico e demonstre que outros Secrets são inacessíveis.

**Requisitos:**
- [ ] Criar ServiceAccount `app-sa` no namespace `default`
- [ ] Criar Secrets `db-secret` e `admin-secret` no mesmo namespace
- [ ] Criar Role `read-db-only` que permite `get` apenas em `db-secret` (usando `resourceNames`)
- [ ] Criar RoleBinding para `app-sa`
- [ ] Criar Pod que usa `app-sa` como ServiceAccount
- [ ] Dentro do Pod, usar o token do ServiceAccount para testar via curl à API:
  - `GET /api/v1/namespaces/default/secrets/db-secret` → deve funcionar (200)
  - `GET /api/v1/namespaces/default/secrets/admin-secret` → deve falhar (403)
  - `GET /api/v1/namespaces/default/secrets` (list) → deve falhar (403)
- [ ] Registrar as respostas de cada chamada
- [ ] Deletar todos os recursos ao final

**Verificação:**
```bash
# Dentro do pod com app-sa:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER=https://kubernetes.default.svc

curl -sk -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/default/secrets/db-secret | jq .status
# null = sucesso (objeto retornado)

curl -sk -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/default/secrets/admin-secret | jq .reason
# "Forbidden"
```

---

## Exercício 7: Secret TLS e Ingress (avançado)

**Contexto:** O time quer criar um Secret TLS manualmente para um subdomínio de estudo (sem usar o cert-manager), e configurar o Ingress para usar esse certificado.

**Missão:** Gere um certificado self-signed, crie o Secret TLS, e configure o Ingress para servir HTTPS com esse certificado.

**Requisitos:**
- [ ] Gerar certificado self-signed com openssl:
  ```bash
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key -out tls.crt \
    -subj "/CN=estudo.staypuff.info/O=homelab"
  ```
- [ ] Criar Secret TLS: `kubectl create secret tls estudo-tls --cert=tls.crt --key=tls.key`
- [ ] Verificar o tipo: `kubernetes.io/tls`
- [ ] Verificar as chaves: deve ter `tls.crt` e `tls.key`
- [ ] Criar Deployment `estudo-app` com nginx e Service ClusterIP
- [ ] Criar Ingress para `estudo.staypuff.info` referenciando o Secret TLS na seção `tls:`
- [ ] Verificar que o Ingress está configurado: `kubectl describe ingress estudo-app | grep -A5 TLS`
- [ ] Deletar todos os recursos ao final (incluindo o Secret TLS)

**Verificação:**
```bash
kubectl get secret estudo-tls -o jsonpath='{.type}'
# kubernetes.io/tls

kubectl describe ingress estudo-app | grep -A3 TLS
# TLS:  estudo-tls terminates estudo.staypuff.info

kubectl describe ingress estudo-app | grep Address
# Deve mostrar 192.168.3.100 (MetalLB)
```

---

## Exercício 8: Detectar base64 errado e diferença entre stringData e data (Staff-level)

**Contexto:** O SRE recebeu um Secret YAML de um desenvolvedor. A aplicação falha na autenticação, mas a senha parece correta. Você precisa debugar o Secret sem expor os valores nos logs.

**Missão:** Identifique e corrija três problemas clássicos de Secrets mal configurados.

**Requisitos:**
- [ ] **Problema 1:** Secret com `data.password` gerado via `echo "senha123" | base64` (com newline) — detectar e corrigir
- [ ] **Problema 2:** Secret com campo `stringData` e `data` simultâneos com valores diferentes para a mesma chave — qual prevalece? Testar e documentar
- [ ] **Problema 3:** Pod que referencia `secretKeyRef.key: password` mas o Secret tem a chave `Password` (maiúsculo) — qual o sintoma e como diagnosticar?
- [ ] Para cada problema: criar o cenário, observar o sintoma, diagnosticar com `kubectl describe` e `kubectl exec`, corrigir
- [ ] Documentar o método de diagnóstico de cada problema sem nunca imprimir a senha em texto puro nos logs

**Verificação:**
```bash
# Problema 1: detectar newline no valor
kubectl exec <pod> -- sh -c 'printf "%s" "$PASSWORD" | wc -c'
# Se retornar 9 em vez de 8 para "senha123" → há newline extra

# Problema 3: sintoma
kubectl describe pod <pod> | grep -A5 Events
# Error: couldn't find key password in Secret default/meu-secret
# → chave não encontrada (case-sensitive)
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura pura — sem risco, faça primeiro.
> Exercícios 2 e 3 são fundamentais — cobrem criação e consumo, faça em sequência.
> Exercício 4 é independente — pode ser feito a qualquer momento.
> Exercício 5 requer dois Deployments simultâneos — faça após entender volume vs env var.
> Exercícios 6, 7 e 8 são avançados — faça após dominar os básicos.

> [!warning] Nunca logar valores de Secret
> Em nenhum exercício imprima valores sensíveis em `kubectl logs` sem intenção. Use `wc -c` para verificar tamanho, `xxd | head` para verificar encoding, mas nunca `echo $PASSWORD` ou `cat /run/secrets/...` em um sistema de produção.
