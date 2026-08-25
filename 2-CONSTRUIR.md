# 2. Construir

Ha dois caminhos. O primeiro da-te **exactamente** a mesma aplicacao. O segundo
serve se preferires que o Claude Code construa a partir da especificacao.

---

## Caminho A: extrair o codigo e compilar (recomendado)

O codigo verdadeiro esta dentro dos blocos em `codigo\`. Cada bloco tem uma marca
invisivel a dizer onde deve ser gravado. Este comando le as marcas e escreve a
arvore de ficheiros.

**Abre o PowerShell na pasta que descarregaste** e cola isto:

~~~~powershell
$destino = "$HOME\prompt-smith"

Get-ChildItem ".\codigo" -Filter *.md -File | ForEach-Object {
    $alvo = $null; $bom = $true; $dentro = $false
    $buffer = New-Object System.Collections.Generic.List[string]
    foreach ($linha in (Get-Content $_.FullName -Encoding UTF8)) {
        if ($linha -match '^<!--\s*destino:\s*(.+?)\s*\|\s*codificacao:\s*(\w+)\s*-->') {
            $alvo = $Matches[1]; $bom = ($Matches[2] -eq "bom"); continue
        }
        if ($linha -like '~~~~*') {
            if (-not $dentro) { $dentro = $true; $buffer.Clear(); continue }
            $dentro = $false
            if ($alvo) {
                $f = Join-Path $destino $alvo
                $pasta = Split-Path $f
                if (-not (Test-Path $pasta)) { New-Item -ItemType Directory -Force $pasta | Out-Null }
                [System.IO.File]::WriteAllLines($f, $buffer, (New-Object System.Text.UTF8Encoding($bom)))
                Write-Host ("  " + $alvo.PadRight(28) + $buffer.Count + " linhas")
                $alvo = $null
            }
            continue
        }
        if ($dentro) { $buffer.Add($linha) }
    }
}
Write-Host "Pronto: $destino"
~~~~

Ficas com isto:

    prompt-smith\
      Modelfile
      app\Nucleo.cs      app\Motor.cs      app\Janela.cs
      app\janela.xaml    app\compilar.cmd
      agentes\           os seis agentes
      regras\regras.md
      scripts\aceitacao.ps1

### Compilar

O compilador de C# **ja vem no Windows**. Nao instalas nada.

~~~~
%USERPROFILE%\prompt-smith\app\compilar.cmd
~~~~

Demora uns segundos e escreve `app\prompt-smith.exe`. Duplo clique e esta a andar.

Se disser que nao encontra o compilador, e porque falta a .NET Framework 4 —
raro, vem com o Windows desde ha muito. O caminho que ele procura e
`C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`.

### Confirmar que ficou bem

~~~~
powershell -ExecutionPolicy Bypass -File %USERPROFILE%\prompt-smith\scripts\aceitacao.ps1
~~~~

Tem de dar **18 / 18**. Se der menos, o modelo nao foi criado como deve — volta ao
[1-INSTALAR.md](1-INSTALAR.md).

---

## Caminho B: dar a especificacao ao Claude Code

Se tens o Claude Code instalado e preferes que ele construa, abre esta pasta com
ele e diz-lhe alguma coisa como:

> Le o 3-ESPECIFICACAO.md, o 4-AGENTES.md e o 5-REGRAS.md desta pasta e constroi
> a aplicacao que la esta descrita. E uma janela WPF em C#, compilada com o csc.exe
> que ja vem no Windows, sem SDK e sem Visual Studio. O XAML e carregado do disco
> em tempo de execucao com XamlReader.Load, e nao ha code-behind: todos os eventos
> sao ligados por codigo. Respeita os quatro contratos que a especificacao lista.

Fica igual no comportamento, mas nao linha a linha — depende do que o Claude Code
decidir. Se queres o mesmo codigo, usa o caminho A.

---

## Se quiseres mudar o aspecto

O `janela.xaml` e **lido do disco quando a aplicacao arranca**, e o ficheiro solto
ganha ao que esta dentro do `.exe`. Ou seja: mexes nas cores ou no layout, gravas,
voltas a abrir a janela, e ja esta. **So mexer nos `.cs` obriga a recompilar.**
