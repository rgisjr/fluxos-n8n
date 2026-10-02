# Alerta WhatsApp

## Objetivo

Avisar no WhatsApp (via Evolution API) quando:
- um fluxo **falha** em produção (este é o `errorWorkflow` padrão de todos os fluxos);
- um fluxo precisa dar um **aviso de negócio** (ex.: "Lead não encontrado na Clint").

## Link e ID no n8n

- ID: `ktpi4gsUslWWAWyG`
- https://n8n.grupotoda.com.br/workflow/ktpi4gsUslWWAWyG
- Não precisa estar publicado: fluxos de erro e sub-fluxos rodam mesmo inativos.

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
| Enviar WhatsApp | `evolution_valeng` (**provisória — confirmar**) | httpHeaderAuth (`apikey`) | `dpb9deNkG8TenXdV` |

## Configuração manual necessária

- [ ] Preencher no nó **Config**: `evolutionUrl` (sem `/` no final), `instancia`, `numeroDestino`
      (formato `55DDDNÚMERO`). Hoje estão como `PENDENTE`.
- [ ] Confirmar se `evolution_valeng` é a credencial certa; se não, indicar/criar outra
      `httpHeaderAuth` com header `apikey`.

## Como testar

Os gatilhos não são executáveis pelo MCP. Usar um fluxo temporário "TESTE alerta"
(Webhook → Execute Workflow `ktpi4gsUslWWAWyG` com `titulo`/`mensagem`), executar duas
vezes seguidas e conferir: **uma** mensagem recebida (a segunda barrada pelo anti-spam).
Arquivar o fluxo temporário depois.

## Limitações conhecidas

- Enquanto o **Config** estiver `PENDENTE`, o envio falha (o erro fica só no histórico de
  execuções do próprio Alerta; ele não tem `errorWorkflow`, para não entrar em loop).
- Anti-spam só funciona em execuções de produção (dados estáticos não persistem em testes manuais).

## Histórico de decisões

- 2026-10-02 — Canal WhatsApp via Evolution (escolha do usuário). Anti-spam de 30 min
  porque gatilhos de polling a cada minuto poderiam gerar 60 alertas/hora. Um único fluxo
  centraliza erro técnico e aviso de negócio.
