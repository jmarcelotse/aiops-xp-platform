## Fluxo de Interação

```mermaid
flowchart LR
    U["👤 Usuário"] --> P["📝 Prompt"]
    P --> L["🧠 LLM"]
    L --> R["💬 Resposta"]
    R --> U

    classDef user fill:#6b2d2d,stroke:#d9d9d9,stroke-width:2px,color:#fff,rx:12,ry:12;
    classDef prompt fill:#0f2b36,stroke:#d9d9d9,stroke-width:2px,color:#fff,rx:12,ry:12;
    classDef llm fill:#2b1d0f,stroke:#d9d9d9,stroke-width:2px,color:#fff,rx:12,ry:12;
    classDef response fill:#0f2f1a,stroke:#d9d9d9,stroke-width:2px,color:#fff,rx:12,ry:12;

    class U user;
    class P prompt;
    class L llm;
    class R response;