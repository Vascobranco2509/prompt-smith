# app\Nucleo.cs

O ficheiro inteiro esta no bloco abaixo. Grava-o em `app\Nucleo.cs`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `app\Nucleo.cs`

<!-- destino: app\Nucleo.cs | codificacao: bom -->
~~~~csharp
// Motor do prompt-smith: regras, Ollama, OCR do Windows e visao.
// Nao tem interface nenhuma. A janela esta em Janela.cs.
using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Text;
using System.Threading;
using Windows.Foundation;
using Windows.Graphics.Imaging;
using Windows.Media.Ocr;
using Windows.Storage;
using Windows.Storage.Streams;
using System.Web.Script.Serialization;

namespace PromptSmith
{
    // ---------------------------------------------------------------------
    // Le o ficheiro regras/regras.md. Editavel sem recompilar nada.
    // ---------------------------------------------------------------------
    public class Regras
    {
        public List<string> Checklist = new List<string>();
        public List<string> Vagas = new List<string>();
        public Dictionary<string,string> Destinos = new Dictionary<string,string>();
        public Dictionary<string,string> Mudancas = new Dictionary<string,string>();
        public Dictionary<string,string> PtPt = new Dictionary<string,string>();
        // Lista, e nao dicionario, porque a ordem do ficheiro e a ordem que o
        // utilizador ve na janela.
        public List<KeyValuePair<string,string>> Accoes = new List<KeyValuePair<string,string>>();
        public string Erro = null;

        public static Regras Carregar(string caminho)
        {
            Regras r = new Regras();
            try
            {
                string seccao = "";
                foreach (string bruta in File.ReadAllLines(caminho, Encoding.UTF8))
                {
                    string l = bruta.Trim();
                    if (l.StartsWith("##")) { seccao = l.TrimStart('#').Trim().ToUpper(); continue; }
                    if (l.Length == 0 || l.StartsWith("#") || l.StartsWith("---")) continue;

                    int barra = l.IndexOf('|');
                    string direita = barra >= 0 ? l.Substring(barra + 1).Trim() : l;
                    string esquerda = barra >= 0 ? l.Substring(0, barra).Trim() : l;

                    if (seccao == "CHECKLIST" && barra >= 0) r.Checklist.Add(direita);
                    else if (seccao == "PALAVRAS-VAGAS" && barra < 0) r.Vagas.Add(l.ToLower());
                    else if (seccao == "DESTINOS" && barra >= 0) r.Destinos[esquerda] = direita;
                    else if (seccao == "O QUE MUDEI" && barra >= 0) r.Mudancas[esquerda] = direita;
                    else if (seccao == "PT-PT" && barra >= 0) r.PtPt[esquerda.ToLower()] = direita;
                    else if (seccao == "ACCOES" && barra >= 0)
                        r.Accoes.Add(new KeyValuePair<string,string>(esquerda, direita));
                }
                if (r.Checklist.Count == 0) r.Erro = "o ficheiro de regras nao tem checklist";
            }
            catch (Exception e) { r.Erro = "nao consegui ler as regras: " + e.Message; }
            return r;
        }

        // Palavras vagas encontradas num texto, para assinalar no diagnostico.
        public List<string> VagasEm(string texto)
        {
            List<string> achadas = new List<string>();
            string t = " " + texto.ToLower() + " ";
            foreach (string v in Vagas)
                if (t.Contains(" " + v + " ") && !achadas.Contains(v)) achadas.Add(v);
            return achadas;
        }
    }

    // ---------------------------------------------------------------------
    // ---------------------------------------------------------------------
    // Biblioteca de prompts guardados. Ficheiro de texto simples e editavel:
    //     ## Nome do prompt
    //     o corpo, com {campos} entre chavetas
    // ---------------------------------------------------------------------
    public class Biblioteca
    {
        public List<KeyValuePair<string, string>> Itens = new List<KeyValuePair<string, string>>();
        public string Caminho;

        public Biblioteca(string caminho) { Caminho = caminho; Ler(); }

        public void Ler()
        {
            Itens.Clear();
            try
            {
                if (!File.Exists(Caminho)) return;
                string nome = null;
                StringBuilder corpo = new StringBuilder();
                foreach (string l in File.ReadAllLines(Caminho, Encoding.UTF8))
                {
                    if (l.TrimStart().StartsWith("## "))
                    {
                        if (nome != null) Itens.Add(new KeyValuePair<string, string>(nome, corpo.ToString().Trim()));
                        nome = l.TrimStart().Substring(3).Trim();
                        corpo.Length = 0;
                    }
                    else if (nome != null) corpo.AppendLine(l);
                }
                if (nome != null) Itens.Add(new KeyValuePair<string, string>(nome, corpo.ToString().Trim()));
            }
            catch { Itens.Clear(); }
        }

