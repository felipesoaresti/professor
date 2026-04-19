---
tags:
  - trilha
  - networking
area: networking
tipo: trilha
---

# Trilha Networking

> Canvas geral: [[trilha-devops-sre]]

## Assuntos Cobertos

| Assunto | Conteúdo | Exercícios | Lab | Status |
|---|---|---|---|---|
| SSH / OpenSSH | [[02-Networking/ssh-openssh]] | [[11-Exercicios/ssh-openssh]] | [[12-Labs/ssh-openssh]] | Conteúdo gerado — pendente exercícios |
| Calico Network — BGP, NetworkPolicy, IPAM, GlobalNetworkPolicy | [[02-Networking/calico-network]] | [[11-Exercicios/calico-network]] | — | Exercícios gerados — pendente execução |

## Sequência Recomendada

### Fundamentos de Acesso e Segurança
1. [[02-Networking/ssh-openssh]] — SSH, OpenSSH, autenticação, port forwarding, hardening
2. TLS/mTLS internals — handshake, certificados X.509, PKI *(a fazer)*
3. DNS profundo — resolução, recursão, DNSSEC, split-horizon *(a fazer)*

### Protocolos de Transporte
4. TCP internals — three-way handshake, estados, congestion control, TIME_WAIT *(a fazer)*
5. HTTP/1.1 vs HTTP/2 vs HTTP/3 — multiplexing, QUIC, headers *(a fazer)*
6. Load balancing — L4 vs L7, algoritmos, health checks *(a fazer)*

### Redes em Kubernetes
7. [[02-Networking/calico-network]] — BGP, Felix, BIRD, IPAM, NetworkPolicy, GlobalNetworkPolicy, NetworkSet ✅
8. Service mesh — Istio/Linkerd, mTLS automático, traffic management *(a fazer)*
9. Ingress e Gateway API — nginx, TLS termination, rate limiting *(a fazer)*

### Diagnóstico e Performance
10. tcpdump e Wireshark — captura, filtros, análise de pacotes *(a fazer)*
11. iptables e nftables — chains, regras, NAT, DNAT *(a fazer)*
12. Diagnóstico de rede — ss, ip, traceroute, mtr, netcat *(a fazer)*

## Próximo Sugerido

**iptables e nftables** — o Calico usa iptables extensivamente. Entender as chains, tables (filter/nat/mangle), e como o Felix as popula permite diagnosticar NetworkPolicy com precisão cirúrgica. Também é a base para entender kube-proxy e MetalLB.

Ou: **tcpdump e Wireshark** — o Exercício 2 do Calico usa tcpdump. Aprofundar filtros BPF, captura em interfaces do cluster, e correlação com NetworkPolicy.

Ou: **TLS/mTLS internals** — complemento do cert-manager e Ingress já cobertos no K8s. Handshake, X.509, PKI.
