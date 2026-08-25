# prompt-smith

Uma equipa de agentes e um punhado de ferramentas de texto, a correr **no teu
computador**. Sem contas, sem nuvem, sem mensalidade. Janela Windows, compilada
com o compilador que ja vem no sistema — nao instalas ambiente de programacao
nenhum.

Este repositorio tem **so ficheiros `.md`**. O codigo verdadeiro esta dentro
deles, em blocos, e sai com um comando. Ver [2-CONSTRUIR.md](2-CONSTRUIR.md).

---

## O que e

```
+--------------+-------------------------------------+-----------------+
| prompt-smith |  (*) Investigador       [Claude]     | FICHA DO AGENTE |
|              |      Pesquisa de mercado            |                 |
| A MINHA      +-------------------------------------+ Nome  [_______] |
| EQUIPA       |                                     | Cargo [_______] |
| (*) Investig.|          +------------------------+ | Missao          |
|     Pesquisa |          | Quem concorre connosco?| | [_____________] |
| (*) Estratega|          +------------------------+ | [_____________] |
|     Posicion.|                                     | Cerebro         |
| (*) Critico  |  +--------------------------+       |  (o) Claude     |
|     Revisao  |  | Os principais rivais sao |       |  ( ) local      |
| (*) Revisor  |  | X, Y e Z. Fontes: ...    |       | Ferramentas     |
|     Clareza  |  +--------------------------+       |  [x] WebSearch  |
|  + agente    |                                     |  [x] WebFetch   |
|              |                                     |  [ ] ler-ficheiro|
| FERRAMENTAS  |  +--------------------------------+ | Cor  * * * * *  |
|  Construir   |  | +  escreve aqui...    [enviar] | |                 |
|  Diagnosticar|  +--------------------------------+ | [   Gravar   ]  |
|  ...         |  copiar   guardar .md    11 seg     | [  Arquivar  ]  |
+--------------+-------------------------------------+-----------------+
```

**A esquerda, a tua equipa.** Cada agente e um ficheiro de texto com um nome, um
cargo, uma missao e um conjunto de ferramentas. Falas com ele em conversa, e ele
lembra-se do que disseste antes — a conversa fica guardada no ficheiro dele e
volta quando reabres a janela.

**Cada agente tem um de dois cerebros:**

| Cerebro | Quanto demora | O que consegue | Quanto custa |
|---|---|---|---|
| **local** | 1 a 5 segundos | moer texto que tu lhe das | nada |
| **Claude** | 20 a 120 segundos | pesquisar na net e citar fontes | a tua subscricao do Claude Code |

**Em baixo a esquerda, as ferramentas.** Oito coisas que nao precisam de agente:

- **Construir** um prompt do zero, por entrevista, uma pergunta de cada vez.
- **Diagnosticar** um prompt que ja tens, sempre na mesma estrutura de 5 pontos.
- **Imagem para markdown**, pelo OCR do Windows ou por um modelo de visao.
- **Reescrever texto teu**: resumir, encurtar, traduzir, tirar tarefas.
- **Biblioteca** dos prompts que guardaste, com campos para preencher.
- **Conversar com um ficheiro**: PDF, Word, txt, md, csv.
- **Procurar dentro dos ficheiros** de uma pasta — pelo conteudo, nao pelo nome.
- **Encontrar duplicados** pelo conteudo. Nunca apaga: move, e podes desfazer.

---

## Duas regras que nao se mexem

**Nenhum agente inventa numeros.** A instrucao esta em todos: se um numero nao lhe
foi dado, diz que nao o tem. Serve para escrever *sobre* os teus dados, nunca para
os produzir.

**Pesquisa sem fontes leva aviso.** Quando um agente com ferramentas de internet
responde sem um unico link, a aplicacao acrescenta em baixo:

```
(atencao: esta resposta veio sem fontes, confirma antes de a usar)
```

Isto e verificado em codigo depois da resposta, e nao pedido ao modelo. Um modelo
pode desobedecer a uma instrucao; uma expressao regular nao.

---

## Do que precisas

| | |
|---|---|
| **Windows** | 10 ou 11. A janela e WPF e o OCR e o do proprio Windows |
| **Ollama** | gratuito — [ollama.com](https://ollama.com). E o que corre o modelo local |
| **Placa grafica** | 4 GB de memoria chegam. Foi desenvolvido numa NVIDIA T500 |
| **Espaco** | ~2,5 GB para o modelo |
| **Compilador** | ja vem no Windows (`csc.exe` da .NET Framework 4). Nada a instalar |
| **Claude Code** | **opcional**. So se quiseres agentes que pesquisam na internet |

Sem placa grafica tambem anda, mas em processador — contem com 4 a 6 vezes mais
lento.

---

## Comecar

1. [**1-INSTALAR.md**](1-INSTALAR.md) — o Ollama e o modelo. Uns 10 minutos.
2. [**2-CONSTRUIR.md**](2-CONSTRUIR.md) — tirar o codigo daqui e compilar. Um comando.
3. Duplo clique no `prompt-smith.exe`.

---

## O resto do que esta aqui

| Ficheiro | Para que serve |
|---|---|
| [3-ESPECIFICACAO.md](3-ESPECIFICACAO.md) | como o ecra esta desenhado, com as medidas, e os quatro contratos que o codigo impoe. Le isto antes de mexer no layout |
| [4-AGENTES.md](4-AGENTES.md) | o que e um agente, o formato do ficheiro, e como fazer os teus |
| [5-REGRAS.md](5-REGRAS.md) | o `regras.md`, que muda o comportamento sem recompilar |
| [6-LICOES.md](6-LICOES.md) | **o mais util.** O que foi medido a tirar trabalho a serio de um modelo de 4 mil milhoes de parametros: o que resultou, e o que piorou |
| `codigo/` | os ficheiros de codigo, cada um dentro de um bloco de markdown |

---

## O que isto nao e

- **Nao e um produto.** E o que uma pessoa construiu para o trabalho dela e
  resolveu partilhar. Nao ha suporte, nao ha instalador, nao ha assinatura digital.
- **O modelo local e pequeno.** 4 mil milhoes de parametros. Resume, reescreve,
  extrai e corrige bem. Nao sabe factos e escreve portugues do Brasil de vez em
  quando — ha correccoes automaticas em codigo para isso, mas nao apanham tudo.
- **Nao ha versao para Mac nem para Linux.** A janela e WPF e o OCR e do Windows.
- **Os agentes nao falam uns com os outros** e nao trabalham sozinhos a horas
  certas. Falam contigo, quando tu falas com eles.

---

## Licenca

MIT — ver [LICENSE.md](LICENSE.md).

Em portugues simples: **faz o que quiseres com isto.** Usar, copiar, mudar,
distribuir, vender. A unica condicao e manteres o aviso de direitos de autor.
Vem sem garantia nenhuma: se correr mal, o problema e teu.