        public void Acrescentar(string nome, string corpo)
        {
            try
            {
                StringBuilder sb = new StringBuilder();
                if (!File.Exists(Caminho))
                {
                    sb.AppendLine("# Biblioteca de prompts");
                    sb.AppendLine();
                    sb.AppendLine("Cada prompt comeca por ## e o nome. Podes editar este ficheiro a mao.");
                    sb.AppendLine("Usa {chavetas} nos campos que mudam de vez para vez.");
                    sb.AppendLine();
                }
                sb.AppendLine();
                sb.AppendLine("## " + nome);
                sb.AppendLine();
                sb.AppendLine(corpo.Trim());
                File.AppendAllText(Caminho, sb.ToString(), new UTF8Encoding(false));
                Ler();
            }
            catch { }
        }
    }

    // ---------------------------------------------------------------------
    // Um ficheiro transformado em texto. PDF pelo pdftotext, que ja esta
    // instalado; Word abrindo o zip por dentro; o resto le-se directamente.
    // ---------------------------------------------------------------------
    public class Documento
    {
        public string Caminho = "", Nome = "", Texto = "", Erro = null;
        public int Palavras = 0;

        public static Documento Abrir(string caminho)
        {
            Documento d = new Documento();
            d.Caminho = caminho;
            d.Nome = Path.GetFileName(caminho);
            try
            {
                if (!File.Exists(caminho)) { d.Erro = "nao encontrei o ficheiro"; return d; }

                // Ficheiros do OneDrive que ainda nao foram descarregados sao so um
                // marcador. Sem isto o utilizador levava um erro de zip danificado.
                if ((File.GetAttributes(caminho) & System.IO.FileAttributes.Offline) != 0)
                {
                    d.Erro = "este ficheiro esta na nuvem e ainda nao foi descarregado. Abre-o uma vez no Explorador e tenta outra vez.";
                    return d;
                }

                string ext = Path.GetExtension(caminho).ToLower();
                if (ext == ".pdf") d.Texto = DePdf(caminho);
                else if (ext == ".docx") d.Texto = DeDocx(caminho);
                else if (ext == ".txt" || ext == ".md" || ext == ".csv" || ext == ".log")
                    d.Texto = File.ReadAllText(caminho, Encoding.UTF8);
                else { d.Erro = "nao sei ler ficheiros " + ext + ". Aceito pdf, docx, txt, md e csv."; return d; }

                // O pdftotext mete quebras de pagina; outros formatos trazem lixo
                // de controlo. Fora, senao aparecem no meio do texto.
                d.Texto = (d.Texto ?? "").Replace("\r\n", "\n");
                d.Texto = System.Text.RegularExpressions.Regex.Replace(d.Texto, "[\u0000-\u0008\u000B\u000C\u000E-\u001F]", "\n").Trim();
                d.Palavras = ContarPalavras(d.Texto);
                if (d.Palavras == 0) d.Erro = "consegui abrir o ficheiro mas nao tem texto nenhum. Se for um PDF digitalizado, e uma imagem: usa o modo Imagem para markdown.";
            }
            catch (InvalidDataException)
            {
                d.Erro = "este ficheiro parece estar na nuvem ou danificado. Abre-o uma vez no Explorador e tenta outra vez.";
            }
            catch (System.ComponentModel.Win32Exception)
            {
                d.Erro = "nao encontrei o pdftotext, que e preciso para ler PDFs.";
            }
            catch (Exception e) { d.Erro = "nao consegui ler: " + e.Message; }
            return d;
        }

        public static int ContarPalavras(string t)
        {
            if (string.IsNullOrWhiteSpace(t)) return 0;
            return t.Split(new char[] { ' ', '\n', '\t', '\r' }, StringSplitOptions.RemoveEmptyEntries).Length;
        }

        static string DePdf(string caminho)
        {
            System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo();
            psi.FileName = "pdftotext";
            psi.Arguments = "-layout -enc UTF-8 \"" + caminho + "\" -";
            psi.UseShellExecute = false;
            psi.RedirectStandardOutput = true;
            psi.CreateNoWindow = true;
            psi.StandardOutputEncoding = Encoding.UTF8;
            using (System.Diagnostics.Process p = System.Diagnostics.Process.Start(psi))
            {
                string s = p.StandardOutput.ReadToEnd();
                p.WaitForExit(60000);
                return s;
            }
        }

        // Um .docx e um zip. O texto vive em word/document.xml, entre etiquetas.
        static string DeDocx(string caminho)
        {
            using (System.IO.Compression.ZipArchive z = System.IO.Compression.ZipFile.OpenRead(caminho))
            {
                System.IO.Compression.ZipArchiveEntry e = z.GetEntry("word/document.xml");
                if (e == null) return "";
                using (StreamReader r = new StreamReader(e.Open(), Encoding.UTF8))
                {
                    string xml = r.ReadToEnd();
                    xml = System.Text.RegularExpressions.Regex.Replace(xml, "</w:p>", "\n");
                    xml = System.Text.RegularExpressions.Regex.Replace(xml, "<[^>]+>", "");
                    return System.Net.WebUtility.HtmlDecode(xml);
                }
            }
        }
    }

