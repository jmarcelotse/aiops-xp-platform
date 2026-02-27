# Visão Geral do Cluster Kubernetes

**Data da Análise:** 27 de Fevereiro de 2026, 16:15 BRT  
**Cluster:** nxt-cluster-staging  
**Versão Kubernetes:** v1.35.0  
**Tipo:** Kind (Kubernetes in Docker)

---

## Infraestrutura

### Nodes

| Node | Role | Status | Versão | CPU | Memória | IP Interno |
|------|------|--------|--------|-----|---------|------------|
| nxt-cluster-staging-control-plane | control-plane | ✅ Ready | v1.35.0 | 8 cores | 15.4 GB | 172.18.0.5 |
| nxt-cluster-staging-worker | worker | ✅ Ready | v1.35.0 | 8 cores | 15.4 GB | 172.18.0.2 |
| nxt-cluster-staging-worker2 | worker | ✅ Ready | v1.35.0 | 8 cores | 15.4 GB | 172.18.0.3 |
| nxt-cluster-staging-worker3 | worker | ✅ Ready | v1.35.0 | 8 cores | 15.4 GB | 172.18.0.4 |

**Total de Nodes:** 4 (1 control-plane + 3 workers)  
**Idade do Cluster:** 7 dias e 1 hora

### Capacidade Total do Cluster

**CPU:**
- Total: 32 cores (8 cores × 4 nodes)
- Alocável: 32 cores
- Uso: Métricas não disponíveis (Metrics Server não instalado)

**Memória:**
- Total: 61.5 GB (15.4 GB × 4 nodes)
- Alocável: 61.5 GB
- Uso: Métricas não disponíveis (Metrics Server não instalado)

**Pods:**
- Capacidade máxima: 440 pods (110 por node)
- Pods em execução: 19 pods

**Container Runtime:** containerd 2.2.0  
**Sistema Operacional:** Debian GNU/Linux 12 (bookworm)  
**Kernel:** 6.17.0-14-generic

---

## Namespaces Ativos

| Namespace | Deployments | Pods | Services | Idade |
|-----------|-------------|------|----------|-------|
| default | 2 | 4 | 3 | 7d1h |
| kube-system | 2 | 10 | 1 | 7d1h |
| local-path-storage | 1 | 1 | 0 | 7d1h |
| kube-node-lease | 0 | 0 | 0 | 7d1h |
| kube-public | 0 | 0 | 0 | 7d1h |

**Total:** 5 namespaces

---

## Workloads

### Deployments

| Namespace | Nome | Réplicas Desejadas | Réplicas Disponíveis | Status | Imagem |
|-----------|------|-------------------|---------------------|--------|--------|
| default | encontros-tech-app | 3 | 3 | ✅ 3/3 | fabricioveronez/encontros-tech-labs:v1 |
| default | postgres-db | 1 | 1 | ✅ 1/1 | postgres:15-alpine |
| kube-system | coredns | 2 | 2 | ✅ 2/2 | registry.k8s.io/coredns/coredns:v1.13.1 |
| local-path-storage | local-path-provisioner | 1 | 1 | ✅ 1/1 | kindest/local-path-provisioner:v20251212 |

**Total:** 4 Deployments (7 réplicas desejadas, 7 disponíveis)

### StatefulSets

Nenhum StatefulSet encontrado no cluster.

### DaemonSets

| Namespace | Nome | Desired | Current | Ready | Imagem |
|-----------|------|---------|---------|-------|--------|
| kube-system | kindnet | 4 | 4 | 4 | kindest/kindnetd:v20251212 |
| kube-system | kube-proxy | 4 | 4 | 4 | registry.k8s.io/kube-proxy:v1.35.0 |

**Total:** 2 DaemonSets (8 pods de sistema)

---

## Pods

### Namespace: default

| Nome | Status | Restarts | Node | IP |
|------|--------|----------|------|-----|
| encontros-tech-app-675dc47fd6-pxxbm | ✅ Running | 1 (12m ago) | nxt-cluster-staging-worker | 10.244.3.2 |
| encontros-tech-app-675dc47fd6-rmbmk | ✅ Running | 2 (12m ago) | nxt-cluster-staging-worker3 | 10.244.2.3 |
| encontros-tech-app-675dc47fd6-v24ts | ✅ Running | 1 (12m ago) | nxt-cluster-staging-worker2 | 10.244.1.2 |
| postgres-db-5f884fcff6-zqw4q | ✅ Running | 0 | nxt-cluster-staging-worker3 | 10.244.2.2 |

**Total:** 4 pods (4 Running)

### Namespace: kube-system

| Nome | Status | Restarts | Node | IP |
|------|--------|----------|------|-----|
| coredns-7d764666f9-lkbc2 | ✅ Running | 0 | nxt-cluster-staging-control-plane | 10.244.0.2 |
| coredns-7d764666f9-z8nx7 | ✅ Running | 0 | nxt-cluster-staging-control-plane | 10.244.0.3 |
| etcd-nxt-cluster-staging-control-plane | ✅ Running | 0 | nxt-cluster-staging-control-plane | 172.18.0.5 |
| kindnet-gmg6k | ✅ Running | 0 | nxt-cluster-staging-control-plane | 172.18.0.5 |
| kindnet-nptfd | ✅ Running | 0 | nxt-cluster-staging-worker | 172.18.0.2 |
| kindnet-rnkxs | ✅ Running | 0 | nxt-cluster-staging-worker3 | 172.18.0.4 |
| kindnet-w4j7l | ✅ Running | 0 | nxt-cluster-staging-worker2 | 172.18.0.3 |
| kube-apiserver-nxt-cluster-staging-control-plane | ✅ Running | 0 | nxt-cluster-staging-control-plane | 172.18.0.5 |
| kube-controller-manager-nxt-cluster-staging-control-plane | ✅ Running | 2 (4d2h ago) | nxt-cluster-staging-control-plane | 172.18.0.5 |
| kube-proxy-jwgm6 | ✅ Running | 0 | nxt-cluster-staging-worker | 172.18.0.2 |
| kube-proxy-kcs4b | ✅ Running | 0 | nxt-cluster-staging-worker2 | 172.18.0.3 |
| kube-proxy-p9nbx | ✅ Running | 0 | nxt-cluster-staging-worker3 | 172.18.0.4 |
| kube-proxy-wmgtp | ✅ Running | 0 | nxt-cluster-staging-control-plane | 172.18.0.5 |
| kube-scheduler-nxt-cluster-staging-control-plane | ✅ Running | 2 (4d2h ago) | nxt-cluster-staging-control-plane | 172.18.0.5 |

