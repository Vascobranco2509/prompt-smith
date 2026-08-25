# regras\regras.md

O ficheiro inteiro esta no bloco abaixo. Grava-o em `regras\regras.md`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `regras\regras.md`

<!-- destino: regras\regras.md | codificacao: bom -->
~~~~markdown
# Regras do prompt-smith

Este ficheiro e lido pela aplicacao. Podes edita-lo num editor de texto.
Nao e preciso recriar o modelo depois de mexer aqui: basta voltar a enviar.

Cada linha tem o nome em portugues, uma barra vertical, e a regra em ingles.
A aplicacao so envia a parte a direita da barra.

---

## CHECKLIST

Estas sao as regras que a segunda passagem usa para corrigir o prompt.
Mantem-nas curtas e imperativas. Acima de 10 o modelo comeca a ignorar as ultimas.

Papel        | Open with one line saying who the model is, for example You are a...
Tarefa       | State exactly one task, as an instruction, not as a wish
Publico      | Say who the answer is for
Formato      | Describe the exact shape of the answer, for example bullets, table, JSON
Tamanho      | Give a length limit in words, lines or items
Proibicoes   | Say plainly what the model must not do
Sucesso      | Give one measurable test of a good answer
Concreto     | Replace vague words like good, impactful or professional with something measurable

---

## PALAVRAS-VAGAS

Palavras que quase sempre indicam um prompt fraco. A aplicacao assinala-as
no diagnostico. Uma por linha.

bom
boa
impacto
impactante
profissional
apelativo
envolvente
criativo
moderno
otimizado
otimizada
eficaz
de qualidade
interessante
cativante
adequado
apropriado

---

## DESTINOS

A forma que o prompt reescrito toma consoante o modelo a que se destina.

Claude  | Put the context inside <context> and </context> tags, then write the instructions after the closing tag.
ChatGPT | Start with a role line beginning with You are, then numbered steps, then the output format at the end.
Gemini  | Start with the task, then the context, and end with an instruction asking for the answer to be split into headed sections.

---

## O QUE MUDEI

Frase que a aplicacao escreve no fim quando adapta a um destino. E escrita por
nos, e nao pelo modelo, porque o modelo dizia que tinha feito coisas que nao fez.

Claude  | passei o contexto para dentro de etiquetas <context> e deixei as instrucoes depois delas.
ChatGPT | comecei por uma linha de papel, depois passos numerados, e o formato de saida no fim.
Gemini  | pus a tarefa primeiro, o contexto a seguir, e pedi a resposta dividida em seccoes.

---

## PT-PT

Correccoes de portugues do Brasil para portugues de Portugal. Sao aplicadas ao texto
depois de o modelo responder, e so as seccoes em portugues: nunca ao ponto 4, que e
em ingles. Determinista: nao depende do modelo e nao pode falhar.

Cada linha tem a palavra brasileira, uma barra vertical, e a portuguesa.
Acrescenta as que te fizerem falta. Nao e preciso recompilar nada.

usuario     | utilizador
usuarios    | utilizadores
arquivo     | ficheiro
arquivos    | ficheiros
tela        | ecra
telas       | ecras
engajamento | envolvimento
engajar     | envolver
time        | equipa
times       | equipas
celular     | telemovel
celulares   | telemoveis
gerenciar   | gerir
gerenciamento | gestao
planilha    | folha de calculo
mouse       | rato
cadastro    | registo
cadastrar   | registar
aplicativo  | aplicacao
aplicativos | aplicacoes
midia       | meios
midias      | meios
copia de seguranca | copia de seguranca

---

## ACCOES

As transformacoes do modo Reescrever. A ordem aqui e a ordem na lista.
Cada linha tem o nome que ves, uma barra vertical, e a instrucao em ingles.
Acrescenta as tuas. Nao e preciso recompilar: mexes, gravas, e ja aparece.

Resumir              | Summarize the text below in at most five bullet points. Keep every important fact. Add nothing.
Encurtar para metade | Rewrite the text below to about half its length, keeping the meaning and the tone.
Tornar formal        | Rewrite the text below in a formal, professional register. Keep every fact.
Tornar simples       | Rewrite the text below so that someone with no knowledge of the subject understands it. Avoid jargon.
Corrigir erros       | Fix spelling, grammar and punctuation in the text below. Change nothing else, keep the same words wherever they are correct.
Transformar em pontos | Turn the text below into a bulleted list, one idea per bullet, in the original order.
Extrair tarefas      | List every task, decision and deadline in the text below. One per line. Name who is responsible only when the text says so.
Extrair os factos    | List only the verifiable facts stated in the text below, one per line. Leave out opinions and anything not written there.
Traduzir para ingles | Translate the text below into English. Output only the translation.
Traduzir para portugues | Translate the text below into European Portuguese, not Brazilian. Output only the translation.
Email a partir de notas | Turn the notes below into a short, polite email in the same language. Keep every fact and invent nothing.
Titulo e resumo      | Give one title of at most eight words, then a blank line, then a summary of at most three lines.

~~~~

