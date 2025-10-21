# 🤖 Automação de Log de Repositório: GitHub para Notion

Este documento fornece um guia detalhado para a configuração de um sistema de log automatizado que rastreia atividades de desenvolvimento (commits, Pull Requests) neste repositório e as registra em um banco de dados centralizado no Notion.

## Visão Geral

Este projeto utiliza **GitHub Actions** para "escutar" eventos-chave que ocorrem no repositório. Quando um evento detectado (como a abertura de um Pull Request) corresponde às nossas regras, a automação executa um script que envia dados formatados para a API do Notion, criando uma nova entrada (linha) em um banco de dados específico.

O objetivo é criar um "log de atividade" imutável e centralizado, visível para toda a equipe, que responde a perguntas como:
* "Quais Pull Requests estão abertos?"
* "O que foi mesclado hoje?"
* "Alguém fez um commit direto na `main` sem um PR?"

## Funcionalidades Rastreáveis

A automação está configurada para rastrear e registrar os seguintes eventos:

* **Pull Request Aberto:** Registra um novo PR no Notion com o status "PR Aberto" assim que ele é criado.
* **Pull Request Mesclado (Merged):** Registra o merge com o status "PR Mesclado" no momento em que ele acontece.
* **Pull Request Fechado (Sem Merge):** Se um PR for fechado e *não* mesclado, ele é registrado como "PR Fechado".
* **Push Direto (Commit Direto):** Monitora branches protegidos (`main`, `master`, `develop`). Se um commit for enviado diretamente para um desses branches (contornando o processo de PR), ele é registrado com o status "Push Direto", sinalizando uma possível quebra de fluxo.
* **Reversão (Revert):** Como uma reversão (via botão "Revert" do GitHub ou `git revert`) é tratada como um novo PR ou um novo push, ela também é capturada e registrada.

---

## 🛠️ Guia de Configuração Detalhado

Para replicar esta automação, você precisa configurar três componentes: O Notion (o destino), o GitHub Secrets (as chaves) e o GitHub Actions (o motor).

### Parte 1: Configuração do Notion (O Destino)

O Notion receberá os dados. Precisamos criar o banco de dados e uma "porta de entrada" segura (a Integração).

#### 1.1. Criar a Integração (API Key)
A integração é o "robô" que tem permissão para escrever no seu Notion.

