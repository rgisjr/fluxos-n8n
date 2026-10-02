# Anotação Reuniao Meet (Meet → Clint)

> Status: **em construção** (baseline salvo; fluxo completo pendente — ver
> [plano](../../docs/plans/2026-10-02-meet-para-clint-plano.md)).

## Objetivo

Quando o Google Meet salvar os documentos de uma reunião comercial de "INSS de obras" na
pasta da Sara, extrair com IA as informações da conversa e gravar um bloco de anotações no
deal do lead na Clint (campo RICH_TEXT "Anotações da reunião", reunião mais nova no topo).

## Link e ID no n8n

- ID: `Owu9NI5ScaU5zZfA`
- https://n8n.grupotoda.com.br/workflow/Owu9NI5ScaU5zZfA

## Gatilho

Google Drive Trigger (conta `sara`), a cada minuto, **pasta criada** dentro de
`12LPKim0euEzdnAkPSorKccTx-6seq7Dk`. O Meet cria uma subpasta por reunião.

## Passo a passo

A definir na T7 do plano. Resumo: aguardar 2 min → título/data/telefone a partir do nome
da subpasta → escolher documento → achar evento na agenda → ID do deal pelo link na
descrição (plano B: telefone) → ler documento → IA → montar HTML → gravar no deal.

## Credenciais usadas

| Uso | Credencial | Tipo | ID |
|---|---|---|---|
| Drive (gatilho e leitura) | `sara` | googleDriveOAuth2Api | `Xz7XkQuj0KDkVtW5` |
| Agenda da Sara | `Google Calendar account` | googleCalendarOAuth2Api | `dVWPfETb214FyT82` |
| Clint | `Header Clint` | httpHeaderAuth | `qiMe1q95EEa3L2HI` |
| IA | `grupotoda` | openAiApi | `ynBZoBP7TIG7AJAZ` |

## Configuração manual necessária

- [ ] Criar na Clint, no deal, o campo **"Anotações da reunião"** do tipo RICH_TEXT.
- [ ] Indicar um deal de teste.

## Como testar

Ver T8 do plano (7 cenários com dados fixados contra um deal de teste).

## Limitações conhecidas

- O dia de retorno fica só no texto (sem tarefa/lembrete na Clint).
- A API da Clint não envia anexos; o link do documento vai no texto.
- Títulos sem telefone (ex.: reuniões com credenciado) dependem do link do deal na agenda.

## Histórico de decisões

- **2026-10-02 — Estrutura real da pasta (inspeção T4).**
  - A pasta monitorada contém **uma subpasta por reunião**, nome
    `{título do evento} - AAAA/MM/DD HH:mm GMT-03:00`. Por isso o gatilho continua em
    **pasta criada** (o plano original previa trocar para "arquivo criado" — descartado).
  - Conteúdo varia por reunião:
    - `{…} - Anotações do Gemini` (Google Doc): resumo, decisões, próximas etapas e
      detalhes do Gemini **mais a transcrição completa** (~36 mil caracteres);
    - ou `{…} - Transcript` (Google Doc, só transcrição) + `{…} - Recording` (mp4, criado
      ~30 min depois).
  - O Doc é criado ~1 s depois da subpasta e preenchido em 15–30 s → espera de 2 min.
  - **Documento usado:** "Anotações do Gemini" se existir; senão "Transcript"/"Transcrição".
  - Padrões de título observados (o telefone varia de formato e pode faltar):
    - `INSS de obras: {Nome} - 5531988302388 CRED {Nome}`
    - `INSS de Obra: {Nome} - +55 11 99297 4103`
    - `INSS de obra / {Nome} +55 (11) 91512 6813`
    - `INSS de obra / {Empresa} / Credenciado: {Nome}` (sem telefone)
  - Telefone = sequência com dígitos, espaços, parênteses e hífens, normalizada para só dígitos.
- **2026-10-02 — Identificação do deal:** link `app.clint.digital/deal/{uuid}` na descrição
  do evento da agenda (principal); telefone do título (plano B).
- **2026-10-02 — Onde gravar:** campo único RICH_TEXT (HTML), reunião nova no topo; sem tarefa.
