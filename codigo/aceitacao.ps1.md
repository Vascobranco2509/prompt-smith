# scripts\aceitacao.ps1

O ficheiro inteiro esta no bloco abaixo. Grava-o em `scripts\aceitacao.ps1`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `scripts\aceitacao.ps1`

<!-- destino: scripts\aceitacao.ps1 | codificacao: bom -->
~~~~powershell
# Teste de aceitacao: corre o prompt-smith como um utilizador o usa e verifica tudo.
# Uso:  .\aceitacao.ps1          (1 ronda)
#       .\aceitacao.ps1 -Rondas 3
param([int]$Rondas = 1)

$raiz = Split-Path -Parent $PSScriptRoot
$psd  = Join-Path $raiz 'bin\psd.cmd'
$tmp  = Join-Path $env:TEMP 'aceitacao-prompt-smith.txt'
'faz-me um resumo deste relatorio de vendas e diz quais sao os produtos que estao a correr mal, quero uma coisa rapida para a reuniao' |
    Out-File $tmp -Encoding utf8

$total = 0; $passou = 0
function Check($n, $c) {
    $script:total++
    if ($c) { $script:passou++; Write-Host "  [OK]   $n" -ForegroundColor Green }
    else    { Write-Host "  [FALHA] $n" -ForegroundColor Red }
}
function Ponto4([string]$t) {
    if ($t -match '(?ms)^\s*4\.[^\r\n]*\r?\n(.*?)(?=^\s*5\.)') { return $Matches[1].Trim() }
    return ''
}

for ($ronda = 1; $ronda -le $Rondas; $ronda++) {
    Write-Host "`n=========== RONDA $ronda ===========" -ForegroundColor Magenta

    # --- diagnostico sem destino ---
    Write-Host "`nDiagnostico simples"
    $t = (& $psd $tmp) -join "`n"
    Check "tem os 5 pontos" (([regex]::Matches($t,'(?m)^\s*[1-5]\.\s')).Count -ge 5)
    Check "comeca por 'Modo: diagnostico'" ($t -match 'Modo:\s*diagnostico')
    Check "sem destino: nao ha linha 'Ajustado para'" (-not ($t -match '(?i)ajustado para'))

    # --- os tres destinos ---
    foreach ($d in 'Claude','ChatGPT','Gemini') {
        Write-Host "`nDestino $d"
        $t = (& $psd $tmp $d) -join "`n"
        $p4 = Ponto4 $t
        Check "tem os 5 pontos" (([regex]::Matches($t,'(?m)^\s*[1-5]\.\s')).Count -ge 5)
        Check "ponto 4 nao esta vazio ($($p4.Length) chars)" ($p4.Length -gt 60)
        $en = ([regex]::Matches($p4,'(?i)\b(the|you|and|with|for|that|this|are|must|should)\b')).Count
        Check "ponto 4 em ingles (palavras EN: $en)" ($en -ge 5)
        $forma = switch ($d) {
            'Claude'  { ($p4 -match '<context>') -and ($p4 -match '</context>') }
            'ChatGPT' { ($p4 -match '(?i)you are') -and (([regex]::Matches($p4,'\d[\.\)]\s')).Count -ge 3) }
            'Gemini'  { $p4 -match '(?i)(section|task:)' }
        }
        Check "ponto 4 na forma do $d" $forma
        Check "tem a linha 'Ajustado para $d'" ($t -match "(?i)ajustado para $d")
    }
}
Write-Host "`nRESULTADO: $passou / $total" -ForegroundColor Yellow

~~~~

