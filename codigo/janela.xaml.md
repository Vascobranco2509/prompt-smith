# app\janela.xaml

O ficheiro inteiro esta no bloco abaixo. Grava-o em `app\janela.xaml`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `app\janela.xaml`

<!-- destino: app\janela.xaml | codificacao: bom -->
~~~~xml
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="forja" Height="780" Width="1240" MinHeight="620" MinWidth="1000"
        WindowStartupLocation="CenterScreen" WindowStyle="None" AllowsTransparency="True"
        Background="Transparent" FontFamily="Segoe UI" UseLayoutRounding="True" TextOptions.TextFormattingMode="Ideal">
  <Window.Resources>
    <!-- As cores sao DynamicResource para o tema poder ser trocado em andamento.
         Os valores iniciais estao aqui; o tema claro e aplicado por codigo.
         O tema claro copia a paleta do Grok Bot; o escuro e o desta casa. -->
    <SolidColorBrush x:Key="Fundo"    Color="#0F1115"/>
    <SolidColorBrush x:Key="Painel"   Color="#161922"/>
    <SolidColorBrush x:Key="Cartao"   Color="#1B1F2A"/>
    <SolidColorBrush x:Key="Bordo"    Color="#252A36"/>
    <SolidColorBrush x:Key="Texto"    Color="#E7E9EF"/>
    <SolidColorBrush x:Key="Apagado"  Color="#868D9E"/>
    <SolidColorBrush x:Key="Realce"   Color="#E08A45"/>
    <SolidColorBrush x:Key="RealceE"  Color="#3A2A1C"/>
    <SolidColorBrush x:Key="Passar"   Color="#20252F"/>
    <SolidColorBrush x:Key="RealceC"  Color="#EC9A57"/>
    <SolidColorBrush x:Key="Inactivo" Color="#2A2F3B"/>
    <SolidColorBrush x:Key="InactivoT" Color="#6C7381"/>
    <SolidColorBrush x:Key="SobreRealce" Color="#1A1206"/>
    <!-- Os baloes da conversa. No Grok Bot: agente cinzento a esquerda,
         utilizador quase preto a direita. No tema escuro, ao contrario. -->
    <SolidColorBrush x:Key="BolhaAgente" Color="#232936"/>
    <SolidColorBrush x:Key="BolhaTu"     Color="#E7E9EF"/>
    <SolidColorBrush x:Key="TextoBolhaTu" Color="#0F1115"/>

    <Style x:Key="Pilula" TargetType="Button">
      <Setter Property="Foreground" Value="{DynamicResource Apagado}"/>
      <Setter Property="Background" Value="Transparent"/>
      <Setter Property="FontSize" Value="13.5"/>
      <Setter Property="Padding" Value="12,8"/>
      <Setter Property="Cursor" Value="Hand"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="Button">
            <Border x:Name="b" Background="{TemplateBinding Background}" CornerRadius="9" Padding="{TemplateBinding Padding}">
              <ContentPresenter HorizontalAlignment="Left" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsMouseOver" Value="True">
                <Setter TargetName="b" Property="Background" Value="{DynamicResource Passar}"/>
              </Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <Style x:Key="Accao" TargetType="Button">
      <Setter Property="Foreground" Value="{DynamicResource SobreRealce}"/>
      <Setter Property="Background" Value="{DynamicResource Realce}"/>
      <Setter Property="FontSize" Value="13"/>
      <Setter Property="FontWeight" Value="SemiBold"/>
      <Setter Property="Cursor" Value="Hand"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="Button">
            <Border x:Name="b" Background="{TemplateBinding Background}" CornerRadius="16" Padding="16,8">
              <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
            </Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsMouseOver" Value="True"><Setter TargetName="b" Property="Background" Value="{DynamicResource RealceC}"/></Trigger>
              <Trigger Property="IsEnabled" Value="False">
                <Setter TargetName="b" Property="Background" Value="{DynamicResource Inactivo}"/>
                <Setter Property="Foreground" Value="{DynamicResource InactivoT}"/>
              </Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <Style x:Key="Discreto" TargetType="Button" BasedOn="{StaticResource Accao}">
      <Setter Property="Background" Value="{DynamicResource Cartao}"/>
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
      <Setter Property="FontWeight" Value="Normal"/>
    </Style>

    <!-- Titulos de seccao da barra lateral -->
    <Style x:Key="Seccao" TargetType="TextBlock">
      <Setter Property="Foreground" Value="{DynamicResource Apagado}"/>
      <Setter Property="FontSize" Value="10.5"/>
      <Setter Property="FontWeight" Value="SemiBold"/>
      <Setter Property="Margin" Value="6,0,0,7"/>
    </Style>

    <!-- Etiquetas da ficha do agente -->
    <Style x:Key="Rotulo" TargetType="TextBlock">
      <Setter Property="Foreground" Value="{DynamicResource Apagado}"/>
      <Setter Property="FontSize" Value="11"/>
      <Setter Property="Margin" Value="2,8,0,3"/>
    </Style>

    <Style x:Key="Janelinha" TargetType="Button">
      <Setter Property="Foreground" Value="{DynamicResource Apagado}"/>
      <Setter Property="FontFamily" Value="Segoe MDL2 Assets"/>
      <Setter Property="FontSize" Value="10"/>
      <Setter Property="Width" Value="42"/>
      <Setter Property="Cursor" Value="Hand"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="Button">
            <Border x:Name="b" Background="Transparent"><ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/></Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsMouseOver" Value="True"><Setter TargetName="b" Property="Background" Value="{DynamicResource Passar}"/></Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <Style TargetType="TextBox">
      <Setter Property="Background" Value="{DynamicResource Cartao}"/>
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
      <Setter Property="CaretBrush" Value="{DynamicResource Texto}"/>
      <Setter Property="BorderThickness" Value="1"/>
      <Setter Property="BorderBrush" Value="{DynamicResource Bordo}"/>
      <Setter Property="Padding" Value="12,10"/>
      <Setter Property="FontSize" Value="14"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="TextBox">
            <Border Background="{TemplateBinding Background}" BorderBrush="{TemplateBinding BorderBrush}"
                    BorderThickness="{TemplateBinding BorderThickness}" CornerRadius="10">
              <ScrollViewer x:Name="PART_ContentHost" Margin="{TemplateBinding Padding}"/>
            </Border>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <Style TargetType="CheckBox">
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
    </Style>
    <Style TargetType="RadioButton">
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
      <Setter Property="FontSize" Value="12.5"/>
      <Setter Property="Margin" Value="2,2,0,2"/>
    </Style>

    <Style TargetType="ComboBoxItem">
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
      <Setter Property="Padding" Value="10,7"/>
      <Setter Property="FontSize" Value="13"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="ComboBoxItem">
            <Border x:Name="b" Background="Transparent" Padding="{TemplateBinding Padding}" CornerRadius="6">
              <ContentPresenter/>
            </Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsHighlighted" Value="True"><Setter TargetName="b" Property="Background" Value="{DynamicResource Passar}"/></Trigger>
              <Trigger Property="IsSelected" Value="True"><Setter Property="Foreground" Value="{DynamicResource Realce}"/></Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <!-- Sem modelo proprio o Windows desenha as caixas de seleccao sempre brancas. -->
    <Style TargetType="ComboBox">
      <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
      <Setter Property="FontSize" Value="13"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="ComboBox">
            <Grid>
              <ToggleButton Focusable="False" ClickMode="Press"
                            IsChecked="{Binding IsDropDownOpen, Mode=TwoWay, RelativeSource={RelativeSource TemplatedParent}}">
                <ToggleButton.Template>
                  <ControlTemplate TargetType="ToggleButton">
                    <Border Background="{DynamicResource Cartao}" BorderBrush="{DynamicResource Bordo}"
                            BorderThickness="1" CornerRadius="9" Padding="12,8">
                      <Grid>
                        <ContentPresenter HorizontalAlignment="Left" VerticalAlignment="Center"/>
                        <Path Data="M0,0 L4,4 L8,0" Stroke="{DynamicResource Apagado}" StrokeThickness="1.4"
                              HorizontalAlignment="Right" VerticalAlignment="Center" Margin="0,2,2,0"/>
                      </Grid>
                    </Border>
                  </ControlTemplate>
                </ToggleButton.Template>
              </ToggleButton>
              <ContentPresenter Content="{TemplateBinding SelectionBoxItem}" IsHitTestVisible="False"
                                Margin="13,0,0,0" VerticalAlignment="Center" TextElement.Foreground="{DynamicResource Texto}"/>
              <Popup IsOpen="{TemplateBinding IsDropDownOpen}" Placement="Bottom" AllowsTransparency="True"
                     Focusable="False" PopupAnimation="Fade">
                <Border Background="{DynamicResource Painel}" BorderBrush="{DynamicResource Bordo}" BorderThickness="1"
                        CornerRadius="9" Padding="4" MinWidth="{TemplateBinding ActualWidth}" Margin="0,4,0,0">
                  <StackPanel IsItemsHost="True"/>
                </Border>
              </Popup>
            </Grid>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>

    <!-- Serve o historico e a lista da equipa -->
    <Style TargetType="ListBoxItem">
      <Setter Property="Foreground" Value="{DynamicResource Apagado}"/>
      <Setter Property="FontSize" Value="11.5"/>
      <Setter Property="Cursor" Value="Hand"/>
      <Setter Property="HorizontalContentAlignment" Value="Stretch"/>
      <Setter Property="Template">
        <Setter.Value>
          <ControlTemplate TargetType="ListBoxItem">
            <Border x:Name="b" Background="Transparent" CornerRadius="8" Padding="8,6" Margin="0,1">
              <ContentPresenter/>
            </Border>
            <ControlTemplate.Triggers>
              <Trigger Property="IsMouseOver" Value="True">
                <Setter TargetName="b" Property="Background" Value="{DynamicResource Passar}"/>
                <Setter Property="Foreground" Value="{DynamicResource Texto}"/>
              </Trigger>
              <Trigger Property="IsSelected" Value="True">
                <Setter TargetName="b" Property="Background" Value="{DynamicResource RealceE}"/>
                <Setter Property="Foreground" Value="{DynamicResource Realce}"/>
              </Trigger>
            </ControlTemplate.Triggers>
          </ControlTemplate>
        </Setter.Value>
      </Setter>
    </Style>
  </Window.Resources>

  <Border x:Name="Raiz" Background="{DynamicResource Fundo}" CornerRadius="12" BorderBrush="{DynamicResource Bordo}" BorderThickness="1">
    <Grid>
      <Grid.RowDefinitions><RowDefinition Height="44"/><RowDefinition Height="*"/></Grid.RowDefinitions>

      <Grid x:Name="Barra" Grid.Row="0" Background="Transparent">
        <TextBlock Text="forja" Foreground="{DynamicResource Apagado}" FontSize="12.5"
                   VerticalAlignment="Center" Margin="18,0,0,0"/>
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Right">
          <Button x:Name="BtTema" Style="{StaticResource Janelinha}" Content="&#xE706;" ToolTip="Tema claro ou escuro"/>
          <Button x:Name="BtLetraMenos" Style="{StaticResource Janelinha}" Content="&#xE8E7;" ToolTip="Letra mais pequena    Ctrl -"/>
          <Button x:Name="BtLetraMais" Style="{StaticResource Janelinha}" Content="&#xE8E8;" ToolTip="Letra maior    Ctrl +"/>
          <Border Width="1" Margin="8,12" Background="{DynamicResource Bordo}"/>
          <Button x:Name="BtMinimizar" Style="{StaticResource Janelinha}" Content="&#xE921;" ToolTip="Minimizar"/>
          <Button x:Name="BtMaximizar" Style="{StaticResource Janelinha}" Content="&#xE922;" ToolTip="Ecra cheio    F11"/>
          <Button x:Name="BtFechar" Style="{StaticResource Janelinha}" Content="&#xE8BB;" ToolTip="Fechar"/>
        </StackPanel>
      </Grid>

      <!-- Tres colunas, como no Grok Bot: equipa | conversa | ficha -->
      <Grid Grid.Row="1">
        <Grid.ColumnDefinitions>
          <ColumnDefinition Width="280"/>
          <ColumnDefinition Width="*" MinWidth="400"/>
          <ColumnDefinition Width="Auto"/>
        </Grid.ColumnDefinitions>

        <Border x:Name="Lado" Grid.Column="0" Background="{DynamicResource Painel}" CornerRadius="0,0,0,11">
          <Grid Margin="14,6,14,12">
            <Grid.RowDefinitions>
              <RowDefinition Height="Auto"/><RowDefinition Height="*"/>
              <RowDefinition Height="Auto"/><RowDefinition Height="Auto"/><RowDefinition Height="Auto"/>
            </Grid.RowDefinitions>

            <StackPanel Grid.Row="0">
              <TextBlock Text="forja" Foreground="{DynamicResource Texto}" FontSize="19" FontWeight="SemiBold" Margin="6,4,0,2"/>
              <TextBlock Text="no teu computador, sem internet" Foreground="{DynamicResource Apagado}" FontSize="11.5" Margin="6,0,0,14"/>
            </StackPanel>

            <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto" HorizontalScrollBarVisibility="Disabled" Padding="0,0,2,0">
              <StackPanel>
                <TextBlock Text="A MINHA EQUIPA" Style="{StaticResource Seccao}"/>
                <ListBox x:Name="Equipa" Background="Transparent" BorderThickness="0" IsSynchronizedWithCurrentItem="False"
                         ScrollViewer.VerticalScrollBarVisibility="Disabled"
                         ScrollViewer.HorizontalScrollBarVisibility="Disabled"/>
                <Button x:Name="BtNovoAgente" Style="{StaticResource Pilula}" Content="+  agente novo"
                        FontSize="11.5" Padding="8,6" Margin="0,2,0,0"
                        ToolTip="Cria um agente novo. Fica um ficheiro .md na pasta agentes."/>

                <Border Height="1" Background="{DynamicResource Bordo}" Margin="6,14,6,12"/>

                <TextBlock Text="FERRAMENTAS" Style="{StaticResource Seccao}"/>
                <Button x:Name="BtConstruir" Style="{StaticResource Pilula}" Content="Construir um prompt"/>
                <Button x:Name="BtDiagnosticar" Style="{StaticResource Pilula}" Content="Diagnosticar um prompt"/>
                <Button x:Name="BtImagem" Style="{StaticResource Pilula}" Content="Imagem para markdown"/>
                <Button x:Name="BtReescrever" Style="{StaticResource Pilula}" Content="Reescrever texto"/>
                <Button x:Name="BtBiblioteca" Style="{StaticResource Pilula}" Content="Biblioteca de prompts"/>
                <Button x:Name="BtFicheiro" Style="{StaticResource Pilula}" Content="Conversar com um ficheiro"/>
                <Button x:Name="BtProcurar" Style="{StaticResource Pilula}" Content="Procurar nos ficheiros"/>
                <Button x:Name="BtDuplicados" Style="{StaticResource Pilula}" Content="Encontrar duplicados"/>
                <Button x:Name="BtAgentes" Style="{StaticResource Pilula}" Content="A minha equipa"/>

                <!-- Os paineis de opcoes ficam sobrepostos: so um esta visivel de cada vez -->
                <Grid Margin="0,12,0,0">
                  <StackPanel x:Name="PainelOpcoes">
                    <TextBlock Text="DESTINO" Style="{StaticResource Seccao}"/>
                    <ComboBox x:Name="Destino" Margin="6,0,6,14">
                      <ComboBoxItem Content="nenhum" IsSelected="True"/>
                      <ComboBoxItem Content="ChatGPT"/>
                      <ComboBoxItem Content="Gemini"/>
                      <ComboBoxItem Content="Claude"/>
                    </ComboBox>
                    <CheckBox x:Name="AutoCritica" IsChecked="True" FontSize="12.5" Margin="6,0,0,4">
                      <TextBlock Text="Rever contra o checklist" TextWrapping="Wrap"/>
                    </CheckBox>
                    <TextBlock Text="Segunda passagem que corrige o prompt. Mais 15 segundos."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="26,2,6,0"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelAccoes" Visibility="Collapsed">
                    <TextBlock Text="O QUE FAZER AO TEXTO" Style="{StaticResource Seccao}"/>
                    <ComboBox x:Name="Accao" Margin="6,0,6,10"/>
                    <TextBlock Text="As accoes vem do regras.md e podes acrescentar as tuas."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="6,0,6,10"/>
                    <Border Height="1" Background="{DynamicResource Bordo}" Margin="6,0,6,10"/>
                    <TextBlock Text="ATALHO EM QUALQUER LADO" Style="{StaticResource Seccao}"/>
                    <TextBlock Text="Selecciona texto em qualquer aplicacao e carrega em Ctrl+Alt+P."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="6,0,6,8"/>
                    <CheckBox x:Name="ColarSozinho" FontSize="12.5" Margin="6,0,0,4">
                      <TextBlock Text="Colar sozinho no sitio" TextWrapping="Wrap"/>
                    </CheckBox>
                    <TextBlock Text="Desligado, o resultado fica na area de transferencia e colas tu. Ligado, ele substitui o texto seleccionado - mais rapido, mas se a janela mudar entretanto cola no sitio errado."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="26,2,6,0"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelBiblioteca" Visibility="Collapsed">
                    <TextBlock Text="Escolhe um prompt na lista em baixo. Ele abre na caixa de escrever, onde preenches os {campos} e copias."
                               Foreground="{DynamicResource Apagado}" FontSize="11" TextWrapping="Wrap" Margin="6,0,6,10"/>
                    <Button x:Name="BtAbrirBiblioteca" Style="{StaticResource Pilula}" Content="Abrir o ficheiro" FontSize="11.5" Padding="9,6"
                            ToolTip="Abre a biblioteca no bloco de notas, para renomear ou apagar prompts"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelAgente" Visibility="Collapsed">
                    <TextBlock Text="Escolhe alguem da equipa em cima e fala com ele aqui ao lado. A ficha dele abre a direita."
                               Foreground="{DynamicResource Apagado}" FontSize="11" TextWrapping="Wrap" Margin="6,0,6,4"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelPasta" Visibility="Collapsed">
                    <TextBlock Text="PASTA" Style="{StaticResource Seccao}"/>
                    <Border Background="{DynamicResource Cartao}" BorderBrush="{DynamicResource Bordo}" BorderThickness="1"
                            CornerRadius="9" Padding="11,9" Margin="6,0,6,10">
                      <StackPanel>
                        <TextBlock x:Name="PastaNome" Text="nenhuma escolhida" Foreground="{DynamicResource Texto}" FontSize="12" TextWrapping="Wrap"/>
                        <TextBlock x:Name="PastaInfo" Text="" Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="0,4,0,0"/>
                      </StackPanel>
                    </Border>
                    <Button x:Name="BtEscolherPasta" Style="{StaticResource Pilula}" Content="Escolher pasta..." FontSize="11.5" Padding="9,6"/>
                    <Button x:Name="BtIndexar" Style="{StaticResource Pilula}" Content="Ler os ficheiros" FontSize="11.5" Padding="9,6"
                            ToolTip="Le o texto de todos os ficheiros uma vez e guarda. Da proxima e quase instantaneo."/>
                    <Button x:Name="BtDesfazer" Style="{StaticResource Pilula}" Content="Desfazer a ultima mudanca" FontSize="11.5" Padding="9,6" Visibility="Collapsed"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelFicheiro" Visibility="Collapsed">
                    <TextBlock Text="FICHEIRO" Style="{StaticResource Seccao}"/>
                    <Border Background="{DynamicResource Cartao}" BorderBrush="{DynamicResource Bordo}" BorderThickness="1"
                            CornerRadius="9" Padding="11,9" Margin="6,0,6,10">
                      <StackPanel>
                        <TextBlock x:Name="FichNome" Text="nenhum escolhido" Foreground="{DynamicResource Texto}"
                                   FontSize="12" TextWrapping="Wrap"/>
                        <TextBlock x:Name="FichInfo" Text="" Foreground="{DynamicResource Apagado}" FontSize="10.5"
                                   TextWrapping="Wrap" Margin="0,4,0,0"/>
                      </StackPanel>
                    </Border>
                    <Button x:Name="BtEscolherFicheiro" Style="{StaticResource Pilula}" Content="Escolher ficheiro..." FontSize="11.5" Padding="9,6"/>
                    <TextBlock Text="Aceita PDF, Word, txt, md e csv. Tambem podes arrastar para a caixa de escrever."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="6,6,6,0"/>
                  </StackPanel>

                  <StackPanel x:Name="PainelImagem" Visibility="Collapsed">
                    <TextBlock Text="COMO LER A IMAGEM" Style="{StaticResource Seccao}"/>
                    <CheckBox x:Name="ForcarVisao" FontSize="12.5" Margin="6,0,0,4">
                      <TextBlock Text="Forcar modelo de visao" TextWrapping="Wrap"/>
                    </CheckBox>
                    <TextBlock Text="Por omissao usa o OCR do Windows. Se a imagem tiver pouco texto passa sozinho ao modelo de visao."
                               Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="26,2,6,0"/>
                  </StackPanel>
                </Grid>
              </StackPanel>
            </ScrollViewer>

            <Grid Grid.Row="2" Margin="0,14,0,6">
              <TextBlock x:Name="TituloLista" Text="HISTORICO" Style="{StaticResource Seccao}" Margin="6,0,0,0"/>
              <Button x:Name="BtLimparHist" Style="{StaticResource Pilula}" Content="limpar" Padding="6,0"
                      FontSize="10.5" HorizontalAlignment="Right" ToolTip="Apagar tudo o que esta guardado"/>
            </Grid>

            <ListBox x:Name="Historico" Grid.Row="3" Background="Transparent" BorderThickness="0" MaxHeight="170" IsSynchronizedWithCurrentItem="False"
                     ScrollViewer.HorizontalScrollBarVisibility="Disabled"/>

            <StackPanel Grid.Row="4">
              <Border Height="1" Background="{DynamicResource Bordo}" Margin="6,10,6,8"/>
              <Button x:Name="BtAbrirRegras" Style="{StaticResource Pilula}" Content="Abrir as regras" FontSize="11.5"
                      Padding="9,6" Margin="0,0,0,6"
                      ToolTip="Abre regras.md no bloco de notas. Mexes, gravas, e a proxima resposta ja usa. Nao e preciso recompilar."/>
              <TextBlock x:Name="Estado" Text="" Foreground="{DynamicResource Apagado}" FontSize="11.5" TextWrapping="Wrap" Margin="6,0,6,0"/>
            </StackPanel>
          </Grid>
        </Border>

        <!-- A conversa -->
        <Grid Grid.Column="1" Margin="20,6,20,14">
          <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/><RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/><RowDefinition Height="Auto"/>
          </Grid.RowDefinitions>

          <Grid Grid.Row="0" Margin="2,2,0,12">
            <Grid.ColumnDefinitions>
              <ColumnDefinition Width="Auto"/><ColumnDefinition Width="*"/><ColumnDefinition Width="Auto"/>
            </Grid.ColumnDefinitions>
            <Ellipse x:Name="CabecaCor" Grid.Column="0" Width="11" Height="11" Fill="Transparent"
                     VerticalAlignment="Top" Margin="0,7,10,0" Visibility="Collapsed"/>
            <StackPanel Grid.Column="1">
              <TextBlock x:Name="Titulo" Text="Construir um prompt" Foreground="{DynamicResource Texto}"
                         FontSize="21" FontWeight="SemiBold" Margin="0,0,0,3"/>
              <TextBlock x:Name="Ajuda" Text="" Foreground="{DynamicResource Apagado}" FontSize="12.5"
                         TextWrapping="Wrap"/>
            </StackPanel>
            <Border x:Name="CrachaCerebro" Grid.Column="2" Visibility="Collapsed" VerticalAlignment="Top"
                    Background="{DynamicResource RealceE}" CornerRadius="9" Padding="9,4" Margin="12,4,0,0">
              <TextBlock x:Name="CrachaTexto" Text="" Foreground="{DynamicResource Realce}" FontSize="10.5" FontWeight="SemiBold"/>
            </Border>
          </Grid>

          <Border x:Name="Moldura" Grid.Row="1" Background="{DynamicResource Cartao}" BorderBrush="{DynamicResource Bordo}"
                  BorderThickness="1" CornerRadius="10">
            <ScrollViewer x:Name="Rolo" VerticalScrollBarVisibility="Auto" Padding="16,14">
              <StackPanel x:Name="Fio">
                <Border x:Name="CartaoSaida" Background="Transparent" CornerRadius="18" Padding="0">
                  <TextBox x:Name="Saida" IsReadOnly="True" Background="Transparent" BorderThickness="0"
                           TextWrapping="Wrap" FontFamily="Cascadia Mono, Consolas" FontSize="12.5"
                           Foreground="{DynamicResource Texto}" Padding="0"/>
                </Border>
              </StackPanel>
            </ScrollViewer>
          </Border>

          <!-- O comprimido de escrever, em baixo, como no Grok Bot -->
          <Border x:Name="Comprimido" Grid.Row="2" Background="{DynamicResource Cartao}" BorderBrush="{DynamicResource Bordo}"
                  BorderThickness="1" CornerRadius="18" Padding="5,4" Margin="0,12,0,0">
            <Grid>
              <Grid.ColumnDefinitions>
                <ColumnDefinition Width="Auto"/><ColumnDefinition Width="*"/><ColumnDefinition Width="Auto"/>
              </Grid.ColumnDefinitions>
              <Button x:Name="BtMais" Grid.Column="0" Style="{StaticResource Pilula}" Content="+" FontSize="17"
                      Padding="11,1" VerticalAlignment="Bottom" Margin="0,0,2,1"
                      ToolTip="Juntar um ficheiro, uma imagem ou uma pasta"/>
              <TextBox x:Name="Entrada" Grid.Column="1" MinHeight="36" MaxHeight="200" AcceptsReturn="True" TextWrapping="Wrap"
                       VerticalScrollBarVisibility="Auto" AllowDrop="True" VerticalAlignment="Bottom"
                       Background="Transparent" BorderThickness="0" Padding="6,8"/>
              <StackPanel Grid.Column="2" Orientation="Horizontal" VerticalAlignment="Bottom" Margin="6,0,0,0">
                <Button x:Name="BtRecomecar" Style="{StaticResource Pilula}" Content="recomecar" FontSize="11.5" Padding="10,7" Margin="0,0,4,0"/>
                <Button x:Name="BtEnviar" Style="{StaticResource Accao}" Content="Enviar    Ctrl+Enter"/>
              </StackPanel>
            </Grid>
          </Border>

          <StackPanel Grid.Row="3" Orientation="Horizontal" Margin="2,10,0,0">
            <Button x:Name="BtEscolherImagem" Style="{StaticResource Pilula}" Content="Escolher imagem..." FontSize="11.5" Padding="9,6" Visibility="Collapsed" Margin="0,0,6,0"/>
            <Button x:Name="BtCopiarPrompt" Style="{StaticResource Pilula}" Content="Copiar prompt" FontSize="11.5" Padding="9,6" Margin="0,0,6,0"
                    ToolTip="So o prompt melhorado, pronto a colar"/>
            <Button x:Name="BtCopiar" Style="{StaticResource Pilula}" Content="Copiar tudo" FontSize="11.5" Padding="9,6" Margin="0,0,6,0"/>
            <Button x:Name="BtGuardar" Style="{StaticResource Pilula}" Content="Guardar .md" FontSize="11.5" Padding="9,6" Margin="0,0,6,0"/>
            <Button x:Name="BtGuardarBiblioteca" Style="{StaticResource Pilula}" Content="Guardar na biblioteca" FontSize="11.5" Padding="9,6"
                    ToolTip="Guarda o prompt final para o reutilizares mais tarde"/>
            <TextBlock x:Name="Rodape" Foreground="{DynamicResource Apagado}" FontSize="11.5"
                       VerticalAlignment="Center" Margin="14,0,0,0"/>
          </StackPanel>
        </Grid>

        <!-- A ficha do agente, so no modo da equipa -->
        <Border x:Name="Ficha" Grid.Column="2" Width="320" Visibility="Collapsed"
                Background="{DynamicResource Painel}" BorderBrush="{DynamicResource Bordo}"
                BorderThickness="1,0,0,0" CornerRadius="0,0,11,0">
          <ScrollViewer VerticalScrollBarVisibility="Auto" Padding="16,12,16,14">
            <StackPanel>
              <TextBlock Text="FICHA DO AGENTE" Style="{StaticResource Seccao}" Margin="2,2,0,4"/>
              <TextBlock Text="O que mudares aqui e gravado no ficheiro .md do agente."
                         Foreground="{DynamicResource Apagado}" FontSize="10.5" TextWrapping="Wrap" Margin="2,0,0,6"/>

              <TextBlock Text="Nome" Style="{StaticResource Rotulo}"/>
              <TextBox x:Name="FiNome" FontSize="13" Padding="10,7"/>
              <TextBlock Text="Cargo" Style="{StaticResource Rotulo}"/>
              <TextBox x:Name="FiCargo" FontSize="13" Padding="10,7"/>
              <TextBlock Text="Missao" Style="{StaticResource Rotulo}"/>
              <TextBox x:Name="FiMissao" Height="104" AcceptsReturn="True" TextWrapping="Wrap"
                       VerticalScrollBarVisibility="Auto" FontSize="12.5"/>

              <TextBlock Text="Cerebro" Style="{StaticResource Rotulo}"/>
              <RadioButton x:Name="FiLocal" GroupName="cerebro" Content="modelo local, sem internet"/>
              <RadioButton x:Name="FiClaude" GroupName="cerebro" Content="Claude, pode pesquisar"/>

              <TextBlock Text="Ferramentas" Style="{StaticResource Rotulo}"/>
              <CheckBox x:Name="FiWebSearch" FontSize="12.5" Margin="2,2,0,2" Content="WebSearch - procurar na net"/>
              <CheckBox x:Name="FiWebFetch" FontSize="12.5" Margin="2,2,0,2" Content="WebFetch - abrir paginas"/>
              <CheckBox x:Name="FiFicheiro" FontSize="12.5" Margin="2,2,0,2" Content="ler-ficheiro - ler documentos"/>
              <TextBlock x:Name="FiAviso" Text="" Foreground="{DynamicResource Apagado}" FontSize="10.5"
                         TextWrapping="Wrap" Margin="2,6,0,0"/>

              <TextBlock Text="Cor" Style="{StaticResource Rotulo}"/>
              <StackPanel x:Name="Cores" Orientation="Horizontal" Margin="2,2,0,0"/>

              <Button x:Name="BtGravarAgente" Style="{StaticResource Accao}" Content="Gravar" Margin="0,16,0,0" HorizontalAlignment="Left"/>
              <Button x:Name="BtAbrirAgente" Style="{StaticResource Pilula}" Content="Abrir o ficheiro .md" FontSize="11.5" Padding="9,6" Margin="0,8,0,0"/>
              <Button x:Name="BtArquivarAgente" Style="{StaticResource Pilula}" Content="Arquivar este agente" FontSize="11.5" Padding="9,6"
                      ToolTip="Move o ficheiro para a subpasta _arquivo. Nunca apaga nada."/>
              <TextBlock x:Name="FiMemoria" Text="" Foreground="{DynamicResource Apagado}" FontSize="10.5"
                         TextWrapping="Wrap" Margin="2,10,0,0"/>
            </StackPanel>
          </ScrollViewer>
        </Border>
      </Grid>
    </Grid>
  </Border>
</Window>
~~~~

