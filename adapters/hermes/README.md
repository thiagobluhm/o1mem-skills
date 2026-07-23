# O(1)mem — porta para o Hermes Agent

O mesmo sistema (índice capado + handover em arquivo + gatilho), adaptado para rodar no **[Hermes Agent](https://github.com/)** em vez do Claude Code. A tese é idêntica; muda só o substrato de ferramentas e — no gatilho — o **modelo de execução**.

> **Por que existe esta pasta.** O O(1)mem nasceu no Claude Code, mas nada nele é do Claude Code: é índice + arquivo + teto O(1). Tirar o lock-in é a evolução natural. As skills aqui são as **mesmas** da raiz, com os nomes de ferramenta traduzidos (ver [`../../PORTABILITY.md`](../../PORTABILITY.md)) e uma diferença real de arquitetura no watchdog, documentada abaixo com honestidade.

## O que muda em relação à raiz (Claude Code)

| | Claude Code (raiz) | Hermes (esta pasta) |
|---|---|---|
| Skills `handover` / `organizador-mem` | idem | mesma espinha; ferramentas traduzidas + `/clear`→`/reset` + Opção B (memory tool nativa) |
| Índice de memória | `MEMORY.md` (+ auto-memory) | `MEMORY.md` **ou** a `memory` tool nativa do Hermes (Opção B, projetos leves) |
| Gatilho ("quando dar handover") | hook `UserPromptSubmit` **síncrono** | cron watchdog **assíncrono** (a cada 10 min) |
| Instalação das skills | `~/.claude/skills/` | `~/AppData/Local/hermes/skills/` |

## A sacada — e o preço honesto dela

O hook original do Claude Code é **síncrono**: roda antes de cada prompt e injeta o aviso direto na conversa. O Hermes não tem esse ponto de entrada, então a porta usa um **cron watchdog** (`no_agent=True`): um script que roda a cada 10 min, lê o `state.db`, e só imprime algo quando o limiar é cruzado (stdout vazio = silêncio).

**Ganho:** o aviso deixou de bloquear o caminho crítico da conversa. É um observador de fora, não um pedágio a cada turno.

**Preço — e isto precisa ficar escrito, não escondido:**

1. **Não é imediato.** O cron tem granularidade de ~10 min; o aviso pode chegar com atraso em relação ao momento em que a conversa cruzou o limiar. O hook síncrono era instantâneo.
2. **`deliver=local` salva, mas não te cutuca.** No TUI, o `deliver=local` grava o output do cron no log/estado — mas **não** empurra uma notificação pra você. Ou seja: o aviso *existe*, mas você só o vê se olhar. Para **realmente** receber o toque, aponte o cron para um gateway ativo (`deliver=telegram` ou outro).
3. **Rede de segurança nativa.** A compressão automática do Hermes (`compression.enabled: true`, `threshold: 0.50`) age como fallback se o nudge passar batido — mas comprimir ≠ destilar: a compressão preserva tokens, o handover preserva *o porquê*. Uma não substitui a outra.

## Considerações honestas (o que você pode querer ajustar)

- **(a) Disparar junto da compressão nativa**, em vez do cron de 10 min — casa o aviso com o momento em que o Hermes já decidiu que a janela está cheia. Mais preciso, menos "polling".
- **(b) `deliver=telegram`** (ou outro gateway) — se você quer de fato ser avisado, e não só ter o aviso salvo. Com `deliver=local` no TUI, o nudge é um bilhete na gaveta.
- **(c) Deixar como está** — as skills já entregam o valor sozinhas; o watchdog é o *extra* que lembra a hora. Se você tem o hábito de dar `/handover` na mão, o gatilho é opcional.

Nenhuma das três é "a certa" — dependem de você querer imediatismo, notificação ativa, ou simplicidade.

## Instalação (Hermes)

```bash
# skills
cp -r adapters/hermes/handover adapters/hermes/organizador-mem ~/AppData/Local/hermes/skills/

# watchdog
cp adapters/hermes/nudge-watchdog/hermes_handover_nudge.py ~/AppData/Local/hermes/scripts/
cp adapters/hermes/nudge-watchdog/handover-nudge.config.json ~/AppData/Local/hermes/
# registre o cron (a cada 10 min, no_agent=True) — ver nudge-watchdog/README.md
```

Detalhes do script (queries do `state.db`, proxy de tokens, estado por sessão, fail-open): [`nudge-watchdog/README.md`](nudge-watchdog/README.md).
Tabela completa de tradução de ferramentas: [`../../PORTABILITY.md`](../../PORTABILITY.md).