    // ---------------------------------------------------------------------
    // Indice de texto de uma pasta. Ler os ficheiros a cada procura custaria
    // mais de um minuto (medido: 326 ms por PDF). Le-se uma vez e guarda-se.
    // ---------------------------------------------------------------------
    public class Indice
    {
        public class Entrada
        {
            public string Caminho = "", Nome = "", Texto = "";
            public long Tamanho = 0;
            public string Data = "";
        }
        public class Resultado
        {
            public string Caminho = "", Nome = "", Excerto = "";
            public int Pontos = 0;
        }

        public List<Entrada> Entradas = new List<Entrada>();
        public string Pasta = "";
        public string Ficheiro;

        public Indice(string ficheiro) { Ficheiro = ficheiro; Ler(); }

        public void Ler()
        {
            Entradas.Clear();
            try
            {
                if (!File.Exists(Ficheiro)) return;
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 200000000;
                Dictionary<string, object> d = (Dictionary<string, object>)js.DeserializeObject(File.ReadAllText(Ficheiro, Encoding.UTF8));
                if (d.ContainsKey("pasta")) Pasta = Convert.ToString(d["pasta"]);
                if (!d.ContainsKey("entradas")) return;
                foreach (object o in (object[])d["entradas"])
                {
                    Dictionary<string, object> e = (Dictionary<string, object>)o;
                    Entrada x = new Entrada();
                    x.Caminho = Convert.ToString(e["caminho"]);
                    x.Nome = Convert.ToString(e["nome"]);
                    x.Texto = Convert.ToString(e["texto"]);
                    x.Tamanho = Convert.ToInt64(e["tamanho"]);
                    x.Data = Convert.ToString(e["data"]);
                    Entradas.Add(x);
                }
            }
            catch { Entradas.Clear(); }
        }

        public void Gravar()
        {
            try
            {
                List<Dictionary<string, object>> lista = new List<Dictionary<string, object>>();
                foreach (Entrada x in Entradas)
                {
                    Dictionary<string, object> e = new Dictionary<string, object>();
                    e["caminho"] = x.Caminho; e["nome"] = x.Nome; e["texto"] = x.Texto;
                    e["tamanho"] = x.Tamanho; e["data"] = x.Data;
                    lista.Add(e);
                }
                Dictionary<string, object> d = new Dictionary<string, object>();
                d["pasta"] = Pasta; d["entradas"] = lista;
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 200000000;
                File.WriteAllText(Ficheiro, js.Serialize(d), new UTF8Encoding(false));
            }
            catch { }
        }

        static readonly string[] EXTENSOES = { ".pdf", ".docx", ".txt", ".md", ".csv", ".log" };

        // Reindexa so o que mudou: mesmo caminho, tamanho e data ficam como estao.
        public int Construir(string pasta, Action<int, int, string> progresso, Func<bool> parar)
        {
            Pasta = pasta;
            Dictionary<string, Entrada> antigas = new Dictionary<string, Entrada>();
            foreach (Entrada x in Entradas) antigas[x.Caminho] = x;

            List<string> todos = new List<string>();
            foreach (string f in Directory.GetFiles(pasta))
                if (Array.IndexOf(EXTENSOES, Path.GetExtension(f).ToLower()) >= 0) todos.Add(f);

            List<Entrada> novas = new List<Entrada>();
            int feitos = 0, lidos = 0;
            foreach (string f in todos)
            {
                if (parar != null && parar()) break;
                feitos++;
                FileInfo fi = new FileInfo(f);
                string data = fi.LastWriteTimeUtc.ToString("s");
                if (antigas.ContainsKey(f) && antigas[f].Tamanho == fi.Length && antigas[f].Data == data)
                { novas.Add(antigas[f]); if (progresso != null) progresso(feitos, todos.Count, fi.Name); continue; }

                if (progresso != null) progresso(feitos, todos.Count, fi.Name);
                Documento d = Documento.Abrir(f);
                if (d.Erro != null || d.Texto.Length == 0) continue;
                Entrada x = new Entrada();
                x.Caminho = f; x.Nome = fi.Name; x.Texto = d.Texto;
                x.Tamanho = fi.Length; x.Data = data;
                novas.Add(x); lidos++;
            }
            Entradas = novas;
            Gravar();
            return lidos;
        }

