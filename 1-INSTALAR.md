# 1. Instalar

Uns 10 minutos, quase tudo a descarregar.

---

## 1. O Ollama

E o programa que corre o modelo no teu computador. Gratuito, sem conta.

Descarrega de [ollama.com](https://ollama.com) e instala. Fica a correr em segundo
plano, com um icone junto ao relogio.

Confirma que esta vivo:

~~~~
ollama --version
~~~~

---

## 2. O modelo de base

~~~~
ollama pull qwen3:4b-instruct
~~~~

Sao **cerca de 2,5 GB**. E um modelo de 4 mil milhoes de parametros — pequeno de
proposito, para caber numa placa grafica de 4 GB e responder em segundos.

### Se quiseres o modo de imagem

O modo "imagem para markdown" usa primeiro o OCR do proprio Windows, que le texto
muito bem e nao precisa de nada. So passa a um modelo de visao quando a imagem tem
pouco texto — um grafico, um diagrama, uma fotografia.

Se quiseres essa parte:

~~~~
ollama pull qwen2.5vl:3b
~~~~

Mais **cerca de 3,2 GB**. **E opcional**: sem ele, o modo imagem continua a
funcionar por OCR.

---

## 3. Criar o modelo `prompt-smith`

O modelo de base sozinho nao chega. O que lhe da o comportamento — a entrevista, a
estrutura fixa de 5 pontos do diagnostico, os exemplos — esta no `Modelfile`, que
esta neste repositorio em [codigo/Modelfile.md](codigo/Modelfile.md).

Extrai os ficheiros primeiro (ver [2-CONSTRUIR.md](2-CONSTRUIR.md)) e depois, na
pasta onde ficou o `Modelfile`:

~~~~
ollama create prompt-smith -f Modelfile
~~~~

Confirma:

~~~~
ollama list
~~~~

Tem de aparecer `prompt-smith:latest`.

---

## 4. Confirmar que ficou bem

Depois de compilares (ver [2-CONSTRUIR.md](2-CONSTRUIR.md)):

~~~~
powershell -ExecutionPolicy Bypass -File scripts\aceitacao.ps1
~~~~

Tem de dar **18 / 18**. Demora uns minutos, porque faz oito pedidos verdadeiros ao
modelo. Se der menos, o modelo nao foi criado a partir do `Modelfile` — repete o
passo 3.

---

## Se ficar lento

Com o modelo carregado, escreve `ollama ps`. A coluna `PROCESSOR` tem de dizer
**100% GPU**. Se disser algo como `30%/70% CPU/GPU`, por ordem de probabilidade:

1. **Muitos separadores do browser abertos.** Roubam memoria a placa grafica.
2. **Mexeste no `num_ctx`.** Acima de 4096 nao cabe em 4 GB. Volta a por 4096.
3. **Tiraste o `PARAMETER num_gpu 99`.** Sem ele perdes cerca de **4x**: o Ollama
   deixa um terco das camadas no processador.

Estes dois numeros nao foram escolhidos ao calhas. Medido numa NVIDIA T500 de 4 GB:

| Configuracao | Velocidade | Onde corre |
|---|---|---|
| omissao do Ollama (contexto 262144) | 7 tokens/s | 95% no processador |
| `num_ctx 4096` + `num_gpu 99` | **27 tokens/s** | **100% na grafica** |

O modo de imagem troca de modelo na placa, o que acrescenta uns 10 segundos de
cada vez que alternas entre modos.

Sob uso continuado um portatil aquece e cai de 26 para uns 15 tokens por segundo.
E normal e passa quando arrefece.

---

## Se quiseres agentes que pesquisam na internet

**E opcional.** Sem isto tens os tres agentes locais e as oito ferramentas, tudo a
funcionar e tudo de graca.

Os agentes com `cerebro: claude` chamam o **Claude Code** que tiveres instalado, em
modo sem interface, e usam a tua subscricao. A aplicacao procura-o em:

~~~~
%APPDATA%\npm\claude.cmd
~~~~

Se nao o encontrar, esses agentes dizem-te isso e nao inventam nada.

Ver [4-AGENTES.md](4-AGENTES.md) para perceber como as ferramentas de cada agente
sao limitadas — e porque essa limitacao e real e nao decorativa.
