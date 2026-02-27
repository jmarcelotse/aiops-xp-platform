Estrutura Base
 - Função: Quem o modelo é
 - Tarefa: O que deve fazer
 - Contexto: informações do seu ambiente
 - Saida: como entregar a resposta

## Estrutura do Prompt

```mermaid
flowchart TB
    F(🛡️ Função)
    T(🎯 Tarefa)
    C(🌎 Contexto)
    S(📄 Saída)

    F --> T --> C --> S

    classDef func fill:#0d2530,stroke:#59a8ff,stroke-width:3px,color:#ffffff;
    classDef task fill:#0f2b17,stroke:#58d26a,stroke-width:3px,color:#ffffff;
    classDef context fill:#2b2030,stroke:#e0a6ff,stroke-width:3px,color:#ffffff;
    classDef output fill:#2b1808,stroke:#ff8a1c,stroke-width:3px,color:#ffffff;

    class F func;
    class T task;
    class C context;
    class S output;

    linkStyle 0 stroke:#7a7a7a,stroke-width:1.5px;
    linkStyle 1 stroke:#7a7a7a,stroke-width:1.5px;
    linkStyle 2 stroke:#7a7a7a,stroke-width:1.5px;
```

## Ciclo Thought, Action e Observation

```mermaid
flowchart TB
    T["🤔 Thought"]

    subgraph row[" "]
        direction LR
        A["⚡ Action"]
        O["👁️ Observation"]
    end

    T --> A
    A --> O
    O --> T

    classDef thought fill:#0d2530,stroke:#59a8ff,stroke-width:3px,color:#ffffff;
    classDef action fill:#2b1808,stroke:#ff8a1c,stroke-width:3px,color:#ffffff;
    classDef observation fill:#0f2b17,stroke:#58d26a,stroke-width:3px,color:#ffffff;

    class T thought;
    class A action;
    class O observation;

    style row fill:transparent,stroke:transparent
    linkStyle 0 stroke:#d9d9d9,stroke-width:2px;
    linkStyle 1 stroke:#d9d9d9,stroke-width:2px;
    linkStyle 2 stroke:#d9d9d9,stroke-width:2px;
```
```mermaid
flowchart TB
    P["🧠 Plan"]
    T["🔧 Tool Use"]
    O["👁️ Observation"]

    P -->|Define ação| T
    T -->|Retorna resultado| O
    O -->|Reavalia e replaneja| P

    classDef plan fill:#0d2530,stroke:#59a8ff,stroke-width:3px,color:#ffffff;
    classDef tool fill:#2b1808,stroke:#ff8a1c,stroke-width:3px,color:#ffffff;
    classDef obs fill:#0f2b17,stroke:#58d26a,stroke-width:3px,color:#ffffff;

    class P plan;
    class T tool;
    class O obs;
```
## Loop Agentic com Tool Use

```mermaid
flowchart TB
    P["🧠 Plan"]
    T["🔧 Tool Use"]
    O["👁️ Observation"]

    P -->|Define ação| T
    T -->|Retorna resultado| O
    O -->|Reavalia e replaneja| P

    classDef plan fill:#0d2530,stroke:#59a8ff,stroke-width:3px,color:#ffffff;
    classDef tool fill:#2b1808,stroke:#ff8a1c,stroke-width:3px,color:#ffffff;
    classDef obs fill:#0f2b17,stroke:#58d26a,stroke-width:3px,color:#ffffff;

    class P plan;
    class T tool;
    class O obs;

    linkStyle 0 stroke:#d9d9d9,stroke-width:2px;
    linkStyle 1 stroke:#d9d9d9,stroke-width:2px;
    linkStyle 2 stroke:#d9d9d9,stroke-width:2px;
```

Analise o projeto /home/jmarcelotse/devops/aiops-xp-platform/encontros-tech e me diga qual é a stack do projeto. Como faço para executar localmente e quais sao os principais recursos

Detalhe mais o modelo da dados da aplicação