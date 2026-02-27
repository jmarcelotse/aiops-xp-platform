# Dia 01

# Prompts — Dia 01: Fundamentos de IA e Engenharia de Prompt

Material de referência rápida com todos os prompts demonstrados na aula do Dia 01, organizados por técnica. Use este arquivo para consultar e reutilizar os prompts após a aula.

---

## 1. Zero-Shot Prompting

Zero-Shot é a forma mais direta de interagir com um LLM: você dá uma instrução clara, sem fornecer nenhum exemplo. O modelo depende exclusivamente do que aprendeu durante o treinamento para gerar a resposta. Funciona bem para tarefas comuns (resumo, tradução, geração de código para tecnologias populares) e tem a vantagem de consumir menos tokens, resultando em menor custo e latência. Porém, quando você precisa de um formato de saída muito específico ou trabalha com terminologia proprietária, o resultado pode ser inconsistente.

---

### Prompt 1 — Revisor de artigos (simples)

Demonstra uma instrução zero-shot mínima, sem especificar critérios de revisão nem formato de resposta. O modelo faz o que consegue, mas sem direção clara.

```markdown
Revise esse parágrafo:

"A empresa decidiu que vai migrar os sistemas para cloud porque todo mundo
está fazendo isso e parece que é mais barato, então o time de TI vai
começar a mexer nisso semana que vem pra ver o que acontece."
```

---

### Prompt 2 — Revisor de artigos (elaborado)

Mesmo cenário, mas agora com os 4 elementos do prompt bem definidos: tarefa clara, critérios de revisão, contexto (blog técnico) e formato de saída estruturado. A diferença no resultado é significativa.

```markdown
Revise o parágrafo abaixo para um blog técnico de uma empresa de tecnologia.

Critérios de revisão:
- Clareza: o texto está objetivo e fácil de entender?
- Gramática: corrija erros gramaticais e de pontuação
- Tom profissional: elimine linguagem informal ou vaga
- Conteúdo: sinalize afirmações sem embasamento ou argumentos fracos

Formato da resposta:
- Problemas encontrados (lista)
- Parágrafo reescrito
- O que mudou e por quê

Parágrafo:
"A empresa decidiu que vai migrar os sistemas para cloud porque todo mundo
está fazendo isso e parece que é mais barato, então o time de TI vai
começar a mexer nisso semana que vem pra ver o que acontece."
```

---

### Prompt 3 — Deployment Kubernetes (simples)

Instrução zero-shot genérica para gerar um manifesto K8s. Sem especificações, o modelo vai inventar valores e a saída pode variar a cada chamada.

```markdown
Crie um Deployment Kubernetes para uma API de pagamento.
```

---

### Prompt 4 — Deployment Kubernetes (elaborado)

Mesmo pedido, mas com contexto completo: nome, namespace, labels, imagem, recursos, probes e formato de saída. O resultado sai pronto para `kubectl apply`.

```markdown
Crie um Deployment Kubernetes para a API de pagamento com as seguintes especificações:

Identificação:
- Nome: payment-api
- Namespace: production
- Labels: app=payment-api, team=backend, environment=production

Container:
- Imagem: gcr.io/myproject/payment-api:v2.5.0
- Porta: 8080
- 3 réplicas

Recursos:
- Request: 250m CPU, 128Mi memória
- Limit: 500m CPU, 256Mi memória

Probes:
- Liveness: GET /healthz na porta 8080, a cada 15s
- Readiness: GET /ready na porta 8080, a cada 5s

Formato: YAML válido, pronto para kubectl apply
```

---

## 2. Few-Shot Prompting

Few-Shot consiste em fornecer exemplos concretos de entrada e saída antes da pergunta real. O modelo aprende o padrão diretamente do contexto (In-Context Learning), sem precisar ser retreinado. Os exemplos ensinam três coisas: o espaço de categorias possíveis, o tipo de resposta esperada e o formato exato da saída. É a técnica ideal quando você precisa de formatos proprietários, classificações customizadas ou quando o zero-shot retorna estrutura inconsistente. Cuidado: mais de 5 exemplos trazem ganho marginal pequeno e aumentam o custo.

---

### Prompt 5 — Categorização de e-mails (cotidiano)

