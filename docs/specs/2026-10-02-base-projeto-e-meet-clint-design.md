# Design — Base do projeto + Alerta WhatsApp + Fluxo Meet → Clint

- **Data:** 2026-10-02
- **Status:** aprovado em brainstorming, aguardando revisão da spec
- **Instância n8n:** https://n8n.grupotoda.com.br

## 1. Objetivo

Este projeto é a base para o Claude criar e manter fluxos no n8n sob demanda, em vez de construí-los manualmente. Os tipos de fluxo não são previsíveis; a base precisa ser genérica.

Esta spec cobre três entregas, nesta ordem de construção:

1. **Base do projeto** — estrutura de pastas, regras de trabalho (`CLAUDE.md`), git.
2. **Fluxo "Alerta WhatsApp"** — alerta de erro e de negócio, reutilizado por todos os fluxos.
3. **Fluxo "Anotação Reuniao Meet"** — transcrição do Google Meet → anotações no deal da Clint.

## 2. Base do projeto

### 2.1 Estrutura

```
fluxos n8n/
├── CLAUDE.md                     regras que o Claude segue em toda sessão
├── .mcp.json                     conexão MCP com o n8n (escopo de projeto)
├── .env                          segredos locais (fora do git)
├── .gitignore
├── docs/
│   ├── specs/                    designs aprovados
│   └── referencias/              materiais de referência (imagens, anotações)
└── fluxos/
    └── <slug-do-fluxo>/
        ├── README.md             explicação em português
        ├── workflow.json         cópia exata do fluxo no n8n
        └── prompts/              prompts de IA (quando houver)
```

### 2.2 Fonte da verdade e sincronização

- O fluxo vive no n8n; o repositório guarda uma **cópia exata** (`workflow.json`, obtida via `get_workflow_details`) e um **README**.
- **Criação:** código SDK do n8n → `validate_workflow` → `create_workflow_from_code`. O código SDK não é guardado; a cópia JSON é.
- **Edição:** `update_workflow` (operações pontuais).
- **Antes de editar:** comparar o `versionId` do n8n com o do `workflow.json` local. Se diferirem, alguém editou na interface: puxar a versão do n8n para o repo (e commitar) antes de alterar.
- **Depois de qualquer mudança:** puxar o JSON atualizado, atualizar o README e commitar.

### 2.3 README de cada fluxo

Seções obrigatórias: Objetivo · Link e ID no n8n · Gatilho · Passo a passo · Credenciais usadas (nome e ID, nunca o segredo) · Configuração manual necessária · Como testar · Limitações conhecidas · Histórico de decisões.

### 2.4 Autonomia

Para fluxos novos ou alterados, o Claude segue sem pedir aprovação até o fim:
criar → validar → testar com dados de exemplo → **ativar** se os testes passarem → salvar cópia e README → commit.
Se um teste falhar e não houver correção óbvia, o fluxo fica **inativo** e o Claude reporta o que falhou.

### 2.5 Regras fixas

- Todo fluxo ativo tem `errorWorkflow` = fluxo "Alerta WhatsApp".
- Chamadas HTTP externas e de IA: `retryOnFail`, 3 tentativas, 2 s entre elas.
- Nenhum segredo (token, senha, API key) no repositório. Fluxos referenciam credenciais do n8n por nome/ID. O `.env` local (fora do git) guarda tokens só para consultas do Claude durante o desenvolvimento.
- O MCP não cria credenciais: quando faltar uma, o Claude informa exatamente qual criar e com que tipo.
- Nomes de fluxo em português, descritivos; pasta em `kebab-case`.

### 2.6 Git

Repositório git local nesta pasta. Sem remoto (GitHub) a menos que o usuário peça. Commits no formato `<tipo>: <descrição>`.

## 3. Fluxo "Alerta WhatsApp"

### 3.1 Entradas

1. **Error Trigger** — disparado automaticamente pelo n8n quando um fluxo que o tem como `errorWorkflow` falha em produção.
2. **Execute Workflow Trigger** — chamado por outros fluxos para avisos de negócio. Entrada: `{ titulo: string, mensagem: string, link?: string }`.

### 3.2 Passos

