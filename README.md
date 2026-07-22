# 🪙 skills — Token Economy for Claude Code

### Duas skills para parar de pagar pedágio de contexto a cada sessão
🇧🇷 Made in Brazil

---

![status](https://img.shields.io/badge/status-battle--tested-green)
![tool](https://img.shields.io/badge/tool-Claude%20Code-8A2BE2)
![focus](https://img.shields.io/badge/focus-token%20economy-brightgreen)
![approach](https://img.shields.io/badge/chunking-agentic-blue)
![lang](https://img.shields.io/badge/docs-pt--BR-yellow)

---

## ⚡ TL;DR

> **Contexto é caro e finito. Estas duas skills param o vazamento e limpam o que já vazou.**

- `organizador-mem` — **enxuga** um arquivo de contexto grande (`CLAUDE.md`, memória) separando o que é sempre-relevante do que é sob-demanda.
- `handover` — **estanca** a perda de fio ao dar `/clear`, destilando a sessão em 3 camadas de custo + um cap que impede a memória de inflar de novo.

👉 Uma faz a faxina. A outra impede que suje de novo. Juntas fecham o ciclo.

---

## 🔥 A dor (talvez você reconheça)

Se você usa Claude Code todo dia, você já sentiu isso: a sessão incha, o `CLAUDE.md` vira um monstro que o modelo relê inteiro toda vez, e dar `/clear` significa perder o fio de uma tarefa pela metade. Eu vivi exatamente esse ciclo — e essas duas skills nasceram dele.

O problema, no fundo, é um só: **toda sessão paga pedágio** para reler os arquivos de instrução do projeto (`CLAUDE.md`, memória, handovers). Estas skills atacam esse custo por duas frentes que se completam.

> ⚠️ **Sobre os percentuais:** os números abaixo são **casos reais que eu observei**, não promessa. O ganho depende do tamanho do seu arquivo e de quanto dele é "sempre-relevante" versus "sob demanda". Trate como ordem de grandeza.

> 🌐 **Agnósticas de domínio.** Nasceram num projeto real meu, mas a mecânica serve qualquer repo com um arquivo de contexto grande demais ou uma memória que precisa sobreviver ao `/clear`. Os exemplos dentro de cada `SKILL.md` são só isso — exemplos.

---

## 🧹 `organizador-mem`

**A dor.** Meu `CLAUDE.md` tinha mais de 1500 linhas. Toda sessão lia tudo — mesmo quando 90% daquilo não tinha nada a ver com a tarefa do dia. Eu pagava, a cada turno, por regras de subsistemas que eu nem ia tocar.

**O que ela faz.** Separa o arquivo grande em **núcleo sempre-relevante** + **documentos-satélite sob demanda**, ligados por um *mapa* enxuto. O corte de cada pedaço é decidido por um **agente que lê e entende a semântica** — não é split cego por regex ou heading. Quando dois trechos parecem acoplados, ou uma seção cabe em dois tópicos, a skill **para e pergunta** antes de aplicar. Aprendi da pior forma que split mecânico fragmenta raciocínio ao meio.

**Por que melhora.** O modelo passa a ler o núcleo curto + o mapa, e só abre o satélite que a tarefa realmente toca. O custo de leitura deixa de ser "o arquivo todo" e vira "núcleo + o que importa agora".

**Quanto rendeu.**

| Antes | Depois | Redução |
|---|---|---|
| `CLAUDE.md` ~1589 linhas lidas/sessão | ~150 linhas de núcleo + mapa | **~90%** |

Faixa típica que eu esperaria: **60–90%**, quando a maior parte do arquivo é tópico-específica.

**A intuição, em uma frase:** *nem toda regra é sempre relevante.* Princípios inegociáveis são núcleo — todo turno. A lei de um subsistema só importa quando você mexe nele. O mapa preserva a *descoberta* ("existe uma regra sobre X, abra tal doc") sem pagar o *conteúdo* até precisar.

**O que controla em `.claude/`.** Vive em `.claude/skills/organizador-mem/SKILL.md`. **Reorganiza** o seu `.claude/CLAUDE.md` (ou qualquer arquivo que você apontar) e cria a pasta de satélites ao lado (ex.: `documentacao/regras/`). Não toca em código — só na camada de instrução que o Claude carrega.

---

## 📤 `handover`

**A dor.** Sessão inchou, tarefa pela metade, e você fica no dilema: carregar a conversa inteira pra frente (caríssimo) ou dar `/clear` e recomeçar reexplicando tudo (lento — e você SEMPRE esquece um porquê importante no caminho).

**O que ela faz.** Prepara a **saída limpa** da sessão. Escreve **um** documento seletivo em `documentacao/` — seletivo é regra, não adjetivo: só entra o que git + código + memória **não** contam sozinhos (o *porquê* das decisões com a alternativa descartada, o estado pendente, o próximo passo exato, os riscos). Atualiza um breadcrumb enxuto na memória e declara um **modo de retomada**: `rapida` (próximo passo não toca runtime) ou `verificada` (toca — e aí a sessão nova é obrigada a reconferir o estado vivo antes de afirmar qualquer coisa).

**Por que melhora.** Distribui o estado em **3 camadas de custo diferente**:

| Camada | Carrega quando | Custo |
|---|---|---|
| **Resume** | você retoma *esta* conversa | alto — e morre no `/clear` |
| **Memória-índice** | **toda** sessão nova | baixo — breadcrumb terso que aponta |
| **Handover-arquivo** | só quando alguém o abre | zero até ser aberto |

Cada informação fica na camada mais barata que ainda a entrega a tempo.

**Quanto rendeu.** O maior ganho é **estrutural** — e foi um bug meu que me ensinou. A 1ª versão preservava o histórico de retomadas para sempre: cada handover depositava uma linha permanente no índice, crescendo **O(n)** sem ninguém perceber. Esta versão traz um **cap de histórico** (no máximo as **2** retomadas anteriores; o resto delega aos ponteiros duráveis) → crescimento **O(1)**.

| Aspecto | Sem cap | Com cap |
|---|---|---|
| Índice de memória | 96 linhas e subindo | **65 linhas (~32%)**, estável |
| Crescimento por sessão | +1 linha permanente (O(n)) | limitado (O(1)) |

Sem o cap, o índice voltaria a inflar em semanas — eu só descobri olhando o painel de context usage e me perguntando por que a memória pesava tanto.

**A intuição, em uma frase:** *a memória é o ÍNDICE — aponta, não repete.* A camada que carrega toda sessão tem que ser a mais enxuta possível: só precisa dizer **qual arquivo abrir** e **qual o próximo passo**. E como "o que era verdade quando escrevi" ≠ "o que é verdade agora", o modo `verificada` existe para uma coisa: economia de token **nunca** vale uma afirmação falsa sobre o runtime.

**O que controla em `.claude/`.** Vive em `.claude/skills/handover/SKILL.md`. **Escreve** o handover em `documentacao/` e **mantém** o índice de memória (`MEMORY.md` + `memory/*.md`) enxuto e capado. É a disciplina de *entrada* da memória; o `organizador-mem` é a *faxina*.

---

## 🔗 Por que as duas juntas

Aqui está a parte que eu demorei a enxergar: `handover` **alimenta** a memória a cada saída de sessão; `organizador-mem` a **reorganiza** quando ela incha. Sem a primeira disciplinada (com o cap), a segunda vira **enxugar gelo** — cada handover deposita mais uma linha e o índice que você acabou de emagrecer engorda de novo. Juntas: entrada capada + faxina agêntica.

---

## 🚀 Como usar

Cada pasta tem um `SKILL.md` autocontido (frontmatter `name` + `description`). Coloque a pasta em um diretório de skills que o seu setup leia — tipicamente `.claude/skills/<nome>/SKILL.md` no projeto, ou o diretório global. O Claude carrega a skill quando a tarefa casa com a `description`, ou quando você a chama pelo nome.

O ciclo de vida de uma sessão longa com o `handover` é sempre este:

| Você quer... | Comando | O que acontece |
|---|---|---|
| **Fechar a sessão sem perder o fio** | rode `/handover` | destila a tarefa em handover + memória e declara o modo de retomada |
| **Limpar o contexto** | `/clear` | zera esta conversa — o handover e a memória já estão em disco, nada se perde (só **você** executa; o modelo não pode) |
| **Retomar depois** | sessão NOVA → *"retomar o handover"* (ou `/handover`) | lê a linha `RETOMADA` do `MEMORY.md`, abre o handover indicado e segue o modo gravado |

👉 A regra de ouro: **`/clear` só depois do `/handover`**. O handover é o que torna o `/clear` seguro.

Se testar num projeto seu e os números baterem (ou não baterem), me conta — os percentuais daqui só valem o que valem porque vieram de caso real, e mais casos reais só melhoram a calibragem.

---

## 🇧🇷 Made in Brazil

Feitas para:

- projetos reais que incharam de verdade
- restrições de custo de token
- agentes que precisam lembrar sem pagar caro por isso
