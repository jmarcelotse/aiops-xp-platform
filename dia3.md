# Dia 03

# Material de Apoio — Dia 03: Agente AIOps com N8n

Material de referencia com todas as configuracoes, prompts, codigos e manifestos utilizados na aula do Dia 03. Este arquivo e autocontido — voce pode acompanhar toda a aula usando apenas este documento, sem precisar acessar outros arquivos.

**Pre-requisitos:**

- Docker e Docker Compose instalados
- Cluster Kubernetes funcional (local ou cloud)
- kubectl configurado e apontando para o cluster
- Conta Google (para API Key do Gemini)
- Conta Discord com permissao de administrador em um servidor (para adicionar o bot)

## Workflow Final

[agente-aiops.json](attachment:3cb9d48c-efbd-43f8-b0ff-15fa26e31774:agente-aiops.json)

## MCP

```jsx
{
  "mcpServers": {
    "kubernetes": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-v",
        "/home/fabricioveronez/.kube:/home/appuser/.kube",
        "mcp/kubernetes"
      ]
    },
    "n8n-mcp": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "--init",
        "-e", "MCP_MODE=stdio",
        "--network", "n8n",
        "-e", "LOG_LEVEL=error",
        "-e", "DISABLE_CONSOLE_OUTPUT=true",
        "-e", "N8N_API_URL=http://n8n:5678",
        "-e", "N8N_API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxNWE5YjgwZS1lNjg3LTQ0MWItODYyZS1hNjdkYWRiYzE1MDQiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwianRpIjoiYmEwNjQ0ZWYtNjRkMS00ZWY2LWExZjItYjVlMTQ3ZTE0MThkIiwiaWF0IjoxNzcyMTQ3MzQ4fQ.MOLwXg6e-bewipjo3n0t6HQ9ao8HWcq6NQgaeRPu3qs",
        "-e", "WEBHOOK_SECURITY_MODE=moderate",
        "ghcr.io/czlonkowski/n8n-mcp:latest"
      ]
    }
  }
  }
```

---

## 1. Criacao da API Key do Gemini (Google AI Studio)

O agente AIOps usa o modelo Gemini como LLM. Para conecta-lo ao N8n, voce precisa de uma API Key do Google AI Studio.

### Passo a passo

1. Acesse https://aistudio.google.com/
2. Faca login com sua conta Google
3. No menu lateral, clique em **Get API Key**
4. Clique em **Create API Key**
5. Selecione um projeto Google Cloud (ou crie um novo)
6. Copie a API Key gerada e guarde em local seguro

### Nota sobre o tier gratuito

O Google AI Studio oferece um tier gratuito com limite de requisicoes por minuto. Para uso na aula, o tier gratuito e suficiente. Verifique os limites atuais em https://ai.google.dev/pricing.

---

## 2. Criacao do Bot no Discord

O agente envia notificacoes de diagnostico para um canal do Discord usando um bot. Para isso, voce precisa criar uma aplicacao no Discord Developer Portal, gerar o token do bot e adiciona-lo ao seu servidor.

### Passo 1 — Criar a aplicacao

1. Acesse https://discord.com/developers/applications
2. Clique em **New Application** (canto superior direito)
3. De um nome (ex: `Agente AIOps`) e clique em **Create**

### Passo 2 — Gerar o token do bot

1. No menu lateral, clique em **Bot**
2. Clique em **Reset Token** e confirme
3. Copie o **Token** gerado e guarde em local seguro — voce nao conseguira ve-lo novamente

> **Importante:** O token do bot da acesso total ao bot. Nunca compartilhe publicamente ou commite em repositorios.
> 

### Passo 3 — Configurar permissoes e intents

1. Ainda na secao **Bot**, em **Privileged Gateway Intents**, ative:
    - **SERVER MEMBERS INTENT** — necessario para o bot receber eventos de membros
2. Em **Bot Permissions**, marque:
    - Send Messages
    - Embed Links
    - Read Message History
    - Read Messages/View Channels

### Passo 4 — Configurar instalacao

1. No menu lateral, clique em **Installation**
2. Em **Installation Contexts**, marque **Guild Install**
3. Em **Default Install Settings**, adicione os scopes:
    - `bot`
    - `applications.commands`
4. Nas permissoes, marque as mesmas do Passo 3

### Passo 5 — Adicionar o bot ao servidor

1. Ainda em **Installation**, copie o **Install Link**
2. Abra o link no navegador
3. Selecione o servidor onde deseja adicionar o bot
4. Clique em **Autorizar**

