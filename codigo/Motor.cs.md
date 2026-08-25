# app\Motor.cs

O ficheiro inteiro esta no bloco abaixo. Grava-o em `app\Motor.cs`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `app\Motor.cs`

<!-- destino: app\Motor.cs | codificacao: bom -->
~~~~csharp
// Pipelines do prompt-smith. Cada passo e uma chamada com uma so tarefa:
// foi assim que a fiabilidade passou de 1 em 4 para 36 em 36.
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;
using System.Text.RegularExpressions;

namespace PromptSmith
{
    public class Motor
    {
        public Ollama IA = new Ollama();
        public Regras R;

        public Motor(string pastaRegras) { R = Regras.Carregar(pastaRegras); }

        const string SYS_REESCREVER =
            "You improve prompts. Output only the improved prompt, in English. No explanation, " +
            "no headings, no code fences, no preamble. Never answer the prompt, never invent example data.";

        const string SYS_MARKDOWN =
            "You turn raw text into clean Markdown. Output only Markdown. Keep every fact from the input. " +
            "Never invent content that is not there. Use headings, lists and tables where they fit.";

        // ---------------- entrevista ----------------
        public string Entrevistar(List<Dictionary<string, string>> historico, string texto, Action<string> pedaco)
        {
            Dictionary<string, string> m = new Dictionary<string, string>();
            m["role"] = "user";
            // O marcador so vai na primeira mensagem. Sem ele o modelo escolhe
            // mal o modo em cerca de metade dos casos.
            m["content"] = historico.Count == 0 ? "ENTREVISTA: " + texto : texto;
            historico.Add(m);
            string resposta = IA.ConversarEmDirecto(historico, pedaco).Trim();
            // As perguntas da entrevista tambem levam a correccao de portugues.
            // O modo imagem nao leva: o texto ai e conteudo teu, e trocar palavras
            // dentro de um documento que fotografaste seria adulterá-lo.
            resposta = CorrigirPortugues(resposta);
            Dictionary<string, string> e = new Dictionary<string, string>();
            e["role"] = "assistant"; e["content"] = resposta;
            historico.Add(e);
            return resposta;
        }

        // ---------------- diagnostico ----------------
        // Devolve o texto final. O primeiro passo vai aparecendo pelo pedaco;
        // os passos seguintes reescrevem o ponto 4, e por isso a janela substitui
        // o texto todo no fim.
        public string Diagnosticar(string prompt, string destino, bool autoCritica, Action<string> estado, Action<string> pedaco)
        {
            // 1) o diagnostico em cinco pontos, sempre sozinho
            if (estado != null) estado("a analisar o prompt...");
            List<Dictionary<string, string>> msgs = new List<Dictionary<string, string>>();
            Dictionary<string, string> m = new Dictionary<string, string>();
            m["role"] = "user";
            m["content"] = "DIAGNOSTICO este prompt:" + Environment.NewLine + Environment.NewLine + prompt;
            msgs.Add(m);
            string texto = IA.ConversarEmDirecto(msgs, pedaco).Trim();

            string ponto4 = ExtrairPonto4(texto);
            string melhor = ponto4;
            bool adaptou = false;   // so escrevemos "Ajustado para" se a adaptacao correu mesmo

            // 2) auto-critica contra o checklist editavel
            if (autoCritica && melhor.Length > 0 && R.Checklist.Count > 0)
            {
                if (estado != null) estado("a rever contra o checklist...");
                StringBuilder regras = new StringBuilder();
                foreach (string c in R.Checklist) regras.AppendLine("- " + c);
                string pedido = "Rewrite this prompt so that every rule below is satisfied." + Environment.NewLine + Environment.NewLine
                              + "Rules:" + Environment.NewLine + regras.ToString() + Environment.NewLine
                              + "Keep it under 180 words." + Environment.NewLine + Environment.NewLine
                              + "Prompt:" + Environment.NewLine + melhor;
                string r = IA.Tarefa(SYS_REESCREVER, pedido);
                if (r.Length > 30) melhor = r;
            }

            // 3) forma do destino, em chamada separada
            if (destino != null && R.Destinos.ContainsKey(destino) && melhor.Length > 0)
            {
                if (estado != null) estado("a adaptar ao " + destino + "...");
                string pedido = "Rewrite this prompt in English. " + R.Destinos[destino]
                              + " Keep every requirement that is already there."
                              + Environment.NewLine + Environment.NewLine + "Prompt:" + Environment.NewLine + melhor;
                string r = IA.Tarefa(SYS_REESCREVER, pedido);
                if (r.Length > 30) { melhor = r; adaptou = true; }
            }

            if (melhor != ponto4 && ponto4.Length > 0) texto = texto.Replace(ponto4, melhor);

            // 4) correccoes deterministas: primeiro os titulos, depois o portugues.
            // Pela ordem certa, porque a correccao do portugues precisa de encontrar
            // o ponto 4, e so o encontra depois de os titulos estarem canonicos.
            texto = NormalizarTitulos(texto);
            texto = CorrigirPortugues(texto);

            // 5) acrescentos escritos por nos, nunca pelo modelo
            StringBuilder fim = new StringBuilder(texto.TrimEnd());
            if (melhor.Length > 0)
                fim.Append(Environment.NewLine + Environment.NewLine + Avaliar(melhor).ToString());
            List<string> vagas = R.VagasEm(prompt);
            if (vagas.Count > 0)
                fim.Append(Environment.NewLine + Environment.NewLine +
                           "Palavras vagas no teu prompt: " + string.Join(", ", vagas.ToArray()));
            if (adaptou && destino != null && R.Mudancas.ContainsKey(destino))
                fim.Append(Environment.NewLine + Environment.NewLine +
                           "Ajustado para " + destino + ": " + R.Mudancas[destino]);
            return fim.ToString();
        }

