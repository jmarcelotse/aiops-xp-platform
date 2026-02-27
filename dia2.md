# Dia 02

# Material de Apoio — Dia 02: Codificação Agêntica e MCP

Material de referência com todos os prompts, configurações e manifestos utilizados na aula do Dia 02. Este arquivo é autocontido — você pode acompanhar toda a aula usando apenas este documento, sem precisar acessar outros arquivos.

**Pré-requisitos:**

- Node.js 18+ (para instalar o Gemini CLI)
- Docker instalado (para rodar o MCP Server Kubernetes)
- Cluster Kubernetes funcional (local ou cloud)
- kubectl configurado e apontando para o cluster
- Conta Google (para autenticação OAuth do Gemini CLI)

---

## 1. Setup: Instalação do Gemini CLI

O Gemini CLI é um agente de codificação e DevOps que roda direto no terminal. Ele analisa projetos, edita arquivos, executa comandos e automatiza workflows usando os modelos Gemini. Além disso, funciona como **Host MCP** — o que permite conectá-lo a MCP Servers que dão acesso a ferramentas externas (como Kubernetes).

### Instalação

Site oficial: https://geminicli.com/

```bash
npm install -g @google/gemini-cli
```

### Autenticação

Execute o comando abaixo e siga o fluxo de autenticação no navegador:

```bash
gemini
```

Na primeira execução, o Gemini CLI abre o navegador para autenticação OAuth com sua conta Google. Após autorizar, o token fica salvo localmente e você não precisa repetir o processo.

**Alternativa com API Key:**

Se preferir usar uma API Key em vez de OAuth, exporte a variável de ambiente antes de executar:

```bash
export GEMINI_API_KEY=sua-chave-aqui
gemini
```

---

## 2. Setup: Configuração do MCP Server Kubernetes

O MCP Server de Kubernetes permite que o Gemini CLI interaja com seu cluster usando ferramentas padronizadas (listar pods, descrever recursos, ler logs, etc.) — sem precisar executar comandos kubectl via shell.

### Arquivo de configuração

Crie ou edite o arquivo `~/.gemini/settings.json` com o seguinte conteúdo:

```json
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
    }
  }
}
```

**Explicação dos campos:**

- `mcpServers` — seção que lista todos os MCP Servers que o Gemini CLI deve iniciar
- `kubernetes` — nome identificador do server (pode ser qualquer nome)
- `command` — executável que inicia o server (neste caso, `docker`)
- `args` — argumentos passados ao Docker: `run -i --rm` cria um container temporário e interativo, `v` monta o kubeconfig local dentro do container, e `mcp/kubernetes` é a imagem Docker do MCP Server oficial de Kubernetes

> **Importante:** substitua `/home/fabricioveronez/.kube` pelo caminho do seu kubeconfig. Se você usa o padrão, será `$HOME/.kube`.
> 

### Verificação

Dentro do Gemini CLI, use os comandos abaixo para confirmar que o MCP Server está ativo:

- `/mcp` — lista os MCP Servers conectados
- `/tools` — lista todas as ferramentas disponíveis (deve mostrar `kubectl_get`, `kubectl_describe`, `kubectl_logs`, etc.)

---

## 3. Manifestos Kubernetes — Cenários de Caos

Estes manifestos contêm erros **propositais** para serem usados como cenário de diagnóstico. Aplique todos no cluster antes de usar os prompts de diagnóstico (seções 7 e 8 deste material).

---

### Cenário 1: Erro de imagem — ImagePullBackOff

O Deployment abaixo usa a imagem `ngin:latest` (falta o "x" de nginx). O Kubernetes não consegue baixar a imagem e o pod entra em estado `ImagePullBackOff`.

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
kubectl apply -f caos/nginx.yaml
```

---

### Cenário 2: Banco de dados inexistente — CrashLoopBackOff

Este manifesto cria o namespace `encontros-tech` com um PostgreSQL e a aplicação Encontros Tech. O erro está na `DATABASE_URL` do Secret: ela aponta para o banco `encontros`, mas o PostgreSQL cria o banco `encontros_tech`. A aplicação não consegue conectar e entra em `CrashLoopBackOff`.

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
kubectl apply -f caos/deploy-encontros-tech.yaml
```

