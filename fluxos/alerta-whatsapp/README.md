# Alerta WhatsApp

## Objetivo

Avisar no WhatsApp (via Evolution API) quando:
- um fluxo **falha** em produção (este é o `errorWorkflow` padrão de todos os fluxos);
- um fluxo precisa dar um **aviso de negócio** (ex.: "Lead não encontrado na Clint").

## Link e ID no n8n

- ID: `ktpi4gsUslWWAWyG` (publicado — exigido pelo n8n para ser `errorWorkflow`)
- https://n8n.grupotoda.com.br/workflow/ktpi4gsUslWWAWyG

## Gatilhos

1. **Erro em outro fluxo** (Error Trigger) — disparado pelo n8n quando um fluxo com
   `errorWorkflow = ktpi4gsUslWWAWyG` falha em execução de produção.
2. **Aviso de outro fluxo** (Execute Workflow Trigger) — chamado por outros fluxos com
   `{ titulo, mensagem, link }`.

## Passo a passo

1. **Normalizar erro** / **Normalizar aviso** — monta `texto` e uma `chave` de deduplicação.
   - Erro: `⚠️ Falha no n8n` + fluxo, passo, erro, horário (America/Sao_Paulo), link da execução.
   - Aviso: `ℹ️ {titulo}` + mensagem + link.
2. **Config** — URL da Evolution, nome da instância, número de destino.
3. **Anti-spam 30 min** — a mesma `chave` só é enviada uma vez a cada 30 minutos
   (estado em `$getWorkflowStaticData`, que só persiste em execuções de produção).
4. **Enviar WhatsApp** — `POST {evolutionUrl}/message/sendText/{instancia}` com
   `{ number, text }`; 3 tentativas, 2 s entre elas.

## Credenciais usadas

| Nó | Credencial | Tipo | ID |
|---|---|---|---|
| Enviar WhatsApp | `evolution_valeng` (**chave errada — 401; trocar**) | httpHeaderAuth (`apikey`) | `dpb9deNkG8TenXdV` |

## Configuração manual necessária

- [x] **Config**: `evolutionUrl` = `https://evolution.vendasx.com`, `instancia` = `rgis`,
      `numeroDestino` = `5562983013515`.
- [ ] **Credencial**: `evolution_valeng` devolve **401 Unauthorized** na instância `rgis`.
      Criar no n8n uma credencial `httpHeaderAuth` (ex.: `evolution_rgis`) com
      Name = `apikey` e Value = chave da instância `rgis`, e trocar no nó "Enviar WhatsApp".

## Como testar

Os gatilhos não são executáveis pelo MCP. Usar um fluxo temporário "TESTE alerta"
(Webhook → Execute Workflow `ktpi4gsUslWWAWyG` com `titulo`/`mensagem`), executar duas
vezes seguidas e conferir: **uma** mensagem recebida (a segunda barrada pelo anti-spam).
Arquivar o fluxo temporário depois.

## Limitações conhecidas

- Enquanto a credencial estiver errada, o envio falha (o erro fica só no histórico de
  execuções do próprio Alerta; ele não tem `errorWorkflow`, para não entrar em loop).
- Anti-spam só funciona em execuções de produção (dados estáticos não persistem em testes manuais).

## Histórico de decisões

- 2026-10-02 — Teste real (execução 304792): chegou à Evolution, mas 401 com a credencial
  provisória. Fluxo publicado mesmo assim (o n8n só aceita fluxo de erro publicado).

- 2026-10-02 — Canal WhatsApp via Evolution (escolha do usuário). Anti-spam de 30 min
  porque gatilhos de polling a cada minuto poderiam gerar 60 alertas/hora. Um único fluxo
  centraliza erro técnico e aviso de negócio.