        // ---------------- correccoes deterministas ----------------
        // Tudo o que se segue acontece depois de o modelo responder. Nao custa
        // tempo, nao custa tokens, e nao pode falhar. E aqui que se resolvem os
        // defeitos que tentar corrigir com instrucoes so piorava.

        static readonly string[] TITULOS = {
            "1. O que esta bem", "2. O que falta", "3. Ambiguidades",
            "4. Prompt melhorado", "5. O que mudei e porque" };

        // O modelo estropia os titulos: ja escreveu "5. O-que-mudei-e-porque",
        // "5. O tqdm e porque" e "5. O short e porque". Reescrevemos pelo numero.
        // Guardas para nao estragar passos numerados dentro do proprio prompt:
        // so a primeira ocorrencia de cada numero, por ordem, em linha curta e
        // com poucas palavras. Um passo de um prompt e sempre mais comprido.
        public static string NormalizarTitulos(string texto)
        {
            string[] linhas = texto.Replace("\r\n", "\n").Split('\n');
            int proximo = 1;
            for (int i = 0; i < linhas.Length && proximo <= 5; i++)
            {
                Match m = Regex.Match(linhas[i], @"^\s*" + proximo + @"\s*[\.\)]\s*(\S.*)$");
                if (!m.Success) continue;
                string resto = m.Groups[1].Value.Trim();
                if (resto.Length > 46) continue;
                if (resto.Split(new char[] { ' ' }, StringSplitOptions.RemoveEmptyEntries).Length > 6) continue;
                linhas[i] = TITULOS[proximo - 1];
                proximo++;
            }
            return string.Join(Environment.NewLine, linhas);
        }

        // Portugues do Brasil para portugues de Portugal, pela tabela editavel.
        // O ponto 4 e em ingles e fica de fora: tiramo-lo, corrigimos, repomos.
        public string CorrigirPortugues(string texto)
        {
            if (R.PtPt.Count == 0) return texto;
            string p4 = ExtrairPonto4(texto);
            string marca = "\u0001PONTO4\u0001";
            if (p4.Length > 0) texto = texto.Replace(p4, marca);

            foreach (KeyValuePair<string, string> par in R.PtPt)
            {
                string padrao = @"\b" + ComAcentos(par.Key) + @"\b";
                texto = Regex.Replace(texto, padrao, delegate (Match m) {
                    // Manter a maiuscula inicial se a original a tinha.
                    string v = par.Value;
                    if (char.IsUpper(m.Value[0]) && v.Length > 0) v = char.ToUpper(v[0]) + v.Substring(1);
                    return v;
                }, RegexOptions.IgnoreCase);
            }

            if (p4.Length > 0) texto = texto.Replace(marca, p4);
            return texto;
        }