        // A procura nao usa o modelo: pontuacao por palavras. Instantanea, e
        // incapaz de inventar um resultado que nao existe.
        public List<Resultado> Procurar(string consulta, int quantos)
        {
            List<string> chave = new List<string>();
            foreach (string w in consulta.ToLower().Split(new char[] { ' ', ',', '.', '?', '!', ';', ':', '"' }, StringSplitOptions.RemoveEmptyEntries))
                if (w.Length > 2 && !chave.Contains(w)) chave.Add(w);

            List<Resultado> saida = new List<Resultado>();
            if (chave.Count == 0) return saida;

            foreach (Entrada x in Entradas)
            {
                string baixo = x.Texto.ToLower();
                int pontos = 0; int onde = -1;
                foreach (string w in chave)
                {
                    int n = 0, i = 0;
                    while ((i = baixo.IndexOf(w, i, StringComparison.Ordinal)) >= 0) { n++; if (onde < 0) onde = i; i += w.Length; if (n > 50) break; }
                    if (n > 0) pontos += n + 5;                       // premio por palavra encontrada
                    if (x.Nome.ToLower().Contains(w)) pontos += 20;   // no nome vale mais
                }
                if (pontos == 0) continue;
                Resultado r = new Resultado();
                r.Caminho = x.Caminho; r.Nome = x.Nome; r.Pontos = pontos;
                if (onde >= 0)
                {
                    int ini = Math.Max(0, onde - 90);
                    int fim = Math.Min(x.Texto.Length, onde + 150);
                    r.Excerto = "..." + x.Texto.Substring(ini, fim - ini).Replace("\n", " ").Trim() + "...";
                }
                saida.Add(r);
            }
            saida.Sort(delegate (Resultado a, Resultado b) { return b.Pontos.CompareTo(a.Pontos); });
            if (saida.Count > quantos) saida.RemoveRange(quantos, saida.Count - quantos);
            return saida;
        }
    }

    // ---------------------------------------------------------------------
    // Duplicados. Agrupa primeiro por tamanho e so calcula a impressao digital
    // dentro de cada grupo: sem isso era preciso ler a pasta toda.
    // ---------------------------------------------------------------------
    public class Duplicados
    {
        public static List<List<string>> Encontrar(string pasta, Action<string> progresso, Func<bool> parar)
        {
            Dictionary<long, List<string>> porTamanho = new Dictionary<long, List<string>>();
            foreach (string f in Directory.GetFiles(pasta))
            {
                FileInfo fi = new FileInfo(f);
                if (fi.Length < 1024) continue;                 // ficheiros minusculos nao interessam
                if (!porTamanho.ContainsKey(fi.Length)) porTamanho[fi.Length] = new List<string>();
                porTamanho[fi.Length].Add(f);
            }

            List<List<string>> grupos = new List<List<string>>();
            int feitos = 0;
            foreach (KeyValuePair<long, List<string>> par in porTamanho)
            {
                if (parar != null && parar()) break;
                if (par.Value.Count < 2) continue;
                Dictionary<string, List<string>> porHash = new Dictionary<string, List<string>>();
                foreach (string f in par.Value)
                {
                    feitos++;
                    if (progresso != null) progresso(Path.GetFileName(f));
                    string h = Hash(f);
                    if (h == null) continue;
                    if (!porHash.ContainsKey(h)) porHash[h] = new List<string>();
                    porHash[h].Add(f);
                }
                foreach (KeyValuePair<string, List<string>> g in porHash)
                    if (g.Value.Count > 1) grupos.Add(g.Value);
            }
            return grupos;
        }

        static string Hash(string caminho)
        {
            try
            {
                using (System.Security.Cryptography.MD5 md5 = System.Security.Cryptography.MD5.Create())
                using (FileStream fs = File.OpenRead(caminho))
                    return BitConverter.ToString(md5.ComputeHash(fs));
            }
            catch { return null; }
        }
    }

    // ---------------------------------------------------------------------
    // Um agente: nome, missao, cerebro, ferramentas e memoria. Cada um e um
    // ficheiro de texto em agentes/<nome>.md, editavel a mao.
    // ---------------------------------------------------------------------
    public class Agente
    {
        public string Nome = "", Cargo = "", Cor = "", Missao = "", Cerebro = "local", Caminho = "";
        public List<string> Ferramentas = new List<string>();
        public List<string> Memoria = new List<string>();
        // Falso quando a cor nao esta no ficheiro e foi so emprestada.
        public bool CorEscolhida = false;

        // A mesma rampa de cores que o Grok Bot usa para distinguir os agentes.
        // Quem nao escolher cor recebe uma, sempre a mesma para o mesmo nome.
        public static readonly string[] CORES = { "#1084fe", "#ff6700", "#00bca6", "#9159fe", "#ff309b", "#97683d" };

        public static string CorPorNome(string nome)
        {
            int soma = 0;
            foreach (char c in nome) soma += c;
            return CORES[soma % CORES.Length];
        }

