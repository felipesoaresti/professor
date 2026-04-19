---
tags:
  - exercicios
  - kubernetes
  - volumes
  - pv
  - pvc
  - storageclass
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/volumes-storage]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Volumes em Kubernetes

> Conteúdo: [[05-Kubernetes/volumes-storage]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**NFS:** `192.168.3.11:/mnt/nfs-data/k8s`
**StorageClasses:** `local-path` (delete/WFC), `nfs-homelab` (retain), `nfs-homelab-delete` (delete)
**Atenção:** Nunca tocar em `databases/postgres-pvc`, `controle-gastos`, `promobot`.

---

## Exercício 1: Explorar o ambiente de storage do cluster

**Contexto:** Você assumiu a administração de um cluster Kubernetes e precisa fazer um inventário completo do storage antes de qualquer operação.

**Missão:** Faça um levantamento completo das StorageClasses, PVs e PVCs existentes no cluster.

**Requisitos:**
- [ ] Listar todas as StorageClasses e identificar qual é a default
- [ ] Para cada StorageClass, registrar: provisioner, reclaimPolicy, volumeBindingMode
- [ ] Listar todos os PVs do cluster e seus estados (Available, Bound, Released)
- [ ] Listar todos os PVCs em todos os namespaces e identificar quais estão Bound vs Pending
- [ ] Para o PV bound ao `postgres-pvc` no namespace `databases`: registrar o tamanho, access mode, reclaimPolicy e storageClass (não tocar no PVC ou Pod, apenas observar)
- [ ] Identificar qual node tem os dados do PVC `local-path` (se houver algum)
- [ ] Documentar: qual StorageClass usaria para um banco de dados crítico? Por quê?

**Verificação:**
```bash
kubectl get sc
kubectl get pv
kubectl get pvc -A
kubectl describe pv <pv-do-postgres>
# Registrar todos os campos sem fazer nenhuma alteração
```

---

## Exercício 2: PVC com local-path e persistência de dados

**Contexto:** O time precisa de um volume para armazenar logs de uma aplicação. Os logs podem ser perdidos se o node morrer (não são críticos), mas devem sobreviver a restarts do Pod.

**Missão:** Crie um PVC com `local-path`, monte em um Pod, escreva dados, delete o Pod, recrie-o e confirme que os dados persistiram.

**Requisitos:**
- [ ] Criar PVC `app-logs` no namespace `default` com `storageClassName: local-path`, `accessModes: RWO`, `storage: 512Mi`
- [ ] Verificar que o PVC está em `Pending` (WaitForFirstConsumer) antes de criar o Pod
- [ ] Criar Pod `writer` que escreve a data atual em `/data/log.txt` a cada 5s
- [ ] Confirmar que o PVC vai para `Bound` após o Pod ser criado
- [ ] Identificar em qual node o PV foi criado e qual é o path no filesystem do node
- [ ] Verificar o arquivo `/data/log.txt` dentro do Pod
- [ ] Deletar o Pod e recriar com o mesmo PVC
- [ ] Confirmar que as linhas anteriores ainda estão em `/data/log.txt`
- [ ] Deletar Pod e PVC ao final — confirmar que o PV foi deletado automaticamente

**Verificação:**
```bash
kubectl get pvc app-logs
# STATUS: Bound após criar o Pod

kubectl describe pv <pv-name> | grep -E "Path|Node"
# Mostra o path no node host

kubectl exec writer -- cat /data/log.txt
# Deve mostrar linhas anteriores mesmo após recriar o Pod

kubectl get pv   # após deletar o PVC: PV deve desaparecer (reclaimPolicy: Delete)
```

---

## Exercício 3: PVC com NFS e sobrevivência de dados (reclaimPolicy: Retain)

**Contexto:** O time de dados precisa de um volume persistente para um banco de dados de desenvolvimento. Os dados não podem ser perdidos se o PVC for deletado acidentalmente.

**Missão:** Crie um PVC com `nfs-homelab` (Retain), escreva dados, delete o PVC, e demonstre que os dados ficaram preservados no PV.

**Requisitos:**
- [ ] Criar PVC `dados-dev` com `storageClassName: nfs-homelab`, `accessModes: RWO`, `storage: 1Gi`
- [ ] Confirmar que o PVC vai para `Bound` imediatamente (Immediate binding mode)
- [ ] Criar Pod que escreve um arquivo `/data/registro.txt` com conteúdo identificável
- [ ] Verificar o conteúdo do arquivo dentro do Pod
- [ ] Deletar o Pod e o PVC
- [ ] Verificar que o PV ficou em estado `Released` (não foi deletado)
- [ ] Inspecionar o PV: os dados ainda existem (o PV aponta para o path no NFS)
- [ ] Recuperar o PV: remover o `claimRef` do spec do PV para que volte a `Available`
- [ ] Criar um novo PVC apontando diretamente para esse PV via `volumeName`
- [ ] Criar um Pod e confirmar que o arquivo `/data/registro.txt` ainda está lá
- [ ] Limpar: deletar Pod, PVC e o PV manualmente

**Verificação:**
```bash
kubectl get pv   # PV em Released após deletar o PVC

kubectl describe pv <nome> | grep -E "Status|Claim|Path"
# Status: Released
# Claim: default/dados-dev   ← ainda referencia o PVC antigo

# Após remover claimRef:
kubectl get pv   # Status: Available

# Após criar novo PVC com volumeName:
kubectl exec novo-pod -- cat /data/registro.txt
# Deve mostrar o conteúdo escrito antes da deleção
```

---

## Exercício 4: RWX com NFS — múltiplos Pods escrevendo no mesmo volume

**Contexto:** A aplicação tem 3 réplicas e todas precisam ler/escrever em um diretório compartilhado (assets estáticos que qualquer réplica pode atualizar).

**Missão:** Crie um PVC RWX com `nfs-homelab-delete` e confirme que múltiplos Pods podem escrever e ler simultaneamente.

**Requisitos:**
- [ ] Criar PVC `shared-assets` com `storageClassName: nfs-homelab-delete`, `accessModes: ReadWriteMany`, `storage: 2Gi`
- [ ] Criar Deployment `webapp` com 3 réplicas, cada container escreve a cada 10s: `echo "$(date): Pod $POD_NAME" >> /data/shared.log`
- [ ] Usar Downward API para injetar `metadata.name` como variável `POD_NAME`
- [ ] Confirmar que os 3 Pods estão em nodes diferentes (ou pelo menos que o PVC está montado nos 3)
- [ ] Fazer `kubectl exec` em cada Pod e ler `/data/shared.log`
- [ ] Confirmar que todas as 3 réplicas aparecem no arquivo (cada uma escrevendo sua própria linha)
- [ ] Deletar o Deployment e o PVC ao final

**Verificação:**
```bash
kubectl get pods -l app=webapp -o wide
# Distribuídos nos workers

kubectl exec <pod-1> -- cat /data/shared.log
# Deve mostrar linhas de pod-1, pod-2 e pod-3 todas no mesmo arquivo
```

---

## Exercício 5: Diagnosticando PVC preso em Pending

**Contexto:** O time reportou que um Pod está preso em Pending. O manifesto foi aplicado mas nada funciona. Você precisa diagnosticar e corrigir.

**Missão:** Crie intencionalmente situações de PVC Pending e pratique o diagnóstico de cada uma.

**Requisitos:**
- [ ] **Caso 1:** Criar PVC com `storageClassName: storage-que-nao-existe` — diagnosticar e registrar a mensagem de erro nos Events
- [ ] **Caso 2:** Criar PVC com `local-path` mas sem criar nenhum Pod — explicar por que está Pending e o que vai fazê-lo sair
- [ ] **Caso 3:** Criar PVC com `nfs-homelab` pedindo `accessModes: ReadWriteMany` — verificar se esse access mode é suportado pelo provisioner (testar e observar)
- [ ] Para cada caso: usar `kubectl describe pvc <nome>` e registrar o campo `Events`
- [ ] Para o caso 2: criar um Pod que use o PVC e confirmar que o PVC vai para Bound
- [ ] Deletar todos os recursos criados neste exercício ao final

**Verificação:**
```bash
kubectl describe pvc <pvc-caso-1> | grep -A5 Events
# "storageclass.storage.k8s.io ... not found"

kubectl describe pvc <pvc-caso-2> | grep -A5 Events
# "waiting for first consumer..."

kubectl describe pvc <pvc-caso-3> | grep -A5 Events
# Verificar se aceita RWX ou retorna erro
```

---

## Exercício 6: emptyDir e compartilhamento entre containers do mesmo Pod

**Contexto:** Uma aplicação tem dois containers: um que gera relatórios em `/tmp/reports` e outro que faz upload dos arquivos gerados. Eles precisam compartilhar um diretório temporário dentro do Pod.

**Missão:** Configure um Pod multi-container usando `emptyDir` para compartilhar dados entre os dois containers.

**Requisitos:**
- [ ] Criar Pod `processador` com dois containers:
  - `gerador`: escreve um arquivo `/shared/report-$(date +%s).txt` a cada 15s
  - `uploader`: lista os arquivos em `/shared/` a cada 10s e imprime no log
- [ ] Ambos os containers devem montar o mesmo `emptyDir` em `/shared`
- [ ] Confirmar que o container `uploader` consegue ver os arquivos gerados pelo `gerador`
- [ ] Verificar os logs de ambos os containers com `kubectl logs processador -c gerador` e `kubectl logs processador -c uploader`
- [ ] Deletar o Pod e observar que os dados em `/shared` foram perdidos (emptyDir não persiste)

**Verificação:**
```bash
kubectl logs processador -c uploader
# Deve mostrar os arquivos criados pelo gerador sendo listados

kubectl get pod processador -o jsonpath='{.spec.volumes}'
# Deve mostrar emptyDir como tipo do volume
```

---

## Exercício 7: Encontrar onde os dados do local-path estão no node (avançado)

**Contexto:** O time de infraestrutura precisa fazer um backup manual dos dados de um PVC local-path. Para isso, você precisa encontrar exatamente onde os dados estão no filesystem do node.

**Missão:** Crie um PVC local-path com dados, encontre o path exato no node e confirme que os dados são acessíveis diretamente no node.

**Requisitos:**
- [ ] Criar PVC `backup-test` com `local-path` e um Pod que escreve dados identificáveis
- [ ] Usar `kubectl describe pv <nome>` para encontrar o campo `Path` (local no node)
- [ ] Identificar em qual node o PV foi criado
- [ ] Acessar o node via SSH e confirmar que os arquivos existem no path do PV
- [ ] Verificar que o diretório pertence ao UID do container
- [ ] Usar `rsync` ou `cp` no node para simular um backup do diretório
- [ ] Deletar Pod e PVC ao final

**Verificação:**
```bash
kubectl describe pv <pv-name> | grep Path
# /opt/local-path-provisioner/pvc-xxx.../default_backup-test

# No node (via SSH):
ls -la /opt/local-path-provisioner/pvc-xxx.../
# Deve mostrar os arquivos criados pelo Pod
```

---

## Exercício 8: Migrar dados de local-path para NFS (Staff-level)

**Contexto:** Uma aplicação foi implantada com `local-path` (rápido para dev) e agora vai para produção. Os dados precisam ser migrados para `nfs-homelab` (com Retain) sem perda e sem downtime prolongado.

**Missão:** Execute uma migração de storage: local-path → nfs-homelab, com transferência dos dados existentes.

**Requisitos:**
- [ ] Criar PVC `app-local` com `local-path` (2Gi) e Pod que escreve dados de teste
- [ ] Criar PVC `app-nfs` com `nfs-homelab` (2Gi) do mesmo tamanho
- [ ] Criar Pod de migração que monta **os dois PVCs** simultaneamente e copia os dados: `cp -av /source/. /dest/`
- [ ] Verificar que os dados foram copiados corretamente no PVC NFS
- [ ] Parar a aplicação original, atualizar o Deployment para usar `app-nfs`, reiniciar
- [ ] Confirmar que a aplicação lê os dados migrados corretamente
- [ ] Após validar, deletar o Pod de migração e o PVC `app-local`
- [ ] Verificar que `app-nfs` ficou em `Released` → não (ainda está Bound pois o Deployment usa)
- [ ] Deletar Deployment e PVC ao final

**Verificação:**
```bash
kubectl exec pod-migracao -- ls /source /dest
# Ambos devem ter os mesmos arquivos

kubectl exec <pod-app-novo> -- cat /data/<arquivo-de-teste>
# Deve mostrar os dados originais, agora servidos pelo NFS

kubectl get pvc
# app-local: Bound → (após deletar) desaparece
# app-nfs: Bound → permanece (Deployment ainda usa)
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura pura — faça primeiro, sem risco.
> Exercícios 2 e 3 cobrem local-path e NFS respectivamente — faça nessa ordem.
> Exercício 4 requer NFS com RWX — faça após o exercício 3.
> Exercício 5 é diagnóstico — pode ser feito a qualquer momento após o 2.
> Exercícios 6, 7 e 8 são independentes e avançados.

> [!warning] Limpeza obrigatória
> PVCs com reclaimPolicy Retain deixam PVs e dados no NFS mesmo após deleção.
> Após os exercícios, verificar: `kubectl get pv | grep Released`
> e deletar manualmente os PVs órfãos.
