# Anotação Reuniao Meet (Meet → Clint)

> Status: **publicado e ativo** desde 2026-10-02 15:41 (versão `902610d7`).

## Objetivo

Quando o Google Meet salva os documentos de uma reunião comercial de "INSS de obras" na
pasta da Sara, a IA extrai as informações da conversa e o fluxo grava um bloco de anotações
no card (deal) do lead na Clint, no campo RICH_TEXT **"Notas da Reunião"**
(`notas_da_reuniao`), com a reunião mais nova no topo.

## Link e ID no n8n

- ID: `Owu9NI5ScaU5zZfA`
- https://n8n.grupotoda.com.br/workflow/Owu9NI5ScaU5zZfA
- `errorWorkflow`: Alerta WhatsApp (`ktpi4gsUslWWAWyG`)

## Gatilho

Google Drive Trigger (conta `sara`), a cada minuto, **pasta criada** dentro de
`12LPKim0euEzdnAkPSorKccTx-6seq7Dk`. O Meet cria uma subpasta por reunião, com o nome
`{título do evento} - AAAA/MM/DD HH:mm GMT-03:00`. Só pastas criadas depois da ativação
são processadas.

## Passo a passo

1. **Aguardar documento** — 2 min (o Doc é criado logo após a pasta e preenchido em 15–30 s).
2. **Extrair dados do título** — título do evento e data/hora de início a partir do nome da pasta.
3. **Listar arquivos da reunião** — Drive API, filhos da subpasta.
4. **Buscar eventos na agenda** — Calendar API (agenda da Sara), ±4 h do início.
5. **Consolidar reunião** — escolhe o documento (`… - Anotações do Gemini`; senão
   `… - Transcript`/`Transcrição`) e o evento com título idêntico; extrai o ID do card do
   link `app.clint.digital/deal/{uuid}` ou `app.useclint.com/deal/{uuid}` da descrição.
6. **Tem documento?** — não → alerta "Reunião sem transcrição".
7. **Buscar deal** — `GET /v1/deals/{id}`. **Escolher deal** só aceita a resposta se o ID
   devolvido for exatamente o do link (nunca usa listas).
8. **Achou deal?** — não (evento sem link ou link que não abre card) → alerta
   "Reunião sem card na Clint". Nada é gravado.
9. **Ler documento** — exporta o Google Doc como texto.
10. **Extrair com IA** — OpenAI `gpt-5.4-mini`, saída JSON validada por schema; prompt em
    [prompts/extracao.md](prompts/extracao.md).
11. **Montar anotação** — bloco HTML (data, mão de obra, recibo, valores, pagamento, objeção,
    retorno, resumo, link da transcrição); bloco novo + separador + conteúdo anterior;
    **todo caractere não-ASCII vira entidade HTML** (a API da Clint corrompe acentos).
12. **Já gravado?** — se o link desse documento já está no campo, não grava de novo.
13. **Gravar no deal** — `POST /v1/deals/{id}` com `{"fields":{"notas_da_reuniao":"…"}}`.

Retry 3× / 2 s em todas as chamadas HTTP e na IA. Alertas não bloqueiam o fluxo
(`waitForSubWorkflow: false`, `onError: continueRegularOutput`).

## Credenciais usadas

| Uso | Credencial | Tipo | ID |
|---|---|---|---|
| Drive (gatilho, listagem e leitura) | `sara` | googleDriveOAuth2Api | `Xz7XkQuj0KDkVtW5` |
| Agenda da Sara | `sara` | googleCalendarOAuth2Api | `cL4FeJhik5kcbnJB` |
| Clint | `Header Clint` | httpHeaderAuth | `qiMe1q95EEa3L2HI` |
| IA | `grupotoda` | openAiApi | `ynBZoBP7TIG7AJAZ` |

## Configuração manual necessária

- [x] Campo "Notas da Reunião" (RICH_TEXT) no deal da Clint.
- [x] Credencial da agenda da Sara.
- [ ] **Todo evento de reunião precisa ter o link do card da Clint na descrição.**
- [ ] Alertas: credencial da Evolution com a `apikey` da instância `rgis` (ver README do Alerta).

## Como testar

O gatilho do Drive não é executável pelo MCP. Para testar, adicionar temporariamente ao
rascunho um Webhook "TESTE pasta" → Code `return { json: $json.body }` → "Extrair dados do
título" (recebe `{ id, name, createdTime }` de uma subpasta real) e/ou "TESTE contexto" →
Code → "Tem documento?" (recebe o contexto já resolvido, apontando para o deal de teste
`40ab3b02-cadf-419d-a842-d8870204971e`). **Desativar "Gravar no deal" ao usar pastas reais**
(senão grava nos cards reais). Remover as entradas de teste antes de publicar.

Cenários validados em 2026-10-02 (execuções 304784–304808):

| Cenário | Resultado |
|---|---|
| Pasta real (Cristiana, só Transcript) → evento → card pelo link | ✅ deal certo, gravação desativada |
| Pasta real (Lucas, Anotações do Gemini) → evento → card pelo link | ✅ deal certo, doc Gemini escolhido |
| Gravar no deal de teste | ✅ HTML com acentos/emoji corretos; outros 43 campos intactos |
| Mesmo documento de novo | ✅ não regrava |
| Segunda reunião no mesmo deal | ✅ bloco novo no topo, anterior preservado |
| Evento sem link / link de card inexistente | ✅ alerta "Reunião sem card na Clint", nada gravado |
| Subpasta sem documento | ✅ alerta "Reunião sem transcrição" |

## Limitações conhecidas

- O dia de retorno fica só no texto (sem tarefa/lembrete na Clint).
- A API da Clint não envia anexos; o link do documento vai no texto.
- A IA pode variar na leitura de valores ditos de forma confusa na conversa (ex.: honorários
  lidos como R$ 3.640,00 numa execução e R$ 3.364,00 em outra) — o vendedor deve conferir
  valores pela transcrição.
- Reprocessar o mesmo documento chama a IA antes de detectar que já foi gravado (custo baixo;
  só acontece em reprocessamento manual).
- O campo cresce a cada reunião, sem limite de histórico.

## Histórico de decisões

- **2026-10-02 — Estrutura real da pasta (inspeção).** Uma subpasta por reunião → gatilho
  "pasta criada" mantido. Conteúdo varia: "Anotações do Gemini" (resumo + transcrição
  completa) ou "Transcript" + "Recording". Documento usado: Gemini, senão Transcript.
- **2026-10-02 — Onde gravar:** campo único RICH_TEXT (HTML), reunião nova no topo; sem tarefa.
- **2026-10-02 — Campo na Clint:** chave `notas_da_reuniao` (rótulo "Notas da Reunião").
  A API corrompe não-ASCII → entidades HTML. `""`/`null` não apagam; `<p></p>` sim.
- **2026-10-02 — Credencial da agenda:** a "Google Calendar account" expirou; trocada pela
  nova credencial `sara` (`cL4FeJhik5kcbnJB`) criada pelo usuário.
- **2026-10-02 — Sem plano B por telefone (decisão do usuário).** O link do card sempre estará
  no evento. Motivo adicional: em teste, a busca `GET /v1/deals?phone=00000000000` foi ignorada
  pela Clint (devolveu todos os deals) e o fluxo gravou num card real errado (André de Paula
  Saliés, `ff404a8c…`); o campo foi restaurado para vazio (`<p></p>`) na hora. Hoje o fluxo só
  aceita o deal cujo ID é exatamente o do link.
