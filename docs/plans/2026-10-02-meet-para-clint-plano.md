# Plano de implementação — Base + Alerta WhatsApp + Meet → Clint

- **Spec:** [docs/specs/2026-10-02-base-projeto-e-meet-clint-design.md](../specs/2026-10-02-base-projeto-e-meet-clint-design.md)
- **Objetivo:** quando o Meet salvar o documento de uma reunião na pasta da Sara, gravar no deal da Clint um bloco de anotações (mão de obra, recibo, valores, pagamento, objeção, retorno, resumo, link), reunião mais nova no topo.
- **Ferramentas:** MCP do n8n (`claude_ai_n8n`), `curl` com o token do `.env` para consultas à Clint, git local.
- **Testes:** n8n não tem testes unitários; cada tarefa termina com uma verificação objetiva (validação do MCP, execução com dados fixados, ou consulta à API). Nada é ativado sem passar na verificação.

## Fatos levantados (2026-10-02)

- Campos personalizados do deal ficam planos em `deal.fields.{chave}`; a chave é o rótulo em snake_case **truncado em 20 caracteres** (ex.: `nome_completo_creden`).
- Tipo `RICH_TEXT` existe e guarda **HTML** (ex.: campo `anexos` = `<p><a href="…">…</a>&nbsp;</p>`). Já existe `resumo_qualificacao_ia` (RICH_TEXT) — precedente de texto gerado por IA.
- `GET /v1/deals/{id}` devolve `fields`, `contact{name, phone, ddi}`, `user`, `stage`, `latest_meeting_link`.
- No deal de referência (`bba448bd-…`), o telefone do título do evento (`5531988302388`) **não** é o telefone do contato (`31987832999`) → o plano B por telefone pode falhar; o link do deal na agenda é o caminho principal.
- O gatilho Google Drive não é executável pelo MCP; só Schedule, Webhook, Form e Chat são. Testes usam dados fixados (pin data) ou fluxos temporários com Webhook.
- O fluxo `Owu9NI5ScaU5zZfA` tem versão publicada separada do rascunho: `update_workflow` altera o rascunho; `publish_workflow` coloca no ar.

## Dependências

```
T1 Base ──────────────┐
T2 Alerta (criar) ────┼──> T3 Alerta (configurar+testar) ──┐
T4 Inspecionar Drive ─┤                                    │
T5 Campo na Clint ────┤                                    │
T6 Prompt ────────────┴──> T7 Construir Meet ──> T8 Testar ┴──> T9 Publicar ──> T10 Acompanhar 1ª reunião real
```

| Pendência do usuário | Bloqueia |
|---|---|
| Criar campo RICH_TEXT **"Anotações da reunião"** no deal (Clint) | T5 → T7 |
| Indicar **deal de teste** (e, se possível, evento de teste na agenda da Sara com o link dele) | T8 |
| Dados da **Evolution** (URL, instância, número) + credencial `httpHeaderAuth` com a `apikey` | T3 |

T1, T2, T4 e T6 não dependem do usuário e podem começar já.

---

## T1 — Base do projeto

**Depende de:** nada.
**Arquivos:** `CLAUDE.md`, `fluxos/.gitkeep`.

- [ ] Criar `CLAUDE.md` com:
  - Propósito do projeto e instância (`https://n8n.grupotoda.com.br`).
  - Estrutura de pastas (spec §2.1) e formato do README de fluxo (spec §2.3).
  - Fluxo de trabalho: novo fluxo e edição (spec §2.2 e §2.4), incluindo a checagem de `versionId` antes de editar.
  - Regras fixas (spec §2.5): `errorWorkflow` obrigatório, retry 3×/2 s, sem segredos, credenciais por ID, avisar quando faltar credencial.
  - Ordem obrigatória no MCP: `get_workflow_sdk_reference` → `get_workflow_best_practices` → `search_nodes` → `get_node_types` → `validate_workflow` → criar.
  - Tabela de credenciais já conhecidas (spec §4.1) e ID do fluxo "Alerta WhatsApp" (preenchido em T2).
  - Clint: base `https://api.clint.digital`, header `api-token`, campos em `fields.{chave≤20}`, RICH_TEXT = HTML; token local em `.env` (nunca commitar).