1. **Config** (Set): `evolutionUrl`, `instancia`, `numeroDestino`. Valores pendentes até o usuário informar.
2. **Montar mensagem**:
   - Erro técnico:
     ```
     ⚠️ Falha no n8n
     Fluxo: {nome do fluxo}
     Passo: {nó que falhou}
     Erro: {mensagem}
     Quando: {dd/MM HH:mm, America/Sao_Paulo}
     Ver execução: {url}
     ```
   - Aviso de negócio: `ℹ️ {titulo}\n{mensagem}\n{link}`.
3. **Anti-spam**: chave = `workflowId + mensagem de erro` (ou `titulo` no aviso). Se a mesma chave foi enviada há menos de **30 min**, não envia. Estado guardado em `$getWorkflowStaticData('global')`.
4. **Enviar** (HTTP Request): `POST {evolutionUrl}/message/sendText/{instancia}` com header `apikey` vindo de credencial `httpHeaderAuth` do n8n; corpo `{ number, text }`.

### 3.3 Pendências do usuário

- URL da Evolution, nome da instância, número de destino.
- Criar (ou indicar) a credencial `httpHeaderAuth` com a `apikey` da Evolution.

### 3.4 Teste

Enviar uma mensagem real de aviso de negócio para o número de destino e confirmar o recebimento com o usuário. Até as pendências serem resolvidas, o fluxo é criado mas não testado.

## 4. Fluxo "Anotação Reuniao Meet" (Meet → Clint)

- **Workflow existente:** `Owu9NI5ScaU5zZfA` — https://n8n.grupotoda.com.br/workflow/Owu9NI5ScaU5zZfA
- Hoje contém só o gatilho do Google Drive. O fluxo será construído **dentro dele** (mesmo ID).
- **Contexto:** reuniões comerciais de "INSS de obras", agendadas na agenda da Sara. Padrão do evento (referência: `docs/referencias/evento-agenda.png`):
  - Título: `INSS de obras: {Nome do lead} - {telefone com DDI} {texto livre}` (ex.: `INSS de obras: Cristiana Aparecida - 5531988302388 CRED ARNALDO`).
  - Descrição: contém o link do deal, `https://app.clint.digital/deal/{dealId UUID}`.
  - Anexos: gravação de vídeo e documento Google Docs do Gemini, com nome iniciado pelo título do evento.

### 4.1 Credenciais

| Uso | Credencial n8n | ID |
|---|---|---|
| Google Drive (gatilho e leitura) | `sara` (googleDriveOAuth2Api) | `Xz7XkQuj0KDkVtW5` |
| Google Agenda da Sara | `Google Calendar account` (googleCalendarOAuth2Api) | `dVWPfETb214FyT82` |
| Clint API (header `api-token`) | `Header Clint` (httpHeaderAuth) | `qiMe1q95EEa3L2HI` |
| IA | `grupotoda` (openAiApi) | `ynBZoBP7TIG7AJAZ` |

### 4.2 Passos

1. **Gatilho Google Drive** — pasta `12LPKim0euEzdnAkPSorKccTx-6seq7Dk`, a cada minuto, evento **arquivo criado** (hoje está "pasta criada"; corrigir).
2. **Filtrar** — aceitar apenas documentos Google Docs (ignora vídeo). Antes de construir, inspecionar um arquivo real da pasta para decidir se o documento usado é a "Transcrição", as "Anotações do Gemini" ou ambos. Regra: **uma execução útil por reunião** (o outro documento, se existir, é ignorado).
3. **Achar o evento** — extrair o título da reunião do nome do arquivo (removendo sufixos do Meet: data/hora, "Transcrição", "Anotações do Gemini") e buscar na agenda da Sara o evento com esse título no dia de criação do arquivo.
4. **Identificar o deal**:
   - **Principal:** extrair o UUID do link `app.clint.digital/deal/{uuid}` da descrição do evento → `GET https://api.clint.digital/v1/deals/{uuid}`.
   - **Plano B** (evento não encontrado, sem link ou deal inexistente): telefone = sequência de ≥ 10 dígitos do título → `GET /v1/deals?phone={telefone}&status=OPEN`; se vazio, repetir sem o DDI `55`; se mais de um, usar o de atualização mais recente.
   - **Nenhum deal** → aviso de negócio "Lead não encontrado na Clint" com título e link do documento; fim.
   - Nome do lead = parte do título entre `INSS de obras:` (ou `INSS de obra`) e o telefone; se não casar, usar o nome do contato do deal.