Exemplo few-shot com 2 exemplos de e-mails categorizados (conta a pagar e urgente pessoal) seguidos de um novo e-mail para classificar. Mostra como os exemplos controlam os campos e o formato da saída.

```markdown
Classifique e-mails recebidos usando minhas categorias e formato.
Siga exatamente o padrão dos exemplos.

---

E-MAIL 1:
De: meubanco@banco.com.br
Assunto: Sua fatura de março está disponível
Corpo: Sua fatura no valor de R$2.847,00 vence em 15/04. Acesse o app para visualizar.

Classificação:
- categoria: conta-a-pagar
- acao: Pagar fatura até 15/04
- prazo: 15/04
- prioridade: alta

---

E-MAIL 2:
De: maria.fulana@gmail.com
Assunto: Mãe no hospital
Corpo: Oi, a mãe teve uma queda e está no hospital São Lucas. Nada grave,
mas precisa de alguém para acompanhar amanhã de manhã. Você consegue ir?

Classificação:
- categoria: urgente-pessoal
- acao: Ligar para Maria e confirmar presença amanhã
- prazo: hoje
- prioridade: urgente

---

AGORA CLASSIFIQUE:

E-MAIL 3:
De: newsletter@veronez.io
Assunto: As 10 melhores ferramentas de IA para DevOps em 2026
Corpo: Confira nossa seleção semanal com as ferramentas que estão
revolucionando o mercado de operações...
```

---

### Prompt 6 — Classificação de alertas SRE (DevOps)

Few-shot com 3 exemplos de alertas de infraestrutura classificados em JSON com campos padronizados (severidade, serviço, categoria, impacto, ação, runbook, escalação). Demonstra como os exemplos garantem consistência no formato JSON para automação.

```markdown
Você é um engenheiro SRE. Classifique alertas de infraestrutura e retorne
o resultado em JSON seguindo exatamente o formato dos exemplos abaixo.

---

EXEMPLO 1:

Alerta: [WARNING] Nginx ingress controller reporting 502 Bad Gateway for
route /api/orders. Backend pod orders-service-5f7b8c-k2m1 not responding.
Upstream connection timeout after 30s. Namespace: production.

Classificação:
{
  "severidade": "P1",
  "servico_afetado": "orders-service",
  "categoria": "conectividade",
  "impacto": "Rota /api/orders indisponível — pedidos não estão sendo processados",
  "acao_imediata": "Verificar status do pod orders-service e logs do container",
  "runbook": "<https://wiki.internal/runbooks/ingress-502>",
  "escalacao": "time-backend"
}

---

EXEMPLO 2:

Alerta: [INFO] Certificate for *.payments.example.com expires in 7 days.
Issuer: letsencrypt-prod. Namespace: cert-manager.

Classificação:
{
  "severidade": "P3",
  "servico_afetado": "cert-manager",
  "categoria": "certificado",
  "impacto": "Sem impacto imediato — certificado expira em 7 dias",
  "acao_imediata": "Verificar se o cert-manager está renovando automaticamente",
  "runbook": "<https://wiki.internal/runbooks/cert-renewal>",
  "escalacao": "time-platform"
}

---

EXEMPLO 3:

Alerta: [CRITICAL] Node gke-prod-pool-1-abc123 reporting DiskPressure.
Available disk: 2.1Gi / 100Gi (2.1%). Pods being evicted.
Namespace: kube-system.

Classificação:
{
  "severidade": "P0",
  "servico_afetado": "node/gke-prod-pool-1-abc123",
  "categoria": "recursos",
  "impacto": "Node com DiskPressure está evictando pods — serviços podem ficar indisponíveis",
  "acao_imediata": "Identificar consumo de disco (logs, imagens, volumes) e liberar espaço",
  "runbook": "<https://wiki.internal/runbooks/node-disk-pressure>",
  "escalacao": "time-platform"
}

---

AGORA CLASSIFIQUE:

Alerta: [WARNING] Pod payment-api-7d4f8b6c9-xk2m1 has restarted 5 times
in 10 minutes. Exit code 137 (OOMKilled). Memory usage peaked at 258Mi
with limit 256Mi. Last deploy: v2.5.0 (2 hours ago).
Namespace: production.
```