- [ ] Criar `fluxos/` (com `.gitkeep`).

**Verificação:** `git status` mostra só `CLAUDE.md` e `fluxos/.gitkeep`; `grep -i "api-token=" -r . --exclude=.env` sem resultados.
**Commit:** `docs: CLAUDE.md com regras do projeto`

---

## T2 — Fluxo "Alerta WhatsApp" (criar)

**Depende de:** T1.
**Recursos:** novo workflow no n8n; `fluxos/alerta-whatsapp/{README.md,workflow.json}`.

- [ ] Carregar `get_workflow_sdk_reference` e `get_workflow_best_practices` (técnica de notificação/erro).
- [ ] `get_node_types` para: `errorTrigger`, `executeWorkflowTrigger`, `set`, `code`, `httpRequest`, `if`.
- [ ] Montar o fluxo:
  1. **Error Trigger** → **Normalizar erro** (Code):
     ```js
     const e = $json;
     return [{ json: {
       tipo: 'erro',
       chave: `${e.workflow?.id}:${e.execution?.error?.message ?? ''}`.slice(0, 300),
       texto: [
         '⚠️ Falha no n8n',
         `Fluxo: ${e.workflow?.name ?? '?'}`,
         `Passo: ${e.execution?.lastNodeExecuted ?? '?'}`,
         `Erro: ${(e.execution?.error?.message ?? '?').slice(0, 300)}`,
         `Quando: ${DateTime.now().setZone('America/Sao_Paulo').toFormat('dd/MM HH:mm')}`,
         e.execution?.url ? `Ver execução: ${e.execution.url}` : '',
       ].filter(Boolean).join('\n'),
     }}];
     ```
  2. **Execute Workflow Trigger** (entradas `titulo`, `mensagem`, `link`) → **Normalizar aviso** (Code):
     ```js
     const { titulo, mensagem, link } = $json;
     return [{ json: {
       tipo: 'aviso',
       chave: `aviso:${titulo}:${mensagem}`.slice(0, 300),
       texto: [`ℹ️ ${titulo}`, mensagem, link].filter(Boolean).join('\n'),
     }}];
     ```
  3. Os dois → **Config** (Set, "incluir outros campos"): `evolutionUrl`, `instancia`, `numeroDestino` (placeholders `PENDENTE`).
  4. **Anti-spam** (Code):
     ```js
     const JANELA_MS = 30 * 60 * 1000;
     const estado = $getWorkflowStaticData('global');
     const enviados = estado.enviados ?? {};
     const agora = Date.now();
     const limpo = Object.fromEntries(
       Object.entries(enviados).filter(([, t]) => agora - t < JANELA_MS),
     );
     const { chave } = $json;
     if (limpo[chave]) { estado.enviados = limpo; return []; }
     estado.enviados = { ...limpo, [chave]: agora };
     return [{ json: $json }];
     ```
  5. **Enviar WhatsApp** (HTTP Request): `POST {{$json.evolutionUrl}}/message/sendText/{{$json.instancia}}`, autenticação `httpHeaderAuth` (credencial definida em T3), corpo JSON `{ "number": numeroDestino, "text": texto }`, retry 3×/2 s.
- [ ] `validate_workflow` → `create_workflow_from_code` (nome "Alerta WhatsApp"). Não precisa estar ativo para funcionar como fluxo de erro.
- [ ] `get_workflow_details` → salvar `fluxos/alerta-whatsapp/workflow.json`; escrever README; anotar o ID no `CLAUDE.md`.

**Verificação:** `validate_workflow` sem erros; fluxo aparece em `search_workflows`.
**Commit:** `feat: fluxo Alerta WhatsApp (pendente configuração Evolution)`