        // Faz a procura tolerante a acentos: quem escrever "usuario" na tabela
        // apanha tambem "usuário". Sem isto era preciso listar todas as variantes,
        // e foi exactamente assim que "usuários" escapou no primeiro teste.
        static string ComAcentos(string palavra)
        {
            StringBuilder sb = new StringBuilder();
            foreach (char c in palavra.ToLower())
            {
                switch (c)
                {
                    case 'a': sb.Append("[aáàâã]"); break;
                    case 'e': sb.Append("[eéèê]"); break;
                    case 'i': sb.Append("[ií]");   break;
                    case 'o': sb.Append("[oóòôõ]"); break;
                    case 'u': sb.Append("[uúü]");  break;
                    case 'c': sb.Append("[cç]");   break;
                    default:  sb.Append(Regex.Escape(c.ToString())); break;
                }
            }
            return sb.ToString();
        }

        // ---------------- avaliacao do prompt final ----------------
        // O checklist so servia para reescrever. Agora tambem verifica, para se
        // ver se a segunda passagem serviu de alguma coisa.
        public class Avaliacao
        {
            public int Cumpridos, Total;
            public List<string> Faltam = new List<string>();
            public override string ToString()
            {
                string s = "Checklist: " + Cumpridos + " de " + Total + " criterios cumpridos.";
                if (Faltam.Count > 0) s += " Faltam: " + string.Join(", ", Faltam.ToArray()) + ".";
                return s;
            }
        }

        public static Avaliacao Avaliar(string prompt)
        {
            string[,] criterios = {
                { "linha de papel",      @"(?i)you are" },
                { "tarefa clara",        @"(?i)\b(write|summarize|summarise|create|generate|explain|analyse|analyze|translate|list|rewrite|produce|draft)\b" },
                { "publico",             @"(?i)\b(for|audience|readers?|aimed at|targeting)\b" },
                { "formato",             @"(?i)(bullet|table|json|section|paragraph|list|format|steps?)" },
                { "limite de tamanho",   @"(?i)\b\d+\s*(words?|lines?|items?|bullets?|sentences?|characters?)\b" },
                { "proibicoes",          @"(?i)(do not|don't|avoid|never|without)" },
                { "criterio de sucesso", @"(?i)(success|good answer|passes|measur|test:)" } };

            Avaliacao a = new Avaliacao();
            a.Total = criterios.GetLength(0) + 1;
            for (int i = 0; i < criterios.GetLength(0); i++)
            {
                if (Regex.IsMatch(prompt, criterios[i, 1])) a.Cumpridos++;
                else a.Faltam.Add(criterios[i, 0]);
            }
            // o oitavo: nao usar palavras vagas
            if (Regex.IsMatch(prompt, @"(?i)\b(good|impactful|professional|engaging|creative|nice|effective)\b"))
                a.Faltam.Add("sem palavras vagas");
            else a.Cumpridos++;
            return a;
        }

        // ---------------- reescrever texto ----------------
        // O modo onde um modelo pequeno e mais fiavel: o conteudo vem todo de ti,
        // por isso ele nao tem nada para inventar. Só transforma.
        const string SYS_TEXTO =
            "You transform text. Output only the transformed text. No explanation, no preamble, " +
            "no headings, no code fences, no quotes around the answer. " +
            "Never add facts that are not in the text. " +
            // As instrucoes vao em ingles porque o modelo as segue melhor assim, mas
            // sem esta linha ele respondia em ingles a um texto portugues.
            "Answer in the same language as the text, unless the instruction tells you to translate.";

        public string Reescrever(string texto, string instrucao, Action<string> pedaco)
        {
            // A lingua e decidida aqui, e nao pedida ao modelo. Pedir nao chegava:
            // com a instrucao em ingles a vista, ele respondia em ingles a texto
            // portugues, mesmo com uma regra geral no system prompt a dizer o contrario.
            string lingua = "";
            if (instrucao.IndexOf("Translate", StringComparison.OrdinalIgnoreCase) < 0)
                lingua = ParecePortugues(texto)
                    ? Environment.NewLine + "Answer in European Portuguese."
                    : Environment.NewLine + "Answer in English.";

            string r = IA.TarefaEmDirecto(SYS_TEXTO,
                instrucao + lingua + Environment.NewLine + Environment.NewLine +
                "Text:" + Environment.NewLine + texto, pedaco).Trim();
            // So corrigimos o portugues se a resposta for mesmo em portugues.
            // Senao estragavamos as traducoes para ingles.
            if (ParecePortugues(r)) r = CorrigirPortugues(r);
            return r;
        }

