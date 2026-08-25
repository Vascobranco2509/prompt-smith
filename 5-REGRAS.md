# 5. As regras

O `regras\regras.md` muda o comportamento da aplicacao **sem recompilar nada**.
Editas num editor de texto, gravas, e a resposta seguinte ja usa. Nem sequer e
preciso recriar o modelo no Ollama.

O ficheiro completo esta em [codigo/regras.md.md](codigo/regras.md.md).

---

## O formato, e a armadilha

Cada regra e uma linha com **o nome em portugues, uma barra vertical, e a regra em
ingles**. A aplicacao so envia ao modelo a parte a direita da barra.

```
Papel        | Open with one line saying who the model is, for example You are a...
Tarefa       | State exactly one task, as an instruction, not as a wish
Publico      | Say who the answer is for
```

Porque em ingles: o modelo e pequeno e segue instrucoes em ingles com bastante mais
fiabilidade. O que **tu** les continua em portugues, que e a parte da esquerda.

> **A armadilha.** Uma linha de texto explicativo que tenha uma barra vertical passa
> a ser lida como regra. Ja aconteceu: uma frase que dizia "Formato: Nome que ves |
> instrucao" criou uma accao chamada *"Formato: Nome que ves"* no menu.
>
> **Regra: nas linhas de explicacao, nunca ponhas uma barra vertical.**

---

## As seis seccoes

| Seccao | Quantas | O que faz |
|---|---|---|
| `## CHECKLIST` | 8 | os criterios que a segunda passagem usa para corrigir o prompt |
| `## PALAVRAS-VAGAS` | 19 | palavras que a aplicacao assinala no fim do diagnostico |
| `## DESTINOS` | 3 | como adaptar o prompt ao ChatGPT, ao Gemini ou ao Claude |
| `## O QUE MUDEI` | 3 | como redigir a explicacao do que foi alterado |
| `## PT-PT` | 23 | correccoes de portugues do Brasil para o de Portugal |
| `## ACCOES` | 12 | o menu do modo "Reescrever texto" |

---

## `## CHECKLIST` — o que mais rende

E a lista que a segunda passagem usa para reescrever o prompt. Foi o que deu mais
resultado de tudo o que se tentou: tres prompts que pontuavam 3/7, 1/7 e 2/7
subiram todos para **6/7** so por existir esta passagem.

**Mantem-nas curtas e imperativas.** Acima de umas 10 o modelo comeca a ignorar as
ultimas — testado.

---

## `## PT-PT` — porque isto esta em codigo

O modelo escreve portugues do Brasil de vez em quando. Tentou-se corrigir por
instrucao, tres vezes, e as tres pioraram o resto da resposta.

A solucao foi trocar as palavras **depois** de o modelo responder, com uma tabela.
Resolveu a primeira e nunca mais falhou.

```
usuário      | utilizador
tela         | ecra
arquivo      | ficheiro
```

Detalhe que custou uma tentativa falhada: a troca tem de ser **tolerante a acentos**,
senao `usuarios` era apanhado e `usuários` escapava.

---

## `## ACCOES` — o menu de reescrever

Cada linha e uma entrada do menu. Acrescentas uma linha, aparece no menu na proxima
vez que abrires a aplicacao.

```
Resumir em 5 pontos     | Summarise this text in exactly five bullet points
Encurtar para metade    | Rewrite this text at half the length, same meaning
Tirar as tarefas        | List every task in this text, one per line, with who and when
```

Este e o modo mais fiavel de todos, e a razao e simples: **o conteudo vem de ti**.
O modelo nao tem nada para inventar — so tem de transformar.

---

## O que **nao** esta aqui

O comportamento de fundo — a entrevista, a estrutura de 5 pontos do diagnostico, os
exemplos — nao esta no `regras.md`. Esta no `Modelfile`, e mexer la **obriga** a
recriar o modelo:

~~~~
ollama create prompt-smith -f Modelfile
~~~~

Antes de mexeres la, le o [6-LICOES.md](6-LICOES.md). Ha coisas nesse ficheiro que
parecem melhorias obvias e que foram medidas a piorar o resultado.
