# Relatório de Análise de Eventos Kubernetes

**Data da análise:** 27 de Fevereiro de 2026, 19:37 BRT  
**Cluster:** nxt-cluster-staging  
**Período analisado:** Últimas 1 hora  
**Metodologia:** ReAct (Reasoning + Acting)

---

## Problema 1: Degradação de Performance do etcd Causando Instabilidade no Control Plane

**Severidade:** Crítica  
**Namespace:** kube-system  
**Recursos afetados:** 
- etcd-nxt-cluster-staging-control-plane
- kube-apiserver-nxt-cluster-staging-control-plane
- kube-controller-manager-nxt-cluster-staging-control-plane
- kube-scheduler-nxt-cluster-staging-control-plane

### Eventos Observados

| Timestamp | Tipo | Razão | Recurso | Mensagem |
|-----------|------|-------|---------|----------|
| 2026-02-27 19:28:04 | Warning | Unhealthy | Pod/etcd | Readiness probe failed: context deadline exceeded |
| 2026-02-27 19:29:03 | Warning | Unhealthy | Pod/kube-apiserver | Readiness probe failed: HTTP probe failed with statuscode: 500 |
| 2026-02-27 19:29:05 | Warning | Unhealthy | Pod/kube-apiserver | Liveness probe failed: HTTP probe failed with statuscode: 500 |
| 2026-02-27 19:28:01 | Warning | Unhealthy | Pod/kube-controller-manager | Liveness probe failed: connection reset by peer |
| 2026-02-27 19:29:10 | Warning | BackOff | Pod/kube-controller-manager | Back-off restarting failed container |
| 2026-02-27 19:29:06 | Warning | Unhealthy | Pod/kube-scheduler | Readiness probe failed: connection refused |
| 2026-02-27 19:29:13 | Warning | BackOff | Pod/kube-scheduler | Back-off restarting failed container |

### Cadeia de Raciocínio

#### 1. Observação Inicial
Eventos de Warning concentrados no período 19:28-19:29, afetando todos os componentes críticos do control plane simultaneamente. Correlação temporal clara indica evento sistêmico, não falhas isoladas.

#### 2. Investigação - Componentes do Control Plane

**kube-scheduler:**
- Logs mostraram falha ao renovar lease de liderança
- Erro: "Failed to update lease: context deadline exceeded"
- Causa: Timeout ao comunicar com kube-apiserver (5s timeout excedido)
- Resultado: Perda de liderança e restart do componente

**kube-controller-manager:**
- Padrão idêntico ao scheduler
- Logs mostraram "leaderelection lost"
- Shutdown ordenado seguido de restart
- 4 restarts totais (último há 8m53s do momento da análise)

**kube-apiserver:**
- Logs críticos mostrando múltiplos timeouts:
  - "apiserver was unable to write a JSON response: http: Handler timeout"
  - "Post-timeout activity" em operações de lease
  - Falhas ao atualizar leases de coordenação
- Causa raiz identificada: **Comunicação com etcd extremamente lenta**

#### 3. Investigação - etcd (Causa Raiz)

**Evidências de Performance Degradada:**

Logs do etcd revelaram performance crítica:

```
"apply request took too long" - Múltiplas ocorrências
- Transações: 1.7s a 3.9s (esperado: <100ms)
- Operações de leitura: 800ms a 2.6s
- Raft requests: 1.7s a 3.7s
```

**Exemplos específicos:**
- Transaction duration: 3.941s (39x mais lento que o esperado)
- Linearizable read: 2.674s (26x mais lento)
- Apply request: 2.674s para operação que deveria levar <100ms

**Erros de comunicação etcd-client:**
```
"retrying of unary invoker failed"
"rpc error: code = DeadlineExceeded desc = context deadline exceeded"
```

#### 4. Correlação - Efeito Cascata

```
etcd lento (causa raiz)
    ↓
kube-apiserver timeout (não consegue ler/escrever no etcd)
    ↓
kube-scheduler/controller-manager não conseguem renovar leases
    ↓
Perda de liderança e restarts
    ↓
Instabilidade temporária no cluster
```