        public static bool ParecePortugues(string t)
        {
            int pt = Regex.Matches(t, @"(?i)\b(de|que|para|com|uma|nao|n[aã]o|dos|das|pelo|s[aã]o|mais|como|est[aá])\b").Count;
            int en = Regex.Matches(t, @"(?i)\b(the|and|with|that|from|this|your|for|are|will|should)\b").Count;
            return pt > en;
        }

        // ---------------- correr um agente ----------------
        public string CorrerAgente(Agente a, string pergunta, Action<string> estado, Action<string> pedaco)
        {
            StringBuilder sb = new StringBuilder();
            sb.AppendLine("You are " + a.Nome + ".");
            sb.AppendLine("Your job: " + a.Missao);
            sb.AppendLine("Answer in European Portuguese.");
            sb.AppendLine("Never invent numbers, percentages or statistics. If a number is not given to you, say you do not have it.");
            if (a.Ferramentas.Contains("WebSearch") || a.Ferramentas.Contains("WebFetch"))
                sb.AppendLine("When you use the web, always end with the sources you used, as links.");
            sb.AppendLine();

            // Se o agente pode ler ficheiros e a mensagem traz um caminho, damos-lhe o texto.
            if (a.Ferramentas.Contains("ler-ficheiro"))
            {
                foreach (string parte in pergunta.Split('"', '\n'))
                {
                    string c = parte.Trim();
                    if (c.Length > 4 && File.Exists(c))
                    {
                        if (estado != null) estado("a ler " + Path.GetFileName(c) + "...");
                        Documento d = Documento.Abrir(c);
                        if (d.Erro == null && d.Texto.Length > 0)
                        {
                            string txt = d.Texto;
                            if (d.Palavras > PALAVRAS_MAX) txt = PartesRelevantes(d.Texto, pergunta, 4);
                            sb.AppendLine("Document " + d.Nome + ":");
                            sb.AppendLine("---");
                            sb.AppendLine(txt);
                            sb.AppendLine("---");
                            sb.AppendLine();
                        }
                        break;
                    }
                }
            }

            // O fio da conversa: turnos a serio, nao linhas soltas do ficheiro.
            // E isto que faz o agente perceber "e qual deles anuncia mais?" sem
            // lhe repetirmos o assunto. O modelo local so tem 4096 de contexto,
            // por isso leva menos turnos e com as respostas cortadas.
            bool ehLocal = a.Cerebro != "claude";
            List<string[]> turnos = a.Turnos(ehLocal ? 3 : 6);
            if (turnos.Count > 0)
            {
                sb.AppendLine("This is the conversation so far. Use it to understand what the user means now.");
                foreach (string[] t in turnos)
                {
                    string resp = t[1];
                    if (ehLocal && resp.Length > 400) resp = resp.Substring(0, 400) + "...";
                    sb.AppendLine("User: " + t[0]);
                    sb.AppendLine("You: " + resp);
                    sb.AppendLine();
                }
            }
            sb.AppendLine("Now do this:");
            sb.AppendLine(pergunta);

            if (a.Cerebro == "claude")
            {
                if (estado != null) estado("a falar com o Claude (leva ~20 s)...");
                string erro;
                List<string> fer = new List<string>();
                foreach (string f in a.Ferramentas) if (f == "WebSearch" || f == "WebFetch") fer.Add(f);
                string r = Claude.Perguntar(sb.ToString(), fer, out erro);
                if (r == null)
                    return "Nao consegui falar com o Claude: " + erro + Environment.NewLine +
                           "Confirma que o Claude Code esta instalado e com sessao iniciada.";
                // Aviso quando um agente de pesquisa responde sem citar nada.
                if (fer.Count > 0 && !Regex.IsMatch(r, @"(?i)(https?://|fontes?\s*:|sources?\s*:)"))
                    r += Environment.NewLine + Environment.NewLine +
                         "(atencao: esta resposta veio sem fontes, confirma antes de a usar)";
                return r;
            }

            if (estado != null) estado("a pensar...");
            string local = IA.TarefaEmDirecto(
                "You are an assistant that follows the role given in the message. Output only the answer, " +
                "no preamble. Never invent facts, numbers or sources. You have no internet access: if the " +
                "answer needs the web, say so plainly.", sb.ToString(), pedaco).Trim();
            return CorrigirPortugues(local);
        }

