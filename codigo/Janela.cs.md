# app\Janela.cs

O ficheiro inteiro esta no bloco abaixo. Grava-o em `app\Janela.cs`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `app\Janela.cs`

<!-- destino: app\Janela.cs | codificacao: bom -->
~~~~csharp
// Janela do prompt-smith. O desenho esta em janela.xaml, ao lado do .exe, e pode
// ser editado sem recompilar. As preferencias e o historico ficam em ficheiros
// na mesma pasta, para poderes levar tudo contigo.
using System;
using System.Collections.Generic;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Interop;
using System.Windows.Markup;
using System.Windows.Media;
using System.Web.Script.Serialization;

namespace PromptSmith
{
    public class App
    {
        static Window J;
        static Motor motor;
        static string pasta;
        static int modo = 0;                    // 0 construir, 1 diagnosticar, 2 imagem
        static string imagemEscolhida = null;
        static List<Dictionary<string, string>> historico = new List<Dictionary<string, string>>();

        static Rect anterior;
        static bool grande = false, ecraCheio = false;
        static bool temaClaro = false;
        static double tamanhoLetra = 12.5;
        static bool colarSozinho = false;
        static bool aGerar = false;
        static Biblioteca biblio;
        static Documento docAberto = null;
        static Indice indice;
        static string pastaEscolhida = null;
        static List<List<string>> gruposDup = new List<List<string>>();
        static List<Agente> agentes = new List<Agente>();
        static Agente agenteActivo = null;
        static string pastaAgentes;

        static List<Dictionary<string, string>> registos = new List<Dictionary<string, string>>();

        static T Achar<T>(string nome) where T : class { return J.FindName(nome) as T; }

        // ------------------------------------------------------------------
        [STAThread]
        public static void Main()
        {
            pasta = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);

            string xaml = null;
            string fora = Path.Combine(pasta, "janela.xaml");
            if (File.Exists(fora)) xaml = File.ReadAllText(fora, Encoding.UTF8);
            else using (Stream s = Assembly.GetExecutingAssembly().GetManifestResourceStream("janela.xaml"))
                if (s != null) using (StreamReader r = new StreamReader(s, Encoding.UTF8)) xaml = r.ReadToEnd();
            if (xaml == null) { MessageBox.Show("Falta o ficheiro janela.xaml."); return; }
            using (MemoryStream ms = new MemoryStream(Encoding.UTF8.GetBytes(xaml)))
                J = (Window)XamlReader.Load(ms);

            string regras = Path.Combine(pasta, "regras.md");
            if (!File.Exists(regras)) regras = Path.Combine(pasta, @"..\regras\regras.md");
            motor = new Motor(regras);

            biblio = new Biblioteca(Path.Combine(pasta, "biblioteca.md"));
            indice = new Indice(Path.Combine(pasta, "indice.json"));
            pastaAgentes = Path.Combine(pasta, "agentes");
            if (!Directory.Exists(pastaAgentes)) pastaAgentes = Path.GetFullPath(Path.Combine(pasta, @"..\agentes"));
            agentes = Agente.Todos(pastaAgentes);
            LerDefinicoes();
            LerRegistos();
            Ligar();
            AplicarTema();
            AplicarLetra();
            Achar<CheckBox>("ColarSozinho").IsChecked = colarSozinho;
            ComboBox ac = Achar<ComboBox>("Accao");
            foreach (KeyValuePair<string, string> a in motor.R.Accoes) ac.Items.Add(a.Key);
            if (ac.Items.Count > 0) ac.SelectedIndex = 0;
            MostrarEquipa();
            MudarModo(0);
            MostrarLista();

            new Thread(delegate () {
                string detalhe; bool vivo = motor.IA.Vivo(out detalhe);
                if (motor.R.Erro != null) { vivo = false; detalhe = motor.R.Erro; }
                J.Dispatcher.Invoke((Action)delegate { Achar<TextBlock>("Estado").Text = detalhe; });
                if (!vivo) return;
                // Pre-aquecer. Medido: a primeira resposta levava 10,1 segundos ate ao
                // primeiro caractere; a segunda, 0,9. Ou seja, a demora nao e carregar o
                // modelo - e ele ler o system prompt e os exemplos. O Ollama guarda esse
                // trabalho, mas so para o caminho das conversas (/api/chat), e so se o
                // inicio da conversa for igual. Por isso o aquecimento tem de ser uma
                // conversa a serio, e nao um pedido qualquer.
                J.Dispatcher.Invoke((Action)delegate { Achar<TextBlock>("Estado").Text = "a preparar o modelo..."; });
                try {
                    List<Dictionary<string, string>> aquecer = new List<Dictionary<string, string>>();
                    Dictionary<string, string> u = new Dictionary<string, string>();
                    u["role"] = "user"; u["content"] = "ok";
                    aquecer.Add(u);
                    motor.IA.Conversar(aquecer);
                } catch { }
                J.Dispatcher.Invoke((Action)delegate {
                    if (!aGerar) Achar<TextBlock>("Estado").Text = "pronto";
                });
            }) { IsBackground = true }.Start();

            new Application().Run(J);
        }

        // ------------------------------------------------------------------
        static void Ligar()
        {
            Achar<Grid>("Barra").MouseLeftButtonDown += delegate (object o, MouseButtonEventArgs a) {
                if (a.ClickCount == 2) { Alternar(false); return; }
                if (!grande && !ecraCheio) J.DragMove();
            };
            Achar<Button>("BtFechar").Click     += delegate { GravarDefinicoes(); J.Close(); };
            Achar<Button>("BtMinimizar").Click  += delegate { J.WindowState = WindowState.Minimized; };
            Achar<Button>("BtMaximizar").Click  += delegate { Alternar(false); };
            Achar<Button>("BtTema").Click       += delegate { temaClaro = !temaClaro; AplicarTema(); GravarDefinicoes(); };
            Achar<Button>("BtLetraMais").Click  += delegate { MudarLetra(+1); };
            Achar<Button>("BtLetraMenos").Click += delegate { MudarLetra(-1); };

            Achar<Button>("BtConstruir").Click    += delegate { MudarModo(0); };
            Achar<Button>("BtDiagnosticar").Click += delegate { MudarModo(1); };
            Achar<Button>("BtImagem").Click       += delegate { MudarModo(2); };
            Achar<Button>("BtReescrever").Click   += delegate { MudarModo(3); };
            Achar<Button>("BtBiblioteca").Click   += delegate { MudarModo(4); };
            Achar<Button>("BtFicheiro").Click     += delegate { MudarModo(5); };
            Achar<Button>("BtProcurar").Click     += delegate { MudarModo(6); };
            Achar<Button>("BtDuplicados").Click   += delegate { MudarModo(7); };
            Achar<Button>("BtAgentes").Click      += delegate { agentes = Agente.Todos(pastaAgentes); MostrarEquipa(); MudarModo(8); };
            Achar<Button>("BtAbrirAgente").Click  += delegate { if (agenteActivo != null) AbrirNoBloco(agenteActivo.Caminho); };
            Achar<Button>("BtNovoAgente").Click   += delegate { NovoAgente(); };
            Achar<Button>("BtGravarAgente").Click += delegate { GravarFicha(); };
            Achar<Button>("BtArquivarAgente").Click += delegate { ArquivarAgente(); };
            Achar<Button>("BtMais").Click         += delegate { Anexar(); };
            // A lista da equipa e so dela: um clique escolhe com quem falas.
            Achar<ListBox>("Equipa").SelectionChanged += delegate (object o, SelectionChangedEventArgs a) {
                if (aMexerNaEquipa) return;
                ListBox l = (ListBox)o;
                if (l.SelectedIndex < 0) return;
                if (modo != 8) { MudarModo(8); return; }   // o MudarModo ja escolhe
                EscolherAgente(l.SelectedIndex);
            };
            Achar<Button>("BtEscolherPasta").Click += delegate { EscolherPasta(); };
            Achar<Button>("BtIndexar").Click      += delegate { Indexar(); };
            Achar<Button>("BtDesfazer").Click     += delegate { Desfazer(); };
            Achar<Button>("BtEscolherFicheiro").Click += delegate { EscolherFicheiro(); };
            Achar<Button>("BtGuardarBiblioteca").Click += delegate { GuardarNaBiblioteca(); };
            Achar<Button>("BtAbrirBiblioteca").Click   += delegate { AbrirNoBloco(biblio.Caminho); };

            Achar<Button>("BtEnviar").Click    += delegate { if (aGerar) Parar(); else Enviar(); };
            Achar<Button>("BtRecomecar").Click += delegate { MudarModo(modo); };
            Achar<Button>("BtCopiar").Click       += delegate { Copiar(false); };
            Achar<Button>("BtCopiarPrompt").Click += delegate { Copiar(true); };
            Achar<Button>("BtAbrirRegras").Click  += delegate { AbrirRegras(); };
            Achar<Button>("BtGuardar").Click   += delegate { Guardar(); };
            Achar<Button>("BtEscolherImagem").Click += delegate { EscolherImagem(); };
            Achar<Button>("BtLimparHist").Click += delegate { LimparRegistos(); };

            // A mesma lista mostra o historico ou a biblioteca, conforme o modo.
            Achar<ListBox>("Historico").SelectionChanged += delegate (object o, SelectionChangedEventArgs a) {
                ListBox l = (ListBox)o;
                if (l.SelectedIndex < 0) return;
                if (modo == 4) { if (l.SelectedIndex < biblio.Itens.Count) AbrirDaBiblioteca(l.SelectedIndex); }
                else if (l.SelectedIndex < registos.Count) AbrirRegisto(l.SelectedIndex);
            };

            TextBox e = Achar<TextBox>("Entrada");
            e.KeyDown += delegate (object o, KeyEventArgs a) {
                if (a.Key == Key.Enter && (Keyboard.Modifiers & ModifierKeys.Control) != 0) { a.Handled = true; Enviar(); }
            };
            e.PreviewDragOver += delegate (object o, DragEventArgs a) { a.Effects = DragDropEffects.Copy; a.Handled = true; };
            e.Drop += delegate (object o, DragEventArgs a) {
                if (a.Data.GetDataPresent(DataFormats.FileDrop)) {
                    string[] f = (string[])a.Data.GetData(DataFormats.FileDrop);
                    if (f.Length == 0) return;
                    string ext = Path.GetExtension(f[0]).ToLower();
                    // Uma imagem vai para o modo imagem; um documento vai para o modo ficheiro.
                    if (ext == ".png" || ext == ".jpg" || ext == ".jpeg" || ext == ".bmp" || ext == ".gif" || ext == ".tif" || ext == ".tiff")
                        { MudarModo(2); UsarImagem(f[0]); }
                    else { MudarModo(5); UsarFicheiro(f[0]); }
                }
            };

            J.Loaded += delegate { RegistarAtalho(); };
            Achar<CheckBox>("ColarSozinho").Click += delegate { GravarDefinicoes(); };
            J.PreviewKeyDown += delegate (object o, KeyEventArgs a) {
                bool ctrl = (Keyboard.Modifiers & ModifierKeys.Control) != 0;
                if (a.Key == Key.F11) { a.Handled = true; Alternar(true); }
                else if (a.Key == Key.Escape && (grande || ecraCheio)) { a.Handled = true; Repor(); }
                else if (ctrl && (a.Key == Key.OemPlus || a.Key == Key.Add)) { a.Handled = true; MudarLetra(+1); }
                else if (ctrl && (a.Key == Key.OemMinus || a.Key == Key.Subtract)) { a.Handled = true; MudarLetra(-1); }
            };
            // Ctrl + roda do rato sobre o resultado muda o tamanho da letra.
            Achar<ScrollViewer>("Rolo").PreviewMouseWheel += delegate (object o, MouseWheelEventArgs a) {
                if ((Keyboard.Modifiers & ModifierKeys.Control) != 0) { a.Handled = true; MudarLetra(a.Delta > 0 ? +1 : -1); }
            };
        }