#### 5. Análise de Recursos

**Configuração atual do etcd:**
- CPU request: 100m (0.1 core)
- Memory request: 100Mi
- Sem limits definidos
- Volume: hostPath em /var/lib/etcd

**Problema identificado:**
- Recursos extremamente limitados para um componente crítico
- Em cluster Kind (Docker), o etcd compete por I/O com outros containers
- Possível contenção de disco ou CPU insuficiente

### Causa Raiz

**Performance degradada do etcd devido a recursos insuficientes**, causando:

1. **Latência excessiva** em operações de leitura/escrita (1-4 segundos vs <100ms esperado)
2. **Timeouts em cascata** no kube-apiserver ao tentar acessar o etcd
3. **Falha de renovação de leases** pelos componentes do control plane
4. **Restarts automáticos** do kube-scheduler e kube-controller-manager
5. **Instabilidade temporária** do cluster durante o período de recuperação

**Fatores contribuintes:**
- Recursos de CPU/memória muito baixos (100m/100Mi)
- Ambiente Kind (Docker) com possível contenção de I/O
- Ausência de tuning de performance do etcd
- Falta de monitoramento de métricas do etcd

### Plano de Correção

#### Ação Imediata (Mitigação)

**1. Aumentar recursos do etcd**

Editar o manifesto estático do etcd no control plane:

```bash
# Acessar o node control plane
docker exec -it nxt-cluster-staging-control-plane bash

# Editar o manifesto
vi /etc/kubernetes/manifests/etcd.yaml
```

Modificar a seção de resources:

```yaml
resources:
  requests:
    cpu: 500m      # Aumentar de 100m para 500m
    memory: 512Mi  # Aumentar de 100Mi para 512Mi
  limits:
    cpu: 1000m     # Adicionar limit
    memory: 1Gi    # Adicionar limit
```

O kubelet reiniciará automaticamente o pod etcd com os novos recursos.

**2. Monitorar recuperação**

Usar ferramentas MCP:
```
kubectl_logs com name="etcd-nxt-cluster-staging-control-plane", namespace="kube-system", tail=50
```

Verificar se as mensagens "apply request took too long" diminuem ou desaparecem.

#### Correção Definitiva

**1. Implementar monitoramento de métricas do etcd**

Criar ServiceMonitor para Prometheus (se disponível):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd-metrics
  namespace: kube-system
  labels:
    component: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: metrics
    port: 2381
    targetPort: 2381
    protocol: TCP
  selector:
    component: etcd
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      component: etcd
  endpoints:
  - port: metrics
    interval: 30s
```

**2. Configurar alertas para métricas críticas do etcd**

Métricas importantes a monitorar:
- `etcd_disk_backend_commit_duration_seconds` (latência de commit)
- `etcd_disk_wal_fsync_duration_seconds` (latência de fsync)
- `etcd_server_leader_changes_seen_total` (mudanças de líder)
- `etcd_server_proposals_failed_total` (propostas falhadas)

Alertar quando:
- Latência de commit > 100ms (p99)
- Latência de fsync > 100ms (p99)
- Mudanças de líder > 0 em 5 minutos

**3. Otimizar configuração do etcd**

Adicionar flags de performance ao manifesto:

```yaml
command:
  - etcd
  # ... flags existentes ...
  - --heartbeat-interval=100
  - --election-timeout=1000
  - --quota-backend-bytes=8589934592  # 8GB
  - --max-request-bytes=10485760      # 10MB
  - --auto-compaction-mode=periodic
  - --auto-compaction-retention=5m
```

**4. Considerar volume dedicado para etcd**

Para ambientes de produção, usar PersistentVolume com:
- SSD/NVMe para baixa latência
- IOPS garantido
- Separação de I/O de outros workloads

#### Prevenção Futura

**1. Implementar Metrics Server**

Instalar para monitoramento de recursos:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Adicionar flags para Kind:
```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

