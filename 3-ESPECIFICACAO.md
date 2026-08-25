# 3. Especificacao

Isto e o que precisas de saber para **mexer** na aplicacao sem a partir, ou para a
reconstruir de raiz. Se so a queres usar, este ficheiro nao te faz falta.

---

## O ecra

Tres colunas por baixo de uma barra de titulo propria. As medidas nao sao
arbitrarias: sao as do **Grok Bot**, lidas da folha de estilo da propria aplicacao
(os tokens `--sand-*`), porque o objectivo era esse aspecto.

| Peca | Medida |
|---|---|
| Barra de titulo | 44 px |
| Coluna da esquerda (equipa + ferramentas) | 280 px |
| Coluna do meio (conversa) | o resto, minimo 400 px |
| Coluna da direita (ficha do agente) | 320 px, escondida fora do modo da equipa |
| Grelha de espacamento | 4 px |

A janela abre a 1240 x 780 e nao encolhe abaixo de 1000 x 620, para as tres
colunas caberem sempre.

### Baloes de mensagem

```
raio      : 18 px
recheio   : 8 px em cima e em baixo, 12 px dos lados
largura   : no maximo 640 px
letra     : 14 px, entrelinha 20
tu        : fundo escuro, texto invertido, encostado a direita
o agente  : fundo cinzento claro, encostado a esquerda
espaco    : 12 px entre baloes
```

### A caixa de escrever

```
altura    : minimo 36 px, cresce ate 200
raio      : 18 px
recheio   : 8 / 12 px
```
A esquerda um botao `+`, que junta um ficheiro, uma imagem ou uma pasta conforme o
modo. A direita, `recomecar` e `Enviar`.

### As cores

Dezasseis fichas de cor, trocadas em bloco quando mudas de tema. **Tem de ficar
todas em `<Window.Resources>`** — ver o contrato B mais abaixo.

| Ficha | Claro | Escuro | Para que serve |
|---|---|---|---|
| `Fundo` | `#FCFCFC` | `#0F1115` | fundo da janela |
| `Painel` | `#F7F7F7` | `#161922` | barra lateral e ficha |
| `Cartao` | `#FFFFFF` | `#1B1F2A` | caixas de texto e cartoes |
| `Bordo` | `#DEDEDF` | `#252A36` | contornos e separadores |
| `Texto` | `#141414` | `#E7E9EF` | texto principal |
| `Apagado` | `#6B6B70` | `#868D9E` | texto secundario |
| `Realce` | `#070707` | `#E08A45` | botao principal, item escolhido |
| `RealceE` | `#E4E4E6` | `#3A2A1C` | fundo do item escolhido |
| `Passar` | `#EFEFEF` | `#20252F` | rato por cima |
| `RealceC` | `#2F2F2F` | `#EC9A57` | botao principal com o rato por cima |
| `Inactivo` | `#E6E6E8` | `#2A2F3B` | botao desligado |
| `InactivoT` | `#A3A3A8` | `#6C7381` | texto de botao desligado |
| `SobreRealce` | `#FCFCFC` | `#1A1206` | texto por cima do realce |
| `BolhaAgente` | `#EEEEEE` | `#232936` | balao do agente |
| `BolhaTu` | `#070707` | `#E7E9EF` | balao teu |
| `TextoBolhaTu` | `#FCFCFC` | `#0F1115` | texto dentro do teu balao |

Cores dos agentes (uma por posicao na lista):
`#1084fe` `#ff6700` `#00bca6` `#9159fe` `#ff309b` `#97683d`

Letra: **Segoe UI** em tudo, excepto o resultado dos modos de ferramenta, que usa
Cascadia Mono para o texto nao dancar.

---

## Os nove modos

O modo e um numero, e manda em tudo o que aparece no ecra.

| # | Botao | Painel lateral | Fala com |
|---|---|---|---|
| 0 | Construir um prompt | — | modelo local, em directo |
| 1 | Diagnosticar um prompt | destino + rever | modelo local, 2 chamadas |
| 2 | Imagem para markdown | forcar visao | OCR do Windows, ou modelo de visao |
| 3 | Reescrever texto | accao + atalho global | modelo local |
| 4 | Biblioteca de prompts | abrir ficheiro | ninguem: le do disco |
| 5 | Conversar com um ficheiro | ficheiro escolhido | modelo local, contexto 6144 |
| 6 | Procurar nos ficheiros | pasta + indexar | **ninguem**: e so codigo |
| 7 | Encontrar duplicados | pasta + desfazer | **ninguem**: e so codigo |
| 8 | A minha equipa | — (a ficha abre a direita) | conforme o agente |