**Total:** 14 pods (14 Running)

### Namespace: local-path-storage

| Nome | Status | Restarts | Node | IP |
|------|--------|----------|------|-----|
| local-path-provisioner-67b8995b4b-ss6zm | ✅ Running | 0 | nxt-cluster-staging-control-plane | 10.244.0.4 |

**Total:** 1 pod (1 Running)

### Resumo Geral de Pods

- **Total de Pods no Cluster:** 19
- **Running:** 19 (100%)
- **Pending/Failed:** 0
- **Pods com Restarts:** 5 pods (4 aplicação + 1 sistema)

---

## Services

| Namespace | Nome | Tipo | Cluster IP | Portas | Selector |
|-----------|------|------|------------|--------|----------|
| default | encontros-tech-app | NodePort | 10.96.66.158 | 8000:30800/TCP | app=encontros-tech, component=application |
| default | postgres-db | ClusterIP | 10.96.49.222 | 5432/TCP | app=encontros-tech, component=database |
| default | kubernetes | ClusterIP | 10.96.0.1 | 443/TCP | - |
| kube-system | kube-dns | ClusterIP | 10.96.0.10 | 53/UDP, 53/TCP, 9153/TCP | k8s-app=kube-dns |

**Total:** 4 Services
- **ClusterIP:** 3
- **NodePort:** 1
- **LoadBalancer:** 0

---

## Itens que Merecem Atenção

### ⚠️ Restarts de Pods da Aplicação

**Pods com Restarts Recentes:**
- `encontros-tech-app-675dc47fd6-pxxbm`: 1 restart (12 minutos atrás)
- `encontros-tech-app-675dc47fd6-rmbmk`: 2 restarts (12 minutos atrás)
- `encontros-tech-app-675dc47fd6-v24ts`: 1 restart (12 minutos atrás)

**Causa Identificada:** Falhas de readiness probe durante a inicialização. Os pods tentaram responder no endpoint `/metrics` antes da aplicação estar completamente pronta. Eventos mostram:
- "Readiness probe failed: connection refused"
- "Back-off restarting failed container"

**Status Atual:** Todos os pods estão Running e estáveis após os restarts iniciais.

**Recomendação:** Considerar aumentar o `initialDelaySeconds` do readiness probe de 10s para 20-30s para dar mais tempo à aplicação Flask/Gunicorn inicializar.

### ⚠️ Metrics Server Não Disponível

O Metrics Server não está instalado no cluster, impossibilitando:
- Visualização de uso de CPU e memória em tempo real
- Comandos `kubectl top nodes` e `kubectl top pods`
- Horizontal Pod Autoscaling (HPA)

**Recomendação:** Instalar o Metrics Server para monitoramento de recursos:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### ℹ️ Restarts de Componentes do Control Plane

- `kube-controller-manager`: 2 restarts (4 dias e 2 horas atrás)
- `kube-scheduler`: 2 restarts (4 dias e 2 horas atrás)

**Status:** Normal para clusters de longa duração. Componentes estão estáveis desde então.

### ✅ Pontos Positivos

1. **Todos os nodes estão Ready** - Nenhum problema de infraestrutura
2. **Todos os deployments estão com réplicas completas** - 7/7 réplicas disponíveis
3. **Nenhum pod em estado Failed ou Pending**
4. **Distribuição equilibrada de pods** - Aplicação distribuída nos 3 workers
5. **Networking funcional** - Todos os DaemonSets (kindnet, kube-proxy) operacionais
6. **DNS operacional** - CoreDNS com 2 réplicas Running
7. **Aplicação acessível** - Service NodePort exposto na porta 30800

---

## Resumo Executivo

O cluster **nxt-cluster-staging** está **operacional e saudável**. Todos os 4 nodes estão Ready, os 19 pods estão Running, e não há workloads em estado crítico. 

A aplicação **encontros-tech** foi deployada com sucesso há 13 minutos com 3 réplicas distribuídas nos workers e um banco PostgreSQL. Os restarts iniciais dos pods da aplicação foram causados por readiness probes falhando durante a inicialização, mas todos os pods estão agora estáveis.

**Principais Recomendações:**
1. Instalar Metrics Server para monitoramento de recursos
2. Ajustar readiness probe da aplicação (aumentar initialDelaySeconds)
3. Considerar implementar PersistentVolume para o PostgreSQL (atualmente usando emptyDir)

**Capacidade Disponível:**
- CPU: 32 cores disponíveis
- Memória: 61.5 GB disponíveis
- Pods: 421 slots disponíveis (19/440 em uso)

O cluster tem ampla capacidade para crescimento e está pronto para receber mais workloads.