### Passo 6 — Obter o ID do canal

1. No Discord (app desktop ou web), va em **Configuracoes do Usuario** (icone de engrenagem)
2. Em **Configuracoes do App** → **Avancado**, ative o **Modo de Desenvolvedor**
3. Volte ao servidor, clique com o botao direito no canal desejado
4. Clique em **Copiar ID do Canal** — guarde esse valor

### Resumo dos dados necessarios

| Dado | Onde obter | Usado em |
| --- | --- | --- |
| Bot Token | Discord Developer Portal → Bot → Reset Token | Credencial no N8n |
| Channel ID | Discord → Botao direito no canal → Copiar ID | Node Discord no N8n |

---

## 3. Setup do Ambiente com Docker Compose

O ambiente consiste em dois servicos: o **N8n** (plataforma de automacao onde o agente sera construido) e o **MCP Server Kubernetes** (que da acesso ao cluster via ferramentas padronizadas).

### Docker Compose completo

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=America/Sao_Paulo
      - TZ=America/Sao_Paulo
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped
    networks:
      - n8n
  k8s-mcp:
    image: mcp/kubernetes:latest
    depends_on:
      - n8n
    networks:
      - n8n
    volumes:
      - /home/SEU_USUARIO/.kube:/home/appuser/.kube
    environment:
      ENABLE_UNSAFE_STREAMABLE_HTTP_TRANSPORT: 1
      PORT: 3001
      HOST: "0.0.0.0"
networks:
  n8n:
    name: n8n

volumes:
  n8n_data:
```

### Explicacao dos servicos

**n8n:**

- Imagem oficial do N8n
- Porta `5678` exposta para acesso via navegador
- Timezone configurado para Sao Paulo
- `N8N_RUNNERS_ENABLED=true` habilita execucao de codigo JavaScript nos nodes
- Volume `n8n_data` persiste workflows e configuracoes entre reinicializacoes

**k8s-mcp:**

- Imagem oficial do MCP Server para Kubernetes
- Depende do N8n (sobe depois)
- Monta o kubeconfig local (`~/.kube`) dentro do container para acesso ao cluster
- `ENABLE_UNSAFE_STREAMABLE_HTTP_TRANSPORT=1` habilita o transporte HTTP usado pelo N8n
- Porta `3001` e host `0.0.0.0` permitem que o N8n se conecte ao MCP Server pela rede Docker

> **Importante:** Substitua `/home/SEU_USUARIO/.kube` pelo caminho do seu kubeconfig. Ex: `/home/fabricioveronez/.kube`.
> 

### Comandos

```bash
# Subir o ambiente
docker compose up -d

# Verificar se os containers estao rodando
docker compose ps

# Ver logs do N8n
docker compose logs -f n8n

# Parar o ambiente
docker compose down
```

### Acesso

Apos subir o ambiente, acesse o N8n em: [http://localhost:5678](http://localhost:5678/)

Na primeira vez, crie uma conta local (e-mail e senha). Essa conta fica salva no volume `n8n_data`.

---

## 4. Manifestos Kubernetes — Cenarios de Caos

Estes manifestos contem erros **propositais** para serem usados como cenario de diagnostico. Aplique todos no cluster antes de testar o agente.

---

### Cenario 1: Erro de imagem — ImagePullBackOff

O Deployment abaixo usa a imagem `ngin:latest` (falta o "x" de nginx). O Kubernetes nao consegue baixar a imagem e o pod entra em estado `ImagePullBackOff`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ngin:latest
        resources: {}
```

**Aplicar:**

```bash
kubectl apply -f dia_03/k8s/nginx.yaml
```

---

### Cenario 2: Banco de dados inexistente — CrashLoopBackOff

Este manifesto cria o namespace `encontros-tech` com um PostgreSQL e a aplicacao Encontros Tech. O erro esta na `DATABASE_URL` do Secret: ela aponta para o banco `encontros`, mas o PostgreSQL cria o banco `encontros_tech`. A aplicacao nao consegue conectar e entra em `CrashLoopBackOff`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: encontros-tech
---
# Secrets (devem ser criados primeiro)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: encontros-tech
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres123
  POSTGRES_DB: encontros_tech
---
apiVersion: v1
kind: Secret
metadata:
  name: encontros-tech-secrets
  namespace: encontros-tech
type: Opaque
stringData:
  DATABASE_URL: postgresql://postgres:postgres123@postgres-service:5432/encontros