        // ---------------- conversar com um ficheiro ----------------
        // A regra que torna isto de confianca: responder so a partir do documento,
        // citar a frase que sustenta, e admitir quando a resposta nao la esta.
        // Um modelo de 4B sabe pouco; se o deixarmos usar o que "sabe", inventa.
        const string SYS_DOC =
            "You answer questions about one document. Follow these rules exactly. " +
            "Use only what is written in the document. Never use your own knowledge. " +
            "If the answer is not in the document, reply with exactly this sentence and nothing else: " +
            "Isso nao esta no documento. " +
            "When the answer is in the document, give it, then a blank line, then a line that starts with " +
            "Citacao: followed by the sentence from the document that supports it, copied word for word. " +
            "Answer in the same language as the question. " +
            // Sem esta frase ele respondia de cabeca a factos comuns: perguntado
            // pela capital de Portugal, dizia Lisboa. Medido: 5/6 passou a 6/6.
            "This holds even for facts you are certain about: if the words are not in the document, " +
            "you do not answer them. A famous fact you happen to know is still not in the document.";

        public const int PALAVRAS_MAX = 4000;   // orcamento para o documento
        public const int CONTEXTO_DOC = 6144;   // medido: 3,5 GB, 100% na grafica

        public string PerguntarAoDocumento(Documento d, List<Dictionary<string, string>> conversa,
                                           string pergunta, Action<string> estado, Action<string> pedaco)
        {
            string corpo;
            if (d.Palavras <= PALAVRAS_MAX) corpo = d.Texto;
            else
            {
                if (estado != null) estado("documento grande; a procurar as partes relevantes...");
                corpo = PartesRelevantes(d.Texto, pergunta, 4);
            }

            StringBuilder sb = new StringBuilder();
            sb.AppendLine("Document: " + d.Nome);
            sb.AppendLine("---");
            sb.AppendLine(corpo);
            sb.AppendLine("---");
            // Ultimas duas trocas, para as perguntas de seguimento fazerem sentido.
            // Bem separadas da pergunta nova: sem isto o modelo repetia a resposta
            // anterior em vez de responder ao que lhe estavam a perguntar agora.
            int inicio = Math.Max(0, conversa.Count - 4);
            if (conversa.Count > 0)
            {
                sb.AppendLine("Earlier in this conversation, for context only:");
                for (int i = inicio; i < conversa.Count; i++)
                    sb.AppendLine((conversa[i]["role"] == "user" ? "Earlier question: " : "Earlier answer: ")
                                  + conversa[i]["content"]);
                sb.AppendLine();
            }
            sb.AppendLine("Now answer this new question, using the document above:");
            sb.AppendLine(pergunta);

            IA.NumCtx = CONTEXTO_DOC;
            try { return IA.TarefaEmDirecto(SYS_DOC, sb.ToString(), pedaco).Trim(); }
            finally { IA.NumCtx = 0; }
        }

        // Rede de seguranca em codigo: se a resposta nao trouxer citacao e nao for
        // a admissao de que nao sabe, avisamos. E o utilizador que decide se confia.
        public static string AvisoSemCitacao(string resposta)
        {
            if (resposta.IndexOf("nao esta no documento", StringComparison.OrdinalIgnoreCase) >= 0) return null;
            if (resposta.IndexOf("nao esta no documento".Replace("nao", "nao"), StringComparison.OrdinalIgnoreCase) >= 0) return null;
            if (Regex.IsMatch(resposta, @"(?im)^\s*cita[cç][aã]o\s*:")) return null;
            return "(atencao: esta resposta veio sem citacao do documento, confirma antes de a usar)";
        }