---

### Cenário 3: Recursos insuficientes — FailedScheduling

Este manifesto cria o namespace `ecommerce` com um PostgreSQL e uma aplicação Flask. O erro está no Deployment `app`: ele solicita **40 réplicas**, cada uma com `250m` de CPU (total: 10 CPUs). O cluster não tem capacidade suficiente e os pods ficam em `Pending` com evento `FailedScheduling`.

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
# Deployment Aplicacao Flask
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: ecommerce
spec:
  replicas: 40
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
kubectl apply -f caos/deploy-ecommerce.yaml
```

---

## 4. Projeto Base: Encontros Tech

O projeto usado nas demonstrações de codificação agêntica (criação de manifestos e README) é o **Encontros Tech** — uma plataforma para gestão de eventos tecnológicos, construída com Flask, PostgreSQL e Docker.

Clone o repositório antes de iniciar as demonstrações:

```bash
git clone <https://github.com/kubedev/encontros-tech.git>
cd encontros-tech
```

Este é o projeto que o agente vai analisar nos Prompts 1 e 2.

---

## 5. Prompt 1 — Criação de Manifestos Kubernetes

Este prompt demonstra **codificação agêntica**: você dá uma instrução de alto nível e o agente investiga o projeto antes de gerar código. O comportamento esperado passa por 4 fases:

1. **Fase de análise** — o agente lê os arquivos do projeto (Dockerfile, docker-compose, dependências, .env)
2. **Fase de decisão** — determina imagem, portas, variáveis de ambiente e dependências externas
3. **Fase de criação** — gera um único arquivo `k8s/deploy.yaml` com todos os recursos
4. **Fase de resumo** — apresenta o que foi criado e instruções de uso

```
Analise este projeto e crie os manifestos Kubernetes necessarios para fazer o deploy em um cluster.

Regras:
- Use APENAS recursos do tipo Deployment e Service
- Gere TUDO em um unico arquivo chamado k8s/deploy.yaml, separando cada recurso com ---
- Analise o projeto para identificar:
  - Linguagem e runtime utilizados
  - Porta(s) que a aplicacao expoe
  - Variaveis de ambiente necessarias (verifique .env, .env.example, docker-compose, codigo fonte)
  - Se existe Dockerfile, use a imagem que seria gerada por ele. Se nao existir, sugira uma imagem base adequada
  - Dependencias externas (banco de dados, cache, fila) — inclua o Deployment e Service de cada dependencia no mesmo arquivo
- Para cada Deployment configure:
  - replicas: 1
  - resources com requests e limits adequados ao tipo de aplicacao
  - containerPort correspondente a porta real da aplicacao
  - envFrom ou env com as variaveis identificadas (use valores placeholder onde necessario)
  - Labels consistentes: app, component
- Para cada Service configure:
  - Tipo ClusterIP por padrao
  - Porta mapeada corretamente para o targetPort do container
  - Selector correspondente ao label do Deployment
- Organize os recursos no arquivo na seguinte ordem:
  1. Deployments e Services das dependencias (banco de dados, cache, etc.)
  2. Deployment e Service da aplicacao principal
- Ao final, apresente um resumo explicando:
  - Quais recursos foram criados e por que
  - Quais variaveis de ambiente precisam ser preenchidas antes do deploy
  - Qual o comando para aplicar o manifesto
```

---

## 6. Prompt 2 — Geração de README

Este prompt faz o agente analisar o projeto e gerar um [README.md](http://readme.md/) completo e profissional. O agente investiga a estrutura de arquivos, dependências e código fonte antes de escrever a documentação.

```
Voce e um especialista em documentacao tecnica. Sua tarefa e analisar um projeto de software e criar um README.md completo, profissional e bem estruturado seguindo o template abaixo.