**2. Configurar Resource Quotas e Limits**

Garantir que workloads de usuário não consumam recursos críticos do control plane.

**3. Estabelecer baseline de performance**

Documentar métricas normais do etcd:
- Latência média de operações
- Taxa de transações por segundo
- Uso de CPU/memória em operação normal

**4. Implementar testes de carga periódicos**

Usar ferramentas como `etcd-benchmark` para validar performance:
```bash
etcd-benchmark put --total=10000 --key-size=8 --val-size=256
```

**5. Backup automatizado do etcd**

Configurar snapshots periódicos:
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
```

**6. Documentação de runbook**

Criar procedimentos para:
- Identificar degradação de performance do etcd
- Escalar recursos rapidamente
- Restaurar de backup em caso de corrupção
- Procedimento de disaster recovery

---

## Problema 2: Falhas Transitórias de Health Probes na Aplicação

**Severidade:** Média  
**Namespace:** default  
**Recursos afetados:**
- encontros-tech-app-675dc47fd6-pxxbm
- encontros-tech-app-675dc47fd6-rmbmk
- encontros-tech-app-675dc47fd6-v24ts

### Eventos Observados

| Timestamp | Tipo | Razão | Recurso | Mensagem |
|-----------|------|-------|---------|----------|
| 2026-02-27 19:28:59 | Warning | Unhealthy | Pod/encontros-tech-app-pxxbm | Liveness probe failed: context deadline exceeded |
| 2026-02-27 19:27:59 | Warning | Unhealthy | Pod/encontros-tech-app-rmbmk | Readiness probe failed: context deadline exceeded |
| 2026-02-27 19:28:59 | Warning | Unhealthy | Pod/encontros-tech-app-rmbmk | Liveness probe failed: context deadline exceeded |
| 2026-02-27 19:28:59 | Warning | Unhealthy | Pod/encontros-tech-app-v24ts | Liveness probe failed: context deadline exceeded |

### Cadeia de Raciocínio

#### 1. Observação Inicial
Falhas de health probes nos 3 pods da aplicação encontros-tech-app no mesmo período (19:27-19:28) dos problemas do control plane.

#### 2. Investigação
- Logs atuais da aplicação mostram funcionamento normal
- Endpoint /metrics respondendo com status 200
- Nenhum erro de aplicação nos logs
- Correlação temporal perfeita com problemas do etcd/control plane

#### 3. Correlação
As falhas de health probe foram **sintomas secundários** da instabilidade do control plane:
- Kubelet no worker node não conseguia comunicar status ao kube-apiserver
- Kube-apiserver estava lento devido ao etcd
- Health probes falharam por timeout de rede/API, não por falha da aplicação

#### 4. Conclusão
Problema transitório causado pela instabilidade do control plane. Aplicação está saudável.

### Causa Raiz

**Efeito colateral da degradação do etcd**. Durante o período de instabilidade do control plane (19:28-19:29), o kubelet teve dificuldade para:
- Reportar status dos pods ao kube-apiserver
- Executar health probes dentro do timeout configurado
- Manter comunicação estável com o control plane

A aplicação em si não apresentou falhas - os timeouts foram de infraestrutura.

### Plano de Correção

#### Ação Imediata (Mitigação)

**Nenhuma ação necessária** - problema já resolvido automaticamente após estabilização do control plane.

Verificação:
```
kubectl_logs com name="encontros-tech-app-675dc47fd6-rmbmk", namespace="default", tail=50
```

Confirma que aplicação está operacional.

#### Correção Definitiva

**1. Ajustar timeouts dos health probes**

Editar o Deployment para aumentar tolerância a latências transitórias:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: encontros-tech-app
  namespace: default
spec:
  template:
    spec:
      containers:
      - name: app
        livenessProbe:
          httpGet:
            path: /metrics
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5        # Aumentar de 1s para 5s
          failureThreshold: 5      # Aumentar de 3 para 5
        readinessProbe:
          httpGet:
            path: /metrics
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3        # Aumentar de 1s para 3s
          failureThreshold: 3
```