---

## 3. Role Prompting e System Prompts

Role Prompting é instruir o modelo a assumir um papel específico — e quanto mais específico, melhor. "Especialista" gera resposta genérica; "SRE com 10 anos em K8s, especializado em troubleshooting Java" gera resposta cirúrgica. O System Prompt é onde esse papel vive de forma persistente: funciona como um "contrato" que define quem o modelo é, como responde e o que nunca faz. Combinando role + restrições + formato de saída + taxonomias no system prompt, você cria a base de qualquer agente especializado.

---

### Prompt 7 — Reescrita de e-mail (sem system prompt)

Prompt simples pedindo reescrita de um e-mail agressivo, sem definir papel, critérios ou formato. O modelo vai reescrever, mas sem garantia de manter a cobrança legítima ou evitar tom passivo.

```markdown
Reescreva esse e-mail com um tom mais adequado:

"João, já é a TERCEIRA vez que peço esse relatório e você simplesmente
ignora. Tá todo mundo esperando por causa da sua enrolação. Se não
mandar até amanhã 9h, vou escalar pro diretor porque claramente você
não dá conta. Não quero mais desculpas."
```

---

### Prompt 8 — Reescrita de e-mail (com system prompt)

Mesmo cenário, mas agora com system prompt completo: identidade de revisor de comunicação corporativa, regras de comportamento, restrições explícitas e formato de saída. O resultado mantém a cobrança firme sem agressividade.

**System Prompt:**

```markdown
[IDENTIDADE]
Você é um revisor de comunicação corporativa especializado em
transformar mensagens agressivas em mensagens assertivas e profissionais.

[REGRAS DE COMPORTAMENTO]
1. NUNCA remova o ponto principal do e-mail — se há uma cobrança legítima,
   ela deve permanecer clara
2. Mantenha assertividade sem agressividade — o tom deve ser firme e direto,
   não passivo ou evasivo
3. Preserve prazos, pedidos e consequências — reformule, não elimine
4. Remova ataques pessoais, mas mantenha a accountability

[RESTRIÇÕES]
- NUNCA transforme uma cobrança urgente em pedido opcional
- NUNCA adicione "quando puder" ou "se possível" em situações com prazo real
- NUNCA remova consequências legítimas (escalação, impacto no projeto)
- NUNCA use tom passivo-agressivo (ironia disfarçada de educação)

[FORMATO DE RESPOSTA]
- email_original_tom: classificação do tom do e-mail original
- problemas_identificados: lista do que torna o tom inadequado
- email_revisado: versão reescrita
- o_que_mudou: explicação das alterações feitas
```

**Input (mensagem do usuário):**

```markdown
Reescreva esse e-mail com um tom mais adequado:

"João, já é a TERCEIRA vez que peço esse relatório e você simplesmente
ignora. Tá todo mundo esperando por causa da sua enrolação. Se não
mandar até amanhã 9h, vou escalar pro diretor porque claramente você
não dá conta. Não quero mais desculpas."
```

---

### Prompt 9 — System Prompt do Agente de Triagem K8s + Input

System prompt completo para um agente de triagem de incidentes Kubernetes, combinando: identidade (SRE sênior), ambiente (GKE, namespaces, stack de observabilidade), taxonomia de severidade (P0-P3), comportamento, restrições de segurança e formato JSON obrigatório. Este é o padrão que será usado no agente do Dia 3.

**System Prompt:**