**Diretrizes Importantes:**
- Use emojis para tornar o documento visualmente atrativo
- Mantenha linguagem clara, tecnica mas acessivel
- Inclua informacoes praticas e acionaveis
- Organize o conteudo de forma progressiva (visao geral -> implementacao -> uso pratico)
- Use formatacao Markdown consistente
- Priorize informacoes relevantes para desenvolvedores e usuarios

## Template de Estrutura

### [Nome do Projeto] - [Descricao Curta]

> **Status**: [Status atual do projeto] | [Breve descricao do estado]

### Visao Geral

**Objetivo**
[Descreva o proposito principal do projeto em 1-2 frases claras]

**Proposta de Valor**
- **Para [Usuario Tipo 1]**: [Beneficio especifico]
- **Para [Usuario Tipo 2]**: [Beneficio especifico]
- **Para [Stakeholder]**: [Impacto no ecossistema]

### Funcionalidades Implementadas

#### [Categoria de Funcionalidade 1]
- **[Feature 1]** com [tecnologia/framework]
- **[Feature 2]** [descricao tecnica]

### Stack Tecnologica

#### **Backend**
- **[Framework Principal]** - [Descricao/versao]
- **[ORM/Database]** - [Funcao especifica]

#### **Frontend**
[Se aplicavel, liste tecnologias frontend]

### Estrutura do Projeto

[nome-projeto]/
├── [diretorio-principal]/
│   ├── [subdiretorio]/     # [Descricao da funcao]
├── [config-dir]/
└── README.md

### Como Executar

#### **Pre-requisitos**
- [Runtime/linguagem] [versao]+
- [Dependencia externa 1]

#### **Instalacao e Execucao**
[Comandos passo a passo]

### Configuracao de Variaveis de Ambiente

| Variavel | Descricao | Valor Padrao | Obrigatoria |
|----------|-----------|--------------|-------------|
| VAR_1    | [Descricao] | -          | Sim         |

### Proximos Passos (Roadmap)

#### **Fase 1 - [Nome da Fase]**
- [Item do roadmap]
```

---

## 7. Prompt 3 — Visão Geral do Cluster (MCP)

Este prompt usa as ferramentas MCP para obter um **inventário completo** do cluster. Diferente do prompt de diagnóstico (que foca em encontrar problemas), este mapeia o estado atual de todos os recursos — nodes, namespaces, workloads, pods e services.

```
Faca uma visao geral completa do cluster Kubernetes.

Use as tools MCP de Kubernetes para coletar dados reais do cluster.

Pense passo a passo e apresente o resultado no formato de relatorio abaixo:

## Visao Geral do Cluster

### Infraestrutura
- Quantidade de nodes, status (Ready/NotReady), versao do Kubernetes
- Capacidade total de CPU e memoria
- Quanto esta alocado vs disponivel (em % de uso)

### Namespaces Ativos
Tabela com cada namespace e a contagem de deployments, pods e services dentro dele.

### Workloads
Tabela com todos os deployments, statefulsets e daemonsets, mostrando namespace, nome e replicas desejadas vs disponiveis.

### Pods
Tabela com todos os pods, mostrando namespace, nome, status e restarts. Agrupe por namespace.

### Services
Tabela com todos os services, mostrando namespace, nome, tipo (ClusterIP/NodePort/LoadBalancer) e portas.

### Itens que Merecem Atencao
Liste qualquer anomalia encontrada durante a coleta: pods nao Running, replicas indisponiveis, nodes NotReady, uso de recursos acima de 80%, etc. Se nao houver problemas, registre que o cluster esta saudavel.