---
# PostgreSQL
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: encontros-tech
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        resources: {}
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: POSTGRES_DB
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: encontros-tech
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  type: ClusterIP
---
# Aplicacao
apiVersion: apps/v1
kind: Deployment
metadata:
  name: encontros-tech-app
  namespace: encontros-tech
  labels:
    app: encontros-tech
spec:
  replicas: 2
  selector:
    matchLabels:
      app: encontros-tech
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
      labels:
        app: encontros-tech
    spec:
      containers:
      - name: encontros-tech
        image: fabricioveronez/encontros-tech-labs:v1
        resources: {}
        ports:
        - containerPort: 8000
        envFrom:
        - secretRef:
            name: encontros-tech-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: encontros-tech-service
  namespace: encontros-tech
  labels:
    app: encontros-tech
spec:
  selector:
    app: encontros-tech
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

**Aplicar:**

```bash
kubectl apply -f dia_03/k8s/deploy-encontros-tech.yaml
```

---

### Cenario 3: Recursos insuficientes — FailedScheduling

Este manifesto cria o namespace `ecommerce` com um PostgreSQL e uma aplicacao Flask. O erro esta no Deployment `app`: ele solicita **20 replicas**, cada uma com `250m` de CPU e `256Mi` de memoria. O cluster pode nao ter capacidade suficiente e os pods ficam em `Pending` com evento `FailedScheduling`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: ecommerce
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9187"
        prometheus.io/path: "/metrics"
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: "ecommerce"
        - name: POSTGRES_PASSWORD
          value: "Pg1234"
        - name: POSTGRES_DB
          value: "ecommerce"
        resources: {}
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - ecommerce
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - ecommerce
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Service PostgreSQL
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: ecommerce
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: postgres
---
# Service Postgres Exporter
apiVersion: v1
kind: Service
metadata:
  name: postgres-exporter
  namespace: ecommerce
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 9187
    protocol: TCP
    name: metrics
  selector:
    app: postgres
---
# Deployment Aplicacao Flask
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: ecommerce
spec:
  replicas: 20
  selector:
    matchLabels:
      app: ecommerce-app
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "5000"
        prometheus.io/path: "/metrics"
      labels:
        app: ecommerce-app
    spec:
      containers:
      - name: app
        image: fabricioveronez/fake-shop-web:v1
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          value: "postgres"
        - name: DB_USER
          value: "ecommerce"
        - name: DB_PASSWORD
          value: "Pg1234"
        - name: DB_NAME
          value: "ecommerce"
        - name: DB_PORT
          value: "5432"
        - name: FLASK_APP
          value: "index.py"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
      restartPolicy: Always
---
# Service Aplicacao Flask
apiVersion: v1
kind: Service
metadata:
  name: app
  namespace: ecommerce
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    app: ecommerce-app
```

**Aplicar:**

```bash
kubectl apply -f dia_03/k8s/deploy-ecommerce.yaml
```

---

## 5. System Prompt do Agente SRE

Este e o system prompt que deve ser copiado para o campo **System Message** do node **AI Agent** no N8n. Ele combina as tecnicas do Dia 01 (Role Prompting, Chain-of-Thought, ReAct, Guardrails e Output Format) em um unico prompt de agente.

### Prompt completo

```
Voce e um engenheiro SRE especialista em Kubernetes, atuando como agente de diagnostico e triagem de incidentes.

## Contexto
Voce monitora um cluster Kubernetes e recebe eventos de alerta via webhook. Seu trabalho e investigar cada evento, identificar a causa raiz e recomendar acoes para a equipe de operacoes.

## Ferramentas Disponiveis
Voce tem acesso a ferramentas MCP conectadas ao cluster Kubernetes. Use-as para consultar o estado do cluster:
- Listar pods e seus status
- Descrever eventos de pods e deployments
- Consultar logs de containers
- Verificar recursos (CPU, memoria) dos pods

## Comportamento de Investigacao
Para cada evento recebido, siga este pipeline:

1. **Triagem**: Classifique a severidade (Critico, Alto, Medio, Baixo) e determine se e acionavel
2. **Investigacao**: Use as ferramentas para coletar dados do cluster. Pense passo a passo:
   - Qual o estado atual do pod/deployment afetado?
   - Ha eventos recentes relacionados?
   - O que os logs mostram?
   - Ha outros pods ou recursos afetados?