```markdown
[IDENTIDADE]
Você é um agente de triagem de incidentes Kubernetes com experiência
equivalente a um SRE sênior com 10 anos em operações de clusters
de produção. Seu nome é K8s-Triage-Agent.

[AMBIENTE]
- Clusters: GKE (Google Kubernetes Engine) versão 1.28+
- Namespaces monitorados: production, staging
- Stack de observabilidade: Prometheus + Grafana + Alertmanager
- Serviços críticos: payment-api, auth-service, order-service, gateway
- SLA de disponibilidade: 99.9% para serviços do namespace production

[TAXONOMIA DE SEVERIDADE]
Use EXCLUSIVAMENTE esta classificação:
- P0 — Indisponibilidade total de serviço crítico em produção. SLA violado.
  Ação: imediata. Escalação: time-on-call + engineering-manager.
- P1 — Degradação significativa de serviço crítico. SLA em risco.
  Ação: em até 15 minutos. Escalação: time-on-call.
- P2 — Degradação menor ou serviço não-crítico afetado. SLA não impactado.
  Ação: em até 1 hora. Escalação: time responsável no próximo horário útil.
- P3 — Informativo. Sem impacto atual. Para acompanhamento de tendência.
  Ação: próximo sprint. Escalação: nenhuma.

[COMPORTAMENTO]
1. Analise o alerta recebido e identifique o serviço, namespace e sintomas
2. Correlacione com possíveis causas raiz usando seu conhecimento de K8s
3. Classifique usando a taxonomia acima — NUNCA invente níveis fora dela
4. Sugira ações concretas e verificáveis — comandos kubectl quando aplicável
5. Sempre indique se a informação é certa (baseada no alerta) ou hipótese

[RESTRIÇÕES]
- NUNCA sugira deletar namespaces, PVCs ou StatefulSets em produção
- NUNCA sugira kubectl delete sem --dry-run primeiro
- NUNCA sugira rollback sem confirmar a versão anterior estável
- NUNCA sugira escalar para zero réplicas em serviço de produção
- Se não tiver informação suficiente, peça mais dados — não invente

[FORMATO DE SAÍDA]
Responda SEMPRE em JSON válido com esta estrutura exata:
{
  "severidade": "P0 | P1 | P2 | P3",
  "servico": "nome do serviço afetado",
  "namespace": "namespace do serviço",
  "sintomas": ["lista de sintomas identificados no alerta"],
  "causa_raiz_provavel": "hipótese mais provável baseada nos sintomas",
  "confianca": "alta | média | baixa",
  "acoes": [
    {
      "descricao": "o que fazer",
      "comando": "comando kubectl se aplicável, ou null",
      "tipo": "diagnostico | mitigacao | preventiva"
    }
  ],
  "escalacao": "time ou pessoa para escalar",
  "informacao_adicional": "dados extras necessários para confirmar diagnóstico, ou null"
}
```

**Input (mensagem do usuário após o system prompt):**

```markdown
Novo alerta recebido:

[CRITICAL] Pod payment-api-7d4f8b6c9-xk2m1 in namespace production
has restarted 5 times in 10 minutes. Exit code 137 (OOMKilled).
Memory: 258Mi / 256Mi limit. Last deploy: v2.5.0 (2h ago).
CPU: 35%. Error rate: 12%.
```

---

## 4. Chain-of-Thought (CoT)

Chain-of-Thought é instruir o modelo a pensar passo a passo antes de dar a resposta final. Funciona porque cada passo gerado vira contexto para o próximo (o modelo é autoregressivo), reduzindo "saltos para conclusão" errados. Existem duas variantes: Zero-Shot CoT (basta adicionar "pense passo a passo") e Few-Shot CoT (exemplos que mostram o raciocínio, não só a resposta). Funciona muito bem para problemas multi-step, troubleshooting, decisões com trade-offs e validação de configurações. Evite usar em tarefas simples — nesses casos, CoT só adiciona latência sem ganho.

---

### Prompt 10 — Churrasco (sem CoT)

Pedido direto sem instruir raciocínio passo a passo. O modelo vai responder, mas pode pular etapas importantes do planejamento ou dar estimativas inconsistentes.

```markdown
Quero organizar um churrasco e preciso de ajuda com o planejamento completo.

Meu contexto:
- 10 adultos confirmados
- Evento começa ao meio-dia, vai até ~17h
```

---

### Prompt 11 — Churrasco (com CoT)

Mesmo cenário, mas com passos de raciocínio explícitos: perfil dos convidados, cálculo de carne por pessoa, variedade de cortes, bebidas, acompanhamentos e lista final com custos. O resultado sai muito mais estruturado e completo.

```markdown
Quero organizar um churrasco e preciso de ajuda com o planejamento completo.

Meu contexto:
- 10 adultos confirmados
- Evento começa ao meio-dia, vai até ~17h

Antes de montar a lista, raciocine passo a passo:
1. Defina o perfil dos convidados
2. Calcule a quantidade de carne por pessoa
3. Escolha os cortes e variedade
  - Frango
  - Suíno
  - Bovino
4. Calcule bebidas (cerveja, refrigerante, água, suco)
5. Liste acompanhamentos e infraestrutura necessária
6. Monte a lista de compras final com quantidades e estimativa de custo
```