Crie um arquivo markdown chamado visao-geral.md
```

---

## 8. Prompt 4 — Diagnóstico com Causa Raiz (MCP)

Este é o prompt mais completo da aula. Ele combina o padrão **ReAct** (Reasoning + Acting) com **Chain of Thought** para investigação autônoma de problemas no cluster. O agente decide sozinho quais ferramentas usar, em que ordem, e quantas iterações são necessárias.

### Ferramentas MCP disponíveis

O prompt referencia as ferramentas do MCP Server de Kubernetes. Estas são as tools que o agente pode usar durante a investigação:

| Ferramenta MCP | Quando usar |
| --- | --- |
| `kubectl_get` | Listar e obter recursos (pods, events, deployments, nodes, pvc, etc.). Suporta `fieldSelector`, `labelSelector`, `allNamespaces` e `sortBy`. |
| `kubectl_describe` | Obter detalhes completos de um recurso específico (conditions, events, status). |
| `kubectl_logs` | Obter logs de pods, deployments ou jobs. Suporta `previous`, `tail`, `since` e `timestamps`. |
| `kubectl_generic` | Executar comandos kubectl não cobertos pelas ferramentas acima (ex: `top nodes`, `top pods`). |
| `explain_resource` | Consultar documentação de campos de recursos Kubernetes quando houver dúvida sobre specs. |
| `exec_in_pod` | Executar comandos dentro de um pod para diagnóstico (ex: verificar DNS, conectividade). |
| `node_management` | Operações em nodes (cordon, drain, uncordon) — usar apenas no plano de correção quando necessário. |
| `kubectl_apply` | Aplicar manifestos YAML de correção — usar apenas quando explicitamente autorizado pelo usuário. |

### Prompt

```
Voce e um engenheiro SRE especialista em Kubernetes. Sua missao e investigar eventos de Warning e Error no cluster, identificar causas raiz e propor planos de correcao.

## Ferramentas Disponiveis

Voce tem acesso ao cluster Kubernetes exclusivamente atraves do MCP de Kubernetes. Use **somente** as ferramentas MCP listadas abaixo para interagir com o cluster. **Nunca execute comandos kubectl via shell/bash.**

| Ferramenta MCP | Quando usar |
|---|---|
| kubectl_get | Listar e obter recursos (pods, events, deployments, nodes, pvc, etc.). Suporta fieldSelector, labelSelector, allNamespaces e sortBy. |
| kubectl_describe | Obter detalhes completos de um recurso especifico (conditions, events, status). |
| kubectl_logs | Obter logs de pods, deployments ou jobs. Suporta previous, tail, since e timestamps. |
| kubectl_generic | Executar comandos kubectl nao cobertos pelas ferramentas acima (ex: top nodes, top pods). |
| explain_resource | Consultar documentacao de campos de recursos Kubernetes quando houver duvida sobre specs. |
| exec_in_pod | Executar comandos dentro de um pod para diagnostico (ex: verificar DNS, conectividade). |
| node_management | Operacoes em nodes (cordon, drain, uncordon) — usar apenas no plano de correcao quando necessario. |
| kubectl_apply | Aplicar manifestos YAML de correcao — usar apenas quando explicitamente autorizado pelo usuario. |

## Metodologia: ReAct (Reasoning + Acting)

Conduza a investigacao em ciclos iterativos. Para cada ciclo:

1. **Thought:** Raciocine sobre o estado atual da investigacao — o que voce ja sabe, o que ainda falta, qual a proxima informacao mais valiosa a coletar.
2. **Action:** Execute **uma** chamada a ferramenta MCP mais adequada para obter essa informacao.
3. **Observation:** Analise o resultado. Identifique dados relevantes, descarte ruido, e decida se precisa de mais informacao ou se pode avancar para a conclusao.

Repita ate ter evidencias suficientes para determinar a causa raiz. Nao existe numero fixo de ciclos — use quantos forem necessarios, mas evite coletar dados redundantes.

### Ponto de Partida

Inicie sempre obtendo os eventos anormais do cluster:
- Use kubectl_get com resourceType: "events" e allNamespaces: true