5. **Ler documento** — download do Google Doc exportado como `text/plain`.
6. **Extrair com IA** (OpenAI, saída JSON estruturada; prompt em `fluxos/meet-para-clint/prompts/extracao.md`):

   | Chave | Conteúdo |
   |---|---|
   | `tipo_mao_de_obra` | Tipo de mão de obra que o cliente usou (texto livre) |
   | `consegue_recibo` | `Sim` / `Não` / `Não mencionado` + detalhe curto |
   | `negociacao_valores` | Valores discutidos |
   | `negociacao_forma_pagamento` | Forma de pagamento discutida |
   | `objecao_decisao` | Desculpa/objeção para a tomada de decisão (ex.: "precisa falar com a esposa") |
   | `dia_retorno` | Data e hora combinadas para o retorno, em texto `dd/MM/yyyy HH:mm` (ou só data), ou `Não mencionado` |
   | `resumo` | Resumo da reunião em 5 a 8 linhas |

   Regras do prompt: não inventar; o que não foi dito é `Não mencionado`. Datas relativas ("semana que vem", "sexta") são resolvidas a partir da data da reunião.
7. **Montar o bloco de texto**:
   ```
   📅 Reunião {dd/MM/yyyy}
   • Mão de obra: {tipo_mao_de_obra}
   • Recibo: {consegue_recibo}
   • Valores: {negociacao_valores}
   • Pagamento: {negociacao_forma_pagamento}
   • Objeção: {objecao_decisao}
   • Retorno: {dia_retorno}
   Resumo: {resumo}
   Transcrição: {link do documento}
   ```
8. **Gravar no deal** — ler o valor atual do campo "Anotações da reunião" (já obtido no passo 4) e gravar `POST /v1/deals/{id}` com **bloco novo + linha separadora + texto anterior** (reunião mais nova no topo). Se o mesmo documento já estiver no texto (link da transcrição presente), não grava de novo (evita duplicar em reprocessamento).

Nenhuma tarefa/atividade é criada na Clint.

### 4.3 Configuração manual na Clint (usuário, uma vez)

Criar no **deal** 1 campo personalizado do tipo **RICH_TEXT** (texto rico): **Anotações da reunião**. Campos RICH_TEXT guardam HTML; o bloco do passo 7 é gravado em HTML (`<p>`, `<strong>`, `<br>`, `<a>`), com o mesmo conteúdo.
Depois, o Claude lê a chave interna via `GET /v1/account/fields` e a usa no passo 8.

### 4.4 Erros

- Clint, Google e OpenAI: 3 tentativas, 2 s entre elas.
- Falha técnica → fluxo "Alerta WhatsApp" (via `errorWorkflow`).
- Caso de negócio (lead não encontrado) → aviso de negócio via sub-workflow; a execução termina com sucesso.

### 4.5 Teste

O gatilho do Google Drive não pode ser executado pelo MCP. Teste:
1. Usar uma transcrição de exemplo (real ou fictícia) como dado fixado (pin data) no gatilho/leitura.
2. Rodar contra um **deal de teste** na Clint (indicado pelo usuário), com evento de agenda contendo o link do deal.
3. Verificar no deal: campo "Anotações da reunião" com o bloco no formato do passo 7.
4. Testar os ramos: evento sem link (plano B por telefone); lead não encontrado; segunda reunião no mesmo deal (bloco novo no topo, anterior preservado); reprocessamento do mesmo documento (não duplica).
5. Passando, ativar o fluxo.

### 4.6 Limitações conhecidas

- O dia de retorno fica só no texto; o vendedor não recebe lembrete (decisão do usuário: não criar tarefa).
- A API da Clint não envia anexos; o link do documento vai no texto.
- Polling de 1 minuto: a atualização chega até ~1 min depois de o Meet salvar o documento.
- O campo cresce a cada reunião; não há limite de histórico (revisar se ficar grande demais).

## 5. Pendências antes/durante a implementação

| Pendência | Responsável | Bloqueia |
|---|---|---|
| Dados da Evolution + credencial `apikey` | Usuário | Teste do Alerta WhatsApp |
| Criar o campo "Anotações da reunião" no deal da Clint | Usuário | Passo 8 do fluxo Meet |
| Indicar um deal de teste (e evento de teste) | Usuário | Teste do fluxo Meet |
| Inspecionar arquivos reais da pasta do Meet | Claude | Passo 2 do fluxo Meet |
