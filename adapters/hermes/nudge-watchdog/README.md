# nudge-watchdog (Hermes) — detalhes técnicos

Cron watchdog que substitui o hook `UserPromptSubmit` do Claude Code. Roda a cada ~10 min, mede o crescimento da conversa da sessão ativa e sugere `/skill handover` ao cruzar o limiar. **Fail-open absoluto:** qualquer erro sai `0` sem imprimir nada.

## Como mede o crescimento

Lê o SQLite do Hermes (`~/AppData/Local/hermes/state.db`):

1. **Sessão ativa** — `SELECT ... FROM sessions WHERE ended_at IS NULL ORDER BY started_at DESC LIMIT 1`. O total de contexto = `input + cache_read + cache_write` (o que ocupa a janela).
2. **Crescimento da conversa** — cascata de fallback, porque o Hermes nem sempre popula `token_count`:
   - **Tentativa 1:** `SUM(token_count)` das mensagens não-compactadas (`compacted = 0`).
   - **Tentativa 2 (proxy):** `SUM(LENGTH(content) + LENGTH(tool_calls) + LENGTH(reasoning_content)) // 4` — ~4 chars por token.

   É o **crescimento da conversa** que importa, não a janela total: `system`/`tools`/`memória` são ~fixos; o que o handover economiza é o custo de arrastar a *conversa*.

## Estado e reaviso

- Estado por sessão: `~/AppData/Local/hermes/handover-nudge-state/<session_id>.json` — `{"last_level": N, "silenced": bool}`.
- Escalada: primeiro aviso em `threshold` (80k), reavisa a cada `reset_step` (70k) — `level = threshold + idx*step`. Só emite se `level > last_level` (nunca repete o mesmo nível).
- Silenciar uma sessão: gravar `{"silenced": true}` no JSON dela.
- Log de eventos (JSONL): `~/AppData/Local/hermes/handover-nudge.log`.

## Config

`~/AppData/Local/hermes/handover-nudge.config.json`:

```json
{ "threshold": 80000, "reset_step": 70000 }
```

Precedência: env (`HERMES_HANDOVER_NUDGE_THRESHOLD`, `_STEP`, `_DISABLE=1`) > `config.json` > default (80k / 70k). Números são `n=1` — ordem de grandeza; calibre com o log ao longo de 10–15 sessões.

## Registrar o cron

Via a ferramenta `cronjob` do Hermes, a cada 10 min, com `no_agent=True` (padrão watchdog: stdout vazio = nada entregue; stdout não-vazio = entregue).

> ⚠️ **Entrega:** com `deliver=local` no TUI o output é salvo mas **não** te notifica ativamente. Para receber o toque de fato, use `deliver=telegram` (ou outro gateway). Ver a seção "o preço honesto" em [`../README.md`](../README.md).

## Texto do aviso

FACTUAL, não imperativo (mesma disciplina anti-prompt-injection do hook original): informa o crescimento, o limiar cruzado, a trava de valor ("handover só vale se há estado durável") e termina em *"se for útil, rode `/skill handover`"* — nunca uma ordem.