        // ---------------- tema ----------------
        static SolidColorBrush Cor(string hex) { return new SolidColorBrush((Color)ColorConverter.ConvertFromString(hex)); }

        static void AplicarTema()
        {
            string[,] escuro = {
                {"Fundo","#0F1115"},{"Painel","#161922"},{"Cartao","#1B1F2A"},{"Bordo","#252A36"},
                {"Texto","#E7E9EF"},{"Apagado","#868D9E"},{"Realce","#E08A45"},{"RealceE","#3A2A1C"},
                {"Passar","#20252F"},{"RealceC","#EC9A57"},{"Inactivo","#2A2F3B"},{"InactivoT","#6C7381"},
                {"SobreRealce","#1A1206"},{"BolhaAgente","#232936"},{"BolhaTu","#E7E9EF"},{"TextoBolhaTu","#0F1115"} };
            string[,] claro = {
                {"Fundo","#FCFCFC"},{"Painel","#F7F7F7"},{"Cartao","#FFFFFF"},{"Bordo","#DEDEDF"},
                {"Texto","#141414"},{"Apagado","#6B6B70"},{"Realce","#070707"},{"RealceE","#E4E4E6"},
                {"Passar","#EFEFEF"},{"RealceC","#2F2F2F"},{"Inactivo","#E6E6E8"},{"InactivoT","#A3A3A8"},
                {"SobreRealce","#FCFCFC"},{"BolhaAgente","#EEEEEE"},{"BolhaTu","#070707"},{"TextoBolhaTu","#FCFCFC"} };
            string[,] t = temaClaro ? claro : escuro;
            for (int i = 0; i < t.GetLength(0); i++) J.Resources[t[i, 0]] = Cor(t[i, 1]);
            Achar<Button>("BtTema").Content = temaClaro ? "\uE708" : "\uE706";
            if (Achar<Button>("BtConstruir") != null) { PintarModos(); MostrarLista(); MostrarEquipa(); ModoConversa(modo == 8); PintarCabecalho(); PintarCores(); }
        }

        // ---------------- tamanho da letra ----------------
        static void MudarLetra(int passo)
        {
            tamanhoLetra = Math.Max(9, Math.Min(24, tamanhoLetra + passo));
            AplicarLetra();
            GravarDefinicoes();
            Achar<TextBlock>("Estado").Text = "letra a " + tamanhoLetra + " pontos";
        }
        static void AplicarLetra()
        {
            Achar<TextBox>("Saida").FontSize = tamanhoLetra;
            Achar<TextBox>("Entrada").FontSize = Math.Max(12, tamanhoLetra + 1.5);
        }

        // ---------------- tamanho da janela ----------------
        [DllImport("user32.dll")] static extern IntPtr MonitorFromWindow(IntPtr h, int flag);
        [DllImport("user32.dll")] static extern bool GetMonitorInfo(IntPtr mon, ref INFO info);
        [StructLayout(LayoutKind.Sequential)] struct CAIXA { public int Esq, Cima, Dir, Baixo; }
        [StructLayout(LayoutKind.Sequential)] struct INFO { public int tamanho; public CAIXA monitor; public CAIXA util; public int bandeiras; }

        // Perguntamos ao Windows qual o monitor onde a janela esta. Sem isto ela
        // saltava sempre para o monitor principal, que aqui nem e o maior.
        static Rect AreaDoMonitor(bool cheio)
        {
            try
            {
                IntPtr h = new WindowInteropHelper(J).Handle;
                IntPtr mon = MonitorFromWindow(h, 2);
                INFO info = new INFO(); info.tamanho = Marshal.SizeOf(typeof(INFO));
                if (!GetMonitorInfo(mon, ref info)) return SystemParameters.WorkArea;
                CAIXA c = cheio ? info.monitor : info.util;
                Matrix m = PresentationSource.FromVisual(J).CompositionTarget.TransformFromDevice;
                return new Rect(m.Transform(new Point(c.Esq, c.Cima)), m.Transform(new Point(c.Dir, c.Baixo)));
            }
            catch { return SystemParameters.WorkArea; }
        }

        static void Alternar(bool cheio)
        {
            if (grande || ecraCheio) { Repor(); if (cheio == ecraCheio) return; }
            anterior = new Rect(J.Left, J.Top, J.Width, J.Height);
            Rect r = AreaDoMonitor(cheio);
            J.Left = r.Left; J.Top = r.Top; J.Width = r.Width; J.Height = r.Height;
            grande = !cheio; ecraCheio = cheio;
            Moldura(false);
            Achar<Button>("BtMaximizar").Content = "\uE923";
        }
        static void Repor()
        {
            if (anterior.Width > 0) { J.Left = anterior.Left; J.Top = anterior.Top; J.Width = anterior.Width; J.Height = anterior.Height; }
            grande = false; ecraCheio = false;
            Moldura(true);
            Achar<Button>("BtMaximizar").Content = "\uE922";
        }
        static void Moldura(bool redondos)
        {
            Achar<Border>("Raiz").CornerRadius = new CornerRadius(redondos ? 12 : 0);
            Achar<Border>("Lado").CornerRadius = new CornerRadius(0, 0, 0, redondos ? 11 : 0);
        }

        // ---------------- modos ----------------
        static void MudarModo(int m)
        {
            modo = m; historico.Clear(); imagemEscolhida = null;
            Achar<TextBox>("Entrada").Text = "";
            Achar<TextBox>("Saida").Text = "";
            Achar<TextBlock>("Rodape").Text = "";
            Achar<ListBox>("Historico").SelectedIndex = -1;
            LimparFio();

            AplicarModoUI(m);
            // No modo da equipa nao se perde a conversa: volta a abrir o agente.
            // Uma escolha ja feita na lista manda sempre: e o clique do utilizador.
            if (m == 8) {
                ListBox eq = Achar<ListBox>("Equipa");
                if (eq.SelectedIndex >= 0) EscolherAgente(eq.SelectedIndex);
                else {
                    int k = IndiceDe(agenteActivo);
                    if (k >= 0) eq.SelectedIndex = k;   // o evento trata do resto
                    else agenteActivo = null;
                }
            }
        }

