# Left 4 Dead 2 — MOTD HTML com tela preta: solução testada no DIGS TV

**Data do teste:** 25/11/2026  
**Projeto:** DIGS TV / XUPACU GAMERS BR — servidor dedicado L4D2, ZoneMod, tickrate configurado para 128  
**Ambiente testado:** Windows 10, SRCDS dedicado e cliente L4D2 no mesmo PC  
**Resultado:** ✅ O **MOTD HTML inline simplificado** foi exibido corretamente no cliente do teste.  
**Limitação:** ⚠️ O **Host Banner superior ainda exibe a URL como texto**; sua imagem **não** foi comprovadamente renderizada. Os testes não garantem o mesmo resultado em todos os clientes.

> Relato comunitário baseado em testes reais, capturas do jogo e auditorias somente leitura dos arquivos e logs. Não é uma correção oficial da Valve.

## 1. Sintoma original

Ao conectar ao servidor, o cliente mostrava o título “MENSAGEM DO DIA”, mas a área de conteúdo HTML permanecia preta/translúcida. Em outras tentativas, o endereço da página aparecia literalmente como texto amarelo. O Host Banner superior também mostrava a URL em vez de uma imagem.

Foram investigadas, sem necessidade de reinstalar o jogo ou trocar drivers, duas questões distintas:

- **Entrega do conteúdo** (SRCDS → cliente);
- **Renderização do conteúdo HTML** no painel VGUI do cliente.

## 2. O que foi confirmado nos diagnósticos

| Teste / evidência | Resultado |
| --- | --- |
| GitHub Pages com caminhos corretos `/DIGS-TV-L4D2-BANNERS/host.html` e `motd.html` | HTTP 200 |
| JPEGs `DIGS_TV_HEADER_800x400.jpg` e `DIGS_TV_MOTD_900x450.jpg` | HTTP 200, image/jpeg |
| Acesso HTTP aos mesmos caminhos no GitHub Pages | Redireciona com HTTP 301 para HTTPS |
| TXT contendo URLs | Verificados sem BOM, sem espaços extras |
| `motd_enabled` consultado no SRCDS | `1` |
| Servidor Python local na porta 8088 | HTTP 200, mas o conteúdo não apareceu no MOTD |
| MOTD com HTML inline anterior | Área HTML vazia |
| MOTD inline simplificado (seção 3) | **Funcionou visualmente no jogo** |
| Hostfile com `<img src=...>` | Tag exibida literalmente como texto |
| Hostfile restaurado para URL HTTPS pública | A URL aparece como texto no painel superior; imagem ainda não comprovada |

Auditoria adicional encontrou no cliente o arquivo `textwindow_temp.html` contendo o HTML recebido do servidor com um byte NUL final. Isso confirma que o conteúdo **chegou** ao cliente na sessão auditada; logo, o vazio não se explicava apenas por falta de entrega do arquivo.

O log do jogo continha erros históricos `wglCreatePbufferARB not available` e `Could not create GLSurface` (de **19/11/2026**, não da sessão final). Eles **não** comprovam defeito da placa de vídeo, incompatibilidade permanente ou a causa exata da falha anterior. Posteriormente o HTML passou a funcionar **sem troca de GPU ou driver**.

## 3. HTML mínimo que funcionou no teste

Arquivo criado no servidor:

`C:\L4D2_Server_OFICIAL\left4dead2\motd_digs_ultra.txt`

Conteúdo testado (HTML inline, sem DOCTYPE):

```html
<html>
<body bgcolor="#14141E" text="#FFFFFF">
<center>
<h1><font color="#FF9900">DIGS TV</font></h1>
<p>ZoneMod | 128 Tick</p>
<hr>
<p>Bem-vindo ao XUPACU GAMERS BR!</p>
</center>
</body>
</html>
```

O arquivo foi validado com **196 bytes** e **UTF-8 sem BOM**. No teste bem-sucedido a captura exibiu título “DIGS TV” em laranja, texto “ZoneMod | 128 Tick”, linha divisória, mensagem de boas-vindas e fundo escuro.

**Importante:** a mudança para HTML simplificado coincidiu com o sucesso, mas não houve ensaio isolado que permita afirmar qual diferença específica (DOCTYPE, estrutura, reinicialização ou estado do cliente) resolveu a falha.

## 4. Procedimento de reprodução com backups

Faça em ambiente autorizado, preferencialmente fora de uma partida pública em andamento. Preserve o iniciador BAT, ZoneMod, plugins e configurações originais.

### A. Criar o arquivo novo (sem sobrescrever um existente)

No PowerShell:

```powershell
$Host.UI.RawUI.WindowTitle = "DIGS TV | PREPARAR MOTD HTML"
$dir = "C:\L4D2_Server_OFICIAL\left4dead2"
$destino = Join-Path $dir "motd_digs_ultra.txt"
$html = @'
<html>
<body bgcolor="#14141E" text="#FFFFFF">
<center>
<h1><font color="#FF9900">DIGS TV</font></h1>
<p>ZoneMod | 128 Tick</p>
<hr>
<p>Bem-vindo ao XUPACU GAMERS BR!</p>
</center>
</body>
</html>
'@
if (Test-Path -LiteralPath $destino) {
    throw "O arquivo ja existe. Nao sobrescreva sem backup."
}
[System.IO.File]::WriteAllText($destino, $html, [System.Text.UTF8Encoding]::new($false))
Get-Item -LiteralPath $destino | Select-Object FullName,Length
```