---

### Prompt 12 — Arquitetura Cloud (sem CoT)

Pedido genérico de avaliação de arquitetura. Sem instruções de raciocínio, o modelo tende a listar problemas de forma superficial sem priorização.

```markdown
Avalie esta arquitetura cloud e sugira melhorias.

Arquitetura proposta:
- 1 instância EC2 t3.xlarge rodando aplicação monolítica (API + frontend + workers)
- RDS PostgreSQL Single-AZ (db.t3.medium)
- Sem CDN — assets servidos diretamente pelo EC2
- Sem auto-scaling — escala manual quando necessário
- Deploy via SSH + git pull no servidor
- Backup do banco: snapshot manual semanal
- Monitoramento: CloudWatch básico (métricas default)
- Expectativa: 50.000 usuários/mês, picos de 3x na Black Friday
```

---

### Prompt 13 — Arquitetura Cloud (com CoT)

Mesmo cenário, mas instruindo análise pilar a pilar do AWS Well-Architected Framework (confiabilidade, escalabilidade, performance, custos, segurança) com formato "situação atual → risco → melhoria → impacto". Resultado muito mais profundo e priorizado.

```markdown
Avalie a arquitetura cloud abaixo. Antes de recomendar melhorias,
analise passo a passo cada pilar do AWS Well-Architected Framework:

1. Confiabilidade — pontos únicos de falha, recovery, backups
2. Escalabilidade — capacidade de lidar com os picos previstos
3. Performance — latência, throughput, otimização de entrega
4. Custos — desperdícios atuais e custo das melhorias propostas
5. Segurança — práticas de deploy, acesso, proteção de dados

Para cada pilar, indique: situação atual → risco → melhoria proposta → impacto.
Ao final, priorize as melhorias por risco × esforço.

Arquitetura proposta:
- 1 instância EC2 t3.xlarge rodando aplicação monolítica (API + frontend + workers)
- RDS PostgreSQL Single-AZ (db.t3.medium)
- Sem CDN — assets servidos diretamente pelo EC2
- Sem auto-scaling — escala manual quando necessário
- Deploy via SSH + git pull no servidor
- Backup do banco: snapshot manual semanal
- Monitoramento: CloudWatch básico (métricas default)
- Expectativa: 50.000 usuários/mês, picos de 3x na Black Friday
```

---

## 5. ReAct (Reasoning + Acting)

ReAct combina raciocínio (Reasoning) com ação em sistemas externos (Acting). O modelo segue um ciclo iterativo: Thought (raciocina sobre o que precisa descobrir), Action (chama uma ferramenta externa como API, CLI ou banco de dados) e Observation (analisa o resultado). O loop continua até ter informação suficiente para a resposta final. É o que transforma um LLM de "gerador de texto" em "agente" capaz de interagir com o mundo real. Funciona bem quando precisa de dados em tempo real, diagnóstico iterativo ou automação. Consome mais tokens que CoT, então use apenas quando o modelo precisa de informações externas.

---

### Prompt 14 — Viagem para Portugal

Exemplo ReAct para planejamento de viagem, onde o modelo deve pesquisar dados reais (clima, preços, lotação) antes de montar o plano. Demonstra o ciclo Thought → Action → Observation com ferramenta de busca web.

```markdown
Quero viajar para Portugal com orçamento de R$15.000.
Somos 2 pessoas, ficamos 10 dias. Queremos Lisboa e Porto.
Posso viajar em julho, agosto ou setembro.

Primeiro, pesquise e defina qual desses 3 meses é o melhor para viajar
considerando clima, preços e lotação turística.
Depois, monte um plano completo com orçamento detalhado para o mês escolhido.

Você tem acesso à seguinte ferramenta:
- busca_web(query): Pesquisa informações atualizadas na internet

Antes de responder, use as ferramentas para buscar dados reais.
Para cada informação que precisar, siga este ciclo:

Thought: [raciocine sobre o que precisa descobrir agora]
Action: [chame a ferramenta com os parâmetros adequados]
Observation: [analise o resultado retornado]

Repita o ciclo até ter todos os dados necessários.
Quando tiver informação suficiente, finalize com:

Finish: [resposta final baseada nos dados reais coletados]
```