        // Tudo o que muda no ecra quando se muda de modo. Fica num so sitio para
        // nao acontecer o que aconteceu: reabrir um diagnostico do historico e o
        // titulo continuar a dizer "Construir um prompt".
        static void AplicarModoUI(int m)
        {
            modo = m;
            PintarModos();
            Achar<StackPanel>("PainelOpcoes").Visibility     = (m == 1) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelImagem").Visibility     = (m == 2) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelAccoes").Visibility     = (m == 3) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelBiblioteca").Visibility = (m == 4) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelFicheiro").Visibility   = (m == 5) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelPasta").Visibility      = (m == 6 || m == 7) ? Visibility.Visible : Visibility.Collapsed;
            Achar<StackPanel>("PainelAgente").Visibility     = (m == 8) ? Visibility.Visible : Visibility.Collapsed;
            Achar<TextBlock>("TituloLista").Text = (m == 4) ? "OS TEUS PROMPTS" : "HISTORICO";
            Achar<Button>("BtIndexar").Visibility            = (m == 6) ? Visibility.Visible : Visibility.Collapsed;
            Achar<Button>("BtDesfazer").Visibility           = (m == 7) ? Visibility.Visible : Visibility.Collapsed;
            Achar<Button>("BtEscolherImagem").Visibility     = (m == 2) ? Visibility.Visible : Visibility.Collapsed;
            Achar<Button>("BtEnviar").Visibility             = (m == 4) ? Visibility.Collapsed : Visibility.Visible;
            Achar<Button>("BtGuardarBiblioteca").Visibility  = (m == 4) ? Visibility.Collapsed : Visibility.Visible;
            Achar<Button>("BtLimparHist").Visibility = (m == 4) ? Visibility.Collapsed : Visibility.Visible;
            // A ficha do agente e o modo de conversa: as duas pecas do Grok Bot.
            Achar<Border>("Ficha").Visibility = (m == 8) ? Visibility.Visible : Visibility.Collapsed;
            Achar<Button>("BtMais").Visibility = (m == 2 || m == 5 || m == 6 || m == 7 || m == 8) ? Visibility.Visible : Visibility.Collapsed;
            ModoConversa(m == 8);

            TextBlock t = Achar<TextBlock>("Titulo"), a = Achar<TextBlock>("Ajuda");
            if (m == 0) { t.Text = "Construir um prompt";
                a.Text = "Escreve o que precisas por palavras tuas. Ele faz seis perguntas, uma de cada vez, e no fim entrega o prompt pronto."; }
            else if (m == 1) { t.Text = "Diagnosticar um prompt";
                a.Text = "Cola o prompt que queres melhorar. Recebes cinco pontos, uma versao corrigida em ingles e as palavras vagas assinaladas."; }
            else if (m == 2) { t.Text = "Imagem para markdown";
                a.Text = "Arrasta uma imagem para a caixa, escolhe um ficheiro, ou cola o caminho."; }
            else if (m == 3) { t.Text = "Reescrever texto";
                a.Text = "Cola texto teu e escolhe o que lhe fazer. Como o conteudo vem de ti, o modelo nao tem nada para inventar - e onde ele e mais fiavel."; }
            else if (m == 4) { t.Text = "Biblioteca de prompts";
                a.Text = "Os prompts que guardaste. Escolhe um na lista, preenche os {campos} na caixa de cima, e copia."; }
            else if (m == 5) { t.Text = "Conversar com um ficheiro";
                a.Text = "Escolhe um ficheiro e faz perguntas sobre ele. Responde so a partir do que la esta, cita a frase que sustenta, e diz-te quando a resposta nao esta no documento."; }
            else if (m == 6) { t.Text = "Procurar nos ficheiros";
                a.Text = "Procura dentro dos ficheiros, nao nos nomes. Escolhe a pasta, le uma vez, e a partir dai as procuras sao instantaneas. Nao usa o modelo, por isso nao inventa resultados."; }
            else if (m == 8) { t.Text = "A minha equipa";
                a.Text = "Escolhe um agente na lista e fala com ele. Cada um tem a sua missao, memoria e ferramentas. Os que usam o Claude podem pesquisar na net; os locais nao tem internet nenhuma."; }
            else { t.Text = "Encontrar duplicados";
                a.Text = "Compara o conteudo a serio, nao o nome. Mostra os grupos e propoe mover as copias para uma pasta _duplicados. Nunca apaga nada, e podes desfazer."; }
            PintarCabecalho();
            MostrarLista();
        }

        // Os botoes de modo sao pintados a mao, por isso tem de ser repintados
        // sempre que o tema muda. Sem isto o botao activo ficava castanho-escuro
        // em cima do tema claro.
        static void PintarModos()
        {
            string[] nomes = { "BtConstruir", "BtDiagnosticar", "BtImagem", "BtReescrever", "BtBiblioteca", "BtFicheiro", "BtProcurar", "BtDuplicados", "BtAgentes" };
            for (int i = 0; i < nomes.Length; i++)
            {
                Button b = Achar<Button>(nomes[i]);
                bool activo = (i == modo);
                b.Background = activo ? (Brush)J.Resources["RealceE"] : Brushes.Transparent;
                b.Foreground = activo ? (Brush)J.Resources["Realce"] : (Brush)J.Resources["Apagado"];
                b.FontWeight = activo ? FontWeights.SemiBold : FontWeights.Normal;
            }
        }

        // ---------------- conversar com um ficheiro ----------------
        // ---------------- procurar e duplicados ----------------
        // ---------------- atalho global Ctrl+Alt+P ----------------
        // Funciona em qualquer aplicacao, com a janela aberta ou minimizada.
        [DllImport("user32.dll")] static extern bool RegisterHotKey(IntPtr h, int id, uint mod, uint vk);
        [DllImport("user32.dll")] static extern bool UnregisterHotKey(IntPtr h, int id);
        [DllImport("user32.dll")] static extern void keybd_event(byte k, byte s, uint f, IntPtr e);
        [DllImport("user32.dll")] static extern IntPtr GetForegroundWindow();
        [DllImport("user32.dll")] static extern bool SetForegroundWindow(IntPtr h);
        [DllImport("user32.dll")] static extern bool GetCursorPos(out PONTO p);
        [StructLayout(LayoutKind.Sequential)] struct PONTO { public int X, Y; }

        const int ATALHO_ID = 9001;
        static IntPtr janelaAnterior = IntPtr.Zero;
        static Window menuAtalho = null;

        static void RegistarAtalho()
        {
            try
            {
                IntPtr h = new WindowInteropHelper(J).Handle;
                HwndSource.FromHwnd(h).AddHook(Gancho);
                // 0x0002 = Ctrl, 0x0001 = Alt, 0x50 = P
                if (!RegisterHotKey(h, ATALHO_ID, 0x0002 | 0x0001, 0x50))
                    Achar<TextBlock>("Estado").Text = "o Ctrl+Alt+P ja esta a ser usado por outro programa";
            }
            catch { }
        }

        static IntPtr Gancho(IntPtr h, int msg, IntPtr wp, IntPtr lp, ref bool tratado)
        {
            if (msg == 0x0312 && wp.ToInt32() == ATALHO_ID) { tratado = true; AoAtalho(); }
            return IntPtr.Zero;
        }

        // Quando o atalho dispara, o utilizador ainda tem o Ctrl e o Alt em baixo.
        // Se enviarmos Ctrl+C assim, o Windows le Ctrl+Alt+C e nao copia nada.
        // Soltamos as modificadoras primeiro. Foi isto que fez o atalho parecer
        // avariado: dizia sempre que nao havia texto seleccionado.
        static void SoltarModificadores()
        {
            keybd_event(0x12, 0, 2, IntPtr.Zero);   // Alt
            keybd_event(0xA4, 0, 2, IntPtr.Zero);   // Alt esquerdo
            keybd_event(0xA5, 0, 2, IntPtr.Zero);   // Alt direito
            keybd_event(0x11, 0, 2, IntPtr.Zero);   // Ctrl
            keybd_event(0xA2, 0, 2, IntPtr.Zero);   // Ctrl esquerdo
            keybd_event(0xA3, 0, 2, IntPtr.Zero);   // Ctrl direito
            keybd_event(0x10, 0, 2, IntPtr.Zero);   // Shift
            keybd_event(0x5B, 0, 2, IntPtr.Zero);   // Windows
            Thread.Sleep(80);
        }

        static void Teclas(byte tecla)
        {
            keybd_event(0x11, 0, 0, IntPtr.Zero);       // Ctrl em baixo
            keybd_event(tecla, 0, 0, IntPtr.Zero);
            keybd_event(tecla, 0, 2, IntPtr.Zero);
            keybd_event(0x11, 0, 2, IntPtr.Zero);       // Ctrl solto
        }

        static void AoAtalho()
        {
            if (menuAtalho != null) return;
            // Guardamos onde estavas, para poder colar de volta la.
            janelaAnterior = GetForegroundWindow();

            // Limpamos a area de transferencia ANTES de copiar. Sem isto, quando nao
            // ha nada seleccionado o Ctrl+C nao faz nada e ficavamos a trabalhar em
            // silencio sobre o que la estava de antes - ja aconteceu no primeiro teste.
            string antes = "";
            try { if (Clipboard.ContainsText()) antes = Clipboard.GetText(); } catch { }
            try { Clipboard.Clear(); } catch { }

            SoltarModificadores();
            Teclas(0x43);                                // Ctrl+C
            Thread.Sleep(320);
            string texto = "";
            try { if (Clipboard.ContainsText()) texto = Clipboard.GetText(); } catch { }
            if (texto.Trim().Length == 0)
            {
                // Devolvemos o que la estava: nao estragamos o que tinhas copiado.
                try { if (antes.Length > 0) Clipboard.SetText(antes); } catch { }
                MostrarAviso("Nao apanhei texto nenhum. Selecciona o texto primeiro e volta a carregar em Ctrl+Alt+P.");
                return;
            }
            MostrarMenu(texto);
        }

        static Window CaixaFlutuante(double largura)
        {
            Window w = new Window();
            w.WindowStyle = WindowStyle.None;
            w.AllowsTransparency = true;
            w.Background = System.Windows.Media.Brushes.Transparent;
            w.ShowInTaskbar = false;
            w.Topmost = true;
            w.SizeToContent = SizeToContent.Height;
            w.Width = largura;
            PONTO p;
            if (GetCursorPos(out p))
            {
                Matrix m = PresentationSource.FromVisual(J).CompositionTarget.TransformFromDevice;
                Point d = m.Transform(new Point(p.X, p.Y));
                w.Left = d.X + 12; w.Top = d.Y + 12;
            }
            else { w.WindowStartupLocation = WindowStartupLocation.CenterScreen; }
            return w;
        }

        static void MostrarAviso(string texto)
        {
            Window w = CaixaFlutuante(300);
            Border b = new Border();
            b.Background = (Brush)J.Resources["Painel"];
            b.BorderBrush = (Brush)J.Resources["Bordo"];
            b.BorderThickness = new Thickness(1);
            b.CornerRadius = new CornerRadius(10);
            b.Padding = new Thickness(14, 12, 14, 12);
            TextBlock t = new TextBlock();
            t.Text = texto; t.TextWrapping = TextWrapping.Wrap;
            t.Foreground = (Brush)J.Resources["Texto"]; t.FontSize = 12.5;
            b.Child = t; w.Content = b;
            w.Show();
            System.Windows.Threading.DispatcherTimer tt = new System.Windows.Threading.DispatcherTimer();
            tt.Interval = TimeSpan.FromSeconds(2.5);
            tt.Tick += delegate { tt.Stop(); try { w.Close(); } catch { } };
            tt.Start();
        }

        // Menu pequeno junto ao rato com as accoes do regras.md.
        static void MostrarMenu(string texto)
        {
            Window w = CaixaFlutuante(268);
            menuAtalho = w;
            Border b = new Border();
            b.Background = (Brush)J.Resources["Painel"];
            b.BorderBrush = (Brush)J.Resources["Bordo"];
            b.BorderThickness = new Thickness(1);
            b.CornerRadius = new CornerRadius(10);
            b.Padding = new Thickness(6);

            StackPanel sp = new StackPanel();
            TextBlock cab = new TextBlock();
            string amostra = texto.Replace(Environment.NewLine, " ").Trim();
            if (amostra.Length > 46) amostra = amostra.Substring(0, 46) + "...";
            cab.Text = amostra;
            cab.Foreground = (Brush)J.Resources["Apagado"];
            cab.FontSize = 11; cab.TextTrimming = TextTrimming.CharacterEllipsis;
            cab.Margin = new Thickness(10, 6, 10, 8);
            sp.Children.Add(cab);

            foreach (KeyValuePair<string, string> a in motor.R.Accoes)
            {
                Button bt = new Button();
                bt.Content = a.Key;
                bt.Style = (Style)J.FindResource("Pilula");
                bt.HorizontalContentAlignment = HorizontalAlignment.Left;
                bt.Padding = new Thickness(10, 7, 10, 7);
                string instrucao = a.Value;
                bt.Click += delegate { FecharMenu(); Aplicar(texto, instrucao); };
                sp.Children.Add(bt);
            }
            b.Child = sp; w.Content = b;
            // So fecha por perda de foco DEPOIS de o ter ganho. Sem isto, se a
            // aplicacao de onde vieste mantiver o foco, o menu fechava-se no
            // instante em que aparecia.
            bool jaActivou = false;
            w.Activated += delegate { jaActivou = true; };
            w.Deactivated += delegate { if (jaActivou) FecharMenu(); };
            w.KeyDown += delegate (object o, KeyEventArgs e) { if (e.Key == Key.Escape) FecharMenu(); };
            w.Show();
            w.Activate();
        }

        static void FecharMenu()
        {
            if (menuAtalho == null) return;
            try { menuAtalho.Close(); } catch { }
            menuAtalho = null;
        }

        static void Aplicar(string texto, string instrucao)
        {
            bool colar = Achar<CheckBox>("ColarSozinho").IsChecked == true;
            IntPtr voltar = janelaAnterior;
            MostrarAviso("a trabalhar...");
            new Thread(delegate () {
                string r;
                try { r = motor.Reescrever(texto, instrucao, null); }
                catch (Exception e) { r = null; }
                J.Dispatcher.Invoke((Action)delegate {
                    if (string.IsNullOrEmpty(r)) { MostrarAviso("nao consegui: o Ollama respondeu?"); return; }
                    try { Clipboard.SetText(r); } catch { }
                    if (colar && voltar != IntPtr.Zero)
                    {
                        SetForegroundWindow(voltar);
                        Thread.Sleep(150);
                        Teclas(0x56);                    // Ctrl+V
                        MostrarAviso("colado");
                    }
                    else MostrarAviso("pronto. Carrega em Ctrl+V para colar.");
                });
            }) { IsBackground = true }.Start();
        }

        static bool aMexerNaEquipa = false;
        static string corEscolhida = null;

        static int IndiceDe(Agente a)
        {
            if (a == null) return -1;
            for (int i = 0; i < agentes.Count; i++)
                if (string.Equals(agentes[i].Caminho, a.Caminho, StringComparison.OrdinalIgnoreCase)) return i;
            return -1;
        }

        // A lista lateral da equipa: bolinha da cor, nome e cargo. Como no Grok Bot,
        // cada agente parece um contacto e nao um botao de funcao.
        static void MostrarEquipa()
        {
            ListBox l = Achar<ListBox>("Equipa");
            if (l == null) return;
            int sel = IndiceDe(agenteActivo);
            aMexerNaEquipa = true;
            l.Items.Clear();
            foreach (Agente ag in agentes)
            {
                Grid g = new Grid();
                ColumnDefinition c1 = new ColumnDefinition(); c1.Width = GridLength.Auto;
                ColumnDefinition c2 = new ColumnDefinition(); c2.Width = new GridLength(1, GridUnitType.Star);
                g.ColumnDefinitions.Add(c1); g.ColumnDefinitions.Add(c2);

                System.Windows.Shapes.Ellipse p = new System.Windows.Shapes.Ellipse();
                p.Width = 8; p.Height = 8; p.Fill = Cor(ag.Cor);
                p.VerticalAlignment = VerticalAlignment.Center;
                p.Margin = new Thickness(1, 0, 9, 0);
                Grid.SetColumn(p, 0);

                StackPanel sp = new StackPanel();
                Grid.SetColumn(sp, 1);
                TextBlock n = new TextBlock();
                n.Text = ag.Nome; n.FontSize = 12.5;
                n.TextTrimming = TextTrimming.CharacterEllipsis;
                TextBlock c = new TextBlock();
                string cargo = ag.Cargo.Length > 0 ? ag.Cargo : "sem cargo";
                c.Text = cargo + (ag.Cerebro == "claude" ? "  -  Claude" : "");
                c.FontSize = 10.5;
                c.TextTrimming = TextTrimming.CharacterEllipsis;
                c.SetResourceReference(TextBlock.ForegroundProperty, "Apagado");
                sp.Children.Add(n); sp.Children.Add(c);

                g.Children.Add(p); g.Children.Add(sp);
                g.ToolTip = ag.Missao;
                l.Items.Add(g);
            }
            if (agentes.Count == 0) { l.Items.Add(Vazio("nenhum agente ainda")); l.SelectedIndex = -1; }

            if (sel >= 0 && sel < l.Items.Count) l.SelectedIndex = sel;
            aMexerNaEquipa = false;
        }

        // ---------------- a conversa em baloes ----------------
        // O truque para nao partir o texto em directo: a Saida continua a ser a
        // mesma caixa de texto de sempre, mas passa a ser o ultimo balao do fio.
        static void ModoConversa(bool sim)
        {
            Border mold = Achar<Border>("Moldura");
            Border cart = Achar<Border>("CartaoSaida");
            ScrollViewer rolo = Achar<ScrollViewer>("Rolo");
            TextBox s = Achar<TextBox>("Saida");
            if (mold == null || cart == null || rolo == null || s == null) return;
            if (sim)
            {
                mold.Background = Brushes.Transparent;
                mold.BorderThickness = new Thickness(0);
                rolo.Padding = new Thickness(2, 4, 2, 4);
                cart.SetResourceReference(Border.BackgroundProperty, "BolhaAgente");
                cart.Padding = new Thickness(12, 8, 12, 8);
                cart.HorizontalAlignment = HorizontalAlignment.Left;
                cart.MaxWidth = 640;
                cart.Margin = new Thickness(0, 0, 0, 12);
                s.FontFamily = new FontFamily("Segoe UI");
            }
            else
            {
                mold.SetResourceReference(Border.BackgroundProperty, "Cartao");
                mold.BorderThickness = new Thickness(1);
                rolo.Padding = new Thickness(16, 14, 16, 14);
                cart.Background = Brushes.Transparent;
                cart.Padding = new Thickness(0);
                cart.HorizontalAlignment = HorizontalAlignment.Stretch;
                cart.MaxWidth = double.PositiveInfinity;
                cart.Margin = new Thickness(0);
                cart.Visibility = Visibility.Visible;
                s.FontFamily = new FontFamily("Cascadia Mono, Consolas");
            }
        }

        // Um balao fechado. Nao e caixa de texto para poder encolher ate ao tamanho
        // do que la esta; para copiar ha o menu do botao direito.
        static void Mensagem(bool tu, string texto)
        {
            StackPanel fio = Achar<StackPanel>("Fio");
            Border cart = Achar<Border>("CartaoSaida");
            if (fio == null || cart == null) return;
            Border b = new Border();
            b.CornerRadius = new CornerRadius(18);
            b.Padding = new Thickness(12, 8, 12, 8);
            b.MaxWidth = 640;
            b.Margin = new Thickness(0, 0, 0, 12);
            b.HorizontalAlignment = tu ? HorizontalAlignment.Right : HorizontalAlignment.Left;
            b.SetResourceReference(Border.BackgroundProperty, tu ? "BolhaTu" : "BolhaAgente");
            b.Tag = tu ? "tu" : "ag";
            TextBlock t = new TextBlock();
            t.Text = texto;
            t.TextWrapping = TextWrapping.Wrap;
            t.FontSize = tamanhoLetra;
            t.LineHeight = tamanhoLetra + 6;
            t.SetResourceReference(TextBlock.ForegroundProperty, tu ? "TextoBolhaTu" : "Texto");
            b.Child = t;
            ContextMenu cm = new ContextMenu();
            MenuItem mi = new MenuItem();
            mi.Header = "Copiar esta mensagem";
            string copia = texto;
            mi.Click += delegate { try { Clipboard.SetText(copia); Achar<TextBlock>("Estado").Text = "mensagem copiada"; } catch { } };
            cm.Items.Add(mi);
            b.ContextMenu = cm;
            fio.Children.Insert(fio.Children.IndexOf(cart), b);
        }

        // Fim de turno: o que estava a ser escrito passa a balao fixo.
        static void FecharBolha()
        {
            TextBox s = Achar<TextBox>("Saida");
            Border cart = Achar<Border>("CartaoSaida");
            if (s == null || cart == null) return;
            string t = s.Text.Trim();
            if (t.Length > 0) Mensagem(false, t);
            s.Text = "";
            cart.Visibility = Visibility.Collapsed;
        }

        static void LimparFio()
        {
            StackPanel fio = Achar<StackPanel>("Fio");
            Border cart = Achar<Border>("CartaoSaida");
            if (fio == null || cart == null) return;
            for (int i = fio.Children.Count - 1; i >= 0; i--)
                if (fio.Children[i] != cart) fio.Children.RemoveAt(i);
            cart.Visibility = Visibility.Visible;
        }

        // A transcricao inteira. Fora do modo da equipa e exactamente a Saida,
        // para os outros modos se comportarem como sempre se comportaram.
        static string TextoTodo()
        {
            if (modo != 8) return Achar<TextBox>("Saida").Text;
            StackPanel fio = Achar<StackPanel>("Fio");
            Border cart = Achar<Border>("CartaoSaida");
            StringBuilder sb = new StringBuilder();
            foreach (UIElement u in fio.Children)
            {
                Border b = u as Border;
                if (b == null) continue;
                if (b == cart)
                {
                    string v = Achar<TextBox>("Saida").Text.Trim();
                    if (v.Length > 0) { sb.AppendLine(v); sb.AppendLine(); }
                    continue;
                }
                TextBlock t = b.Child as TextBlock;
                if (t == null) continue;
                sb.AppendLine(("tu".Equals(b.Tag) ? "tu: " : "") + t.Text);
                sb.AppendLine();
            }
            return sb.ToString().Trim();
        }

        // ---------------- a equipa de agentes ----------------
        static void EscolherAgente(int i)
        {
            if (i < 0 || i >= agentes.Count) return;
            agenteActivo = agentes[i];
            corEscolhida = agenteActivo.Cor;
            LimparFio();
            Achar<TextBox>("Saida").Text = "";
            Achar<Border>("CartaoSaida").Visibility = Visibility.Collapsed;
            foreach (string[] t in agenteActivo.Turnos(0))
            {
                Mensagem(true, t[0]);
                if (t[1].Length > 0) Mensagem(false, t[1]);
            }
            PintarCabecalho();
            AbrirFicha();
            Achar<TextBlock>("Estado").Text = "fala com o " + agenteActivo.Nome;
            Achar<TextBox>("Entrada").Focus();
            // A barra so sabe onde e o fim depois de o ecra estar desenhado.
            J.Dispatcher.BeginInvoke((Action)delegate {
                ScrollViewer r = Achar<ScrollViewer>("Rolo"); if (r != null) r.ScrollToEnd();
            }, System.Windows.Threading.DispatcherPriority.Loaded);
        }

        static void PintarCabecalho()
        {
            System.Windows.Shapes.Ellipse e = Achar<System.Windows.Shapes.Ellipse>("CabecaCor");
            Border cr = Achar<Border>("CrachaCerebro");
            if (e == null || cr == null) return;
            Agente a = agenteActivo;
            if (modo != 8 || a == null) { e.Visibility = Visibility.Collapsed; cr.Visibility = Visibility.Collapsed; return; }
            e.Fill = Cor(a.Cor); e.Visibility = Visibility.Visible;
            Achar<TextBlock>("Titulo").Text = a.Nome;
            Achar<TextBlock>("Ajuda").Text = a.Cargo.Length > 0 ? a.Cargo : a.Missao;
            bool net = a.Cerebro == "claude" && (a.Ferramentas.Contains("WebSearch") || a.Ferramentas.Contains("WebFetch"));
            Achar<TextBlock>("CrachaTexto").Text = a.Cerebro == "claude"
                ? (net ? "Claude - pesquisa na net" : "Claude - sem pesquisa")
                : "local - sem internet";
            cr.Visibility = Visibility.Visible;
        }

        // ---------------- a ficha do agente, a direita ----------------
        static void AbrirFicha()
        {
            Agente a = agenteActivo;
            Achar<TextBox>("FiNome").Text = a == null ? "" : a.Nome;
            Achar<TextBox>("FiCargo").Text = a == null ? "" : a.Cargo;
            Achar<TextBox>("FiMissao").Text = a == null ? "" : a.Missao;
            Achar<RadioButton>("FiLocal").IsChecked = (a != null && a.Cerebro != "claude");
            Achar<RadioButton>("FiClaude").IsChecked = (a != null && a.Cerebro == "claude");
            Achar<CheckBox>("FiWebSearch").IsChecked = (a != null && a.Ferramentas.Contains("WebSearch"));
            Achar<CheckBox>("FiWebFetch").IsChecked = (a != null && a.Ferramentas.Contains("WebFetch"));
            Achar<CheckBox>("FiFicheiro").IsChecked = (a != null && a.Ferramentas.Contains("ler-ficheiro"));
            Achar<TextBlock>("FiAviso").Text = (a != null && a.Cerebro != "claude")
                ? "O modelo local nao tem internet nenhuma: o WebSearch e o WebFetch so valem com o cerebro Claude."
                : "";
            Achar<TextBlock>("FiMemoria").Text = a == null ? ""
                : (a.Turnos(0).Count + " conversas guardadas neste agente.");
            PintarCores();
        }

        static void PintarCores()
        {
            StackPanel sp = Achar<StackPanel>("Cores");
            if (sp == null) return;
            sp.Children.Clear();
            foreach (string hex in Agente.CORES)
            {
                string h = hex;
                Border b = new Border();
                b.Width = 22; b.Height = 22;
                b.CornerRadius = new CornerRadius(11);
                b.Margin = new Thickness(0, 0, 7, 0);
                b.Background = Cor(h);
                b.Cursor = Cursors.Hand;
                b.ToolTip = h;
                b.BorderThickness = new Thickness(string.Equals(corEscolhida, h, StringComparison.OrdinalIgnoreCase) ? 2.5 : 0);
                b.SetResourceReference(Border.BorderBrushProperty, "Texto");
                b.MouseLeftButtonDown += delegate { corEscolhida = h; PintarCores(); };
                sp.Children.Add(b);
            }
        }

        static void GravarFicha()
        {
            if (agenteActivo == null) { Achar<TextBlock>("Estado").Text = "escolhe primeiro alguem da equipa"; return; }
            string nome = Achar<TextBox>("FiNome").Text.Trim();
            if (nome.Length == 0) { Achar<TextBlock>("Estado").Text = "o nome nao pode ficar vazio"; return; }
            string missao = Achar<TextBox>("FiMissao").Text.Trim();
            if (missao.Length == 0) { Achar<TextBlock>("Estado").Text = "escreve a missao: e o que diz ao agente o que fazer"; return; }
            Agente a = agenteActivo;
            a.Nome = nome;
            a.Cargo = Achar<TextBox>("FiCargo").Text.Trim();
            a.Missao = missao;
            a.Cerebro = (Achar<RadioButton>("FiClaude").IsChecked == true) ? "claude" : "local";
            a.Ferramentas.Clear();
            if (Achar<CheckBox>("FiWebSearch").IsChecked == true) a.Ferramentas.Add("WebSearch");
            if (Achar<CheckBox>("FiWebFetch").IsChecked == true) a.Ferramentas.Add("WebFetch");
            if (Achar<CheckBox>("FiFicheiro").IsChecked == true) a.Ferramentas.Add("ler-ficheiro");
            if (corEscolhida != null && corEscolhida.Length > 0) { a.Cor = corEscolhida; a.CorEscolhida = true; }
            string erro;
            if (!a.Gravar(out erro)) { Achar<TextBlock>("Estado").Text = "nao consegui gravar: " + erro; return; }
            MostrarEquipa();
            PintarCabecalho();
            AbrirFicha();
            Achar<TextBlock>("Estado").Text = "gravado em " + Path.GetFileName(a.Caminho);
        }

        static void NovoAgente()
        {
            string baseN = "Agente novo", nome = baseN; int n = 2;
            while (File.Exists(Path.Combine(pastaAgentes, nome + ".md"))) nome = baseN + " " + (n++);
            string erro;
            Agente novo = Agente.Criar(pastaAgentes, nome, out erro);
            if (novo == null) { Achar<TextBlock>("Estado").Text = "nao consegui criar: " + erro; return; }
            agentes = Agente.Todos(pastaAgentes);
            agenteActivo = novo;
            MostrarEquipa();
            MudarModo(8);
            Achar<TextBlock>("Estado").Text = "agente criado. Muda o nome e a missao na ficha a direita e carrega em Gravar.";
            TextBox fn = Achar<TextBox>("FiNome");
            if (fn != null) { fn.Focus(); fn.SelectAll(); }
        }

        static void ArquivarAgente()
        {
            if (agenteActivo == null) return;
            string nome = agenteActivo.Nome;
            if (MessageBox.Show("Arquivar o " + nome + "?" + Environment.NewLine + Environment.NewLine
                + "O ficheiro vai para a subpasta _arquivo dentro de agentes. Nao e apagado, e podes trazer de volta a mao.",
                "prompt-smith", MessageBoxButton.YesNo, MessageBoxImage.Question) != MessageBoxResult.Yes) return;
            string erro;
            if (!agenteActivo.Arquivar(out erro)) { Achar<TextBlock>("Estado").Text = "nao consegui arquivar: " + erro; return; }
            agenteActivo = null;
            agentes = Agente.Todos(pastaAgentes);
            MostrarEquipa();
            MudarModo(8);
            Achar<TextBlock>("Estado").Text = nome + " foi para a pasta _arquivo";
        }

        // O botao + do comprimido de escrever. Faz o que faz sentido em cada modo.
        static void Anexar()
        {
            if (modo == 2) { EscolherImagem(); return; }
            if (modo == 6 || modo == 7) { EscolherPasta(); return; }
            if (modo == 8)
            {
                Microsoft.Win32.OpenFileDialog d = new Microsoft.Win32.OpenFileDialog();
                d.Title = "Escolhe um ficheiro para dar ao agente";
                d.Filter = "Documentos|*.pdf;*.docx;*.txt;*.md;*.csv|Todos os ficheiros|*.*";
                if (d.ShowDialog() != true) return;
                TextBox e = Achar<TextBox>("Entrada");
                e.Text = "\"" + d.FileName + "\"" + Environment.NewLine + e.Text;
                e.Focus(); e.CaretIndex = e.Text.Length;
                Achar<TextBlock>("Estado").Text =
                    (agenteActivo != null && agenteActivo.Ferramentas.Contains("ler-ficheiro"))
                    ? "ficheiro juntou-se a mensagem"
                    : "atencao: este agente nao tem a ferramenta ler-ficheiro, por isso nao o vai abrir";
                return;
            }
            MudarModo(5); EscolherFicheiro();
        }

        static void EscolherPasta()
        {
            // Sem dependencias extra: usamos o dialogo de ficheiros a apontar para
            // a pasta, e ficamos com a pasta do ficheiro escolhido.
            Microsoft.Win32.OpenFileDialog d = new Microsoft.Win32.OpenFileDialog();
            d.Title = "Entra na pasta que queres e carrega em Abrir (o ficheiro escolhido nao importa)";
            d.CheckFileExists = false;
            d.FileName = "escolher esta pasta";
            if (d.ShowDialog() != true) return;
            string pastaEsc = Path.GetDirectoryName(d.FileName);
            if (!Directory.Exists(pastaEsc)) { Achar<TextBlock>("Estado").Text = "pasta invalida"; return; }
            pastaEscolhida = pastaEsc;
            Achar<TextBlock>("PastaNome").Text = pastaEscolhida;
            int n = 0;
            try { n = Directory.GetFiles(pastaEscolhida).Length; } catch { }
            bool indexada = indice.Pasta == pastaEscolhida && indice.Entradas.Count > 0;
            Achar<TextBlock>("PastaInfo").Text = n + " ficheiros. " +
                (indexada ? indice.Entradas.Count + " ja lidos." : "ainda por ler.");
            Achar<TextBlock>("Estado").Text = "pasta escolhida";
        }

        static void Indexar()
        {
            if (pastaEscolhida == null) { Achar<TextBlock>("Estado").Text = "escolhe primeiro a pasta"; return; }
            Button b = Achar<Button>("BtIndexar");
            b.IsEnabled = false; b.Content = "a ler...";
            string alvo = pastaEscolhida;
            new Thread(delegate () {
                DateTime t0 = DateTime.Now;
                int lidos = indice.Construir(alvo,
                    delegate (int feito, int total, string nome) {
                        if (feito % 5 != 0 && feito != total) return;
                        J.Dispatcher.Invoke((Action)delegate {
                            Achar<TextBlock>("Estado").Text = "a ler " + feito + " de " + total + ": " + nome;
                        });
                    },
                    delegate { return motor.IA.Cancelado; });
                int seg = (int)(DateTime.Now - t0).TotalSeconds;
                J.Dispatcher.Invoke((Action)delegate {
                    b.IsEnabled = true; b.Content = "Ler os ficheiros";
                    Achar<TextBlock>("PastaInfo").Text = indice.Entradas.Count + " ficheiros lidos e guardados.";
                    Achar<TextBlock>("Estado").Text = "pronto: " + lidos + " novos em " + seg + "s";
                });
            }) { IsBackground = true }.Start();
        }

        static void Procurar(string consulta)
        {
            if (indice.Entradas.Count == 0) { Achar<TextBlock>("Estado").Text = "carrega primeiro em Ler os ficheiros"; return; }
            List<Indice.Resultado> res = indice.Procurar(consulta, 15);
            StringBuilder sb = new StringBuilder();
            if (res.Count == 0) sb.AppendLine("Nao encontrei nada com essas palavras nos " + indice.Entradas.Count + " ficheiros lidos.");
            else
            {
                sb.AppendLine(res.Count + " ficheiros, do mais parecido para o menos:");
                sb.AppendLine();
                foreach (Indice.Resultado r in res)
                {
                    sb.AppendLine(r.Nome);
                    sb.AppendLine("   " + r.Excerto);
                    sb.AppendLine("   " + r.Caminho);
                    sb.AppendLine();
                }
                sb.AppendLine("Para conversares com um deles, copia o caminho e cola-o em Conversar com um ficheiro.");
            }
            Achar<TextBox>("Saida").Text = sb.ToString();
            Achar<TextBlock>("Estado").Text = res.Count + " resultados";
        }

        static void ProcurarDuplicados()
        {
            if (pastaEscolhida == null) { Achar<TextBlock>("Estado").Text = "escolhe primeiro a pasta"; return; }
            string alvo = pastaEscolhida;
            Achar<TextBox>("Saida").Text = "";
            Achar<TextBlock>("Estado").Text = "a comparar...";
            new Thread(delegate () {
                List<List<string>> g = Duplicados.Encontrar(alvo,
                    delegate (string nome) { J.Dispatcher.Invoke((Action)delegate { Achar<TextBlock>("Estado").Text = "a comparar " + nome; }); },
                    delegate { return motor.IA.Cancelado; });
                J.Dispatcher.Invoke((Action)delegate {
                    gruposDup = g;
                    StringBuilder sb = new StringBuilder();
                    int copias = 0;
                    foreach (List<string> grupo in g) copias += grupo.Count - 1;
                    if (g.Count == 0) sb.AppendLine("Nao encontrei ficheiros repetidos nesta pasta.");
                    else
                    {
                        long bytes = 0;
                        foreach (List<string> gr in g) { try { bytes += new FileInfo(gr[0]).Length * (gr.Count - 1); } catch { } }
                        sb.AppendLine(g.Count + " grupos de ficheiros iguais, " + copias + " copias a mais.");
                        sb.AppendLine("Libertam " + (bytes / 1024 / 1024) + " MB se as moveres.");
                        sb.AppendLine("O primeiro de cada grupo fica; os outros seriam movidos para _duplicados.");
                        sb.AppendLine();
                        foreach (List<string> grupo in g)
                        {
                    OrdenarGrupo(grupo);
                            sb.AppendLine("--- " + new FileInfo(grupo[0]).Length / 1024 + " KB");
                            for (int i = 0; i < grupo.Count; i++)
                                sb.AppendLine((i == 0 ? "  FICA  " : "  move  ") + Path.GetFileName(grupo[i]));
                            sb.AppendLine();
                        }
                        sb.AppendLine("Escreve MOVER na caixa de cima e carrega em Enviar para os mover.");
                        sb.AppendLine("Nada e apagado, e podes desfazer a seguir.");
                    }
                    Achar<TextBox>("Saida").Text = sb.ToString();
                    Achar<TextBlock>("Estado").Text = g.Count + " grupos";
                });
            }) { IsBackground = true }.Start();
        }

        // Qual fica: o de nome mais curto. "Relatorio.pdf" tem nome mais
        // curto do que "Relatorio (1).pdf", por isso e o original
        // que sobrevive. Em caso de empate, o mais antigo.
        static void OrdenarGrupo(List<string> grupo)
        {
            grupo.Sort(delegate (string a, string b) {
                int na = Path.GetFileName(a).Length, nb = Path.GetFileName(b).Length;
                if (na != nb) return na.CompareTo(nb);
                return File.GetLastWriteTimeUtc(a).CompareTo(File.GetLastWriteTimeUtc(b));
            });
        }

        // Move as copias. Nunca apaga. Escreve o que fez para se poder desfazer.
        static void MoverDuplicados()
        {
            if (gruposDup.Count == 0) { Achar<TextBlock>("Estado").Text = "procura primeiro os duplicados"; return; }
            string destino = Path.Combine(pastaEscolhida, "_duplicados");
            List<string[]> feitos = new List<string[]>();
            try
            {
                if (!Directory.Exists(destino)) Directory.CreateDirectory(destino);
                foreach (List<string> grupo in gruposDup)
                {
                    OrdenarGrupo(grupo);
                    for (int i = 1; i < grupo.Count; i++)
                    {
                        string novo = Path.Combine(destino, Path.GetFileName(grupo[i]));
                        int n = 1;
                        while (File.Exists(novo))
                            novo = Path.Combine(destino, Path.GetFileNameWithoutExtension(grupo[i]) + " (" + (++n) + ")" + Path.GetExtension(grupo[i]));
                        File.Move(grupo[i], novo);
                        feitos.Add(new string[] { novo, grupo[i] });
                    }
                }
            }
            catch (Exception e) { Achar<TextBlock>("Estado").Text = "parei a meio: " + e.Message; }

            try
            {
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 80000000;
                File.WriteAllText(Path.Combine(pasta, "desfazer.json"), js.Serialize(feitos), new UTF8Encoding(false));
            }
            catch { }
            gruposDup.Clear();
            Achar<TextBox>("Saida").Text = "Movi " + feitos.Count + " ficheiros para:" + Environment.NewLine + destino
                + Environment.NewLine + Environment.NewLine + "Nada foi apagado. O botao Desfazer repoe tudo.";
            Achar<TextBlock>("Estado").Text = feitos.Count + " movidos";
        }

        static void Desfazer()
        {
            string f = Path.Combine(pasta, "desfazer.json");
            if (!File.Exists(f)) { Achar<TextBlock>("Estado").Text = "nao ha nada para desfazer"; return; }
            int repostos = 0;
            try
            {
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 80000000;
                object[] lista = (object[])js.DeserializeObject(File.ReadAllText(f, Encoding.UTF8));
                foreach (object o in lista)
                {
                    object[] par = (object[])o;
                    string agora = Convert.ToString(par[0]), antes = Convert.ToString(par[1]);
                    if (File.Exists(agora) && !File.Exists(antes)) { File.Move(agora, antes); repostos++; }
                }
                File.Delete(f);
            }
            catch (Exception e) { Achar<TextBlock>("Estado").Text = "erro a desfazer: " + e.Message; return; }
            Achar<TextBox>("Saida").Text = "Repus " + repostos + " ficheiros nos sitios originais.";
            Achar<TextBlock>("Estado").Text = repostos + " repostos";
        }

        static void EscolherFicheiro()
        {
            Microsoft.Win32.OpenFileDialog d = new Microsoft.Win32.OpenFileDialog();
            d.Filter = "Documentos|*.pdf;*.docx;*.txt;*.md;*.csv|Todos os ficheiros|*.*";
            if (d.ShowDialog() == true) UsarFicheiro(d.FileName);
        }

        static void UsarFicheiro(string caminho)
        {
            Achar<TextBlock>("FichNome").Text = Path.GetFileName(caminho);
            Achar<TextBlock>("FichInfo").Text = "a ler...";
            Achar<TextBox>("Saida").Text = "";
            historico.Clear();

            new Thread(delegate () {
                Documento d = Documento.Abrir(caminho);
                J.Dispatcher.Invoke((Action)delegate {
                    docAberto = d;
                    if (d.Erro != null)
                    {
                        Achar<TextBlock>("FichInfo").Text = d.Erro;
                        Achar<TextBlock>("Estado").Text = "nao consegui abrir";
                        return;
                    }
                    bool inteiro = d.Palavras <= Motor.PALAVRAS_MAX;
                    Achar<TextBlock>("FichInfo").Text = d.Palavras + " palavras. " +
                        (inteiro ? "Cabe inteiro no contexto." : "E grande: vou usar as partes mais relevantes de cada vez.");
                    Achar<TextBlock>("Estado").Text = "pronto. Faz a tua pergunta.";
                    Achar<TextBox>("Entrada").Focus();
                });
            }) { IsBackground = true }.Start();
        }

        // Guarda a troca, para as perguntas de seguimento fazerem sentido.
        static void GuardarTroca(string pergunta, string resposta)
        {
            Dictionary<string, string> q = new Dictionary<string, string>();
            q["role"] = "user"; q["content"] = pergunta;
            historico.Add(q);
            Dictionary<string, string> a = new Dictionary<string, string>();
            a["role"] = "assistant"; a["content"] = resposta;
            historico.Add(a);
        }

        static void EscolherImagem()
        {
            Microsoft.Win32.OpenFileDialog d = new Microsoft.Win32.OpenFileDialog();
            d.Filter = "Imagens|*.png;*.jpg;*.jpeg;*.bmp;*.gif;*.tif;*.tiff|Todos os ficheiros|*.*";
            if (d.ShowDialog() == true) UsarImagem(d.FileName);
        }
        static void UsarImagem(string caminho)
        {
            imagemEscolhida = caminho;
            Achar<TextBox>("Entrada").Text = caminho;
            Achar<TextBlock>("Rodape").Text = "imagem pronta; carrega em Enviar";
        }

        // ---------------- enviar ----------------
        static void Enviar()
        {
            string texto = Achar<TextBox>("Entrada").Text.Trim();
            string img = imagemEscolhida;
            if (modo == 2)
            {
                if (texto.Length > 0 && File.Exists(texto)) img = texto;
                if (img == null || !File.Exists(img)) { Achar<TextBlock>("Estado").Text = "escolhe uma imagem, arrasta-a, ou cola o caminho"; return; }
            }
            if (modo == 4) { Achar<TextBlock>("Estado").Text = "escolhe um prompt na lista"; return; }
            if (modo == 8 && agenteActivo == null) { Achar<TextBlock>("Estado").Text = "escolhe primeiro um agente na lista"; return; }
            // Colar o caminho de uma pasta escolhe-a, nos dois modos.
            if ((modo == 6 || modo == 7) && texto.Length > 0 && Directory.Exists(texto))
            {
                pastaEscolhida = texto;
                Achar<TextBlock>("PastaNome").Text = texto;
                int nf = 0; try { nf = Directory.GetFiles(texto).Length; } catch { }
                bool ja = indice.Pasta == texto && indice.Entradas.Count > 0;
                Achar<TextBlock>("PastaInfo").Text = nf + " ficheiros. " + (ja ? indice.Entradas.Count + " ja lidos." : "ainda por ler.");
                Achar<TextBlock>("Estado").Text = "pasta escolhida";
                Achar<TextBox>("Entrada").Text = "";
                return;
            }
            // Estes dois nao falam com o modelo: sao trabalho do programa.
            if (modo == 6) { if (texto.Length == 0) { Achar<TextBlock>("Estado").Text = "escreve o que procuras"; return; } Procurar(texto); return; }
            if (modo == 7) {
                if (texto.Trim().ToUpper() == "MOVER") { MoverDuplicados(); Achar<TextBox>("Entrada").Text = ""; return; }
                ProcurarDuplicados(); return;
            }
            if (modo == 5)
            {
                // Colar o caminho tambem serve, como no modo imagem.
                if (texto.Length > 0 && File.Exists(texto)) { UsarFicheiro(texto); return; }
                if (docAberto == null || docAberto.Erro != null)
                { Achar<TextBlock>("Estado").Text = "escolhe um ficheiro, arrasta-o, ou cola o caminho"; return; }
            }
            if (modo != 2 && texto.Length == 0) { Achar<TextBlock>("Estado").Text = "escreve alguma coisa primeiro"; return; }

            string accao = null;
            if (modo == 3)
            {
                int i = Achar<ComboBox>("Accao").SelectedIndex;
                if (i < 0 || i >= motor.R.Accoes.Count) { Achar<TextBlock>("Estado").Text = "escolhe o que fazer ao texto"; return; }
                accao = motor.R.Accoes[i].Value;
            }

            int m = modo;
            bool critica = Achar<CheckBox>("AutoCritica").IsChecked == true;
            bool visao = Achar<CheckBox>("ForcarVisao").IsChecked == true;
            string destino = null;
            ComboBoxItem sel = Achar<ComboBox>("Destino").SelectedItem as ComboBoxItem;
            if (sel != null && (string)sel.Content != "nenhum") destino = (string)sel.Content;

            Button enviar = Achar<Button>("BtEnviar");
            aGerar = true; motor.IA.Cancelado = false;
            enviar.Content = "Parar";
            if (m == 8) {
                // O que escreveste vira balao teu, e abre-se um balao vazio para a resposta.
                Mensagem(true, texto);
                Achar<TextBox>("Saida").Text = "";
                Achar<Border>("CartaoSaida").Visibility = Visibility.Visible;
                Achar<TextBox>("Entrada").Text = "";
                Achar<ScrollViewer>("Rolo").ScrollToEnd();
            }
            else if (m == 0 || m == 5) { Juntar("tu: " + texto + Environment.NewLine + Environment.NewLine); Achar<TextBox>("Entrada").Text = ""; }
            else Achar<TextBox>("Saida").Text = "";

            Action<string> estado = delegate (string s) { J.Dispatcher.Invoke((Action)delegate { Achar<TextBlock>("Estado").Text = s; }); };
            // Cada pedaco que o modelo escreve aparece logo, em vez de esperar pelo fim.
            Action<string> pedaco = delegate (string s) { J.Dispatcher.Invoke((Action)delegate { Juntar(s); }); };

            new Thread(delegate () {
                string r; DateTime inicio = DateTime.Now;
                try
                {
                    if (m == 0) { estado("a pensar na proxima pergunta..."); r = motor.Entrevistar(historico, texto, pedaco); }
                    else if (m == 1) r = motor.Diagnosticar(texto, destino, critica, estado, pedaco);
                    else if (m == 3) { estado("a reescrever..."); r = motor.Reescrever(texto, accao, pedaco); }
                    else if (m == 5) { estado("a ler o documento..."); r = motor.PerguntarAoDocumento(docAberto, historico, texto, estado, pedaco); }
                    else if (m == 8) { r = motor.CorrerAgente(agenteActivo, texto, estado, agenteActivo.Cerebro == "claude" ? null : pedaco); }
                    else r = motor.ImagemParaMarkdown(img, visao, estado, pedaco);
                }
                catch (Exception ex)
                {
                    r = "Nao consegui falar com o Ollama." + Environment.NewLine
                      + "Confirma que esta a correr e tenta outra vez." + Environment.NewLine
                      + Environment.NewLine + "Detalhe: " + ex.Message;
                }
                int seg = (int)(DateTime.Now - inicio).TotalSeconds;
                string entradaGuardar = (m == 2) ? Path.GetFileName(img) : texto;
                bool interrompido = motor.IA.Cancelado;
                // Rede de seguranca: um erro a actualizar o ecra nao deve fechar a
                // aplicacao. Foi um indice fora dos limites aqui que a matou uma vez.
                J.Dispatcher.Invoke((Action)delegate {
                  try {
                    if (interrompido) Juntar(Environment.NewLine + "(interrompido)" + Environment.NewLine);
                    // O texto final pode diferir do que foi aparecendo: os passos
                    // seguintes reescrevem o ponto 4. Por isso substituimos no fim.
                    else if (m == 0) Juntar(Environment.NewLine + Environment.NewLine);
                    else if (m == 8) {
                        // Com o Claude nao ha texto em directo: aparece tudo no fim.
                        if (agenteActivo.Cerebro == "claude") Juntar(r);
                        agenteActivo.GuardarNaMemoria(texto, r);
                        FecharBolha();
                        // a contagem da ficha nao pode ficar a mentir depois de uma conversa nova
                        Achar<TextBlock>("FiMemoria").Text = agenteActivo.Turnos(0).Count + " conversas guardadas neste agente.";
                    }
                    else if (m == 5) {
                        // O texto ja apareceu enquanto era escrito; so falta o aviso.
                        string aviso = Motor.AvisoSemCitacao(r);
                        if (aviso != null) Juntar(Environment.NewLine + aviso);
                        Juntar(Environment.NewLine + Environment.NewLine);
                        GuardarTroca(texto, r);
                    }
                    else Achar<TextBox>("Saida").Text = r;

                    aGerar = false;
                    enviar.Content = "Enviar    Ctrl+Enter";
                    Achar<TextBlock>("Estado").Text = interrompido ? "interrompido" : "pronto";
                    // Na entrevista mostramos em que pergunta vais, das seis.
                    if (m == 0 && !interrompido)
                    {
                        int n = Math.Min(6, historico.Count / 2);
                        Achar<TextBlock>("Rodape").Text = "pergunta " + n + " de 6   -   " + seg + " segundos";
                    }
                    else Achar<TextBlock>("Rodape").Text = seg + " segundos";
                    Achar<TextBox>("Entrada").Focus();
                    if (!interrompido) GuardarRegisto(m, entradaGuardar, TextoTodo());
                  }
                  catch (Exception ex2) {
                    Achar<TextBlock>("Estado").Text = "erro interno: " + ex2.Message;
                    aGerar = false;
                    Achar<Button>("BtEnviar").Content = "Enviar    Ctrl+Enter";
                  }
                });
            }) { IsBackground = true }.Start();
        }

        static void Juntar(string t)
        {
            TextBox s = Achar<TextBox>("Saida");
            s.AppendText(t);
            Achar<ScrollViewer>("Rolo").ScrollToEnd();
        }

        // Dois botoes de proposito: o que se quer 90% das vezes e so o prompt.
        static void Copiar(bool soPrompt)
        {
            string t = TextoTodo();
            if (t.Trim().Length == 0) { Achar<TextBlock>("Estado").Text = "nao ha nada para copiar"; return; }
            string conteudo = (soPrompt && modo != 2) ? Motor.PromptFinal(t) : t;
            Clipboard.SetText(conteudo);
            Achar<TextBlock>("Estado").Text = soPrompt ? "prompt copiado" : "copiado tudo";
        }

        static void AbrirRegras()
        {
            string f = Path.Combine(pasta, "regras.md");
            if (!File.Exists(f)) f = Path.GetFullPath(Path.Combine(pasta, @"..\regras\regras.md"));
            AbrirNoBloco(f);
        }

        static void AbrirNoBloco(string f)
        {
            if (!File.Exists(f)) { Achar<TextBlock>("Estado").Text = "o ficheiro ainda nao existe: " + Path.GetFileName(f); return; }
            try { System.Diagnostics.Process.Start("notepad.exe", f); }
            catch (Exception e) { Achar<TextBlock>("Estado").Text = "nao consegui abrir: " + e.Message; }
        }

        // ---------------- biblioteca de prompts ----------------
        static void AbrirDaBiblioteca(int i)
        {
            KeyValuePair<string, string> it = biblio.Itens[i];
            Achar<TextBox>("Entrada").Text = it.Value;
            Achar<TextBox>("Saida").Text = it.Value;
            Achar<TextBlock>("Rodape").Text = it.Key;
            int campos = System.Text.RegularExpressions.Regex.Matches(it.Value, @"\{[^}]+\}").Count;
            Achar<TextBlock>("Estado").Text = campos > 0
                ? campos + " campos para preencher na caixa de cima"
                : "pronto a copiar";
        }

        // Guarda o prompt final. O nome sai das primeiras palavras do que pediste,
        // para nao teres de responder a uma caixa de dialogo de cada vez.
        static void GuardarNaBiblioteca()
        {
            string saida = Achar<TextBox>("Saida").Text;
            if (saida.Trim().Length == 0) { Achar<TextBlock>("Estado").Text = "nao ha nada para guardar"; return; }
            string corpo = (modo == 2 || modo == 3) ? saida.Trim() : Motor.PromptFinal(saida);
            if (corpo.Length < 10) { Achar<TextBlock>("Estado").Text = "nao encontrei um prompt para guardar"; return; }

            string origem = Achar<TextBox>("Entrada").Text.Trim();
            if (origem.Length == 0) origem = corpo;
            string[] palavras = origem.Replace(Environment.NewLine, " ").Split(new char[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
            string nome = string.Join(" ", palavras, 0, Math.Min(7, palavras.Length));
            if (nome.Length > 60) nome = nome.Substring(0, 60);

            biblio.Acrescentar(nome, corpo);
            Achar<TextBlock>("Estado").Text = "guardado na biblioteca como: " + nome;
        }

        // Aborta o pedido em curso. O texto ja recebido fica no ecra.
        static void Parar()
        {
            motor.IA.Cancelar();
            Achar<TextBlock>("Estado").Text = "interrompido";
        }

        static void Guardar()
        {
            string t = TextoTodo();
            if (t.Trim().Length == 0) { Achar<TextBlock>("Estado").Text = "nao ha nada para guardar"; return; }
            string conteudo = (modo == 2) ? t : Motor.PromptFinal(t);
            Microsoft.Win32.SaveFileDialog d = new Microsoft.Win32.SaveFileDialog();
            d.Filter = "Markdown|*.md";
            d.FileName = (modo == 2) ? "notas.md" : "prompt.md";
            if (d.ShowDialog() != true) return;
            File.WriteAllText(d.FileName, conteudo.Trim() + Environment.NewLine, new UTF8Encoding(false));
            Achar<TextBlock>("Estado").Text = "guardado em " + Path.GetFileName(d.FileName);
        }

        // ---------------- historico ----------------
        static string FicheiroRegistos { get { return Path.Combine(pasta, "historico.json"); } }
        static string FicheiroDefinicoes { get { return Path.Combine(pasta, "definicoes.txt"); } }

        static List<Dictionary<string, string>> LerDoDisco()
        {
            List<Dictionary<string, string>> saida = new List<Dictionary<string, string>>();
            try
            {
                if (!File.Exists(FicheiroRegistos)) return saida;
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 80000000;
                object[] lista = (object[])js.DeserializeObject(File.ReadAllText(FicheiroRegistos, Encoding.UTF8));
                foreach (object o in lista)
                {
                    Dictionary<string, object> d = (Dictionary<string, object>)o;
                    Dictionary<string, string> r = new Dictionary<string, string>();
                    foreach (string k in d.Keys) r[k] = Convert.ToString(d[k]);
                    saida.Add(r);
                }
            }
            catch { saida.Clear(); }
            return saida;
        }

        static void LerRegistos() { registos = LerDoDisco(); }

        static void GravarRegistos()
        {
            try
            {
                JavaScriptSerializer js = new JavaScriptSerializer(); js.MaxJsonLength = 80000000;
                File.WriteAllText(FicheiroRegistos, js.Serialize(registos), new UTF8Encoding(false));
            }
            catch { }
        }

        static void GuardarRegisto(int m, string entrada, string saida)
        {
            if (saida.Trim().Length == 0) return;
            string[] nomes = { "construir", "diagnosticar", "imagem", "reescrever", "biblioteca", "ficheiro", "procurar", "duplicados", "agente" };
            Dictionary<string, string> r = new Dictionary<string, string>();
            r["modo"] = nomes[m];
            r["entrada"] = entrada;
            r["saida"] = saida;
            r["quando"] = DateTime.Now.ToString("yyyy-MM-dd HH:mm");
            // Relemos o ficheiro mesmo antes de acrescentar. Assim nada se perde
            // se houver duas copias da aplicacao abertas, ou se o ficheiro tiver
            // sido mexido por fora. Foi um registo perdido nos testes que ensinou isto.
            List<Dictionary<string, string>> disco = LerDoDisco();
            registos = disco;
            registos.Insert(0, r);
            while (registos.Count > 50) registos.RemoveAt(registos.Count - 1);   // guardamos os ultimos 50
            GravarRegistos();
            MostrarLista();
        }

        static void MostrarLista()
        {
            ListBox l = Achar<ListBox>("Historico");
            l.Items.Clear();
            if (modo == 4)
            {
                foreach (KeyValuePair<string, string> it in biblio.Itens)
                {
                    TextBlock b = new TextBlock();
                    b.Text = it.Key;
                    b.TextTrimming = TextTrimming.CharacterEllipsis;
                    b.ToolTip = it.Value.Length > 300 ? it.Value.Substring(0, 300) + "..." : it.Value;
                    l.Items.Add(b);
                }
                if (biblio.Itens.Count == 0) { l.Items.Add(Vazio("ainda nao guardaste nenhum")); l.SelectedIndex = -1; }

                return;
            }
            foreach (Dictionary<string, string> r in registos)
            {
                string e = r.ContainsKey("entrada") ? r["entrada"] : "";
                e = e.Replace(Environment.NewLine, " ").Replace("\n", " ").Trim();
                if (e.Length > 42) e = e.Substring(0, 42) + "...";
                TextBlock t = new TextBlock();
                t.Text = e;
                t.TextTrimming = TextTrimming.CharacterEllipsis;
                t.ToolTip = (r.ContainsKey("quando") ? r["quando"] + "  -  " : "") + (r.ContainsKey("modo") ? r["modo"] : "");
                l.Items.Add(t);
            }
            if (registos.Count == 0) { l.Items.Add(Vazio("ainda vazio")); l.SelectedIndex = -1; }

        }

        static TextBlock Vazio(string texto)
        {
            TextBlock v = new TextBlock();
            v.Text = texto;
            v.Foreground = (Brush)J.Resources["Apagado"];
            v.FontSize = 11;
            v.Margin = new Thickness(9, 4, 0, 0);
            return v;
        }

        static void AbrirRegisto(int i)
        {
            Dictionary<string, string> r = registos[i];
            string[] nomes = { "construir", "diagnosticar", "imagem", "reescrever", "biblioteca", "ficheiro", "procurar", "duplicados", "agente" };
            int m = Array.IndexOf(nomes, r.ContainsKey("modo") ? r["modo"] : "diagnosticar");
            if (m < 0) m = 1;
            historico.Clear();
            AplicarModoUI(m);
            Achar<TextBox>("Entrada").Text = r.ContainsKey("entrada") ? r["entrada"] : "";
            Achar<TextBox>("Saida").Text = r.ContainsKey("saida") ? r["saida"] : "";
            Achar<TextBlock>("Rodape").Text = "guardado em " + (r.ContainsKey("quando") ? r["quando"] : "");
            Achar<TextBlock>("Estado").Text = "a ver um registo antigo";
        }

        static void LimparRegistos()
        {
            if (registos.Count == 0) return;
            if (MessageBox.Show("Apagar os " + registos.Count + " registos guardados?", "prompt-smith",
                MessageBoxButton.YesNo, MessageBoxImage.Question) != MessageBoxResult.Yes) return;
            registos.Clear(); GravarRegistos(); MostrarLista();
            Achar<TextBlock>("Estado").Text = "historico apagado";
        }

        // ---------------- preferencias ----------------
        static void LerDefinicoes()
        {
            try
            {
                if (!File.Exists(FicheiroDefinicoes)) return;
                foreach (string l in File.ReadAllLines(FicheiroDefinicoes))
                {
                    int i = l.IndexOf('=');
                    if (i < 0) continue;
                    string k = l.Substring(0, i).Trim(), v = l.Substring(i + 1).Trim();
                    if (k == "tema") temaClaro = (v == "claro");
                    else if (k == "colar") colarSozinho = (v == "sim");
                    else if (k == "letra") { double d; if (double.TryParse(v, System.Globalization.NumberStyles.Any, System.Globalization.CultureInfo.InvariantCulture, out d)) tamanhoLetra = d; }
                }
            }
            catch { }
        }

        static void GravarDefinicoes()
        {
            try
            {
                File.WriteAllText(FicheiroDefinicoes,
                    "tema=" + (temaClaro ? "claro" : "escuro") + Environment.NewLine +
                    "colar=" + (Achar<CheckBox>("ColarSozinho").IsChecked == true ? "sim" : "nao") + Environment.NewLine +
                    "letra=" + tamanhoLetra.ToString(System.Globalization.CultureInfo.InvariantCulture) + Environment.NewLine,
                    new UTF8Encoding(false));
            }
            catch { }
        }
    }
}



























~~~~