---

## T3 — Alerta WhatsApp (configurar e testar)

**Depende de:** T2 + dados da Evolution do usuário.

- [ ] `list_credentials` → achar a credencial `httpHeaderAuth` da Evolution indicada pelo usuário; `setNodeCredential` em "Enviar WhatsApp".
- [ ] `updateNodeParameters` em **Config** com URL, instância e número.
- [ ] Teste: criar fluxo temporário "TESTE alerta" (Webhook → Execute Workflow "Alerta WhatsApp" com `titulo: "Teste"`, `mensagem: "Alerta configurado"`); `execute_workflow`; disparar duas vezes seguidas.
- [ ] Arquivar o fluxo temporário (`archive_workflow`).
- [ ] Atualizar `workflow.json` e README.

**Verificação:** usuário confirma **uma** mensagem recebida (a segunda foi barrada pelo anti-spam); execução do Alerta com status `success`.
**Commit:** `feat: Alerta WhatsApp configurado com Evolution`

---

## T4 — Inspecionar arquivos reais da pasta do Meet

**Depende de:** nada.

- [ ] Criar fluxo temporário "TESTE inspeção Drive": Webhook → Google Drive "search files" (credencial `sara`, pasta `12LPKim0euEzdnAkPSorKccTx-6seq7Dk`, campos `id,name,mimeType,createdTime,webViewLink`, últimos 20) → Google Drive "download" do Google Doc mais recente convertido para `text/plain`.
- [ ] `execute_workflow`; ler a execução com `get_workflow_execution`.
- [ ] Registrar no README do fluxo Meet (seção "Histórico de decisões"):
  - padrão exato do nome dos arquivos (sufixos de data/hora, "Transcrição", "Anotações do Gemini");
  - se as Anotações do Gemini já contêm a transcrição (abas) ou se são arquivos separados;
  - **decisão:** qual documento processar (preferir o que tiver a conversa completa) e o filtro de nome correspondente.
- [ ] Ajustar as regex de T7 (passo 3) ao padrão real.
- [ ] Arquivar o fluxo temporário.

**Verificação:** decisão escrita, com 1 exemplo real de nome de arquivo (sem dados sensíveis além do título).
**Commit:** `docs: padrão dos arquivos do Meet e documento escolhido`

---

## T5 — Confirmar o campo na Clint

**Depende de:** usuário criar o campo RICH_TEXT "Anotações da reunião" no deal.

- [ ] `curl -H "api-token: $TOKEN" https://api.clint.digital/v1/account/fields` → localizar a chave (esperado: `anotacoes_da_reuniao`, 20 caracteres) e confirmar `type: RICH_TEXT`.
- [ ] No **deal de teste**: `POST /v1/deals/{id}` com `{"fields":{"<chave>":"<p>teste</p>"}}` → `GET` e confirmar que gravou e que os **outros campos não foram apagados**. Se o corpo exigir outro formato (ex.: `fields.deal`), registrar o formato correto.
- [ ] Limpar o valor de teste.
- [ ] Registrar chave e formato do corpo no README do fluxo Meet.

**Verificação:** `GET` do deal de teste mostra o HTML gravado e os demais `fields` intactos.
**Commit:** `docs: chave do campo Anotações da reunião na Clint`

---

## T6 — Prompt de extração

**Depende de:** T4 (para ajustar ao formato real do documento).
**Arquivo:** `fluxos/meet-para-clint/prompts/extracao.md`