        public static Agente Ler(string caminho)
        {
            Agente a = new Agente();
            a.Caminho = caminho;
            a.Nome = Path.GetFileNameWithoutExtension(caminho);
            try
            {
                bool naCabeca = false, naMemoria = false;
                foreach (string bruta in File.ReadAllLines(caminho, Encoding.UTF8))
                {
                    string l = bruta.TrimEnd();
                    if (l.Trim() == "---") { naCabeca = !naCabeca; continue; }
                    if (l.Trim().ToLower().StartsWith("## memoria")) { naMemoria = true; continue; }
                    if (naMemoria) { a.Memoria.Add(l); continue; }
                    if (!naCabeca) continue;
                    int d = l.IndexOf(':');
                    if (d < 0) continue;
                    string k = l.Substring(0, d).Trim().ToLower(), v = l.Substring(d + 1).Trim();
                    if (k == "nome") a.Nome = v;
                    else if (k == "cargo") a.Cargo = v;
                    else if (k == "cor") { a.Cor = v; a.CorEscolhida = v.Length > 0; }
                    else if (k == "missao") a.Missao = v;
                    else if (k == "cerebro") a.Cerebro = v.ToLower();
                    else if (k == "ferramentas")
                        foreach (string f in v.Split(','))
                            if (f.Trim().Length > 0) a.Ferramentas.Add(f.Trim());
                }
            }
            catch { }
            // Os ficheiros antigos nao tem cor nenhuma: damos-lhe uma sem gravar nada.
            if (a.Cor.Length == 0) a.Cor = CorPorNome(a.Nome);
            return a;
        }

        public static List<Agente> Todos(string pastaAgentes)
        {
            List<Agente> lista = new List<Agente>();
            try
            {
                if (!Directory.Exists(pastaAgentes)) return lista;
                // Sem recursao: a pasta de arquivo fica de fora de proposito.
                foreach (string f in Directory.GetFiles(pastaAgentes, "*.md")) lista.Add(Ler(f));
                // Quem nao escolheu cor leva uma pela posicao na lista, para que
                // dois agentes seguidos nunca fiquem com a mesma bolinha.
                for (int i = 0; i < lista.Count; i++)
                    if (!lista[i].CorEscolhida) lista[i].Cor = CORES[i % CORES.Length];
            }
            catch { }
            return lista;
        }

        // Reescreve a cabeca do ficheiro. A memoria passa intacta, linha a linha.
        // O ficheiro nunca muda de nome, mesmo que o agente mude: assim nada se perde.
        public bool Gravar(out string erro)
        {
            erro = null;
            try
            {
                StringBuilder sb = new StringBuilder();
                sb.AppendLine("---");
                sb.AppendLine("nome: " + UmaLinha(Nome));
                sb.AppendLine("cargo: " + UmaLinha(Cargo));
                sb.AppendLine("cor: " + UmaLinha(Cor));
                sb.AppendLine("missao: " + UmaLinha(Missao));
                sb.AppendLine("cerebro: " + UmaLinha(Cerebro));
                sb.AppendLine("ferramentas: " + string.Join(", ", Ferramentas.ToArray()));
                sb.AppendLine("---");
                sb.AppendLine();
                sb.AppendLine("## memoria");
                foreach (string l in Memoria) sb.AppendLine(l);
                File.WriteAllText(Caminho, sb.ToString(), new UTF8Encoding(false));
                return true;
            }
            catch (Exception e) { erro = e.Message; return false; }
        }

        // O frontmatter e uma linha por campo. Quebras de linha partiam-no.
        static string UmaLinha(string t)
        {
            if (t == null) return "";
            return t.Replace("\r", " ").Replace("\n", " ").Trim();
        }

        public static Agente Criar(string pasta, string nome, out string erro)
        {
            erro = null;
            try
            {
                if (!Directory.Exists(pasta)) Directory.CreateDirectory(pasta);
                string limpo = nome == null ? "" : nome.Trim();
                foreach (char c in Path.GetInvalidFileNameChars()) limpo = limpo.Replace(c, ' ');
                limpo = limpo.Trim();
                if (limpo.Length == 0) { erro = "o nome esta vazio"; return null; }
                string caminho = Path.Combine(pasta, limpo + ".md");
                if (File.Exists(caminho)) { erro = "ja existe um agente com esse nome"; return null; }
                Agente a = new Agente();
                a.Caminho = caminho;
                a.Nome = limpo;
                a.Cargo = "";
                a.Cerebro = "local";
                a.Missao = "Escreve aqui o que este agente faz.";
                a.Cor = CorPorNome(limpo);
                if (!a.Gravar(out erro)) return null;
                return a;
            }
            catch (Exception e) { erro = e.Message; return null; }
        }