A partir dos eventos retornados, voce decide autonomamente quais recursos investigar e com quais ferramentas, com base no tipo de problema observado.

## Analise Final: Chain of Thought

Apos coletar evidencias suficientes, construa sua analise passo a passo:

1. **Fatos:** Liste objetivamente os eventos anormais e dados coletados.
2. **Correlacao temporal:** Os eventos tem relacao temporal? Existe sequencia causal (A causou B que causou C)?
3. **Correlacao de recursos:** Os eventos compartilham namespace, node, deployment ou outro recurso comum?
4. **Hipoteses e eliminacao:** Liste as causas possiveis e para cada uma indique quais evidencias a confirmam ou refutam.
5. **Causa raiz:** Determine a causa mais provavel com base nas evidencias.

## Formato de Saida

Ao final da analise, crie um arquivo markdown chamado relatorio-k8s.md com todos os dados estruturados abaixo.

Para cada problema identificado:

## Problema [N]: [Titulo curto]

**Severidade:** Critica | Alta | Media | Baixa
**Namespace:** <namespace>
**Recursos afetados:** <lista>

### Eventos Observados
| Timestamp | Tipo | Razao | Recurso | Mensagem |
|-----------|------|-------|---------|----------|
| ...       | ...  | ...   | ...     | ...      |

### Cadeia de Raciocinio

1. **Observacao inicial:** ...
2. **Investigacao:** ...
3. **Correlacao:** ...
4. **Conclusao:** ...

### Causa Raiz
[Descricao clara e objetiva]

### Plano de Correcao

**Acao imediata (mitigacao):**
- [Passos com manifestos YAML ou indicacao de qual ferramenta MCP usar e com quais parametros]

**Correcao definitiva:**
- [Passos com manifestos YAML ou indicacao de qual ferramenta MCP usar e com quais parametros]

**Prevencao futura:**
- [Recomendacoes]

Ao final, gere o resumo:

## Resumo Executivo

**Data da analise:** [data]
**Total de eventos anormais:** [numero]
**Problemas identificados:** [numero]

| # | Problema | Severidade | Causa Raiz | Status |
|---|----------|------------|------------|--------|
| 1 | ...      | ...        | ...        | Requer acao |

### Acoes Prioritarias
1. [Acao mais urgente]
2. [Segunda acao]
3. [Terceira acao]

## Regras

1. **Nunca assuma** — sempre colete dados reais do cluster antes de concluir.
2. **Use somente ferramentas MCP** — nunca execute kubectl via bash/shell.
3. **Seja especifico** — inclua nomes reais de recursos, namespaces e mensagens de erro.
4. **Nao pare no sintoma** — se um Pod esta em CrashLoopBackOff, investigue os logs para encontrar o erro real.
5. **Considere efeitos cascata** — um problema em um Node pode causar dezenas de eventos em Pods. Agrupe antes de analisar.
6. **Priorize por impacto** — problemas que afetam disponibilidade vem primeiro.
7. **Correcoes com manifestos** — planos de correcao devem incluir YAML aplicavel ou parametros exatos das ferramentas MCP, prontos para uso.
8. **Nunca aplique correcoes sem autorizacao** — apenas proponha. Use kubectl_apply somente se o usuario autorizar explicitamente.
```

---

## 9. Referências

- [Gemini CLI](https://geminicli.com/) — Agente de codificação e DevOps do Google Gemini
- [MCP Server Kubernetes](https://hub.docker.com/r/mcp/kubernetes) — Imagem Docker do MCP Server oficial para Kubernetes
- [Model Context Protocol](https://modelcontextprotocol.io/) — Especificação do protocolo MCP
- [Diretório de MCP Servers](https://github.com/modelcontextprotocol/servers) — Lista de MCP Servers disponíveis
- [Projeto Encontros Tech](https://github.com/kubedev/encontros-tech) — Aplicação usada nas demonstrações de codificação agêntica