- [ ] Escrever o prompt (sistema), em português:

  ```
  Você analisa a transcrição de uma reunião comercial de "INSS de obras" entre um
  vendedor e um lead (dono de obra ou credenciado). Extraia as informações abaixo
  e responda SOMENTE com JSON válido neste formato:

  {
    "tipo_mao_de_obra": string,
    "consegue_recibo": string,
    "negociacao_valores": string,
    "negociacao_forma_pagamento": string,
    "objecao_decisao": string,
    "dia_retorno": string,
    "resumo": string
  }

  Definições:
  - tipo_mao_de_obra: tipo de mão de obra que o cliente usou na obra
    (ex.: própria, empreiteiro, autônomos, empresa contratada).
  - consegue_recibo: começa com "Sim", "Não" ou "Não mencionado"; depois, um
    detalhe curto (ex.: "Sim – tem recibos do pedreiro").
  - negociacao_valores: valores discutidos (proposta, contraproposta, descontos).
  - negociacao_forma_pagamento: forma de pagamento discutida (à vista, parcelado,
    PIX, cartão, entrada + parcelas).
  - objecao_decisao: o motivo/desculpa que o lead deu para não decidir na hora
    (ex.: "precisa falar com a esposa", "vai comparar preço").
  - dia_retorno: data e hora combinadas para o próximo contato no formato
    dd/MM/yyyy HH:mm (ou dd/MM/yyyy se não houver hora). Resolva datas relativas
    ("sexta", "semana que vem") a partir da DATA DA REUNIÃO informada.
  - resumo: 5 a 8 linhas com os pontos principais da conversa.

  Regras:
  - Não invente. Se algo não foi dito, use exatamente "Não mencionado".
  - Use as palavras do lead quando forem relevantes.
  - Valores em reais no formato R$ 1.234,56.
  ```
- [ ] Mensagem do usuário (montada no nó): `DATA DA REUNIÃO: {dd/MM/yyyy (dia da semana)}\n\nTRANSCRIÇÃO:\n{texto}`.

**Verificação:** revisar o prompt contra a lista manuscrita (`docs/referencias/campos-extracao-manuscrito.jpeg`) — os 5 itens + resumo estão cobertos.
**Commit:** `feat: prompt de extração do fluxo Meet → Clint`

---

## T7 — Construir o fluxo Meet → Clint

**Depende de:** T2 (ID do Alerta), T4, T5, T6.
**Recurso:** rascunho do workflow `Owu9NI5ScaU5zZfA` via `update_workflow` (versão publicada continua a atual até T9).