        // Arquivar, nunca apagar. Vai para a subpasta de arquivo e pode voltar a mao.
        public bool Arquivar(out string erro)
        {
            erro = null;
            try
            {
                string pasta = Path.Combine(Path.GetDirectoryName(Caminho), "_arquivo");
                if (!Directory.Exists(pasta)) Directory.CreateDirectory(pasta);
                string baseNome = Path.GetFileNameWithoutExtension(Caminho);
                string destino = Path.Combine(pasta, baseNome + ".md");
                int n = 2;
                while (File.Exists(destino))
                    destino = Path.Combine(pasta, baseNome + " (" + (n++) + ").md");
                File.Move(Caminho, destino);
                return true;
            }
            catch (Exception e) { erro = e.Message; return false; }
        }

        public void GuardarNaMemoria(string pergunta, string resposta)
        {
            try
            {
                StringBuilder sb = new StringBuilder();
                sb.AppendLine();
                sb.AppendLine("### " + DateTime.Now.ToString("yyyy-MM-dd HH:mm"));
                sb.AppendLine("**Tu:** " + pergunta.Replace(Environment.NewLine, " "));
                sb.AppendLine();
                sb.AppendLine(resposta);
                File.AppendAllText(Caminho, sb.ToString(), new UTF8Encoding(false));
                foreach (string l in sb.ToString().Split('\n')) Memoria.Add(l.TrimEnd());
            }
            catch { }
        }

        // A conversa desfeita em pares (pergunta, resposta). E isto que da o fio:
        // sem isto cada pergunta seria independente da anterior.
        public List<string[]> Turnos(int quantos)
        {
            List<string[]> todos = new List<string[]>();
            string p = null;
            StringBuilder r = new StringBuilder();
            foreach (string bruta in Memoria)
            {
                string l = bruta == null ? "" : bruta.TrimEnd();
                string t = l.TrimStart();
                if (t.StartsWith("### "))
                {
                    if (p != null) todos.Add(new string[] { p, r.ToString().Trim() });
                    p = null; r.Length = 0;
                    continue;
                }
                if (t.StartsWith("**Tu:**"))
                {
                    if (p != null) todos.Add(new string[] { p, r.ToString().Trim() });
                    p = t.Substring(7).Trim(); r.Length = 0;
                    continue;
                }
                if (p != null) r.AppendLine(l);
            }
            if (p != null) todos.Add(new string[] { p, r.ToString().Trim() });
            if (quantos > 0 && todos.Count > quantos)
                todos = todos.GetRange(todos.Count - quantos, quantos);
            return todos;
        }

        // As ultimas trocas, para o agente se lembrar sem encher o contexto.
        public string MemoriaRecente(int linhas)
        {
            if (Memoria.Count == 0) return "";
            int ini = Math.Max(0, Memoria.Count - linhas);
            StringBuilder sb = new StringBuilder();
            for (int i = ini; i < Memoria.Count; i++) sb.AppendLine(Memoria[i]);
            return sb.ToString().Trim();
        }
    }

    // ---------------------------------------------------------------------
    // O Claude Code que o utilizador ja tem instalado, chamado sem interface.
    // Usa a subscricao dele: nao e precisa chave de API nenhuma.
    // Verificado: responde em ~6 s, e pesquisa na web em ~21 s com fontes.
    // ---------------------------------------------------------------------
    public class Claude
    {
        public static string Caminho()
        {
            string npm = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "npm", "claude.cmd");
            return File.Exists(npm) ? npm : null;
        }

        public static bool Existe() { return Caminho() != null; }

