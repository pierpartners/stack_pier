# n8n Dependency Visualizer

Visualizador interativo de dependências entre workflows n8n e suas fontes de dados (Supabase, Notion, BigQuery, Google, Microsoft, OpenAI).

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub Actions                               │
│  (n8n-visualizer.yml) - Roda diariamente ou manualmente             │
│                                                                      │
│  1. Executa n8n_export_workflows.py                                  │
│  2. Copia {index.html, style.css, app.js} + n8n_data.json para /public│
│  3. Deploy no GitHub Pages                                           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    n8n_export_workflows.py                           │
│                                                                      │
│  - Conecta na API do n8n (N8N_BASE_URL + N8N_API_KEY)               │
│  - Lista todos os workflows ativos                                   │
│  - Baixa detalhes de cada workflow                                   │
│  - Salva arquivos individuais em n8n_workflows_export/               │
│  - Gera n8n_workflows_export/n8n_data.json (bundle completo)        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  index.html + style.css + app.js                     │
│                                                                      │
│  Ao carregar:                                                        │
│  1. app.js tenta fetch('n8n_data.json')                            │
│  2. Se encontrar, processa os workflows automaticamente              │
│  3. Se não encontrar, solicita o GitHub PAT para atualizar/gerar     │
│                                                                      │
│  Estrutura Modular:                                                  │
│  - style.css: Toda a parte visual e skins do grafo                  │
│  - app.js: Lógica de processamento e D3.js                          │
└─────────────────────────────────────────────────────────────────────┘
```

## Fluxo de Dados

### 1. Extração (Python)

O script `n8n_export_workflows.py`:

```python
# Conecta na API do n8n
N8N_BASE_URL    # ex: https://n8n.seudominio.com
N8N_API_KEY     # API Key do n8n

# Exporta para:
# - n8n_workflows_export/{nome_workflow}.json (individual)
# - n8n_workflows_export/n8n_data.json (bundle para o visualizador)
```

### 2. Deploy (GitHub Actions)

O workflow `.github/workflows/n8n-visualizer.yml`:

- **Trigger**: Diariamente à meia-noite OU manualmente
- **Secrets necessários**: `N8N_BASE_URL`, `N8N_API_KEY`
- **Resultado**: GitHub Pages com index.html + n8n_data.json

O arquivo `index.html` (com lógica em `app.js`):

```javascript
// Auto-carregamento (dentro de app.js)
async function tryAutoLoad() {
    const response = await fetch('n8n_data.json');
    const data = await response.json();
    processWorkflows(data);
}
```

## Formato do n8n_data.json

```json
[
  {
    "id": "abc123",
    "name": "Meu Workflow",
    "nodes": [
      {
        "type": "n8n-nodes-base.supabase",
        "parameters": {
          "tableName": { "value": "minha_tabela" }
        },
        "credentials": {
          "supabaseApi": { "name": "Supabase Prod" }
        }
      }
    ]
  }
]
```

## Entidades Detectadas

| Tipo       | Cor       | Detecção no node.type                    |
|------------|-----------|------------------------------------------|
| Workflow   | Vermelho  | (nó principal)                           |
| Supabase   | Verde     | `supabase`, `vectorStoreSupabase`        |
| Notion     | Roxo      | `notion`                                 |
| BigQuery   | Azul      | `bigquery`                               |
| Microsoft  | Laranja   | `microsoftOutlook`, SharePoint via HTTP  |
| Google     | Magenta   | `googleCalendar`                         |
| OpenAI     | Branco    | `openai`, `OpenAi`                       |

## Funcionalidades do Visualizador

- **Agrupamento**: Clique na legenda para expandir/colapsar grupos
- **Busca**: Campo de texto filtra nós por nome
- **Análise de Impacto**: Clique em uma fonte para ver workflows afetados
- **Exportar MD**: Gera documentação em Markdown
- **Zoom/Pan**: Mouse scroll + arrastar

## Configuração Local

1. Crie um arquivo `.env`:
```
N8N_BASE_URL=https://seu-n8n.com
N8N_API_KEY=sua-api-key
```

2. Execute o script:
```bash
pip install -r requirements.txt
python n8n_export_workflows.py
```

3. Abra `index.html` no navegador

## Configuração GitHub Actions

1. Adicione os secrets no repositório:
   - `N8N_BASE_URL`
   - `N8N_API_KEY`

2. O workflow roda automaticamente ou via "Run workflow" na aba Actions

3. Acesse via GitHub Pages: `https://{usuario}.github.io/{repo}/`