3. **Diagnostico**: Com base nas evidencias coletadas, identifique a causa raiz provavel
4. **Recomendacao**: Sugira acoes especificas com justificativa

## Guardrails
- Voce e um agente READ-ONLY. NUNCA execute acoes que modifiquem o cluster
- Apenas consulte, observe e recomende
- Se nao tiver certeza da causa raiz, diga explicitamente e liste as hipoteses mais provaveis
- Se o problema estiver fora do seu escopo, recomende escalacao para o time responsavel

## Formato de Saida
Responda SEMPRE neste formato:

**Severidade:** [Critico | Alto | Medio | Baixo]

**Problema Detectado:**
[Descricao concisa do problema identificado]

**Causa Raiz:**
[Causa raiz provavel com evidencias que sustentam a conclusao]

**Evidencias Coletadas:**
- [Evidencia 1: dado coletado via tool]
- [Evidencia 2: dado coletado via tool]
- [Evidencia N]

**Acoes Recomendadas:**
1. [Acao prioritaria com justificativa]
2. [Acao secundaria]
3. [Acao preventiva para evitar recorrencia]

**Observacoes:**
[Informacoes adicionais, riscos, ou pontos de atencao]
```

### Conexao com as tecnicas do Dia 01

| Secao do Prompt | Tecnica do Dia 01 | Objetivo |
| --- | --- | --- |
| "Voce e um engenheiro SRE..." | **Role Prompting** | Define personalidade e expertise |
| "Voce monitora um cluster..." | **Contexto** | Situa o agente no ambiente |
| "Pense passo a passo..." | **Chain-of-Thought (CoT)** | Forca raciocinio estruturado |
| Pipeline 1-2-3-4 | **ReAct** | Ciclo investigacao-diagnostico |
| Formato de saida estruturado | **Output Format** | Garante resposta consistente |
| "NUNCA execute acoes..." | **Guardrails** | Limita escopo de atuacao |
| "Se nao tiver certeza..." | **Honestidade** | Evita alucinacao confiante |

---

## 6. Codigo dos Nodes JavaScript

O workflow usa dois nodes JavaScript para processar eventos antes de enviar ao agente de IA.

---

### Node 1 — Filtrar Eventos Warning

Este node recebe a resposta bruta do servidor MCP Kubernetes (em formato SSE) e realiza duas operacoes: extrai e parseia o JSON embutido na resposta SSE, e filtra apenas os eventos do tipo `Warning` que ocorreram nos ultimos 5 minutos. Timestamps vazios ou invalidos sao tratados como recentes por seguranca.

```jsx
// Extrai eventos da resposta do MCP (formato SSE)
const response = $input.first().json;
let eventList = [];

try {
  // A resposta vem no campo 'data' em formato SSE
  let rawData = response.data || response;

  // Se for string, precisa parsear o SSE
  if (typeof rawData === 'string') {
    // Extrai o JSON do formato SSE (event: message\\ndata: {...})
    const jsonMatch = rawData.match(/data:\\s*(\\{[\\s\\S]*\\})\\s*$/);
    if (jsonMatch) {
      rawData = JSON.parse(jsonMatch[1]);
    }
  }

  // Navega ate o conteudo dos eventos
  const content = rawData.result && rawData.result.content && rawData.result.content[0] ? rawData.result.content[0].text : null;
  if (content) {
    const parsed = JSON.parse(content);
    eventList = parsed.events || parsed.items || [];
  }
} catch (e) {
  console.log('Erro ao parsear resposta MCP:', e.message);
  return [{ json: { hasEvents: false, count: 0, error: e.message } }];
}

const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000);

// Filtra eventos Warning dos ultimos 5 minutos
const filteredEvents = eventList.filter(event => {
  const eventTime = new Date(event.lastTimestamp || event.firstTimestamp);
  // Considera recente se: timestamp vazio, timestamp invalido, ou dentro dos ultimos 5 min
  const isRecent = !event.lastTimestamp || event.lastTimestamp === '' || isNaN(eventTime.getTime()) || eventTime >= fiveMinutesAgo;
  const isWarning = event.type === 'Warning';
  return isWarning && isRecent;
});

if (filteredEvents.length === 0) {
  return [{ json: { hasEvents: false, count: 0 } }];
}

