# 6. Licoes

O que se aprendeu a tirar trabalho a serio de um modelo de **4 mil milhoes de
parametros** a correr numa placa grafica de 4 GB. Tudo o que esta aqui foi medido,
nao imaginado. Varias destas coisas parecem melhorias obvias e sao o contrario.

Se so leres um ficheiro deste repositorio, le este.

---

## 1. O que ele faz mal de forma sistematica, corrige-se **depois**, em codigo

**Esta e a licao maior.** Um modelo pequeno erra sempre nas mesmas coisas. A
tentacao e escrever instrucoes melhores. Nao resulta.

Dois exemplos, ambos com o mesmo desfecho:

| Problema | Por instrucao | Em codigo |
|---|---|---|
| Escreve portugues do Brasil | 3 tentativas, **todas pioraram** o resto da resposta | tabela de troca depois da resposta: resolvido a primeira |
| Estropia os titulos das seccoes | idem | apanhar entre a linha `4.` e a linha `5.`: nunca mais falhou |

Corolario, e uma correccao a mim proprio: eu tinha escrito que o portugues do
Brasil "so fine-tuning resolveria". Era falso. So era verdade para mudar o
**modelo** — nao para corrigir o **texto**.

---

## 2. Uma tarefa de cada vez

Pedir o diagnostico e a adaptacao ao destino na mesma chamada partia o formato:
o resultado para o ChatGPT saiu correcto **1 vez em 4**.

Separado em duas chamadas, cada uma com um so trabalho: **36 em 36**.

Sempre que se tentou juntar dois pedidos, o modelo piorou nos dois.

---

## 3. Mais texto no pedido piora, nao melhora

Aconteceu tres vezes, com coisas diferentes, sempre com o mesmo resultado.

O caso mais claro: descrever ao modelo a forma exacta que se queria — *"a linha
`<context>`, depois o contexto, depois..."* — baixou o acerto de **3/4 para 0/4**.
Descrever em prosa curta funcionou melhor do que mostrar o molde.

Outro: uma regra sobre persistencia do modo, colocada no bloco onde o modelo
escolhe o modo, **contaminou a escolha**. Teve de ir para dentro da seccao do
proprio modo.

---

## 4. Exemplos batem instrucoes — com uma armadilha

Pares de exemplo no `Modelfile` (`MESSAGE`) subiram o prompt final sair em ingles
de **2/6 para 5/6**. Vale mais um exemplo do que tres frases a explicar.

Duas coisas que e preciso saber:

- **Exigem aspas triplas.** Sem elas, o `\n` fica gravado a letra e o modelo passa
  a escrever `\n` no meio do texto em vez de mudar de linha. Custou uma tarde.
- **So contam em `/api/chat`.** Num pedido a `/api/generate` sao ignorados sem
  aviso nenhum.

---

## 5. Nunca deixar o modelo dizer o que fez

Ele escrevia *"inclui as etiquetas `<context>`"* sem as ter incluido.

As descricoes do que mudou passaram a ser escritas pelo programa, que sabe o que
pediu. O modelo produz; quem relata e o codigo.

Variante do mesmo erro, do lado do programa: a linha *"Ajustado para ChatGPT"* era
impressa mesmo quando a adaptacao falhava em silencio. Passou a depender de a
chamada ter mesmo corrido. **Nao afirmes o que nao verificaste** — vale para o
modelo e vale para o codigo a volta dele.

---

## 6. Quando ele tem de decidir, decide-se por ele

Deixar o modelo detectar sozinho se o pedido era uma entrevista ou um diagnostico:
acertava **pouco mais de metade** das vezes.

Com um marcador explicito no inicio da mensagem (`DIAGNOSTICO:` / `ENTREVISTA:`),
escolhido pelo programa: **16 em 16**.

Um modelo pequeno nao e um bom juiz. Se a decisao pode ser tomada no codigo, toma-a
no codigo.

---

## 7. Duas passagens rendem mais do que um prompt perfeito

Uma segunda chamada que reescreve o resultado contra um checklist de 8 criterios
subiu tres prompts de **3/7, 1/7 e 2/7** para **6/7** — os tres.

Custa mais uns 15 segundos. Vale sempre a pena.

Bonus: essa passagem tambem corrige a lingua sozinha, sem lho pedirmos.

---

## 8. Os dois numeros que valem 4x

O Ollama, por omissao, reservava contexto para 262144 fichas. Resultado: **43 GB**
pedidos, 95% do trabalho a cair no processador, **7 fichas por segundo**.

```
PARAMETER num_ctx 4096
PARAMETER num_gpu 99
```

| | Velocidade | Onde corre |
|---|---|---|
| omissao | 7 fichas/s | 95% processador |
| com estes dois | **27 fichas/s** | **100% grafica** |

O `num_gpu 99` forca todas as camadas para a placa. Sem ele, o Ollama deixa um
terco no processador e perdes cerca de 4x.

---

## 9. O aquecimento tem de ser uma conversa a serio

A primeira resposta demorava **10,1 segundos** ate ao primeiro caractere. A segunda,
**0,9**.

A demora nunca foi carregar o modelo — foi ele **ler o pedido de sistema e os
exemplos**. O Ollama guarda esse trabalho, mas com duas condicoes:

1. so no mesmo ponto de entrada (`/api/chat` nao aproveita o de `/api/generate`);
2. so se o **inicio** da conversa for igual.

A primeira tentativa de aquecimento nao serviu de nada, precisamente por isto:
aquecia com um pedido de forma diferente. Aquecer com uma conversa igual as reais
poe a primeira resposta a **0,9 s**.

---

## 10. Medir antes de construir

Antes de construir um indice de significado (embeddings) para o modo de conversar
com ficheiros, mediu-se: **128 de 129 documentos cabiam inteiros no contexto**.

O indice teria custado 262 MB, mais tempo de construcao, e teria dado respostas
**piores** — porque partir um documento em pedacos perde o fio. Nao se construiu.

---

## 11. O que nao tem solucao neste tamanho

Limites do modelo, nao configuracoes por afinar:

- **De vez em quando salta uma seccao inteira** do diagnostico. Ves quatro pontos
  em vez de cinco.
- **O modelo de visao e aproximado em graficos.** Acerta a ordem das barras e as
  cores; nao le valores que nao estejam escritos. Confirma sempre.
- **Nao sabe factos.** Nao lhe perguntes coisas do mundo: da-lhe o texto.
- **Fine-tuning nao e opcao** com 4 GB de memoria grafica. Precisa de 16 a 24.

---

## 12. Duas armadilhas do ambiente, se fores mexer nisto

- **Em PowerShell 5.1**, texto lido com `Get-Content -Raw` fica embrulhado num
  objecto e o `ConvertTo-Json` produz `{"prompt": {"value": "..."}}` em vez de uma
  cadeia. Qualquer API recusa. Correccao: `'' + $texto`.
- **Texto com acentos** partia os pedidos ao Ollama ate o corpo passar a ser
  enviado como bytes: `[Text.Encoding]::UTF8.GetBytes($json)`.

E uma nota de metodo que poupa horas: **substituicoes de varias linhas de uma vez
falham em silencio** em ficheiros com fins de linha misturados. Trabalha por
numero de linha, e confirma depois de gravar — nunca antes.
