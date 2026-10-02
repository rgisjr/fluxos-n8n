# Design — Base do projeto + Alerta WhatsApp + Fluxo Meet → Clint

- **Data:** 2026-10-02
- **Status:** aprovado em brainstorming, aguardando revisão da spec
- **Instância n8n:** https://n8n.grupotoda.com.br

## 1. Objetivo

Este projeto é a base para o Claude criar e manter fluxos no n8n sob demanda, em vez de construí-los manualmente. Os tipos de fluxo não são previsíveis; a base precisa ser genérica.

Esta spec cobre três entregas, nesta ordem de construção:

1. **Base do projeto** — estrutura de pastas, regras de trabalho (`CLAUDE.md`), git.
2. **Fluxo "Alerta WhatsApp"** — alerta de erro e de negócio, reutilizado por todos os fluxos.
3. **Fluxo "Anotação Reuniao Meet"** — transcrição do Google Meet → informações no deal da Clint.

## 2. Base do projeto

### 2.1 Estrutura

```
fluxos n8n/
├── CLAUDE.md                     regras que o Claude segue em toda sessão
├── .mcp.json                     conexão MCP com o n8n (escopo de projeto)
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
- Nenhum segredo (token, senha, API key) no repositório. Fluxos referenciam credenciais do n8n por nome/ID.
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

### 4.1 Credenciais

| Uso | Credencial n8n | ID |
|---|---|---|
| Google Drive (gatilho e leitura) | `sara` (googleDriveOAuth2Api) | `Xz7XkQuj0KDkVtW5` |
| Clint API (header `api-token`) | `Header Clint` (httpHeaderAuth) | `qiMe1q95EEa3L2HI` |
| IA | `grupotoda` (openAiApi) | `ynBZoBP7TIG7AJAZ` |

### 4.2 Passos

1. **Gatilho Google Drive** — pasta `12LPKim0euEzdnAkPSorKccTx-6seq7Dk`, a cada minuto, evento **arquivo criado** (hoje está "pasta criada"; corrigir).
2. **Filtrar** — aceitar apenas o documento Google Docs da reunião. Antes de construir, inspecionar um arquivo real da pasta para decidir se o documento usado é a "Transcrição", as "Anotações do Gemini" ou ambos. Regra: **uma execução útil por reunião** (o outro documento, se existir, é ignorado).
3. **Extrair do título** — o título da reunião contém nome e telefone do lead. Telefone = sequência de dígitos (≥ 10). Nome = restante do título sem o telefone, separadores e sufixos do Meet (data/hora, "Transcrição", "Anotações do Gemini").
   - Sem telefone no título → aviso de negócio "Reunião sem telefone no título" e fim.
4. **Buscar deal** — `GET https://api.clint.digital/v1/deals?phone={telefone}&status=OPEN`.
   - Nenhum resultado → repetir sem o DDI `55`.
   - Mais de um → usar o de `updated_at` mais recente.
   - Nenhum após as duas tentativas → aviso de negócio "Lead não encontrado na Clint" com título e link do documento; fim.
5. **Ler documento** — download do Google Doc exportado como `text/plain`.
6. **Extrair com IA** (OpenAI, saída JSON estruturada; prompt em `fluxos/meet-para-clint/prompts/extracao.md`):

   | Chave | Conteúdo |
   |---|---|
   | `tipo_mao_de_obra` | Tipo de mão de obra que o cliente usou (texto livre) |
   | `consegue_recibo` | `Sim` / `Não` / `Não mencionado` + detalhe curto |
   | `negociacao_valores` | Valores discutidos |
   | `negociacao_forma_pagamento` | Forma de pagamento discutida |
   | `objecao_decisao` | Desculpa/objeção para a tomada de decisão (ex.: "precisa falar com a esposa") |
   | `dia_retorno` | Data ISO `YYYY-MM-DD` ou `null` |
   | `hora_retorno` | `HH:mm` ou `null` |
   | `resumo` | Resumo da reunião em 5 a 8 linhas |

   Regra do prompt: não inventar; o que não foi dito é `Não mencionado` (ou `null` nas datas). Datas relativas ("semana que vem", "sexta") são resolvidas a partir da data da reunião.
7. **Atualizar deal** — `POST /v1/deals/{id}` com os campos personalizados. Um campo só é enviado se o novo valor **não** for `Não mencionado` (não apaga informação de reuniões anteriores). `dia_retorno` é gravado como `dd/MM/yyyy HH:mm` (09:00 se sem hora); se `null`, o campo não é enviado.
8. **Criar tarefa** — `POST /v2/activities`:
   - Com data: `type: TASK`, `title: "Retorno: {nome}"`, `due_at` = data + hora (09:00 se sem hora), fuso America/Sao_Paulo convertido para ISO UTC.
   - Sem data: `title: "Definir retorno: {nome}"`, `due_at` = próximo dia útil (seg–sex, sem considerar feriados) após a data da execução, 09:00.
   - `content`: resumo + itens extraídos + link do documento da transcrição.

### 4.3 Configuração manual na Clint (usuário, uma vez)

Criar no **deal** um grupo "Reunião" com 6 campos do tipo texto:
Tipo de mão de obra · Consegue recibo · Negociação – valores · Negociação – forma de pagamento · Objeção para decisão · Dia de retorno.
Depois, o Claude lê as chaves internas via `GET /v1/account/fields` e as usa no passo 7.

### 4.4 Erros

- Clint e OpenAI: 3 tentativas, 2 s entre elas.
- Falha técnica → fluxo "Alerta WhatsApp" (via `errorWorkflow`).
- Casos de negócio (sem telefone, lead não encontrado) → aviso de negócio via sub-workflow; a execução termina com sucesso.

### 4.5 Teste

O gatilho do Google Drive não pode ser executado pelo MCP. Teste:
1. Usar uma transcrição de exemplo (real ou fictícia) como dado fixado (pin data) no gatilho/leitura.
2. Rodar contra um **deal de teste** na Clint (telefone informado pelo usuário).
3. Verificar no deal: campos preenchidos, tarefa criada com data e conteúdo corretos.
4. Testar os ramos: telefone não encontrado; transcrição sem dia de retorno.
5. Passando, ativar o fluxo.

### 4.6 Limitações conhecidas

- A tarefa fica atribuída ao dono do token `Header Clint`; a API não permite escolher o responsável.
- A API da Clint não envia anexos; o link do documento vai no conteúdo da tarefa.
- Polling de 1 minuto: a atualização chega até ~1 min depois de o Meet salvar o documento.

## 5. Pendências antes/durante a implementação

| Pendência | Responsável | Bloqueia |
|---|---|---|
| Dados da Evolution + credencial `apikey` | Usuário | Teste do Alerta WhatsApp |
| Criar os 6 campos do deal na Clint | Usuário | Passo 7 do fluxo Meet |
| Telefone de um deal de teste | Usuário | Teste do fluxo Meet |
| Inspecionar arquivos reais da pasta do Meet | Claude | Passo 2 do fluxo Meet |
