---
tags:
  - kubernetes
  - ingress
  - nginx
  - traefik
  - cert-manager
  - tls
  - networking
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/services]]"
next:
  - "[[11-Exercicios/ingress]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Ingress — nginx, Traefik, cert-manager e TLS

> Pré-requisitos: [[05-Kubernetes/kubernetes-teoria-inicial]] | [[05-Kubernetes/services]] | Exercícios: [[11-Exercicios/ingress]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

Um **Service LoadBalancer** aloca um IP por Service. Com 20 aplicações, você precisa de 20 IPs — caro em cloud, limitado no homelab.

**Ingress** é uma camada de roteamento HTTP/HTTPS que permite que **um único IP** atenda múltiplas aplicações, roteando por host (`app1.dominio.com`, `app2.dominio.com`) ou por path (`/api`, `/admin`):

```
Internet
    │
    ▼
192.168.3.100 (MetalLB/LoadBalancer)
    │
    ▼
Ingress Controller (nginx ou Traefik)
    ├── app1.staypuff.info  → Service app1-svc → Pods app1
    ├── app2.staypuff.info  → Service app2-svc → Pods app2
    └── api.staypuff.info/v1 → Service api-svc → Pods api
```

O Ingress resolve dois problemas de uma vez:
1. **Consolidação de IPs** — um LoadBalancer para tudo HTTP/HTTPS
2. **TLS centralizado** — terminação TLS no Ingress Controller, aplicações falam HTTP internamente

> [!info] Ingress vs Gateway API
> A Gateway API é a sucessora do Ingress no Kubernetes, mais expressiva e com melhor separação de responsabilidades. O Ingress ainda é o padrão amplamente usado e exigido no CKA. Este documento cobre Ingress — Gateway API é tópico separado.

---

## Como funciona internamente

### O objeto Ingress e o IngressClass

O objeto `Ingress` define as regras de roteamento. O `IngressClass` define qual controller processa cada Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minha-app
  namespace: estudo
spec:
  ingressClassName: nginx       # ← qual controller vai processar
  rules:
  - host: estudo.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minha-app-svc
            port:
              number: 80
```

### Como o nginx Ingress Controller funciona

O nginx controller é um Pod que:
1. Faz **watch** em objetos Ingress via apiserver
2. Para cada mudança, **regenera** o `nginx.conf` completo
3. Executa `nginx -s reload` — sem downtime (o nginx termina conexões antigas graciosamente)

```
Ingress criado/modificado
       │
       ▼
nginx controller (Pod)
  │ lê todas as regras Ingress
  │ gera nginx.conf com upstream, server blocks, SSL
  ▼
nginx.conf atualizado → nginx -s reload
```

O `nginx.conf` gerado tem um `upstream` por Service e `server` blocks por host/path. Você pode inspecionar:

```bash
kubectl exec -n ingress-nginx -it <pod-nginx-controller> -- cat /etc/nginx/nginx.conf
```

### Como o Traefik funciona

O Traefik usa um modelo diferente — **configuração dinâmica** direto da API do Kubernetes, sem regenerar arquivo de config:

```
Ingress criado/modificado
       │
       ▼
Traefik (Pod)
  │ watch no apiserver via informers
  │ atualiza roteamento em memória (sem reload)
  ▼
Roteamento atualizado em tempo real (sem restart)
```

O Traefik também suporta seus próprios CRDs (`IngressRoute`, `Middleware`) mais expressivos que o Ingress padrão — mas funciona também com Ingress padrão via `ingressClassName: traefik`.

### PathType — como as rotas são combinadas

| PathType | Comportamento |
|---|---|
| `Exact` | Combina exatamente `/foo` — não combina `/foo/` |
| `Prefix` | Combina `/foo` e qualquer coisa começando com `/foo/` |
| `ImplementationSpecific` | O controller decide (nginx usa regex, Traefik tem comportamento próprio) |

### TLS — fluxo da requisição

Com TLS configurado, o Ingress Controller termina o TLS antes de encaminhar para o Service:

```
Cliente
  │ TLS handshake (certificado do Secret tls.crt/tls.key)
  ▼
Ingress Controller (porta 443)
  │ desencripta, inspeciona Host header
  │ encaminha HTTP puro (sem TLS)
  ▼
Service → Pod
```

O certificado fica num Secret do tipo `kubernetes.io/tls` com as chaves `tls.crt` e `tls.key`.

---

## cert-manager — emissão automática de certificados TLS

### O que é cert-manager

cert-manager é um controller Kubernetes que automatiza a emissão e renovação de certificados TLS. Ele integra com Let's Encrypt, Vault, e outras CAs.

Recursos principais:
- **Issuer / ClusterIssuer** — define como emitir certificados (qual CA, qual challenge)
- **Certificate** — define qual certificado emitir (domínio, validade, qual Issuer usar)
- **CertificateRequest** — pedido individual de certificado (gerenciado automaticamente)
- **Order / Challenge** — processo de validação com Let's Encrypt (ACME)

### ClusterIssuer vs Issuer

| | `Issuer` | `ClusterIssuer` |
|---|---|---|
| Escopo | Um namespace | Todo o cluster |
| Quando usar | Isolamento por equipe/projeto | Shared entre todos os namespaces |
| Referência no Certificate | `issuerRef.kind: Issuer` | `issuerRef.kind: ClusterIssuer` |

No homelab, usar **ClusterIssuer** é o padrão — um único Issuer para todos os namespaces.

### Let's Encrypt: Staging vs Produção

| | Staging | Produção |
|---|---|---|
| Certificados válidos? | Não (CA de teste) | Sim |
| Rate limits | Generosos | Rígidos (5 certs/domínio/semana) |
| Quando usar | Testar a configuração | Após validar no staging |
| URL ACME | `acme-staging-v02.api.letsencrypt.org` | `acme-v02.api.letsencrypt.org` |

> [!warning] Sempre use staging primeiro
> Se errar a configuração e tentar repetidamente, você atinge o rate limit de produção e fica **bloqueado por uma semana**. Use staging para validar o fluxo completo antes de emitir o cert de produção.

### DNS-01 vs HTTP-01

| Challenge | Como funciona | Quando usar |
|---|---|---|
| `HTTP-01` | Let's Encrypt faz GET em `http://dominio/.well-known/acme-challenge/<token>` | Domínios publicamente acessíveis via HTTP |
| `DNS-01` | Adiciona registro TXT `_acme-challenge.dominio` no DNS | Wildcards (`*.dominio.com`), domínios internos, IPs privados |

No homelab, **DNS-01 via Cloudflare** é obrigatório porque:
- O IP 192.168.3.100 é privado — Let's Encrypt não consegue fazer HTTP-01
- Wildcard `*.staypuff.info` só é possível via DNS-01

### ClusterIssuer — Staging e Produção

```yaml
# Staging
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: felipe.staypuff@gmail.com
    privateKeySecretRef:
      name: letsencrypt-staging-key    # chave privada ACME — gerada automaticamente
    solvers:
    - dns01:
        cloudflare:
          email: felipe.staypuff@gmail.com
          apiTokenSecretRef:
            name: cloudflare-api-token  # Secret com token da Cloudflare
            key: api-token
```

```yaml
# Produção
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: felipe.staypuff@gmail.com
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
    - dns01:
        cloudflare:
          email: felipe.staypuff@gmail.com
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

### Issuer (namespace-scoped)

```yaml
# Issuer em namespace específico
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: estudo              # só funciona neste namespace
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: felipe.staypuff@gmail.com
    privateKeySecretRef:
      name: letsencrypt-staging-key-estudo
    solvers:
    - dns01:
        cloudflare:
          email: felipe.staypuff@gmail.com
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

### Fluxo completo de emissão de certificado

```
1. Ingress criado com annotation cert-manager.io/cluster-issuer
   OU Certificate criado explicitamente
       │
       ▼
2. cert-manager detecta → cria CertificateRequest
       │
       ▼
3. CertificateRequest → cria Order com Let's Encrypt ACME
       │
       ▼
4. Order → cria Challenge(s)
       │  (DNS-01: adiciona TXT record via Cloudflare API)
       │  (HTTP-01: cria Pod/Service temporário para servir o token)
       ▼
5. Let's Encrypt valida o Challenge
       │  (verifica o TXT record ou o endpoint HTTP)
       ▼
6. Let's Encrypt emite o certificado
       │
       ▼
7. cert-manager armazena em Secret kubernetes.io/tls
       │  (tls.crt + tls.key)
       ▼
8. Ingress Controller usa o Secret para TLS termination
```

---

## Na prática — nginx Ingress Controller

### Ingress HTTP simples

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: estudo
spec:
  ingressClassName: nginx
  rules:
  - host: estudo.staypuff.info
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

### Ingress HTTPS com cert-manager (annotation approach)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: estudo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-production"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - estudo.staypuff.info
    secretName: estudo-tls          # cert-manager cria este Secret automaticamente
  rules:
  - host: estudo.staypuff.info
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

### Ingress HTTPS com Certificate explícito (abordagem declarativa)

```yaml
# Certificate separado — mais controle sobre validade, renovação, etc.
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: estudo-cert
  namespace: estudo
spec:
  secretName: estudo-tls
  duration: 2160h   # 90 dias
  renewBefore: 360h # renovar 15 dias antes
  dnsNames:
  - estudo.staypuff.info
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: estudo
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - estudo.staypuff.info
    secretName: estudo-tls         # mesmo nome do Certificate.spec.secretName
  rules:
  - host: estudo.staypuff.info
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

### Múltiplos hosts e paths

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
  namespace: estudo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-production"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app1.staypuff.info
    - app2.staypuff.info
    secretName: multi-app-tls
  rules:
  - host: app1.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-svc
            port:
              number: 80
  - host: app2.staypuff.info
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

### Anotações nginx mais usadas

```yaml
annotations:
  # Redirect HTTP → HTTPS
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

  # Rewrite de path (ex: /api/v1 → /)
  nginx.ingress.kubernetes.io/rewrite-target: /$2
  # (com path: /api(/|$)(.*))

  # Tamanho máximo de upload
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  # Timeouts
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "120"

  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "10"

  # Basic Auth
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Área restrita"

  # Backend protocol (se o backend fala HTTPS)
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

  # CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.staypuff.info"
```

---

## Na prática — Traefik Ingress Controller

### Instalação via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=false   # não interferir com nginx
```

### IngressClass do Traefik

```yaml
# Verificar se foi criado pelo Helm
kubectl get ingressclass
# NAME      CONTROLLER                      PARAMETERS   AGE
# nginx     k8s.io/ingress-nginx            <none>       30d
# traefik   traefik.io/ingress-controller   <none>       1m
```

### Ingress via Traefik (objeto padrão)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-traefik
  namespace: estudo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: traefik          # ← diferença: traefik em vez de nginx
  tls:
  - hosts:
    - estudo-traefik.staypuff.info
    secretName: estudo-traefik-tls
  rules:
  - host: estudo-traefik.staypuff.info
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

### IngressRoute (CRD nativo do Traefik — mais expressivo)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: webapp-route
  namespace: estudo
spec:
  entryPoints:
  - websecure               # porta 443
  routes:
  - match: Host(`estudo-traefik.staypuff.info`)
    kind: Rule
    services:
    - name: webapp-svc
      port: 80
  tls:
    secretName: estudo-traefik-tls
```

### Middleware Traefik (equivalente às annotations do nginx)

```yaml
# Redirect HTTP → HTTPS
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: estudo
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
# Rate limiting
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: estudo
spec:
  rateLimit:
    average: 100
    burst: 50
```

### Comparação nginx vs Traefik

| Aspecto | nginx Ingress | Traefik |
|---|---|---|
| Config update | Regenera nginx.conf + reload | Dinâmico em memória, sem reload |
| API nativa K8s | Ingress padrão | Ingress padrão + IngressRoute CRD |
| Dashboard | Não (uso o ingress para expor métricas) | Dashboard web integrado |
| Maturidade | Mais maduro, mais usado | Crescendo, mais moderno |
| Annotations | Extenso (nginx.ingress.kubernetes.io/*) | Limitado — usa Middlewares CRD |
| Performance | Alta (nginx battle-tested) | Alta (Go, sem reload) |
| cert-manager | Integração via annotation ou Certificate | Idem |

---

## Comandos essenciais de diagnóstico

```bash
# Ver todos os Ingresses do cluster
kubectl get ingress -A

# Detalhes de um Ingress (address, rules, backend)
kubectl describe ingress <nome> -n <ns>

# Ver IngressClasses disponíveis
kubectl get ingressclass

# Verificar o IP atribuído ao Ingress Controller
kubectl get svc -n ingress-nginx
kubectl get svc -n traefik

# Status dos certificados
kubectl get certificate -n <ns>
kubectl describe certificate <nome> -n <ns>

# Ver Orders (processo ACME)
kubectl get order -n <ns>
kubectl describe order <nome> -n <ns>

# Ver Challenges (validação DNS/HTTP)
kubectl get challenge -n <ns>
kubectl describe challenge <nome> -n <ns>

# Logs do cert-manager (diagnóstico de emissão)
kubectl logs -n cert-manager -l app=cert-manager --tail=50

# Logs do nginx controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50

# Logs do Traefik
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=50

# Testar resolução DNS e TLS
curl -v https://estudo.staypuff.info
curl -v --resolve estudo.staypuff.info:443:192.168.3.100 https://estudo.staypuff.info

# Ver o nginx.conf gerado pelo controller
kubectl exec -n ingress-nginx <pod> -- cat /etc/nginx/nginx.conf | grep -A10 "server {"
```

---

## Casos de uso e boas práticas

**Use staging antes de produção** — sempre. Um cert staging tem a mesma configuração, o mesmo fluxo DNS-01, mas não consome rate limit.

**Um Ingress por namespace/aplicação** — facilita troubleshooting e controle de acesso. Um Ingress com 50 rules é difícil de depurar.

**`secretName` único por domínio** — dois Ingresses com o mesmo `secretName` no mesmo namespace compartilham o cert. Isso é válido mas precisa ser intencional.

**Wildcard para o homelab** — um único Certificate `*.staypuff.info` cobre todos os subdomínios, sem emitir cert individual por aplicação:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-staypuff
  namespace: cert-manager      # ou onde o IngressController lê
spec:
  secretName: wildcard-staypuff-tls
  dnsNames:
  - "*.staypuff.info"
  - "staypuff.info"
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
```

**HTTP → HTTPS redirect** — sempre habilitar em produção:

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

**Separar `tls:` por Secret por namespace** — o cert-manager e o Ingress Controller precisam ter acesso ao Secret. Se o Ingress está num namespace diferente do Secret, o controller não consegue lê-lo.

---

## Troubleshooting — cenários reais de produção

### Ingress sem address (EXTERNAL-IP vazio)

```bash
kubectl get ingress -n estudo
# ADDRESS vazio

kubectl describe ingress <nome> -n estudo
# Events: vazio ou "no IngressClass found"
```

**Causas:**
- `ingressClassName` errado ou ausente — verificar `kubectl get ingressclass`
- IngressClass não existe no cluster
- Ingress Controller não está rodando — `kubectl get pods -n ingress-nginx`

### Certificado em estado `False` ou `Pending`

```bash
kubectl get certificate -n estudo
# READY: False

kubectl describe certificate <nome> -n estudo
# Status.conditions: "Certificate does not exist"

kubectl get order -n estudo
kubectl describe order <nome> -n estudo
# Status: pending
```

Seguir a cadeia: Certificate → CertificateRequest → Order → Challenge.

### Challenge falhando (DNS-01)

```bash
kubectl describe challenge <nome> -n estudo
# reason: "Waiting for DNS-01 challenge propagation"
# ou: "Error presenting challenge: Cloudflare API error"
```

**Causas comuns:**
- Token da Cloudflare inválido ou sem permissão de escrita no DNS — verificar o Secret
- Propagação DNS ainda não concluída — aguardar (pode levar 30s–5min)
- Domínio não gerenciado pela Cloudflare referenciada no ClusterIssuer

```bash
# Verificar token da Cloudflare
kubectl get secret cloudflare-api-token -n cert-manager -o jsonpath='{.data.api-token}' | base64 -d

# Verificar logs do cert-manager
kubectl logs -n cert-manager -l app=cert-manager | grep -i cloudflare
```

### TLS não funciona — browser diz "certificado inválido"

**Staging vs Produção:** O staging emite certificados assinados por uma CA de teste ("(STAGING) Fake LE Root X1") — browsers não confiam. Se o hostname está servindo um cert de staging, precisa mudar o annotation para `letsencrypt-production`.

**Secret não existe ou está em namespace errado:**
```bash
kubectl get secret estudo-tls -n estudo
# Error from server (NotFound) → cert-manager ainda não emitiu OU está em outro ns
```

### 502 Bad Gateway

```bash
kubectl logs -n ingress-nginx <pod> | grep "502"
```

O Ingress Controller chegou ao Service, mas o Pod não respondeu. Verificar:
1. `kubectl get endpoints <svc> -n estudo` — há IPs?
2. `kubectl get pods -n estudo` — Pods estão Running?
3. `kubectl logs <pod> -n estudo` — a aplicação está respondendo?

### Path not routing correctly

```bash
kubectl describe ingress <nome> -n estudo
# Rules: verificar path e pathType
```

`pathType: Exact` com `/api` não roteia `/api/v1`. Precisa de `Prefix`.

---

## Nível avançado — edge cases e cenários CKA/Staff

### Wildcard com cert compartilhado entre namespaces

O Secret TLS precisa existir no mesmo namespace do Ingress. Para compartilhar um wildcard entre namespaces, use o projeto **reflector** ou **kubed** para sincronizar Secrets entre namespaces automaticamente:

```bash
# Annotation no Secret para o reflector sincronizar
annotations:
  reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
  reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "estudo,producao"
```

### Forçar renovação de certificado

```bash
# Deletar o Secret faz o cert-manager reemitir automaticamente
kubectl delete secret estudo-tls -n estudo
# cert-manager detecta o Secret sumiu e reinicia o processo de emissão
```

### Inspecionar o nginx.conf gerado

```bash
kubectl exec -n ingress-nginx <controller-pod> -- \
  nginx -T 2>/dev/null | grep -A30 "server {" | head -60
```

### Rate limiting por IP com nginx

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-connections: "10"
  nginx.ingress.kubernetes.io/limit-rps: "5"
  nginx.ingress.kubernetes.io/limit-whitelist: "192.168.3.0/24"  # whitelist local
```

### Traefik dashboard exposto via IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
  - websecure
  routes:
  - match: Host(`traefik.staypuff.info`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
    kind: Rule
    services:
    - name: api@internal         # serviço interno do Traefik
      kind: TraefikService
  tls:
    secretName: traefik-tls
```

### Múltiplos IngressControllers coexistindo

No homelab, nginx e Traefik podem coexistir — cada Ingress usa o `ingressClassName` correto:
- `ingressClassName: nginx` → processado pelo nginx controller
- `ingressClassName: traefik` → processado pelo Traefik

Cada controller tem seu próprio LoadBalancer IP (MetalLB aloca IPs distintos).

### Annotations de timeout para WebSockets e SSE

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"    # 1 hora para WS
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
  nginx.ingress.kubernetes.io/configuration-snippet: |
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
```

---

## Referências

- [Kubernetes Ingress Docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [nginx Ingress Controller Docs](https://kubernetes.github.io/ingress-nginx/)
- [Traefik Docs — Kubernetes](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [cert-manager — Cloudflare DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- Relacionados: [[05-Kubernetes/services]] | [[05-Kubernetes/secrets]] | [[05-Kubernetes/networking-policy]]