        // As ferramentas vao explicitas na chamada, agente a agente. Sem isto o
        // Claude bloqueia a pesquisa e avisa - nao inventa, mas tambem nao ajuda.
        public static string Perguntar(string pedido, List<string> ferramentas, out string erro)
        {
            erro = null;
            string exe = Caminho();
            if (exe == null) { erro = "nao encontrei o Claude Code instalado"; return null; }

            System.Diagnostics.ProcessStartInfo psi = new System.Diagnostics.ProcessStartInfo();
            psi.FileName = "cmd.exe";
            StringBuilder args = new StringBuilder();
            args.Append("/c \"\"").Append(exe).Append("\" -p");
            if (ferramentas != null && ferramentas.Count > 0)
            {
                args.Append(" --allowedTools");
                foreach (string f in ferramentas) args.Append(" ").Append(f);
            }
            args.Append("\"");
            psi.Arguments = args.ToString();
            psi.UseShellExecute = false;
            psi.RedirectStandardInput = true;
            psi.RedirectStandardOutput = true;
            psi.RedirectStandardError = true;
            psi.CreateNoWindow = true;
            psi.StandardOutputEncoding = Encoding.UTF8;

            try
            {
                using (System.Diagnostics.Process p = System.Diagnostics.Process.Start(psi))
                {
                    // O pedido vai pela entrada padrao: assim nao ha problemas de
                    // aspas nem de acentos, por muito comprido que seja.
                    p.StandardInput.Write(pedido);
                    p.StandardInput.Close();
                    string saida = p.StandardOutput.ReadToEnd();
                    p.WaitForExit(600000);
                    if (saida.Trim().Length == 0) { erro = "o Claude nao devolveu nada"; return null; }
                    return saida.Trim();
                }
            }
            catch (Exception e) { erro = e.Message; return null; }
        }
    }

    // Ligacao ao Ollama. Tudo local, em http://localhost:11434
    // ---------------------------------------------------------------------
    public class Ollama
    {
        public const string BASE = "http://localhost:11434";
        public string Modelo = "prompt-smith";
        // Guardamos o pedido em curso para o botao Parar o poder abortar.
        HttpWebRequest emCurso = null;
        public bool Cancelado = false;
        // Quando maior que zero, pede este contexto no pedido em vez do que esta
        // no Modelfile. O modo de conversar com ficheiros precisa de mais espaco.
        public int NumCtx = 0;

        void PorContexto(Dictionary<string, object> c)
        {
            if (NumCtx <= 0) return;
            Dictionary<string, object> o = new Dictionary<string, object>();
            o["num_ctx"] = NumCtx; o["num_gpu"] = 99;
            c["options"] = o;
        }

        public void Cancelar()
        {
            Cancelado = true;
            HttpWebRequest p = emCurso;
            if (p != null) { try { p.Abort(); } catch { } }
        }
        public string ModeloVisao = "qwen2.5vl:3b";

        string Enviar(string caminho, Dictionary<string, object> corpo)
        {
            JavaScriptSerializer js = new JavaScriptSerializer();
            js.MaxJsonLength = 80000000;
            byte[] dados = Encoding.UTF8.GetBytes(js.Serialize(corpo));
            HttpWebRequest p = (HttpWebRequest)WebRequest.Create(BASE + caminho);
            p.Method = "POST"; p.ContentType = "application/json";
            p.ContentLength = dados.Length; p.Timeout = 900000; p.ReadWriteTimeout = 900000;
            emCurso = p;
            using (Stream s = p.GetRequestStream()) s.Write(dados, 0, dados.Length);
            using (HttpWebResponse res = (HttpWebResponse)p.GetResponse())
            using (StreamReader l = new StreamReader(res.GetResponseStream(), Encoding.UTF8))
                return l.ReadToEnd();
        }

        Dictionary<string, object> Ler(string json)
        {
            JavaScriptSerializer js = new JavaScriptSerializer();
            js.MaxJsonLength = 80000000;
            return (Dictionary<string, object>)js.DeserializeObject(json);
        }

        // Conversa. Aproveita os exemplos gravados no Modelfile, que so contam aqui.
        public string Conversar(List<Dictionary<string, string>> mensagens)
        {
            Dictionary<string, object> c = new Dictionary<string, object>();
            c["model"] = Modelo; c["messages"] = mensagens;
            c["stream"] = false; c["keep_alive"] = "10m";
            Dictionary<string, object> d = Ler(Enviar("/api/chat", c));
            if (!d.ContainsKey("message")) return "";
            Dictionary<string, object> m = (Dictionary<string, object>)d["message"];
            return m.ContainsKey("content") ? Convert.ToString(m["content"]) : "";
        }

        // Uma tarefa isolada, com o system prompt substituido. Sem os exemplos.
        public string Tarefa(string sistema, string pedido)
        {
            Dictionary<string, object> c = new Dictionary<string, object>();
            c["model"] = Modelo; c["system"] = sistema; c["prompt"] = pedido;
            c["stream"] = false; c["keep_alive"] = "10m";
            Dictionary<string, object> d = Ler(Enviar("/api/generate", c));
            return d.ContainsKey("response") ? Convert.ToString(d["response"]).Trim() : "";
        }

        // Modelo de visao: descreve ou transcreve uma imagem.
        public string Visao(string pedido, string caminhoImagem)
        {
            string b64 = Convert.ToBase64String(File.ReadAllBytes(caminhoImagem));
            Dictionary<string, object> c = new Dictionary<string, object>();
            c["model"] = ModeloVisao; c["prompt"] = pedido;
            c["images"] = new string[] { b64 };
            c["stream"] = false; c["keep_alive"] = "5m";
            Dictionary<string, object> d = Ler(Enviar("/api/generate", c));
            return d.ContainsKey("response") ? Convert.ToString(d["response"]).Trim() : "";
        }

        // ---- em directo ----
        // O Ollama devolve uma linha de JSON por cada pedaco de texto que escreve.
        // Lemos linha a linha e vamos entregando, para o texto aparecer a medida
        // que sai em vez de o utilizador esperar 30 segundos a olhar para o nada.
        string EmDirecto(string caminho, Dictionary<string, object> corpo, string qual, Action<string> pedaco)
        {
            corpo["stream"] = true;
            PorContexto(corpo);
            JavaScriptSerializer js = new JavaScriptSerializer();
            js.MaxJsonLength = 80000000;
            byte[] dados = Encoding.UTF8.GetBytes(js.Serialize(corpo));
            HttpWebRequest p = (HttpWebRequest)WebRequest.Create(BASE + caminho);
            p.Method = "POST"; p.ContentType = "application/json";
            p.ContentLength = dados.Length; p.Timeout = 900000; p.ReadWriteTimeout = 900000;
            emCurso = p;
            using (Stream s = p.GetRequestStream()) s.Write(dados, 0, dados.Length);

            StringBuilder tudo = new StringBuilder();
            using (HttpWebResponse res = (HttpWebResponse)p.GetResponse())
            using (StreamReader l = new StreamReader(res.GetResponseStream(), Encoding.UTF8))
            {
                string linha;
                while ((linha = l.ReadLine()) != null)
                {
                    if (linha.Trim().Length == 0) continue;
                    Dictionary<string, object> d;
                    try { d = (Dictionary<string, object>)js.DeserializeObject(linha); }
                    catch { continue; }
                    string parte = "";
                    if (qual == "response")
                    {
                        if (d.ContainsKey("response")) parte = Convert.ToString(d["response"]);
                    }
                    else if (d.ContainsKey("message"))
                    {
                        Dictionary<string, object> m = (Dictionary<string, object>)d["message"];
                        if (m.ContainsKey("content")) parte = Convert.ToString(m["content"]);
                    }
                    if (parte.Length > 0) { tudo.Append(parte); if (pedaco != null) pedaco(parte); }
                    if (d.ContainsKey("done") && Convert.ToString(d["done"]) == "True") break;
                }
            }
            return tudo.ToString();
        }

        public string ConversarEmDirecto(List<Dictionary<string, string>> mensagens, Action<string> pedaco)
        {
            Dictionary<string, object> c = new Dictionary<string, object>();
            c["model"] = Modelo; c["messages"] = mensagens; c["keep_alive"] = "10m";
            return EmDirecto("/api/chat", c, "message", pedaco);
        }

        public string TarefaEmDirecto(string sistema, string pedido, Action<string> pedaco)
        {
            Dictionary<string, object> c = new Dictionary<string, object>();
            c["model"] = Modelo; c["system"] = sistema; c["prompt"] = pedido; c["keep_alive"] = "10m";
            return EmDirecto("/api/generate", c, "response", pedaco);
        }

        public bool Vivo(out string detalhe)
        {
            try
            {
                HttpWebRequest p = (HttpWebRequest)WebRequest.Create(BASE + "/api/tags");
                p.Timeout = 4000;
                using (HttpWebResponse r = (HttpWebResponse)p.GetResponse())
                using (StreamReader l = new StreamReader(r.GetResponseStream()))
                {
                    string corpo = l.ReadToEnd();
                    if (!corpo.Contains(Modelo)) { detalhe = "o Ollama responde mas falta o modelo " + Modelo; return false; }
                    detalhe = corpo.Contains(ModeloVisao) ? "pronto" : "pronto (sem modelo de visao)";
                    return true;
                }
            }
            catch { detalhe = "o Ollama nao esta a responder"; return false; }
        }
    }

    // ---------------------------------------------------------------------
    // OCR que ja vem dentro do Windows. Nao descarrega nada.
    // ---------------------------------------------------------------------
    public class Ocr
    {
        // Espera por uma operacao do Windows sem precisar de await, que o
        // compilador do Windows nao consegue resolver sem o SDK instalado.
        static T Esperar<T>(IAsyncOperation<T> op)
        {
            ManualResetEventSlim ev = new ManualResetEventSlim(false);
            T resultado = default(T); Exception falha = null;
            op.Completed = delegate (IAsyncOperation<T> o, AsyncStatus s)
            {
                try { if (s == AsyncStatus.Completed) resultado = o.GetResults(); else falha = new Exception("operacao " + s); }
                catch (Exception e) { falha = e; }
                ev.Set();
            };
            ev.Wait();
            if (falha != null) throw falha;
            return resultado;
        }

        public static string Ler(string caminho)
        {
            StorageFile f = Esperar(StorageFile.GetFileFromPathAsync(caminho));
            IRandomAccessStream fluxo = Esperar(f.OpenAsync(FileAccessMode.Read));
            BitmapDecoder dec = Esperar(BitmapDecoder.CreateAsync(fluxo));
            SoftwareBitmap img = Esperar(dec.GetSoftwareBitmapAsync());
            OcrEngine motor = OcrEngine.TryCreateFromUserProfileLanguages();
            if (motor == null) return "";
            OcrResult r = Esperar(motor.RecognizeAsync(img));
            StringBuilder sb = new StringBuilder();
            foreach (OcrLine linha in r.Lines) sb.AppendLine(linha.Text);
            return sb.ToString().Trim();
        }

        public static bool Disponivel()
        {
            try { return OcrEngine.TryCreateFromUserProfileLanguages() != null; }
            catch { return false; }
        }
    }
}







~~~~