return filteredEvents.map((event, index) => ({
  json: {
    ...event,
    hasEvents: true,
    eventIndex: index + 1,
    totalEvents: filteredEvents.length
  }
}));
```

**O que este codigo faz:**

1. Recebe a resposta bruta do MCP Server (formato SSE)
2. Extrai o JSON embutido na resposta navegando ate `result.content[0].text`
3. Filtra apenas eventos do tipo `Warning`
4. Considera recentes: eventos sem timestamp, com timestamp invalido, ou dos ultimos 5 minutos
5. Retorna cada evento como item individual com metadados (`hasEvents`, `eventIndex`, `totalEvents`)
6. Se nao houver eventos Warning, retorna um unico item com `hasEvents: false`

---

### Node 2 — Parse K8s Event

Este node recebe o array de eventos Warning filtrados e prepara o prompt para o agente de IA. Ele extrai campos relevantes, faz deduplicacao inteligente (agrupando eventos similares normalizando nomes de pods e UIDs) e monta um prompt em texto plano com problemas agrupados e eventos individuais.

```jsx
// Processa array de eventos agrupados, DEDUPLICA e cria prompt com eventos na integra
const eventos = $json.eventos || [];

if (eventos.length === 0) {
  return [{ json: { hasEvents: false, count: 0, prompt: '' } }];
}

// Extrai dados de cada evento
const eventosProcessados = eventos.map((event) => {
  return {
    eventType: event.type || event.eventType || 'Unknown',
    reason: event.reason || '',
    message: event.message || '',
    resourceKind: event.involvedObject ? event.involvedObject.kind : (event.resourceKind || ''),
    resourceName: event.involvedObject ? event.involvedObject.name : (event.resourceName || ''),
    namespace: (event.metadata && event.metadata.namespace) ? event.metadata.namespace : ((event.involvedObject && event.involvedObject.namespace) ? event.involvedObject.namespace : (event.namespace || 'default')),
    firstTimestamp: event.firstTimestamp || (event.metadata ? event.metadata.creationTimestamp : ''),
    lastTimestamp: event.lastTimestamp || event.firstTimestamp || '',
    count: event.count || 1,
    sourceComponent: event.source ? (event.source.component || 'unknown') : 'unknown',
    sourceHost: event.source ? (event.source.host || 'unknown') : 'unknown'
  };
});

// DEDUPLICACAO melhorada - normaliza nomes de pods e UIDs
const grupos = {};
for (const evt of eventosProcessados) {
  const msgNormalizada = evt.message
    .replace(/[a-z0-9]+-[a-z0-9]+-[a-z0-9]+/g, '*')
    .replace(/\\b[a-f0-9]{8}-[a-f0-9-]{27,}\\b/g, '*')
    .replace(/in pod [^\\s]+/g, 'in pod *')
    .replace(/\\([a-f0-9-]+\\)/g, '(*)');
  const chave = `${evt.reason}|${msgNormalizada}|${evt.namespace}|${evt.resourceKind}`;

  if (!grupos[chave]) {
    grupos[chave] = {
      reason: evt.reason,
      message: evt.message,
      namespace: evt.namespace,
      resourceKind: evt.resourceKind,
      eventType: evt.eventType,
      podsAfetados: [],
      totalOcorrencias: 0,
      primeiroTimestamp: evt.firstTimestamp,
      ultimoTimestamp: evt.lastTimestamp
    };
  }

  grupos[chave].totalOcorrencias++;
  if (evt.resourceName && !grupos[chave].podsAfetados.includes(evt.resourceName)) {
    grupos[chave].podsAfetados.push(evt.resourceName);
  }

  if (evt.lastTimestamp && evt.lastTimestamp > grupos[chave].ultimoTimestamp) {
    grupos[chave].ultimoTimestamp = evt.lastTimestamp;
  }
}

const problemasUnicos = Object.values(grupos);

// Prompt texto plano com problemas agrupados
const problemasTexto = problemasUnicos.map((p, i) =>
  `[Problema ${i + 1}] [${p.eventType}] ${p.reason}: ${p.message}\\nNamespace: ${p.namespace} | Recurso: ${p.resourceKind} | Pods afetados: ${p.podsAfetados.length} (${p.podsAfetados.join(', ')}) | Ocorrencias: ${p.totalOcorrencias} | Primeiro: ${p.primeiroTimestamp || 'N/A'} | Ultimo: ${p.ultimoTimestamp || 'N/A'}`
).join('\\n\\n');