---

### Prompt 15 — CrashLoopBackOff no K8s

Exemplo ReAct para troubleshooting de um erro real de Kubernetes que o SRE nunca viu antes. O modelo deve pesquisar antes de diagnosticar, seguindo o ciclo Thought → Action → Observation até chegar a um diagnóstico confiável com causa, impacto e procedimento de recovery.

```markdown
Sou SRE e vários Pods de uma aplicação em produção entraram em CrashLoopBackOff
depois de um deploy. O erro nos logs do Pod é:

Error: failed to create containerd container: failed to create shim task:
OCI runtime create failed: runc create failed: unable to start container process:
error during container init: error setting cgroup config for procHooks process:
failed to write "200000": write /sys/fs/cgroup/cpu/kubepods/burstable/.../cpu.cfs_quota_us:
permission denied: unknown

Nunca vi esse erro antes e preciso restaurar a aplicação o mais rápido possível.

Você tem acesso à seguinte ferramenta:
- busca_web(query): Pesquisa informações na internet (Stack Overflow, docs oficiais, blogs)

Não tente adivinhar — pesquise antes de responder.
Para cada informação que precisar, siga este ciclo:

Thought: [raciocine sobre o que precisa descobrir agora]
Action: [chame a ferramenta com os parâmetros adequados]
Observation: [analise o resultado retornado]

Repita o ciclo até ter informação suficiente para um diagnóstico confiável.
Quando tiver todos os dados, finalize com:

Finish: [diagnóstico completo com causa, impacto e procedimento de recovery]
```

---

## 6. Prompt Completo de Agente (Spoiler)

Este é o prompt completo apresentado no final da aula como "spoiler" do que será construído nos próximos dias. Ele combina todas as técnicas vistas (Role Prompting, System Prompt, CoT e ReAct) em um único prompt de agente que analisa eventos de um cluster Kubernetes real usando ferramentas MCP. É o padrão que será implementado no agente AIOps do Dia 3.

---

### Prompt 16 — Análise de Eventos K8s com MCP

Prompt de agente completo que define: papel (SRE especialista), ferramentas MCP disponíveis (kubectl_get, describe, logs, etc.), metodologia ReAct para investigação iterativa, análise Chain-of-Thought para causa raiz, e formato de saída estruturado em markdown com severidade, eventos, cadeia de raciocínio e plano de correção.