- [ ] `get_workflow_details` → salvar o estado atual em `fluxos/meet-para-clint/workflow.json` (baseline) e commitar antes de alterar.
- [ ] `get_node_types` para: `googleDriveTrigger`, `filter`, `code`, `googleCalendar` (event getAll), `if`, `httpRequest`, `googleDrive` (download), `extractFromFile`, `openAi` (message model), `executeWorkflow`. Escolher o modelo OpenAI com `explore_node_resources` (preferir o "mini" mais recente com saída JSON).
- [ ] Nós, em ordem (nomes exatos — usados nas expressões):

  1. **Google Drive Trigger** — `event: fileCreated` (era `folderCreated`); mesma pasta e credencial.
  2. **Só documentos da reunião** (Filter) — `mimeType == application/vnd.google-apps.document` E nome casa com o padrão escolhido em T4.
  3. **Extrair título** (Code, por item):
     ```js
     const f = $json;
     const titulo = f.name
       .replace(/\s*[-–]\s*(Anotações do Gemini|Transcrição|Transcript|Notes by Gemini)\s*$/i, '')
       .replace(/\s*[-–(]\s*\d{4}[\/-]\d{2}[\/-]\d{2}[\s,]+\d{2}:\d{2}.*$/, '')
       .trim();
     const telefone = titulo.match(/\d{10,13}/)?.[0] ?? null;
     const nomeLead = titulo
       .replace(/^INSS de obras?\s*[:\/-]\s*/i, '')
       .split(/\s*[-–]?\s*\d{10,13}/)[0]
       .trim();
     return { json: {
       titulo, telefone, nomeLead,
       docId: f.id,
       docUrl: f.webViewLink ?? `https://docs.google.com/document/d/${f.id}`,
       criadoEm: f.createdTime,
     }};
     ```
     (regex finais ajustadas em T4)
  4. **Buscar evento** (Google Calendar, getAll, credencial `Google Calendar account`, calendário primário) — `timeMin` = início do dia de `criadoEm` em America/Sao_Paulo, `timeMax` = fim do dia, `query` = `titulo`; `alwaysOutputData: true`; retry.
  5. **Resolver deal ID** (Code, executar uma vez):
     ```js
     const ctx = $('Extrair título').first().json;
     const norm = (s) => (s ?? '').toLowerCase().replace(/\s+/g, ' ').trim();
     const eventos = $input.all().map((i) => i.json).filter((e) => e.id);
     const evento = eventos.find((e) => norm(e.summary) === norm(ctx.titulo)) ?? eventos[0];
     const dealId = evento?.description
       ?.match(/app\.clint\.digital\/deal\/([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})/i)?.[1] ?? null;
     return [{ json: { ...ctx, dealId } }];
     ```
  6. **Tem deal ID?** (If `dealId` não vazio)
     - **true →** **Buscar deal por ID** (HTTP GET `https://api.clint.digital/v1/deals/{{dealId}}`, cred. `Header Clint`, "never error", resposta completa) → **Achou por ID?** (If `statusCode == 200`) → true: **Deal encontrado**; false: vai para o plano B.
     - **false →** plano B.
  7. **Plano B — Buscar por telefone** (HTTP GET `/v1/deals?phone={{telefone}}&status=OPEN`) → **Buscar sem DDI** (HTTP GET com `telefone` sem o `55` inicial; executa sempre, é barato) → **Escolher deal** (Code):
     ```js
     const lista = [
       ...($('Buscar por telefone').first().json.data ?? []),
       ...($('Buscar sem DDI').first().json.data ?? []),
     ];
     const deal = lista.sort((a, b) => (b.updated_at ?? '').localeCompare(a.updated_at ?? ''))[0];
     return [{ json: { ...$('Resolver deal ID').first().json, deal: deal ?? null } }];
     ```
     Telefone ausente → pular as buscas e seguir com `deal: null`.
  8. **Lead encontrado?** (If `deal` não nulo) — false → **Avisar lead não encontrado** (Execute Workflow "Alerta WhatsApp": `titulo: "Lead não encontrado na Clint"`, `mensagem: titulo`, `link: docUrl`) → fim.
  9. **Deal encontrado** (Set/Code) — unifica as duas origens em `{ ctx..., deal }` (no caminho por ID, `deal = $json.body.data`).
  10. **Baixar documento** (Google Drive download, cred. `sara`, `fileId = docId`, conversão Google Docs → `text/plain`) → **Ler texto** (Extract From File, text).
  11. **Extrair com IA** (OpenAI message model, cred. `grupotoda`, saída JSON; system = prompt de T6; user = data da reunião + texto; retry).
  12. **Montar anotação** (Code):
      ```js
      const CAMPO = '<chave de T5>';
      const ctx = $('Deal encontrado').first().json;
      const ia = JSON.parse($json.message?.content ?? $json.content ?? '{}');
      const esc = (s) => String(s ?? 'Não mencionado')
        .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
      const data = DateTime.fromISO(ctx.criadoEm).setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy');
      const linha = (rotulo, valor) => `• <strong>${rotulo}:</strong> ${esc(valor)}`;
      const bloco = [
        `<p><strong>📅 Reunião ${data}</strong><br>`,
        [
          linha('Mão de obra', ia.tipo_mao_de_obra),
          linha('Recibo', ia.consegue_recibo),
          linha('Valores', ia.negociacao_valores),
          linha('Pagamento', ia.negociacao_forma_pagamento),
          linha('Objeção', ia.objecao_decisao),
          linha('Retorno', ia.dia_retorno),
        ].join('<br>'),
        `</p><p><strong>Resumo:</strong> ${esc(ia.resumo)}</p>`,
        `<p><a href="${ctx.docUrl}">Transcrição</a></p>`,
      ].join('');
      const anterior = ctx.deal.fields?.[CAMPO] ?? '';
      const jaGravado = anterior.includes(ctx.docId);
      const valor = anterior ? `${bloco}<p>──────────</p>${anterior}` : bloco;
      return [{ json: { dealId: ctx.deal.id, jaGravado, corpo: { fields: { [CAMPO]: valor } } } }];
      ```
      (formato de `corpo` conforme confirmado em T5)
  13. **Já gravado?** (If `jaGravado`) — true → fim; false → **Gravar no deal** (HTTP POST `/v1/deals/{{dealId}}`, cred. `Header Clint`, body `corpo`, retry).