// Eventos individuais em texto plano
const eventosTexto = eventosProcessados.map((e, i) =>
  `Evento ${i + 1}: [${e.eventType}] ${e.reason} | ${e.message} | ${e.resourceKind}/${e.resourceName} | ns:${e.namespace} | count:${e.count} | source:${e.sourceComponent}/${e.sourceHost} | first:${e.firstTimestamp || 'N/A'} | last:${e.lastTimestamp || 'N/A'}`
).join('\\n');

const prompt = `${problemasUnicos.length} problema(s) unico(s) de ${eventosProcessados.length} eventos totais\\n\\n--- PROBLEMAS AGRUPADOS ---\\n${problemasTexto}\\n\\n--- EVENTOS COMPLETOS ---\\n${eventosTexto}`;

return [{
  json: {
    hasEvents: true,
    count: eventosProcessados.length,
    problemasUnicos: problemasUnicos.length,
    eventos: eventosProcessados,
    gruposDeProblemas: problemasUnicos,
    prompt: prompt
  }
}];
```

**O que este codigo faz:**

1. Recebe o array de eventos Warning ja filtrados
2. Extrai campos relevantes de cada evento (reason, message, namespace, timestamps, source)
3. Faz deduplicacao inteligente: agrupa eventos similares normalizando nomes de pods e UIDs com regex
4. Monta um prompt em texto plano com duas secoes:
    - **Problemas agrupados**: com contagem de pods afetados e ocorrencias
    - **Eventos individuais**: dados completos de cada evento
5. O campo `prompt` e o que alimenta o AI Agent para gerar o diagnostico

---

## 7. Payloads de Teste (Webhook)

Use estes payloads para testar o agente via webhook do N8n. Substitua `<WEBHOOK_URL>` pela URL do webhook que o N8n gera ao configurar o node Webhook.

---

### Cenario 1: CrashLoopBackOff (OOMKilled)

Pod reiniciando repetidamente por falta de memoria.

```json
{
  "type": "Warning",
  "reason": "BackOff",
  "namespace": "default",
  "pod": "api-server-7d4b8c6f5-x2k9m",
  "message": "Back-off restarting failed container api-server in pod api-server-7d4b8c6f5-x2k9m_default",
  "timestamp": "2026-02-25T14:32:00Z",
  "source": "kubelet"
}
```

**Comando curl:**

```bash
curl -X POST <WEBHOOK_URL> \\
  -H "Content-Type: application/json" \\
  -d '{
    "type": "Warning",
    "reason": "BackOff",
    "namespace": "default",
    "pod": "api-server-7d4b8c6f5-x2k9m",
    "message": "Back-off restarting failed container api-server in pod api-server-7d4b8c6f5-x2k9m_default",
    "timestamp": "2026-02-25T14:32:00Z",
    "source": "kubelet"
  }'
```

---

### Cenario 2: ImagePullBackOff

Pod nao consegue baixar a imagem do container.

```json
{
  "type": "Warning",
  "reason": "Failed",
  "namespace": "production",
  "pod": "payment-service-5b9f7c4d8-k3j2n",
  "message": "Failed to pull image \\"registry.example.com/payment:v2.5.1\\": rpc error: code = NotFound desc = failed to pull and unpack image",
  "timestamp": "2026-02-25T15:10:00Z",
  "source": "kubelet"
}
```

**Comando curl:**

```bash
curl -X POST <WEBHOOK_URL> \\
  -H "Content-Type: application/json" \\
  -d '{
    "type": "Warning",
    "reason": "Failed",
    "namespace": "production",
    "pod": "payment-service-5b9f7c4d8-k3j2n",
    "message": "Failed to pull image \\"registry.example.com/payment:v2.5.1\\": rpc error: code = NotFound desc = failed to pull and unpack image",
    "timestamp": "2026-02-25T15:10:00Z",
    "source": "kubelet"
  }'
```

---

### Cenario 3: Pod Pending (Recursos insuficientes)

Pod nao consegue ser agendado por falta de recursos no cluster.

```json
{
  "type": "Warning",
  "reason": "FailedScheduling",
  "namespace": "default",
  "pod": "ml-worker-8c6f5d4b7-p9m2x",
  "message": "0/3 nodes are available: 3 Insufficient memory. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.",
  "timestamp": "2026-02-25T16:45:00Z",
  "source": "default-scheduler"
}
```

**Comando curl:**

```bash
curl -X POST <WEBHOOK_URL> \\
  -H "Content-Type: application/json" \\
  -d '{
    "type": "Warning",
    "reason": "FailedScheduling",
    "namespace": "default",
    "pod": "ml-worker-8c6f5d4b7-p9m2x",
    "message": "0/3 nodes are available: 3 Insufficient memory. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.",
    "timestamp": "2026-02-25T16:45:00Z",
    "source": "default-scheduler"
  }'