Aplicar com:
```
kubectl_apply com manifest="<yaml acima>"
```

**2. Implementar endpoint de health dedicado**

Criar endpoint `/health` mais leve que `/metrics`:

```python
@app.route('/health')
def health():
    return {'status': 'healthy'}, 200
```

Atualizar probes para usar `/health` em vez de `/metrics`.

#### Prevenção Futura

**1. Monitorar latência de health probes**

Adicionar métricas customizadas para rastrear:
- Tempo de resposta do endpoint de health
- Taxa de falhas de probes
- Correlação com eventos do control plane

**2. Implementar circuit breaker**

Se múltiplos pods falharem health probes simultaneamente, pode indicar problema de infraestrutura, não da aplicação.

**3. Alertas inteligentes**

Configurar alertas que diferenciem:
- Falha de um pod (problema da aplicação)
- Falha de múltiplos pods simultaneamente (problema de infraestrutura)

---

## Resumo Executivo

**Data da análise:** 27 de Fevereiro de 2026, 19:37 BRT  
**Total de eventos anormais:** 14 eventos de Warning  
**Problemas identificados:** 2 (1 crítico, 1 médio)

| # | Problema | Severidade | Causa Raiz | Status |
|---|----------|------------|------------|--------|
| 1 | Degradação de performance do etcd causando instabilidade no control plane | Crítica | Recursos insuficientes (CPU: 100m, Mem: 100Mi) causando latência de 1-4s em operações que deveriam levar <100ms | Requer ação imediata |
| 2 | Falhas transitórias de health probes na aplicação | Média | Efeito colateral do Problema 1 - kubelet não conseguia comunicar com kube-apiserver durante instabilidade | Resolvido automaticamente |

### Acões Prioritárias

#### 1. **[CRÍTICO]** Aumentar recursos do etcd imediatamente

**Ação:**
```bash
# Acessar control plane
docker exec -it nxt-cluster-staging-control-plane bash

# Editar manifesto
vi /etc/kubernetes/manifests/etcd.yaml

# Modificar resources para:
# CPU: 500m request, 1000m limit
# Memory: 512Mi request, 1Gi limit
```

**Impacto:** Reduzirá latência de operações do etcd de 1-4s para <100ms, estabilizando todo o control plane.

**Validação:** Monitorar logs do etcd por 15 minutos após restart. Mensagens "apply request took too long" devem desaparecer.

#### 2. **[ALTO]** Implementar monitoramento de métricas do etcd

**Ação:**
- Expor métricas do etcd via Service
- Configurar Prometheus para coletar métricas
- Criar dashboards no Grafana (se disponível)
- Configurar alertas para latência > 100ms

**Impacto:** Permitirá detecção proativa de degradação antes que cause instabilidade.

**Prazo:** 48 horas

#### 3. **[MÉDIO]** Instalar Metrics Server

**Ação:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Adicionar flags para Kind:
```yaml
--kubelet-insecure-tls
--kubelet-preferred-address-types=InternalIP
```

**Impacto:** Permitirá monitoramento de uso de recursos em tempo real via `kubectl top`.

**Prazo:** 24 horas

### Observações Finais

**Cluster atual:** Operacional, mas com risco de nova degradação se carga aumentar.

**Prioridade máxima:** Aumentar recursos do etcd. Este é um single point of failure - se o etcd falhar completamente, todo o cluster para.

**Ambiente Kind:** Lembrar que Kind (Kubernetes in Docker) não é recomendado para produção. Para ambientes críticos, considerar:
- Cluster gerenciado (EKS, GKE, AKS)
- Kubeadm em VMs dedicadas
- Distribuições enterprise (OpenShift, Rancher)

**Próximos passos após correção:**
1. Estabelecer baseline de performance
2. Documentar runbook de troubleshooting
3. Implementar testes de carga periódicos
4. Planejar estratégia de backup/restore do etcd
