---
tags:
  - exercicios
  - kubernetes
  - ingress
  - nginx
  - traefik
  - cert-manager
  - tls
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/ingress]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Ingress, nginx, Traefik e cert-manager

> Conteúdo: [[05-Kubernetes/ingress]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `estudo`
**Ingress Controller existente:** nginx em 192.168.3.100
**Domínio:** `*.staypuff.info` (Cloudflare DNS)
**cert-manager:** já instalado com ClusterIssuers `letsencrypt-staging` e `letsencrypt-production`
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Inventário do Ingress Controller existente (leitura apenas)

**Contexto:** Antes de criar qualquer Ingress, você precisa entender o que já está configurado no cluster — qual controller existe, quais regras estão ativas, e quais certificados estão em vigor.

**Missão:** Mapeie completamente a infraestrutura de Ingress existente sem alterar nada.

**Requisitos:**
- [ ] Listar todos os Ingresses do cluster: `kubectl get ingress -A`
- [ ] Identificar o IP do nginx Ingress Controller: `kubectl get svc -n ingress-nginx`
- [ ] Ver as IngressClasses disponíveis: `kubectl get ingressclass`
- [ ] Para cada Ingress existente: anotar host, path, Service de backend e se usa TLS
- [ ] Verificar os certificados em vigor: `kubectl get certificate -A`
- [ ] Para um certificado existente: verificar validade, Issuer usado e Secret gerado
- [ ] Verificar os ClusterIssuers configurados: `kubectl get clusterissuer`
- [ ] Descrever um ClusterIssuer e identificar: servidor ACME, tipo de challenge, Secret do token Cloudflare
- [ ] Documentar: qual namespace tem o Secret do token da Cloudflare? Qual o nome do Secret?

**Verificação:**
```bash
kubectl get ingress -A
# Deve mostrar os Ingresses de controle-gastos, promobot, etc.

kubectl get certificate -A
# READY: True para todos os certs válidos

kubectl get clusterissuer
# letsencrypt-staging e letsencrypt-production (ou nomes equivalentes)
```

---

## Exercício 2: Primeiro Ingress HTTP (sem TLS)

**Contexto:** O time quer expor uma aplicação de teste em `estudo.staypuff.info` para validar a conectividade antes de configurar TLS. Sem certificado por enquanto — só HTTP.

**Missão:** Crie um Deployment + Service + Ingress HTTP e confirme o acesso pelo hostname.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Criar Deployment `webapp` com 2 réplicas usando `nginx:1.27-alpine`
- [ ] Criar Service `webapp-svc` ClusterIP na porta 80
- [ ] Criar Ingress `webapp-ingress` com `ingressClassName: nginx`:
  - host: `estudo.staypuff.info`
  - path: `/` (Prefix)
  - backend: `webapp-svc:80`
- [ ] Confirmar que o Ingress recebeu address: `kubectl get ingress -n estudo`
- [ ] Adicionar entrada no `/etc/hosts` local (WSL) se necessário, ou verificar que o DNS `*.staypuff.info` já aponta para 192.168.3.100
- [ ] Testar: `curl -v http://estudo.staypuff.info` — qual resposta?
- [ ] Verificar nos logs do nginx controller que a requisição chegou

**Verificação:**
```bash
kubectl get ingress webapp-ingress -n estudo
# ADDRESS: 192.168.3.100

curl -s http://estudo.staypuff.info | grep -i nginx
# Página de boas-vindas do nginx

kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=5
# Log de acesso para estudo.staypuff.info
```

---

## Exercício 3: TLS com ClusterIssuer staging

**Contexto:** O time quer validar o fluxo completo de emissão de certificado antes de usar o ClusterIssuer de produção. Usar staging evita consumir o rate limit do Let's Encrypt.

**Missão:** Configure TLS no Ingress usando o ClusterIssuer de staging e acompanhe todo o fluxo de emissão.

**Requisitos:**
- [ ] Adicionar ao Ingress `webapp-ingress` a annotation do ClusterIssuer staging e a seção `tls:`
- [ ] Usar `secretName: estudo-tls-staging`
- [ ] Monitorar a emissão em tempo real:
  ```bash
  kubectl get certificate -n estudo -w
  kubectl get order -n estudo -w
  kubectl get challenge -n estudo -w
  ```
- [ ] Acompanhar a cadeia completa: Certificate → CertificateRequest → Order → Challenge
- [ ] Verificar quando o Challenge DNS-01 é resolvido: `kubectl describe challenge <nome> -n estudo`
- [ ] Confirmar que o Secret `estudo-tls-staging` foi criado: `kubectl get secret estudo-tls-staging -n estudo`
- [ ] Testar: `curl -v --insecure https://estudo.staypuff.info` — qual CA assinou o cert? (deve ser staging)
- [ ] Verificar a validade e as datas do certificado: `echo | openssl s_client -connect estudo.staypuff.info:443 2>/dev/null | openssl x509 -noout -dates -issuer`

**Verificação:**
```bash
kubectl get certificate estudo-tls-staging -n estudo
# READY: True

kubectl describe certificate estudo-tls-staging -n estudo
# Status: Certificate is up to date and has not expired

curl -vk https://estudo.staypuff.info 2>&1 | grep -i "issuer\|subject\|expire"
# Issuer deve conter "Fake LE" ou "(STAGING)"
```

---

## Exercício 4: TLS com ClusterIssuer produção

**Contexto:** O staging funcionou. Hora de emitir o certificado real, confiável pelos browsers.

**Missão:** Troque o ClusterIssuer para produção e confirme que o certificado é válido e confiável.

**Requisitos:**
- [ ] Antes de trocar: deletar o Secret de staging para forçar reemissão
  ```bash
  kubectl delete secret estudo-tls-staging -n estudo
  ```
- [ ] Criar um novo Ingress (ou duplicar o anterior) com:
  - annotation: `cert-manager.io/cluster-issuer: letsencrypt-production`
  - `secretName: estudo-tls-prod`
- [ ] Monitorar o novo fluxo de emissão até `READY: True`
- [ ] Testar: `curl -v https://estudo.staypuff.info` — sem `--insecure`, deve funcionar sem erro de TLS
- [ ] Verificar o certificado no browser (acessar `https://estudo.staypuff.info` e verificar o cadeado)
- [ ] Confirmar o redirecionamento HTTP → HTTPS: `curl -v http://estudo.staypuff.info` — deve retornar 301/308?
- [ ] Se não redirecionar, adicionar a annotation: `nginx.ingress.kubernetes.io/ssl-redirect: "true"`

**Verificação:**
```bash
kubectl get certificate estudo-tls-prod -n estudo
# READY: True

curl -v https://estudo.staypuff.info 2>&1 | grep -E "< HTTP|issuer|subject"
# HTTP/2 200 (ou 301 se redirect)
# Issuer: Let's Encrypt (sem "(STAGING)")

curl -v http://estudo.staypuff.info 2>&1 | grep "< HTTP\|Location"
# HTTP/1.1 308 ou 301, Location: https://estudo.staypuff.info
```

---

## Exercício 5: Múltiplos hosts e paths no mesmo Ingress

**Contexto:** O time adicionou dois serviços: um frontend e uma API. Precisam ser roteados no mesmo Ingress — um por subdomínio, um por path.

**Missão:** Configure um Ingress com múltiplas regras de roteamento e TLS compartilhado.

**Requisitos:**
- [ ] Criar dois Deployments no namespace `estudo`:
  - `api-app`: imagem `hashicorp/http-echo:latest` com arg `-text=resposta-da-api`
  - `frontend-app`: imagem `nginx:1.27-alpine`
- [ ] Criar Services para cada um (ports diferentes: api na 5678, frontend na 80)
- [ ] Criar Ingress `multi-app` com TLS (staging) cobrindo dois hosts:
  - `api.staypuff.info/api` → `api-svc:5678`
  - `estudo.staypuff.info/` → `frontend-svc:80`
- [ ] Testar que cada host/path roteia para o serviço correto:
  - `curl https://api.staypuff.info/api` → resposta da API
  - `curl https://estudo.staypuff.info/` → página nginx
- [ ] Verificar que path `/outros` em `api.staypuff.info` retorna 404 do nginx controller (não do Service)
- [ ] Documentar: o cert gerado cobre os dois domínios? Ver SANs: `openssl s_client -connect api.staypuff.info:443 2>/dev/null | openssl x509 -noout -text | grep -A2 "Subject Alternative"`

**Verificação:**
```bash
curl -sk https://api.staypuff.info/api
# resposta-da-api

curl -sk https://estudo.staypuff.info/
# Welcome to nginx

kubectl get certificate -n estudo
# Um único cert deve ter ambos os domínios nas SANs
```

---

## Exercício 6: Instalar Traefik e criar rota equivalente

**Contexto:** O time está avaliando Traefik como alternativa ao nginx. Você precisa instalar Traefik em paralelo (sem interferir com o nginx) e criar a mesma rota de estudo via Traefik.

**Missão:** Instale o Traefik via Helm em namespace próprio e exponha a mesma aplicação via `IngressClass: traefik`.

**Requisitos:**
- [ ] Instalar Traefik via Helm no namespace `traefik`:
  ```bash
  helm repo add traefik https://traefik.github.io/charts
  helm install traefik traefik/traefik \
    --namespace traefik --create-namespace \
    --set ingressClass.isDefaultClass=false \
    --set service.type=LoadBalancer
  ```
- [ ] Verificar o IP alocado pelo MetalLB para o Traefik: `kubectl get svc -n traefik`
- [ ] Confirmar que o IP é diferente de 192.168.3.100 (não conflitar com nginx)
- [ ] Criar Ingress `webapp-traefik` com `ingressClassName: traefik` no namespace `estudo`:
  - host: `estudo-traefik.staypuff.info`
  - annotation: `cert-manager.io/cluster-issuer: letsencrypt-staging`
  - TLS com `secretName: estudo-traefik-tls`
- [ ] Criar entrada DNS `estudo-traefik.staypuff.info` apontando para o IP do Traefik (no Cloudflare ou via `/etc/hosts`)
- [ ] Verificar que o cert-manager emitiu o certificado via staging
- [ ] Testar: `curl -vk https://estudo-traefik.staypuff.info`
- [ ] Comparar: o tempo de atualização de rota do Traefik vs nginx (adicionar um path novo e cronometrar)

**Verificação:**
```bash
kubectl get svc -n traefik
# EXTERNAL-IP: 192.168.3.x (diferente de .100)

kubectl get ingressclass
# nginx e traefik presentes

kubectl get ingress webapp-traefik -n estudo
# ADDRESS: IP do Traefik

curl -vk https://estudo-traefik.staypuff.info
# Página nginx servida via Traefik com cert staging
```

---

## Exercício 7: Issuer namespace-scoped vs ClusterIssuer

**Contexto:** O time de segurança exige que cada namespace gerencie seus próprios Issuers — sem depender de um ClusterIssuer compartilhado. Você vai criar um Issuer namespace-scoped e demonstrar a diferença.

**Missão:** Crie um Issuer no namespace `estudo` e emita um certificado usando-o (sem ClusterIssuer).

**Requisitos:**
- [ ] Identificar o Secret do token da Cloudflare (qual namespace e nome ele está?): `kubectl get secret -A | grep cloudflare`
- [ ] O Secret precisa estar no mesmo namespace do Issuer. Copiar o Secret para o namespace `estudo`:
  ```bash
  kubectl get secret cloudflare-api-token -n cert-manager -o yaml | \
    sed 's/namespace: cert-manager/namespace: estudo/' | \
    kubectl apply -f -
  ```
- [ ] Criar `Issuer` (não ClusterIssuer) no namespace `estudo` com o mesmo servidor ACME staging
- [ ] Criar um `Certificate` explícito referenciando `issuerRef.kind: Issuer` (não ClusterIssuer):
  ```yaml
  issuerRef:
    name: letsencrypt-staging-local
    kind: Issuer             # ← namespace-scoped
    group: cert-manager.io
  ```
- [ ] Monitorar a emissão
- [ ] Comparar: qual a diferença no `kubectl describe order` entre usar Issuer vs ClusterIssuer?
- [ ] Documentar: quando usar cada um? Quais são as implicações de segurança?

**Verificação:**
```bash
kubectl get issuer -n estudo
# READY: True

kubectl get certificate <nome> -n estudo
# READY: True, via Issuer namespace-scoped

kubectl describe certificate <nome> -n estudo
# IssuerRef.Kind: Issuer (não ClusterIssuer)
```

---

## Exercício 8: Troubleshooting — cert não emite, 502, path errado (Staff-level)

**Contexto:** São 14h de sexta-feira. O time reporta 3 problemas simultâneos na nova configuração de Ingress. Nenhum tem documentação. Você tem 30 minutos para resolver todos.

**Missão:** Aplique os manifests abaixo e corrija todos os problemas sem olhar dicas.

**Manifest 1 — Ingress com backend errado:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-quebrado-1
  namespace: estudo
spec:
  ingressClassName: nginx
  rules:
  - host: debug1.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: servico-que-nao-existe
            port:
              number: 80
```

**Manifest 2 — Certificate com Issuer errado:**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert-quebrado
  namespace: estudo
spec:
  secretName: cert-quebrado-tls
  dnsNames:
  - debug2.staypuff.info
  issuerRef:
    name: issuer-que-nao-existe
    kind: ClusterIssuer
```

**Manifest 3 — Ingress com ingressClassName inexistente:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-quebrado-3
  namespace: estudo
spec:
  ingressClassName: controller-que-nao-existe
  rules:
  - host: debug3.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

**Requisitos:**
- [ ] Aplicar os 3 manifests
- [ ] Para cada problema: identificar o sintoma exato, a causa raiz e a correção
- [ ] Usar a cadeia de diagnóstico correta para cada tipo:
  - Ingress: `describe ingress` → `get endpoints` → `get pods`
  - Certificate: `describe certificate` → `describe certificaterequest` → `describe order` → `describe challenge`
  - IngressClass: `get ingressclass` → `describe ingress`
- [ ] Corrigir Manifest 1: criar o Service correto apontando para um Pod existente
- [ ] Corrigir Manifest 2: referenciar um ClusterIssuer existente
- [ ] Corrigir Manifest 3: usar um `ingressClassName` válido
- [ ] Verificar que os 3 Ingresses recebem ADDRESS após as correções
- [ ] Limpeza final: `kubectl delete namespace estudo && helm uninstall traefik -n traefik && kubectl delete namespace traefik`

**Verificação:**
```bash
kubectl get ingress -n estudo
# ADDRESS: 192.168.3.100 para os 3 ingresses após correção

kubectl get certificate -n estudo
# READY: True para cert-quebrado após correção

kubectl describe ingress ingress-quebrado-1 -n estudo
# Events: deve mostrar o Service correto nos backends
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura obrigatória — entender o que existe antes de criar.
> Exercícios 2, 3 e 4 são sequenciais — HTTP → staging → produção.
> Exercício 5 pode ser feito após o 4 — usa recursos criados nos anteriores.
> Exercício 6 é independente — pode ser feito em qualquer ordem, mas requer Helm.
> Exercício 7 requer entender ClusterIssuer antes — faça após o 4.
> Exercício 8 é o mais desafiador — faça por último, com todos os conceitos dominados.

> [!warning] Rate limit do Let's Encrypt
> Use **sempre** o ClusterIssuer de staging para testar. O rate limit de produção é 5 certificados por domínio por semana. Se errar 5 vezes com produção, você fica bloqueado por uma semana inteira.

> [!warning] Limpeza de recursos de staging
> Certificados staging não são válidos. Ao final dos exercícios de staging, deletar os Secrets TLS gerados evita que o Ingress Controller tente usá-los acidentalmente: `kubectl delete secret -l cert-manager.io/certificate-name -n estudo`

> [!info] DNS para exercícios
> Os subdomínios de teste (`estudo.staypuff.info`, `api.staypuff.info`, etc.) precisam ter registros DNS apontando para 192.168.3.100 (nginx) ou para o IP do Traefik. Crie os registros A no Cloudflare antes de iniciar os exercícios de TLS.