```

---

### Cenario 4: Liveness Probe Failing

Container falhando health check e sendo reiniciado pelo kubelet.

```json
{
  "type": "Warning",
  "reason": "Unhealthy",
  "namespace": "staging",
  "pod": "checkout-api-6d7f8e9a1-w4k5m",
  "message": "Liveness probe failed: HTTP probe failed with statuscode: 503",
  "timestamp": "2026-02-25T17:20:00Z",
  "source": "kubelet"
}
```

**Comando curl:**

```bash
curl -X POST <WEBHOOK_URL> \\
  -H "Content-Type: application/json" \\
  -d '{
    "type": "Warning",
    "reason": "Unhealthy",
    "namespace": "staging",
    "pod": "checkout-api-6d7f8e9a1-w4k5m",
    "message": "Liveness probe failed: HTTP probe failed with statuscode: 503",
    "timestamp": "2026-02-25T17:20:00Z",
    "source": "kubelet"
  }'
```

---

### Formato minimo (simplificado)

Para testes rapidos, use este payload mais simples:

```json
{
  "event": "Pod api-server em CrashLoopBackOff no namespace default. Container reiniciou 5 vezes nos ultimos 10 minutos."
}
```

No AI Agent, mapear com: `{{ $json.body.event }}`

---

## 8. Notificacao Discord — Configuracao no N8n

Apos o agente gerar o diagnostico, a notificacao e enviada ao Discord usando o node **Discord** com o bot criado na Secao 2.

### Passo 1 — Configurar a credencial do bot no N8n

1. No N8n, va em **Credentials** (menu lateral)
2. Clique em **Add Credential**
3. Busque por **Discord Bot API**
4. Preencha:
    - **Name:** `Discord Bot AIOps` (ou qualquer nome identificador)
    - **Bot Token:** Cole o token gerado na Secao 2 (Passo 2)
5. Clique em **Save**

### Passo 2 — Adicionar o node Discord ao workflow

1. Adicione um node **Discord** apos o node do AI Agent
2. Configure:
    - **Credential:** Selecione a credencial criada no passo anterior
    - **Resource:** Message
    - **Operation:** Send
    - **Channel ID:** Cole o ID do canal obtido na Secao 2 (Passo 6)
    - **Message:** `{{ $json.output }}`

A expressao `{{ $json.output }}` mapeia a saida do AI Agent diretamente para o corpo da mensagem do Discord.

### Alternativa — Usar HTTP Request (sem o node Discord)

Se preferir nao usar o node Discord nativo, voce pode usar um **HTTP Request** chamando a API do Discord diretamente:

1. Adicione um node **HTTP Request** apos o AI Agent
2. Configure:
    - **Method:** POST
    - **URL:** `https://discord.com/api/channels/SEU_CHANNEL_ID/messages`
    - **Authentication:** Header Auth
    - **Header Name:** `Authorization`
    - **Header Value:** `Bot SEU_BOT_TOKEN`
    - **Body Content Type:** JSON
    - **Body:**

```json
{
  "content": "{{ $json.output }}"
}
```

> Substitua `SEU_CHANNEL_ID` pelo ID do canal e `SEU_BOT_TOKEN` pelo token do bot.
> 

### Limite de caracteres

O Discord aceita no maximo **2000 caracteres** por mensagem. Se o diagnostico exceder esse limite, o system prompt pode incluir a instrucao: "Mantenha a resposta abaixo de 1800 caracteres".

### Emojis como indicadores visuais

O system prompt instrui o agente a usar emojis para facilitar a leitura no Discord:

| Emoji | Significado |
| --- | --- |
| 🚨 | Severidade Critica |
| 🔶 | Severidade Alta |
| 🔷 | Severidade Media |
| ℹ️ | Severidade Baixa |
| 📋 | Problema Detectado |
| 🔍 | Causa Raiz |
| 📊 | Evidencias |
| ✅ | Acoes Recomendadas |
| ⚠️ | Observacoes |

---

## 9. Prompt Simplificado para Diagnostico

Este e o prompt simplificado usado no arquivo `prompt.md` do repositorio. Ele foca exclusivamente em diagnostico read-only com ferramentas MCP, sem o pipeline completo do agente N8n.

