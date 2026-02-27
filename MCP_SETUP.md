# Configuração de Servidores MCP para Kiro CLI

## O que é MCP?

Model Context Protocol (MCP) é um protocolo que permite ao Kiro CLI se comunicar com servidores externos para estender suas capacidades com ferramentas adicionais.

## Onde Configurar

Os servidores MCP podem ser configurados em dois níveis:

### 1. Configuração Global
- **Localização**: `~/.config/kiro-cli/mcp_config.json`
- **Escopo**: Disponível em todos os projetos
- **Uso**: Para servidores que você usa frequentemente em qualquer projeto

### 2. Configuração de Workspace (Recomendado)
- **Localização**: `.kiro/settings/mcp.json` (na raiz do projeto)
- **Escopo**: Disponível apenas neste projeto
- **Uso**: Para servidores específicos do projeto
- **Vantagem**: Pode ser versionado no Git para compartilhar com a equipe

## Como Adicionar um Servidor MCP

### Método 1: Usando o CLI (Recomendado)

```bash
# Adicionar ao workspace atual
kiro-cli mcp add --name <nome> --command <comando> --args '<arg1>' --args '<arg2>'

# Exemplo: Kubernetes MCP
kiro-cli mcp add --name kubernetes --command npx --args '-y' --args 'kubernetes-mcp-server'
```

### Método 2: Editando o arquivo JSON manualmente

Edite `.kiro/settings/mcp.json` ou `~/.config/kiro-cli/mcp_config.json`:

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "npx",
      "args": ["-y", "kubernetes-mcp-server"]
    }
  }
}
```

## Comandos Úteis

```bash
# Listar servidores configurados
kiro-cli mcp list

# Ver status de um servidor
kiro-cli mcp status --name <nome>

# Remover um servidor
kiro-cli mcp remove --name <nome>

# Importar configuração de outro arquivo
kiro-cli mcp import <caminho-do-arquivo>
```

## Ativando as Mudanças

Após adicionar ou modificar servidores MCP:

1. Saia do Kiro CLI com `/quit`
2. Inicie novamente com `kiro-cli chat`
3. As novas ferramentas estarão disponíveis

## Servidores MCP Populares

- **kubernetes-mcp-server**: Gerenciamento de clusters Kubernetes
- **@modelcontextprotocol/server-filesystem**: Operações avançadas de sistema de arquivos
- **@modelcontextprotocol/server-github**: Integração com GitHub
- **@modelcontextprotocol/server-postgres**: Acesso a bancos PostgreSQL

## Verificando se Funcionou

Após reiniciar o Kiro CLI, execute:

```bash
kiro-cli mcp list
```

O servidor deve aparecer listado em um dos agentes (default ou workspace).