> No Windows PowerShell 5.1, evite depender de `Set-Content -Encoding UTF8` quando for essencial gravar **sem BOM**.

### B. Fazer backup da configuração e alterar somente a linha `motdfile`

`server.cfg` usado no teste: `C:\L4D2_Server_OFICIAL\left4dead2\cfg\server.cfg`.

```powershell
$Host.UI.RawUI.WindowTitle = "DIGS TV | BACKUP MOTD CFG"
$cfg = "C:\L4D2_Server_OFICIAL\left4dead2\cfg\server.cfg"
$backup = "$cfg.bak_$(Get-Date -Format yyyyMMdd_HHmmss)"
Copy-Item -LiteralPath $cfg -Destination $backup -ErrorAction Stop
Get-FileHash -LiteralPath @($cfg,$backup) -Algorithm SHA256
```

Depois de verificar o backup e a linha existente, configure **apenas**:

```cfg
hostfile "digs_tv_host_banner_v2.txt"
motdfile "motd_digs_ultra.txt"
```

O nome do arquivo deve ser ajustado conforme a instalação de quem reproduzir o procedimento. **Não substitua o arquivo completo `server.cfg` por esse exemplo.**

### C. Confirmar o valor efetivo no console físico do SRCDS

O teste mostrou que **editar `server.cfg` em disco não atualizou imediatamente a CVar carregada**. No console do SRCDS, foram usadas consultas e, em seguida, a atualização:

```text
motdfile; hostfile; motd_enabled
```

```text
motdfile "motd_digs_ultra.txt"; motdfile
```

A resposta confirmou `"motdfile" = "motd_digs_ultra.txt"`. A configuração em disco também foi atualizada para persistência no próximo início. Em versões/configurações diferentes, pode ser necessária nova conexão, troca de mapa ou reinício controlado para reapresentar o conteúdo; a simples alteração da CVar não prova que todos os clientes já receberam o novo texto.

### D. Verificar no cliente

Conectar ao servidor e abrir o MOTD (na instalação do teste, tecla **H**). O critério de sucesso é ver **HTML visual**, e não apenas a URL ou o título da janela. O resultado positivo foi registrado numa captura de 25/11/2026.

## 5. Host Banner: resultado parcial, não concluído

A configuração foi restaurada para:

```cfg
hostfile "digs_tv_host_banner_v2.txt"
```

Com `digs_tv_host_banner_v2.txt` contendo apenas:

```text
https://digsawtv.github.io/DIGS-TV-L4D2-BANNERS/host.html
```

Tentamos inserir diretamente uma tag `<img>` nesse TXT: **a tag apareceu literalmente no jogo**. Restauramos a URL nua por backup e SHA-256 idêntico. Isso **não comprova** que a imagem do Host Banner tenha funcionado; a captura mais recente ainda mostra a URL como texto. O Host Banner segue **pendente**.

## 6. Tentativas que não solucionaram o MOTD na época

- URL HTTPS direta do GitHub Pages dentro do TXT;
- HTTP local `127.0.0.1:8088`;
- HTTP LAN `192.168.100.2:8088`;
- Primeiro HTML inline;
- Alterar somente as dimensões da imagem, limpeza de cache ou trocar DNS **não foram identificados como soluções**.

O teste HTTPS direto pode ter sido interpretado como texto por uma rotina antiga do cliente que verifica `http://` explicitamente; isso foi indicado por análise de strings e comparação com código Source correspondente, **não** deve ser generalizado a todas as versões do jogo.

O IP privado `192.168.100.2` só atende a rede local, não é hospedagem para jogadores públicos.

## 7. O que não foi necessário mudar

- Placa de vídeo AMD Radeon RX 7600 ou seu driver;
- Windows, Hyper-V, WHPX ou BCD;
- Tickrate 128, ZoneMod, SourceMod e Metamod;
- Iniciador BAT especial que abre Steam/jogo antes do SRCDS;
- Arquivos originais do servidor, sem backup.

O plugin `cfg_motd.smx` presente no ZoneMod foi inspecionado: registra `sm_cfg` e `sm_changelog` para um painel de descrição da configuração e não mostrou evidência de substituir automaticamente o MOTD nativo.

## 8. Limitações, pendências e referências

**Comprovado neste caso:** HTML inline simples foi exibido com sucesso no cliente DIGS TV, com ZoneMod. **Não comprovado:** causa única do problema antigo, funcionamento em todos os computadores, acessibilidade externa do servidor na porta do jogo ou imagem no Host Banner.

Se a janela continuar preta em outro cliente, considere primeiro confirmar a CVar ativa, recebimento do HTML e logs locais, sem assumir que o hardware é culpado ou que o bug seja irreversível.

Referências comunitárias / bugs documentados:

- Valve — MOTD vazio/translúcido: https://github.com/ValveSoftware/Source-1-Games/issues/3414
- Valve — Host Window: https://github.com/ValveSoftware/Source-1-Games/issues/3415
- AlliedModders — exemplo de MOTD L4D2: https://forums.alliedmods.net/showthread.php?t=329328

**Créditos:** projeto e testes **DIGS TV / XUPACU GAMERS BR**; auditorias e análise colaborativa com ferramentas de IA. Relato fornecido à comunidade sem garantia de correção universal.
