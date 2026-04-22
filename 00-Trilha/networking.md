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

## Sequência Recomendada

### Fundamentos de Acesso e Segurança
1. TLS/mTLS internals — handshake, certificados X.509, PKI *(a fazer)*
3. DNS profundo — resolução, recursão, DNSSEC, split-horizon *(a fazer)*

### Protocolos de Transporte
4. TCP internals — three-way handshake, estados, congestion control, TIME_WAIT *(a fazer)*
5. HTTP/1.1 vs HTTP/2 vs HTTP/3 — multiplexing, QUIC, headers *(a fazer)*
6. Load balancing — L4 vs L7, algoritmos, health checks *(a fazer)*

### Diagnóstico e Performance
10. tcpdump e Wireshark — captura, filtros, análise de pacotes *(a fazer)*
11. iptables e nftables — chains, regras, NAT, DNAT *(a fazer)*
12. Diagnóstico de rede — ss, ip, traceroute, mtr, netcat *(a fazer)*

## Próximo Sugerido

**TLS/mTLS internals** — complemento do cert-manager e Ingress já cobertos no K8s. Handshake, X.509, PKI.

Ou: **iptables e nftables** — chains, tables (filter/nat/mangle), NAT, DNAT. Base para entender kube-proxy e MetalLB.

Ou: **tcpdump e Wireshark** — filtros BPF, captura em interfaces do cluster, análise de pacotes.