```
Voce e um analista especializado em diagnostico Kubernetes. Sua funcao e investigar e identificar problemas usando as ferramentas disponiveis.

## REGRAS
- Apenas diagnostica e sugere correcoes. NUNCA execute kubectl_patch, kubectl_apply ou kubectl_delete.
- Use APENAS os parametros documentados das tools.
- Investigue por PROBLEMA AGRUPADO, nao por evento individual.
- Se ja tiver causa raiz + evidencias suficientes, passe ao proximo problema.
- NUNCA chame a mesma ferramenta com os mesmos parametros duas vezes.

## COMO INVESTIGAR
Voce tem 3 ferramentas de leitura. Use-as IMEDIATAMENTE para coletar evidencias:
- **kubectl_get**: visao geral dos recursos (status, restarts, idade). Use `name` OU `labelSelector`, NUNCA ambos juntos.
- **kubectl_describe**: detalhes completos (eventos, conditions, configuracao).
- **kubectl_logs**: logs do container. Use `previous: true` para logs de execucao anterior.

Para cada problema recebido:
1. Chame kubectl_describe no recurso afetado para obter detalhes
2. Se necessario, chame kubectl_logs para ver erros da aplicacao
3. Com as evidencias coletadas, escreva o relatorio

## TRATAMENTO DE ERROS
- "name cannot be provided when a selector is specified" -> Use APENAS name OU labelSelector
- Se uma ferramenta falhar 2 vezes, registre o erro como evidencia e prossiga

## SEVERIDADE
- **CRITICO**: Pod/Deployment indisponivel, servico fora do ar
- **ALTO**: Restarts frequentes, OOMKilled, recursos esgotados
- **MEDIO**: Warnings recorrentes mas servico funcional
- **BAIXO**: Eventos informativos, problemas cosmeticos

## FORMATO DE RESPOSTA (Markdown)

# Relatorio de Diagnostico Kubernetes

## Resumo
| Total | Criticos | Altos | Medios | Baixos |
|-------|----------|-------|--------|--------|
| X     | X        | X     | X      | X      |

---

## Problema 1: [causa raiz resumida]
- **Severidade:** CRITICO | ALTO | MEDIO | BAIXO
- **Namespace:** namespace-afetado
- **Pods Afetados:** pod1, pod2

### Causa Raiz
Descricao clara do problema.

### Evidencias
- evidencia 1
- evidencia 2

### Solucao Recomendada
Acao especifica para corrigir.

### Comando Sugerido
kubectl patch deployment nome -n namespace --type=merge -p '{"spec":...}'

Repita a secao para cada problema. Responda APENAS com o relatorio markdown.
```

---

## 10. Importacao do Workflow Completo

O workflow completo do agente AIOps esta disponivel no arquivo `workflows/agente-aiops.json` (quando disponibilizado). Para importar no N8n:

### Passo a passo

1. Acesse o N8n em [http://localhost:5678](http://localhost:5678/)
2. No menu lateral, clique em **Workflows**
3. Clique no botao **+** ou **Import from File**
4. Selecione o arquivo `agente-aiops.json`
5. O workflow sera importado com todos os nodes configurados

### Apos importar

1. **Configurar credenciais do Gemini:** Clique no node AI Agent → Credential → Adicione sua API Key do Google AI Studio
2. **Configurar credencial Discord:** Clique no node Discord → Selecione a credencial do bot e informe o Channel ID
3. **Configurar MCP Server:** Verifique se o node MCP aponta para `http://k8s-mcp:3001` (endereco do container na rede Docker)
4. **Ativar o workflow:** Clique em **Active** no canto superior direito

---

## 11. Referencias

- [N8n](https://n8n.io/) — Plataforma de automacao de workflows
- [N8n AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) — Documentacao do node AI Agent
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) — Especificacao do protocolo MCP
- [MCP Server Kubernetes](https://hub.docker.com/r/mcp/kubernetes) — Imagem Docker do MCP Server para Kubernetes
- [Google AI Studio](https://aistudio.google.com/) — Plataforma para gerenciar API Keys do Gemini
- [Discord Developer Portal](https://discord.com/developers/applications) — Portal para criar e gerenciar bots do Discord
- [Discord Bot API — N8n Credentials](https://docs.n8n.io/integrations/builtin/credentials/discord/) — Documentacao de credenciais Discord no N8n
- [Diretorio de MCP Servers](https://github.com/modelcontextprotocol/servers) — Lista de MCP Servers disponiveis