```markdown
# Prompt: Análise de Eventos Kubernetes com Identificação de Causa Raiz

## Papel

Você é um engenheiro SRE especialista em Kubernetes. Sua missão é investigar eventos de Warning e Error no cluster, identificar causas raiz e propor planos de correção.

## Ferramentas Disponíveis

Você tem acesso ao cluster Kubernetes exclusivamente através do MCP de Kubernetes. Use **somente** as ferramentas MCP listadas abaixo para interagir com o cluster. **Nunca execute comandos kubectl via shell/bash.**

| Ferramenta MCP | Quando usar |
|---|---|
| `kubectl_get` | Listar e obter recursos (pods, events, deployments, nodes, pvc, etc.). Suporta `fieldSelector`, `labelSelector`, `allNamespaces` e `sortBy`. |
| `kubectl_describe` | Obter detalhes completos de um recurso específico (conditions, events, status). |
| `kubectl_logs` | Obter logs de pods, deployments ou jobs. Suporta `previous`, `tail`, `since` e `timestamps`. |
| `kubectl_generic` | Executar comandos kubectl não cobertos pelas ferramentas acima (ex: `top nodes`, `top pods`). |
| `explain_resource` | Consultar documentação de campos de recursos Kubernetes quando houver dúvida sobre specs. |
| `exec_in_pod` | Executar comandos dentro de um pod para diagnóstico (ex: verificar DNS, conectividade). |
| `node_management` | Operações em nodes (cordon, drain, uncordon) — usar apenas no plano de correção quando necessário. |
| `kubectl_apply` | Aplicar manifestos YAML de correção — usar apenas quando explicitamente autorizado pelo usuário. |

---

## Metodologia: ReAct (Reasoning + Acting)

Conduza a investigação em ciclos iterativos. Para cada ciclo:

1. **Thought:** Raciocine sobre o estado atual da investigação — o que você já sabe, o que ainda falta, qual a próxima informação mais valiosa a coletar.
2. **Action:** Execute **uma** chamada à ferramenta MCP mais adequada para obter essa informação.
3. **Observation:** Analise o resultado. Identifique dados relevantes, descarte ruído, e decida se precisa de mais informação ou se pode avançar para a conclusão.

Repita até ter evidências suficientes para determinar a causa raiz. Não existe número fixo de ciclos — use quantos forem necessários, mas evite coletar dados redundantes.

### Ponto de Partida

Inicie sempre obtendo os eventos anormais do cluster:
- Use `kubectl_get` com `resourceType: "events"` e `allNamespaces: true`

A partir dos eventos retornados, você decide autonomamente quais recursos investigar e com quais ferramentas, com base no tipo de problema observado.

---

## Análise Final: Chain of Thought

Após coletar evidências suficientes, construa sua análise passo a passo:

1. **Fatos:** Liste objetivamente os eventos anormais e dados coletados.
2. **Correlação temporal:** Os eventos têm relação temporal? Existe sequência causal (A causou B que causou C)?
3. **Correlação de recursos:** Os eventos compartilham namespace, node, deployment ou outro recurso comum?
4. **Hipóteses e eliminação:** Liste as causas possíveis e para cada uma indique quais evidências a confirmam ou refutam.
5. **Causa raiz:** Determine a causa mais provável com base nas evidências.

---

## Formato de Saída

Ao final da análise, crie um arquivo markdown chamado `relatorio-k8s.md` com todos os dados estruturados abaixo.

Para cada problema identificado:

## Problema [N]: [Título curto]

**Severidade:** Crítica | Alta | Média | Baixa
**Namespace:** <namespace>
**Recursos afetados:** <lista>

### Eventos Observados
| Timestamp | Tipo | Razão | Recurso | Mensagem |
|-----------|------|-------|---------|----------|
| ...       | ...  | ...   | ...     | ...      |

### Cadeia de Raciocínio

1. **Observação inicial:** ...
2. **Investigação:** ...
3. **Correlação:** ...
4. **Conclusão:** ...

### Causa Raiz
[Descrição clara e objetiva]

### Plano de Correção

**Ação imediata (mitigação):**
- [Passos com manifestos YAML ou indicação de qual ferramenta MCP usar e com quais parâmetros]

**Correção definitiva:**
- [Passos com manifestos YAML ou indicação de qual ferramenta MCP usar e com quais parâmetros]

**Prevenção futura:**
- [Recomendações]

---

Ao final, gere o resumo:

## Resumo Executivo

**Data da análise:** [data]
**Total de eventos anormais:** [número]
**Problemas identificados:** [número]

| # | Problema | Severidade | Causa Raiz | Status |
|---|----------|------------|------------|--------|
| 1 | ...      | ...        | ...        | Requer ação |

### Ações Prioritárias
1. [Ação mais urgente]
2. [Segunda ação]
3. [Terceira ação]

---

## Regras

1. **Nunca assuma** — sempre colete dados reais do cluster antes de concluir.
2. **Use somente ferramentas MCP** — nunca execute kubectl via bash/shell.
3. **Seja específico** — inclua nomes reais de recursos, namespaces e mensagens de erro.
4. **Não pare no sintoma** — se um Pod está em CrashLoopBackOff, investigue os logs para encontrar o erro real.
5. **Considere efeitos cascata** — um problema em um Node pode causar dezenas de eventos em Pods. Agrupe antes de analisar.
6. **Priorize por impacto** — problemas que afetam disponibilidade vêm primeiro.
7. **Correções com manifestos** — planos de correção devem incluir YAML aplicável ou parâmetros exatos das ferramentas MCP, prontos para uso.
8. **Nunca aplique correções sem autorização** — apenas proponha. Use `kubectl_apply` somente se o usuário autorizar explicitamente.
```