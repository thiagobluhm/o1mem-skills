# ⏰ handover-nudge-hook — o gatilho do ciclo

As skills `handover` e `organizador-mem` resolvem *como* estancar e limpar o contexto.
Falta *quando* disparar — e "quando" é justo o que a gente esquece no meio de uma
tarefa boa. Este hook fecha essa lacuna: mede o contexto a cada turno e, ao passar de
um limiar, **sugere** um `/handover`.

## O que ele mede (e por que não é o total)

O número que importa **não** é o total da janela. `system prompt`, `tools`, `memória`
e `skills` são ~fixos por sessão — não é isso que o handover economiza. O que ele
economiza é o custo de **re-pagar a CONVERSA** ao arrastar a sessão para frente.

Então o hook mede **crescimento da conversa = `total_atual − baseline_da_sessão`**,
lendo o campo `usage` (`input + cache_creation + cache_read`) do primeiro e do último
turno do transcript. É esse delta que cruza o limiar.

> Evidência real (uma sessão): retomar via handover custou **19.5k** de `Messages`
> contra os **124.8k** que a sessão inchada carregava — o fio voltou por ~16% do custo.
> Ver a seção "Evidência" no README raiz.

## As duas travas que impedem o tiro no pé

1. **Trava de valor.** O texto injetado carrega a disciplina do Passo 0 como **fato**:
   um handover só vale com estado durável a preservar; exploração descartável ou tarefa
   concluída dispensa — "aqui basta memória". Sem isso, o automatismo fabricaria
   handovers vazios com timestamp.
2. **Rota de silêncio.** A decisão fica com o usuário: *preparar handover* / *agora não*
   / **silenciar nesta sessão**. Sem repetição entre níveis — o aviso de 150k não cai em
   cima de um "agora não" de 80k. É o antídoto da fadiga de alerta, que mataria o
   mecanismo.

## Por que o texto injetado é *factual*, não imperativo (isso é crítico)

A [doc oficial de hooks](https://code.claude.com/docs/en/hooks) é explícita: **escreva o
contexto injetado como afirmações factuais, não como instruções de sistema imperativas.**
Texto em registro de comando out-of-band ("faça X", "NÃO faça Y") dispara as defesas
anti-prompt-injection do Claude — que passa a **exibir** o texto na sua tela em vez de
tratá-lo como contexto. Ou seja: injetar *"ofereça um handover ao usuário"* é justo a
forma que **falha**. Este hook afirma só fatos — *"a conversa cresceu 82k tokens; o
limiar foi cruzado; um handover só vale com estado durável; silenciar é gravar tal
arquivo"* — e deixa o Claude reagir. É a diferença entre o hook funcionar e virar uma
linha estranha na tela.

## Restrições do `UserPromptSubmit` (respeitadas no script)

Só três eventos injetam stdout que o Claude vê: `UserPromptSubmit`, `UserPromptExpansion`
e `SessionStart` — por isso é este o evento. Mas ele roda **antes** de cada prompt e
**bloqueia** o processamento até terminar, então:

- **Timeout baixo (30s default).** Um hook travado congela a sessão. Registre com
  `"timeout": 10` no `settings.json` e mantenha o script rápido (este lê só 1ª/última
  linha de `usage`).
- **Nunca sair com exit 2.** No `UserPromptSubmit`, exit 2 **bloqueia o processamento e
  apaga o prompt que você digitou**. Este script é *fail-open absoluto*: qualquer erro
  sai 0 silencioso. Um bug no medidor jamais pode engolir o seu prompt.

## Rede de segurança final (roadmap): `PreCompact/auto`

O `UserPromptSubmit` avisa **cedo** (por limiar). Falta a **última chamada**: o evento
`PreCompact` dispara antes da compactação de contexto e, com matcher `auto`, intercepta
o momento exato em que o *autocompact* — o concorrente real do handover — ia comprimir
sem você escolher o quê. `PreCompact` pode inclusive **bloquear** a compactação (exit 2).
Combinar os dois cobre "avisei cedo" + "última chamada antes da perda". Ainda não
incluído aqui porque o stdout de `PreCompact` **não** é canal visível (ver acima) — o
aviso teria de vir por uma ponte via marca de estado + o próximo `UserPromptSubmit`.

## Calibragem honesta (n=1)

O limiar default (**80k**) é chute educado de **uma** sessão observada — ordem de
grandeza, não verdade. Por isso:

- é **configurável** (nunca hardcoded): `handover-nudge.config.json` ou env
  `CLAUDE_HANDOVER_NUDGE_THRESHOLD` / `_STEP`; `CLAUDE_HANDOVER_NUDGE_DISABLE=1` desliga;
- cada aviso é **logado** em `~/.claude/handover-nudge.log` (`nudge_emitted` com o
  crescimento + `nudge_outcome` com accepted/declined/silenced).

Depois de 10–15 sessões, olhe o log e ajuste o número **com dado**. Se você ignora todo
aviso de 80k e só age nos de 150k, o limiar real é outro — o log te conta.

## Instalação

1. Copie `handover_nudge.py` e `handover-nudge.config.json` para `~/.claude/`
   (o `.py` pode ir em `~/.claude/hooks/`).
2. Registre o hook no seu `~/.claude/settings.json` (global) ou no
   `.claude/settings.json` do projeto:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          { "type": "command", "command": "python \"C:/Users/SEU_USUARIO/.claude/hooks/handover_nudge.py\"" }
        ]
      }
    ]
  }
}
```

3. Ajuste o caminho do `command` ao seu SO. Hooks carregam no **início** da sessão do
   Claude Code — abra uma sessão nova para ativar.

**Fail-open absoluto:** qualquer erro (transcript ausente, JSON malformado, etc.) faz o
hook sair silencioso sem imprimir nada. Ele nunca bloqueia o seu turno.
