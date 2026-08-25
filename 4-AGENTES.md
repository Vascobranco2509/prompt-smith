# 4. Os agentes

Um agente e **um ficheiro de texto**. Nao ha base de dados, nao ha configuracao
escondida. Abres, mudas, gravas, e vale na resposta seguinte.

Estao em `agentes\`. Os seis que vem com isto estao em
[codigo/agentes.md](codigo/agentes.md), com a memoria vazia.

---

## O formato

~~~~markdown
---
nome: Investigador
cargo: Pesquisa de mercado
cor: #9159fe
missao: Investigas na internet concorrencia, mercado, tendencias e anuncios de
  outras marcas. Citas SEMPRE as fontes com links. Quando nao encontras, dizes que
  nao encontraste em vez de preencheres com suposicoes.
cerebro: claude
ferramentas: WebSearch, WebFetch
---

## memoria

### 2026-08-25 19:27
**Tu:** a pergunta que fizeste
a resposta que ele deu
~~~~

| Campo | O que faz |
|---|---|
| `nome` | como aparece na lista. Se faltar, usa-se o nome do ficheiro |
| `cargo` | a linha pequena por baixo do nome. Opcional |
| `cor` | a bolinha na lista. Opcional: sem ela, recebe uma pela posicao |
| `missao` | **o que mais importa.** E isto que diz ao agente o que fazer |
| `cerebro` | `local` ou `claude` |
| `ferramentas` | separadas por virgulas. Ver a tabela abaixo |

Tudo o que estiver por baixo de `## memoria` e a conversa. Enche-se sozinha e fica
**so no teu computador**.

---

## Os dois cerebros

| | `local` | `claude` |
|---|---|---|
| Quem responde | o modelo no teu computador | o Claude Code que tiveres instalado |
| Demora | 1 a 5 segundos | 20 a 120 segundos |
| Internet | **nenhuma** | se tiver as ferramentas |
| Custo | zero | a tua subscricao |
| Bom para | moer texto que tu lhe das | investigar, estruturar, criticar |

O cracha no topo da conversa diz sempre qual esta a ser usado. Nunca ficas na
duvida sobre se aquilo foi de graca ou nao.

---

## As ferramentas

| Ferramenta | O que da | So com que cerebro |
|---|---|---|
| `WebSearch` | procurar na internet | `claude` |
| `WebFetch` | abrir uma pagina e ler | `claude` |
| `ler-ficheiro` | quando a mensagem traz um caminho, o ficheiro e lido e vai junto | os dois |

Um agente `local` com `WebSearch` na lista **nao ganha internet nenhuma** — o
modelo corre no teu computador e nao tem por onde sair. A ficha avisa-te disso.

### A limitacao e real, nao decorativa

Isto foi verificado. Perguntei a um agente com `cerebro: claude` mas **sem**
`WebSearch` nem `WebFetch` uma coisa que exigia internet — quantos anuncios uma
marca estava a passar naquela semana. A resposta comecou assim:

> Nao consigo responder a isso, e nao vou inventar um numero.

E explicou que as vias de pesquisa estavam bloqueadas. **Nao inventou.** O mesmo
pedido, feito ao agente com as duas ferramentas ligadas, veio com nomes certos e
links verdadeiros.

A razao tecnica: a aplicacao passa as ferramentas explicitamente na chamada ao
Claude Code (`--allowedTools WebSearch WebFetch`). Sem esse argumento, o Claude
bloqueia essas ferramentas e **diz que o fez** em vez de fingir.

---

## A conversa tem fio

O agente le os ultimos turnos do proprio ficheiro antes de responder. Por isso isto
funciona:

```
tu:  Quem sao os principais concorrentes desta marca?
ele: Sao a X, a Y e a Z, por estas razoes...

tu:  E qual deles achas que investe mais em publicidade?
     ^ sem repetir um unico nome
ele: A X - dona da Y e da Z - e de longe a que mais investe.
```

Quanto e que ele le:

| Cerebro | Turnos | Porque |
|---|---|---|
| `local` | os ultimos **3**, com as respostas cortadas a 400 caracteres | so tem 4096 de contexto; mais do que isto estoura |
| `claude` | os ultimos **6**, inteiros | tem contexto de sobra |

Fechas a aplicacao, voltas a abrir, clicas no agente: a conversa esta la, porque
esta no ficheiro.

---

## Fazer os teus

**+ agente novo** na barra lateral cria um ficheiro com um esqueleto. Mudas o nome,
o cargo e a missao na ficha a direita, escolhes o cerebro e as ferramentas, e
carregas em **Gravar**. Grava no `.md` sem tocar na memoria.

Tambem podes criar o ficheiro a mao em `agentes\` e voltar a clicar em
**A minha equipa**.

**Arquivar este agente** move o ficheiro para `agentes\_arquivo\`. **Nunca apaga
nada** — se te arrependeres, arrastas de volta.

### Escrever uma missao que funciona

O que se aprendeu a escrever estas seis:

1. **Uma tarefa, nao tres.** "Les um briefing e dizes o objectivo, o publico-alvo,
   as entregas e sobretudo o que falta" funciona. "Les, resumes, criticas e
   propoes" sai confuso, sobretudo no cerebro local.
2. **Diz o que fazer quando nao sabe.** Sem isso, um modelo preenche o vazio.
   Todas as seis missoes tem uma frase deste tipo: *"quando o texto nao diz quem,
   escreves nao especificado em vez de adivinhares"*.
3. **Curta.** Duas ou tres frases. Missoes compridas pioram o resultado — isto foi
   medido varias vezes, ver [6-LICOES.md](6-LICOES.md).

---

## As duas regras de seguranca

Estao no codigo, nao na missao. Um agente nao lhes pode fugir editando o `.md`.

**Nenhum agente produz numeros.** Vai em todos os pedidos, em ingles:

```
Never invent numbers, percentages or statistics.
If a number is not given to you, say you do not have it.
```

**Pesquisa sem fontes leva aviso.** Quando um agente com ferramentas de internet
responde sem um unico link, a aplicacao acrescenta:

```
(atencao: esta resposta veio sem fontes, confirma antes de a usar)
```

Isto e uma verificacao em codigo, depois da resposta. Nao e um pedido ao modelo —
um modelo pode desobedecer, uma expressao regular nao.