        // Sem modelo de indexacao: parte em pedacos e escolhe os que tem mais
        // palavras em comum com a pergunta. Chega para o caso raro do documento
        // grande, e nao obriga a descarregar nada.
        public static string PartesRelevantes(string texto, string pergunta, int quantos)
        {
            string[] palavras = texto.Split(new char[] { ' ', '\n', '\t' }, StringSplitOptions.RemoveEmptyEntries);
            List<string> pedacos = new List<string>();
            for (int i = 0; i < palavras.Length; i += 700)
                pedacos.Add(string.Join(" ", palavras, i, Math.Min(700, palavras.Length - i)));

            List<string> chave = new List<string>();
            foreach (string w in pergunta.ToLower().Split(new char[] { ' ', ',', '.', '?', '!', ';', ':' }, StringSplitOptions.RemoveEmptyEntries))
                if (w.Length > 3 && !chave.Contains(w)) chave.Add(w);

            int[] pontos = new int[pedacos.Count];
            for (int i = 0; i < pedacos.Count; i++)
            {
                string baixo = pedacos[i].ToLower();
                foreach (string w in chave)
                    pontos[i] += Regex.Matches(baixo, @"\b" + Regex.Escape(w)).Count;
            }

            List<int> ordem = new List<int>();
            for (int i = 0; i < pedacos.Count; i++) ordem.Add(i);
            ordem.Sort(delegate (int a, int b) { return pontos[b].CompareTo(pontos[a]); });

            List<int> escolhidos = new List<int>();
            for (int i = 0; i < Math.Min(quantos, ordem.Count); i++) escolhidos.Add(ordem[i]);
            escolhidos.Sort();   // pela ordem do documento, para o texto fazer sentido

            StringBuilder sb = new StringBuilder();
            foreach (int i in escolhidos)
                sb.AppendLine("[parte " + (i + 1) + " de " + pedacos.Count + "]").AppendLine(pedacos[i]).AppendLine();
            return sb.ToString();
        }

        // ---------------- imagem para markdown ----------------
        public string ImagemParaMarkdown(string caminho, bool forcarVisao, Action<string> estado, Action<string> pedaco)
        {
            string bruto = "";
            string origem = "";

            if (!forcarVisao && Ocr.Disponivel())
            {
                if (estado != null) estado("a ler o texto da imagem...");
                try { bruto = Ocr.Ler(caminho); } catch (Exception e) { bruto = ""; origem = e.Message; }
                origem = "OCR do Windows";
            }

            // Pouco texto quer dizer diagrama, grafico ou fotografia: passa ao modelo de visao.
            if (bruto.Length < 40)
            {
                if (estado != null) estado("pouco texto; a usar o modelo de visao...");
                // Este pedido foi afinado a medida. O anterior, mais curto, fazia o
                // modelo inventar um titulo que nao existia e devolver so as etiquetas.
                bruto = IA.Visao(
                    "Write a plain-text description of this image. Only report what is actually drawn. " +
                    "Never invent a title, a caption, a unit or a number that is not written in the image; " +
                    "if there is no title, say the chart has no title. If it is a chart, name the kind of chart, " +
                    "list every bar or slice in left-to-right order with its written label, and rank them from " +
                    "largest to smallest. If it is a table, give it row by row. Transcribe every visible word exactly.",
                    caminho);
                origem = "modelo de visao qwen2.5vl";
            }

            if (bruto.Trim().Length == 0) return "Nao consegui extrair nada desta imagem.";

            if (estado != null) estado("a organizar em markdown...");
            string md = IA.TarefaEmDirecto(SYS_MARKDOWN,
                "Turn the text below into clean Markdown. Keep every fact. Do not add anything."
                + Environment.NewLine + Environment.NewLine + bruto, pedaco);
            if (md.Length < 10) md = bruto;

            return md.Trim() + Environment.NewLine + Environment.NewLine
                 + "<!-- extraido por " + origem + " a partir de " + Path.GetFileName(caminho) + " -->";
        }

        // ---------------- auxiliares ----------------
        public static string ExtrairPonto4(string texto)
        {
            // Tolerante de proposito: o modelo ja escreveu "5. O-que-mudei-e-porque".
            // Apanha tudo entre a linha que comeca por 4. e a linha que comeca por 5.
            Match m = Regex.Match(texto, @"^\s*4\.[^\r\n]*\r?\n([\s\S]*?)(?=^\s*5\.)", RegexOptions.Multiline);
            return m.Success ? m.Groups[1].Value.Trim() : "";
        }

        // Para o botao de guardar: so o prompt final, limpo.
        public static string PromptFinal(string texto)
        {
            string p = ExtrairPonto4(texto);
            if (p.Length > 0) return p;
            Match m = Regex.Match(texto, @"PROMPT:\s*([\s\S]*?)(COMO USAR:|$)");
            return m.Success ? m.Groups[1].Value.Trim() : texto.Trim();
        }
    }
}














~~~~

