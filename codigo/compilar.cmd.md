# app\compilar.cmd

O ficheiro inteiro esta no bloco abaixo. Grava-o em `app\compilar.cmd`.
O `2-CONSTRUIR.md` traz um comando que faz isto por ti, e pela codificacao certa.

## `app\compilar.cmd`

<!-- destino: app\compilar.cmd | codificacao: sembom -->
~~~~bat
@echo off
REM Recria o prompt-smith.exe com o compilador que ja vem no Windows.
REM Nao descarrega nada. Duplo clique chega.
setlocal
set NET=C:\Windows\Microsoft.NET\Framework64\v4.0.30319
set WPF=%NET%\WPF
set WM=C:\Windows\System32\WinMetadata
if not exist "%NET%\csc.exe" ( echo Nao encontrei o compilador em %NET% & pause & exit /b 1 )
echo A compilar...
"%NET%\csc.exe" /nologo /target:winexe /platform:anycpu /optimize+ ^
  /out:"%~dp0prompt-smith.exe" ^
  /resource:"%~dp0janela.xaml",janela.xaml ^
  /reference:"%NET%\System.dll" /reference:"%NET%\System.Core.dll" /reference:"%NET%\System.Xml.dll" ^
  /reference:"%NET%\System.Web.Extensions.dll" /reference:"%NET%\System.Runtime.dll" /reference:"%NET%\System.Xaml.dll" ^
  /reference:"%NET%\System.IO.Compression.dll" /reference:"%NET%\System.IO.Compression.FileSystem.dll" ^
  /reference:"%WPF%\PresentationFramework.dll" /reference:"%WPF%\PresentationCore.dll" /reference:"%WPF%\WindowsBase.dll" ^
  /reference:"%WM%\Windows.Foundation.winmd" /reference:"%WM%\Windows.Graphics.winmd" ^
  /reference:"%WM%\Windows.Media.winmd" /reference:"%WM%\Windows.Storage.winmd" ^
  "%~dp0Nucleo.cs" "%~dp0Motor.cs" "%~dp0Janela.cs"
if errorlevel 1 ( echo. & echo A compilacao falhou. Le as mensagens acima. & pause & exit /b 1 )
echo.
echo Pronto: %~dp0prompt-smith.exe
pause

~~~~