Os modos 6 e 7 nao chamam modelo nenhum de proposito: procurar e comparar sao
trabalho de programa, e um modelo so acrescentaria hipoteses de inventar.

---

## O truque dos baloes com texto em directo

O ponto mais delicado do desenho, e vale a pena perceber antes de mexer.

O texto do modelo aparece **enquanto e escrito**, o que exige uma caixa de texto a
que se vai acrescentando (`AppendText`) e uma barra que segue (`ScrollToEnd`). Mas
uma conversa quer baloes separados.

A solucao: a area de conversa e um `ScrollViewer` chamado `Rolo`, que contem um
`StackPanel` chamado `Fio`. Os baloes ja fechados sao filhos do `Fio`. E o **ultimo
filho** e a caixa de texto de sempre, dentro de um `Border` chamado `CartaoSaida`:
e o balao vivo, onde o texto vai caindo.

No fim do turno, `FecharBolha()` converte o que la esta num balao fixo e limpa a
caixa. Nos modos de ferramenta o `Fio` fica vazio e o `CartaoSaida` e o unico
filho — pelo que esses modos se comportam exactamente como se comportavam antes de
existirem baloes.

---

## Os quatro contratos

O codigo **nunca anda pela arvore visual**: nao ha `VisualTreeHelper`, nem
`Children[n]`, nem casts para o elemento pai. Ha exactamente dois acessos:
`FindName` e `FindResource`. Por isso podes remontar o XAML a vontade — desde que
respeites isto:

**A. Os nomes e os tipos.** O codigo procura os elementos por nome com um
auxiliar que faz `as T`. Se renomeares um elemento, **ou lhe mudares o tipo**, o
auxiliar devolve `null` em silencio e a aplicacao rebenta so no uso seguinte, longe
da causa. Em concreto: `Raiz` e `Lado` tem de ser `Border`; `Barra` um `Grid`; os
sete `Painel*` tem de ser `StackPanel`; `Rolo` um `ScrollViewer`; `Entrada` e
`Saida` `TextBox`; `Historico` e `Equipa` `ListBox`. Os nove botoes de modo tem de
ser `Button` — um `ToggleButton` **nao** deriva de `Button` e daria `null`.

**B. As cores ficam em `<Window.Resources>`.** O codigo le-as por indexador
(`J.Resources["Realce"]`), e o indexador **nao** sobe a arvore. Se as puseres num
dicionario mais abaixo, a leitura devolve nulo e a troca de tema deixa de pegar.

**C. As listas sao indexadas por posicao.** O indice escolhido na `ListBox` e usado
como indice directo em tres listas diferentes. Logo: sem `ItemsSource`, sem
`ItemTemplate`, sem ordenacao, sem agrupamento e sem cabecalhos. Um item a mais no
inicio desalinha tudo.

**D. Os botoes de modo sao pintados a mao.** O codigo escreve `Background` e
`Foreground` directamente neles, o que ganha ao estilo. Por isso o modelo do botao
tem de continuar a usar `{TemplateBinding Background}` — se lhe fixares uma cor, o
realce do modo activo desaparece.

### Duas armadilhas do WPF que custaram tempo

- Uma `ListBox` com `IsEnabled="False"` ganha **fundo branco** por omissao do
  Windows, que fica horrivel no tema escuro. Nao a desactives: testa o indice.
- Sem `IsSynchronizedWithCurrentItem="False"`, uma `ListBox` **escolhe sozinha** o
  primeiro item quando o ecra e desenhado, e dispara o evento de escolha.
- Elementos construidos em codigo so seguem a troca de tema se usarem
  `SetResourceReference`. Com uma cor fixa, ficam presos ao tema em que nasceram.
- Um `Border` com uma caixa de texto dentro **nao encolhe** ate ao tamanho do
  texto. Para baloes, usa `TextBlock`.

---

## Como o XAML e carregado

Nao ha `x:Class`, nao ha code-behind, nao ha nada compilado a partir do XAML. O
ficheiro e lido do disco quando a aplicacao arranca e transformado em janela com
`XamlReader.Load`. Todos os eventos sao ligados por codigo, num so sitio.

**O ficheiro solto ao lado do `.exe` ganha ao que esta embebido nele.** Consequencia
pratica: mexes no `janela.xaml`, gravas, voltas a abrir a janela, e ja esta. So
mexer nos `.cs` obriga a recompilar.

Limitacoes que isto traz: o elemento de topo tem de continuar a ser uma `Window`,
e nao podes usar `{x:Static}` nem tipos teus dentro do XAML.