- [ ] Configurações do workflow: `errorWorkflow` = ID do Alerta, `timezone: America/Sao_Paulo`, `saveDataErrorExecution: all`.
- [ ] Retry 3×/2 s em todos os nós HTTP, Calendar, Drive download e OpenAI.
- [ ] `validate_workflow` no estado final; corrigir até zerar erros.

**Verificação:** validação sem erros; `get_workflow_details` mostra os 13 passos conectados e a versão publicada ainda é a antiga.
**Commit:** `feat: fluxo Meet → Clint (rascunho)`

---

## T8 — Testar o fluxo Meet → Clint

**Depende de:** T7 + deal de teste do usuário.

Fixar (pin) a saída do **Google Drive Trigger** com um arquivo real ou fictício (`prepare_workflow_pin_data`) e rodar com `test_workflow`. Quando necessário, fixar também a saída de **Buscar evento** e **Ler texto**.

| # | Cenário | Esperado |
|---|---|---|
| 1 | Evento com link do deal de teste + transcrição completa | Campo do deal de teste com bloco no formato do passo 12; demais campos intactos |
| 2 | Mesmo arquivo de novo | `jaGravado = true`; nada é gravado |
| 3 | Outro arquivo, mesmo deal | Bloco novo **no topo**, separador, bloco anterior preservado |
| 4 | Evento sem link, telefone do título = telefone do deal de teste | Achado pelo plano B; gravado |
| 5 | Sem evento e telefone inexistente | Execute Workflow do Alerta chamado com "Lead não encontrado"; execução `success` |
| 6 | Transcrição sem falar de recibo/retorno | Itens com "Não mencionado"; nada inventado |
| 7 | Arquivo de vídeo | Barrado no filtro; nada mais executa |

- [ ] Rodar os 7 cenários; conferir cada um via `get_workflow_execution` e `GET` do deal de teste.
- [ ] Ajustar prompt/regex se algum falhar; repetir.
- [ ] Limpar o campo do deal de teste ao final (ou deixar, se o usuário preferir).

**Verificação:** os 7 cenários passam.
**Commit:** `test: cenários do fluxo Meet → Clint validados`

---

## T9 — Publicar

**Depende de:** T8 (e T3 para os alertas chegarem de fato).

- [ ] Remover pin data.
- [ ] `publish_workflow` em `Owu9NI5ScaU5zZfA`.
- [ ] `get_workflow_details` → atualizar `workflow.json`; finalizar o README (todas as seções da spec §2.3).
- [ ] Se T3 ainda estiver pendente, registrar no README que os alertas só chegam após a configuração da Evolution.

**Verificação:** `active: true`, `activeVersion.sameAsDraft: true`, `errorWorkflow` apontando para o Alerta.
**Commit:** `feat: publica fluxo Meet → Clint`

---

## T10 — Acompanhar a primeira reunião real

**Depende de:** T9.

- [ ] Após a próxima reunião da Sara: `search_workflow_executions` no fluxo → conferir a execução e o deal real.
- [ ] Mostrar ao usuário o bloco gravado e pedir feedback do vendedor (formato, itens, tom do resumo).
- [ ] Ajustes → seguem o fluxo de edição (checar `versionId`, alterar, testar, publicar, commit).

**Verificação:** pelo menos 1 deal real atualizado corretamente, confirmado pelo usuário.