1.  Vá para a página de integrações do Notion: **[https://www.notion.so/my-integrations](https://www.notion.so/my-integrations)**.
2.  Clique no botão **"+ New integration"** (Nova integração).
3.  Dê um nome a ela (ex: `GitHub Action Logger`).
4.  Selecione o Workspace correto.
5.  Clique em "Submit".
6.  Na próxima tela, clique em "Show" e copie o **Internal Integration Token** (Token de Integração Interna). Ele começa com `secret_...`.
7.  **Guarde este token.** Ele será o nosso `NOTION_API_KEY`.

#### 1.2. Criar o Banco de Dados (Database ID)
Este é o banco de dados que armazenará o log.

1.  Em uma página do Notion, digite `/table` e escolha "Table - Full page" para criar uma nova base de dados.
2.  Dê o nome de **"Log de Atividade do Repositório"**.
3.  **Obtenha o ID da Base de Dados:**
    * Abra o banco de dados em tela cheia. A URL no seu navegador será algo como:
        `https://www.notion.so/username/`**`a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`**`?v=...`
    * O ID é a longa sequência de caracteres entre o `/` e o `?`.
    * **Guarde este ID.** Ele será o nosso `NOTION_DATABASE_ID`.

#### 1.3. Definir a Estrutura (Colunas)
O script da automação espera encontrar nomes de colunas e tipos **exatos**. Crie as seguintes colunas no seu banco de dados (é sensível a maiúsculas/minúsculas):

| Propriedade (Coluna) | Tipo de Propriedade |
| :--- | :--- |
| **Atividade** | `Title` (Título) |
| **Status** | `Status` |
| **Tipo** | `Select` (Seleção) |
| **Autor** | `Rich Text` (Texto) |
| **Data** | `Date` (Data) |
| **Branch** | `Rich Text` (Texto) |
| **Link** | `URL` (URL) |
| **Destino** | `Rich Text` (Texto) |
| **Descrição** | `Rich Text` (Texto) |

* **Importante:** Na coluna `Status`, adicione as opções (com os nomes exatos): `PR Aberto`, `PR Mesclado`, `PR Fechado` e `Push Direto`.
* **Importante:** Na coluna `Tipo`, adicione as opções: `Pull Request` e `Push`.

#### 1.4. Conectar a Integração ao Banco
**Este é o passo mais comum de falha.** O seu "robô" (Integração) precisa de permissão para escrever na sua "tabela" (Banco de Dados).

1.  Abra seu banco de dados "Log de Atividade do Repositório".
2.  Clique no menu `...` no canto superior direito.
3.  Clique em **"+ Add connections"** (Adicionar conexões).
4.  Encontre e selecione a integração que você criou no passo 1.1 (ex: `GitHub Action Logger`).
5.  Clique em "Confirm".

---

### Parte 2: Configuração do GitHub (O Gatilho)

Agora, vamos configurar o GitHub para que ele saiba como falar com o Notion.

#### 2.1. Adicionar os Secrets do Repositório
Nunca devemos colar chaves de API diretamente no código. Usamos os "Secrets" do GitHub, que são criptografados e seguros.

1.  No seu repositório do GitHub, vá para **Settings** (Configurações).
2.  No menu à esquerda, vá para **Secrets and variables** > **Actions**.
3.  Clique no botão **"New repository secret"** e crie os dois secrets a seguir:
    * **Nome:** `NOTION_API_KEY`
        * **Valor:** Cole o Token da API que você copiou no passo 1.1 (o que começa com `secret_...`).
    * **Nome:** `NOTION_DATABASE_ID`
        * **Valor:** Cole o ID da Base de Dados que você copiou no passo 1.2 (a sequência longa da URL).

#### 2.2. Criar o Arquivo de Workflow (.yml)
Este é o "roteiro" da automação.

1.  No seu repositório, vá para a aba **"< > Code"** (Código).
2.  Clique em **"Add file"** > **"Create new file"**.
3.  No campo de nome do arquivo, digite o caminho exato: `.github/workflows/notion-log.yml`
    *(O GitHub criará as pastas `.github` e `workflows` automaticamente)*.
4.  No editor de texto, cole o código completo da seção abaixo.

---

### 📜 O Código Completo do Workflow (`notion-log.yml`)

Este é o código de "boa prática" que equilibra o rastreamento (logs de PRs para *qualquer* destino) com a redução de ruído (logs de push *apenas* para branches protegidos).

```yaml
name: Sincronizar com Log de Atividade

# GATILHOS (Quando deve rodar)
on:
  # 1. GATILHO DE PUSH (COMMIT DIRETO):
  # Rastrear pushes diretos APENAS em branches protegidos
  push:
    branches: 
      - main
      - master
      - develop

  # 2. GATILHO DE PULL REQUEST:
  # Rastrear PRs abertos/fechados para QUALQUER branch de destino
  pull_request:
    types: [opened, closed] 

# TRABALHOS (O que deve fazer)
jobs:
  sync-to-notion:
    runs-on: ubuntu-latest
    steps:

      # --- TRABALHO 1: Se o evento for um PUSH ---
      - name: Enviar Push para o Notion
        if: github.event_name == 'push'
        run: |
          COMMIT_MSG=$(echo "${{ github.event.head_commit.message }}" | head -n 1)
          
          # A flag -f faz o script falhar (dar erro ❌) se o Notion retornar um erro
          curl -f -X POST [https://api.notion.com/v1/pages](https://api.notion.com/v1/pages) \
          -H "Authorization: Bearer ${{ secrets.NOTION_API_KEY }}" \
          -H "Content-Type: application/json" \
          -H "Notion-Version: 2022-06-28" \
          --data '{
            "parent": { "database_id": "${{ secrets.NOTION_DATABASE_ID }}" },
            "properties": {
              "Atividade": { "title": [{ "text": { "content": "Push: ${{ env.COMMIT_MSG }}" } }] },
              "Status": { "status": { "name": "Push Direto" } },
              "Tipo": { "select": { "name": "Push" } },
              "Autor": { "rich_text": [{ "text": { "content": "${{ github.event.pusher.name }}" } }] },
              "Data": { "date": { "start": "${{ github.event.head_commit.timestamp }}" } },
              "Branch": { "rich_text": [{ "text": { "content": "${{ github.ref_name }}" } }] },
              "Link": { "url": "${{ github.event.compare }}" },
              "Destino": { "rich_text": [{ "text": { "content": "${{ github.ref_name }}" } }] }
            }
          }'

      # --- TRABALHO 2: Se o evento for um PULL REQUEST (ABERTO) ---
      - name: Enviar PR Aberto para o Notion
        if: github.event_name == 'pull_request' && github.event.action == 'opened'
        run: |
          PR_BODY=$(echo "${{ github.event.pull_request.body || 'Sem descrição.' }}" | tr -d '"' | tr -d "'" | tr -d '\n' | tr -d '\r' | head -c 1000)

          curl -f -X POST [https://api.notion.com/v1/pages](https://api.notion.com/v1/pages) \
          -H "Authorization: Bearer ${{ secrets.NOTION_API_KEY }}" \
          -H "Content-Type: application/json" \
          -H "Notion-Version: 2022-06-28" \
          --data '{
            "parent": { "database_id": "${{ secrets.NOTION_DATABASE_ID }}" },
            "properties": {
              "Atividade": { "title": [{ "text": { "content": "PR: ${{ github.event.pull_request.title }}" } }] },
              "Status": { "status": { "name": "PR Aberto" } },
              "Tipo": { "select": { "name": "Pull Request" } },
              "Autor": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.user.login }}" } }] },
              "Data": { "date": { "start": "${{ github.event.pull_request.created_at }}" } },
              "Branch": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.head.ref }}" } }] },
              "Link": { "url": "${{ github.event.pull_request.html_url }}" },
              "Destino": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.base.ref }}" } }] },
              "Descrição": { "rich_text": [{ "text": { "content": "${{ env.PR_BODY }}" } }] }
            }
          }'

      # --- TRABALHO 3: Se o evento for um PULL REQUEST (MESCLADO) ---
      - name: Enviar PR Mesclado para o Notion
        if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true
        run: |
          curl -f -X POST [https://api.notion.com/v1/pages](https://api.notion.com/v1/pages) \
          -H "Authorization: Bearer ${{ secrets.NOTION_API_KEY }}" \
          -H "Content-Type: application/json" \
          -H "Notion-Version: 2022-06-28" \
          --data '{
            "parent": { "database_id": "${{ secrets.NOTION_DATABASE_ID }}" },
            "properties": {
              "Atividade": { "title": [{ "text": { "content": "PR Mesclado: ${{ github.event.pull_request.title }}" } }] },
              "Status": { "status": { "name": "PR Mesclado" } },
              "Tipo": { "select": { "name": "Pull Request" } },
              "Autor": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.user.login }}" } }] },
              "Data": { "date": { "start": "${{ github.event.pull_request.closed_at }}" } },
              "Branch": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.head.ref }}" } }] },
              "Link": { "url": "${{ github.event.pull_request.html_url }}" },
              "Destino": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.base.ref }}" } }] }
            }
          }'

      # --- TRABALHO 4: Se o evento for um PULL REQUEST (FECHADO SEM MERGE) ---
      - name: Enviar PR Fechado para o Notion
        if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == false
        run: |
          curl -f -X POST [https://api.notion.com/v1/pages](https://api.notion.com/v1/pages) \
          -H "Authorization: Bearer ${{ secrets.NOTION_API_KEY }}" \
          -H "Content-Type: application/json" \
          -H "Notion-Version: 2022-06-28" \
          --data '{
            "parent": { "database_id": "${{ secrets.NOTION_DATABASE_ID }}" },
            "properties": {
              "Atividade": { "title": [{ "text": { "content": "PR Fechado: ${{ github.event.pull_request.title }}" } }] },
              "Status": { "status": { "name": "PR Fechado" } },
              "Tipo": { "select": { "name": "Pull Request" } },
              "Autor": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.user.login }}" } }] },
              "Data": { "date": { "start": "${{ github.event.pull_request.closed_at }}" } },
              "Branch": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.head.ref }}" } }] },
              "Link": { "url": "${{ github.event.pull_request.html_url }}" },
              "Destino": { "rich_text": [{ "text": { "content": "${{ github.event.pull_request.base.ref }}" } }] }
            }
          }'
