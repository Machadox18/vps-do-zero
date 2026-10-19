# Notas - VPS do zero

## Dia 1 - 08/10/2026-09/10/2026

### 1. Chave SSH
- Comando: `ssh-keygen -t ed25519 -C "gabriel-vps"`
- Cria um par de chaves: a privada (`id_ed25519`) fica só no meu PC, a pública (`id_ed25519.pub`) vai para o servidor
- A passphrase protege a chave privada se alguém pegar meu PC
- `cat ~/.ssh/id_ed25519.pub` mostra a chave pública para copiar
- Erro que tive: digitei `.pu` em vez de `.pub` e o arquivo "não existia"

### 2. Criação da VPS
- Vultr, Shared CPU, São Paulo, Ubuntu 24.04, 1 vCPU / 2 GB
- Desliguei Automatic Backups (custo extra)
- O "/mo" é o teto mensal; a cobrança é por hora (~US$ 0,03/h)
- Desligar não para a cobrança, só Destroy
- O painel deu erro ao cadastrar a chave SSH (bug da Vultr)

### 3. Primeiro acesso
- `ssh root@IP`: conecta no servidor
- Na 1ª vez pergunta se confio no servidor (impressão digital); respondi yes
- `apt update && apt upgrade -y`: atualiza a lista de pacotes e instala as atualizações
- `&&` só roda o segundo comando se o primeiro der certo
- `timedatectl set-timezone America/Sao_Paulo`: fuso horário
- `hostnamectl set-hostname lab-vps`: nome da máquina
- Erro que tive: colei a senha depois de já ter entrado, e o terminal tentou rodar a senha como comando. Troquei a senha com `passwd`

### 4. Navegação e arquivos
- `pwd`: mostra em qual pasta estou
- `ls -la`: lista tudo, inclusive ocultos (nomes que começam com ponto)
  - Colunas: permissões, links, dono, grupo, tamanho, data, nome
  - `drwx------`: d = pasta; depois permissões de dono, grupo e outros (r ler, w escrever, x executar)
- `cd /etc`: entra na pasta de configurações do sistema
- `cd -`: volta para a pasta anterior; `cd ~`: vai para a pasta pessoal
- `;` roda um comando depois do outro, mesmo se o primeiro falhar
- `mkdir -p ~/lab`: cria pasta (-p não dá erro se já existir)
- `touch arquivo`: cria arquivo vazio
- `echo "ola" > a.txt`: escreve no arquivo (substitui); `>>` adiciona no final
- `cat a.txt`: mostra o conteúdo
- `cp` copia, `mv` move ou renomeia, `rm` apaga (sem lixeira!)
- `less /etc/passwd`: lê arquivo com rolagem (q sai)
- `grep root /etc/passwd`: mostra só as linhas com "root"
- `/etc/passwd`: lista de usuários. Formato: usuário:x:UID:GID:descrição:pasta:shell
- UID 0 = root
- `journalctl -u ssh`: logs do serviço ssh; `-f` acompanha ao vivo; `-n 20` últimas 20 linhas

### Incidente: apaguei o /etc/passwd
- O que houve: copiei duas linhas juntas e rodei `rm c.txt less /etc/passwd`
- Sintoma: novo SSH dava "Connection closed"
- Diagnóstico: `journalctl -u ssh` mostrou "Privilege separation user sshd does not exist"
- Correção:
  - Restaurei do backup: `cp /etc/passwd- /etc/passwd`
  - O backup era antigo e não tinha o usuário sshd; recriei com `useradd --system ... sshd`
  - Conferi com `getent passwd sshd` e `pwck -r`
- Lições:
  - Um comando por linha, ler antes do Enter, cuidado com `rm` como root
  - Nunca fechar a sessão aberta antes de testar o acesso numa nova janela
  - Logs mostram a causa real do problema

## Parei em
- Dia 1, passo 5 (usuários e permissões)

### 5. Usuários e permissões
- `adduser teste`: cria usuário (com pasta em /home/teste e senha)
- `id teste`: mostra UID, GID e grupos do usuário
  - Saída: uid=1001(teste) gid=1001(teste) groups=1001(teste),100(users)
  - teste não está no grupo sudo, então não é administrador
- `deluser --remove-home teste`: apaga o usuário e a pasta dele
- `/tmp`: pasta de arquivos temporários, todos os usuários usam, é limpa no reboot
- `su - teste -c "comando"`: roda um comando como outro usuário
  - `-` carrega o ambiente dele; `-c` roda só aquele comando e volta
  - root vira qualquer usuário sem senha
- `chmod 600 arquivo`: muda permissões
  - Conta: ler = 4, escrever = 2, executar = 1. Ordem: dono, grupo, outros
  - 600 = só o dono lê e escreve; 644 = dono escreve, todos leem
  - 700 = pasta privada; 755 = script que todos executam
- Teste antes/depois: com 644 o teste leu o arquivo; com 600 deu "Permission denied"
- Root ignora permissões, por isso o teste precisa ser feito com outro usuário

### 6. Pacotes, serviços e logs
- `apt install -y pacote`: instala (-y confirma sozinho)
- `apt purge -y pacote`: remove o programa E as configurações
  - `apt remove` remove só o programa, mantém configurações
- `apt autoremove --purge -y`: remove dependências que sobraram sem uso
- "set to manually installed": o apt marca o que eu pedi para não remover sozinho
- "2 not upgraded": atualizações do ESM (Ubuntu Pro), pode ignorar
- `htop`: monitor de CPU, memória e processos (q sai)
- `tree pasta`: mostra a pasta em árvore
- nginx: servidor web, instalei só para praticar serviços
  - Depois de instalado, já abria "Welcome to nginx" no navegador pelo IP (sem firewall ainda)
- `systemctl status nginx`: ficha do serviço
  - Loaded ... enabled = liga sozinho no boot
  - Active: active (running) = rodando; inactive/failed = parado/com erro
  - Main PID = número do processo; master process coordena, worker atende
  - status=0/SUCCESS: no Linux, 0 = sucesso
- `systemctl stop|start|restart|enable nginx`: controla o serviço
- `journalctl -u nginx -n 30`: últimas 30 linhas de log do serviço
- systemctl = controle remoto (ações); journalctl = diário (logs)
- `ss -tulpn`: mostra quais programas escutam em quais portas
  - 0.0.0.0 / [::] = aberto para qualquer lugar da internet
  - 127.0.0.x = só o próprio servidor (localhost)
  - Abertas no meu servidor: 22 (sshd) e 80 (nginx, removido depois)
  - 53 = DNS local (systemd-resolve), 68 = DHCP

### Chave pública x privada
- Pública (.pub): "cadeado", vai para o servidor no ~/.ssh/authorized_keys, pode compartilhar
- Privada: "chave", fica só no meu PC, nunca compartilhar
- No login o servidor manda um desafio, meu PC assina com a privada, o servidor confere com a pública
- A chave privada nunca viaja pela rede; senha pode ser adivinhada, chave não

### Erros que tive
- Copiar várias linhas do guia de uma vez junta tudo numa linha só
  - `groups teste deluser --remove-home teste` deu erro de opção inválida
  - Solução: um comando por vez, Enter, esperar o prompt voltar
- `systemctl -u nginx`: errado, o `-u` é do journalctl
- Ler a mensagem de erro com calma quase sempre mostra o problema

### Checklist do Dia 1
- [x] Entro na VPS sem digitar senha do servidor
- [x] Sei explicar a diferença entre chave pública e privada
- [x] Sei o que chmod 600 significa
- [x] Liguei, desliguei um serviço e li seus logs
- [x] Anotei tudo

## Parei em
- Dia 1 concluído. Próximo: Dia 2 (criar meu usuário e acessar com chave)

## Dia 2 - 10/10/2026

### Conceitos
- Não trabalhar como root: um erro como root pode destruir o servidor (vivi isso no Dia 1)
- Princípio do menor privilégio: cada um tem só o poder de que precisa
- `sudo`: executa UM comando como root, pede a MINHA senha e fica registrado em log
- Prompt: `#` = root, `$` = usuário comum

### 1. Criar meu usuário
- `adduser gabriel`: cria o usuário (a senha dele é a que o sudo pede)
- `usermod -aG sudo gabriel`: coloca no grupo sudo
  - `-G` define grupos extras; `-a` ADICIONA sem tirar dos outros
  - Sem o `-a`, o usuário perde todos os outros grupos (erro clássico)
- `id gabriel`: conferir se aparece 27(sudo)
- O /etc/sudoers do Ubuntu já libera o sudo para quem está no grupo sudo

### 2. Copiar a chave para o gabriel
- `rsync --archive --chown=gabriel:gabriel ~/.ssh /home/gabriel`
  - Copia o .ssh do root (com o authorized_keys) para a pasta do gabriel
  - `--archive` mantém permissões e datas; `--chown` troca o dono para gabriel
  - Sem o dono e as permissões certas, o SSH recusa a chave
- `ls -la /home/gabriel/.ssh`: pasta 700 (drwx------), authorized_keys 600 (-rw-------)

### 3. Testar o acesso
- Sempre testar numa NOVA janela, mantendo a do root aberta
- `ssh gabriel@IP`, depois `sudo whoami` deve pedir senha e responder root
- O sudo não transforma o usuário em root de vez: `whoami` volta a ser gabriel depois

### 4. Atalho de conexão (no Windows, não no servidor)
- Arquivo `C:\Users\admin\.ssh\config` (sem extensão)
- Conteúdo: Host lab / HostName IP / User gabriel / IdentityFile ~/.ssh/id_ed25519
- Agora basta `ssh lab`
- Pasta .ssh do Windows:
  - id_ed25519 = chave privada; id_ed25519.pub = pública
  - known_hosts = impressões digitais dos servidores em que confiei
  - known_hosts.old = backup (criado quando recriei a VPS com o mesmo IP)

### 5. Logs
- `sudo journalctl -n 20`: últimas 20 linhas de log do sistema inteiro
  - `--no-pager` mostra as linhas completas
  - audit type=1400 = AppArmor (segurança do Ubuntu), normal
  - apt-news / esm-cache / packagekit = tarefas automáticas de pacotes
  - [UFW BLOCK] = firewall bloqueando conexão vinda da internet
- `/var/log/auth.log`: logins, sudo, su, criação de usuários (precisa de sudo para ler)
  - `sudo tail -n 30 /var/log/auth.log`: últimas linhas
  - `sudo tail -f /var/log/auth.log`: ao vivo
  - `sudo grep "Accepted" ...`: logins que deram certo
  - `sudo grep "Failed password" ...` e `"Invalid user"`: tentativas dos robôs
  - `grep -c`: conta as linhas
  - Horário do auth.log em UTC (+00:00): 20:26 UTC = 17:26 em São Paulo
- Cada sudo fica registrado: quem, terminal, pasta, como quem e comando exato

### Robôs atacando
- Programas automáticos varrem todos os IPs da internet atrás de SSH aberto
- Testam usuário/senha comuns (root/123456, admin/admin...)
- Não é pessoal: a VPS foi atacada horas depois de criada
- Vêm de outros servidores já invadidos (corrente)
- Querem o servidor para: minerar cripto, botnet/DDoS, spam, phishing, atacar outros, roubar dados
- `[preauth]` = conexão encerrada antes de completar o login
- Defesa: senha forte agora; no Dia 3 desligar senha + fail2ban; no Dia 4 firewall

### Descoberta
- A imagem Ubuntu da Vultr já vem com o UFW ligado (só porta 22 liberada)
- `sudo ufw status verbose` para ver as regras
- No Dia 4 vou ajustar um firewall que já existe, não criar do zero

### Erros que tive
- `rsync ... ~/.ssh/home/gabriel`: faltou espaço entre origem e destino
  - Sem destino, o rsync só lista o que copiaria
  - Digitar só `/home/gabriel` tenta executar a pasta ("Is a directory")
- Rodei `sudo whoami` como root e não pediu senha: root não precisa; o teste é como gabriel
- Arquivo config ficou vazio (0 bytes): o Bloco de Notas não salvou
  - Resolvi gravando pelo PowerShell com `Set-Content -Encoding ascii`
- `ssh: Could not resolve hostname lab`: o SSH não achou o atalho e procurou "lab" na internet
- `systemctl` x `journalctl`: confundi as opções de novo; -u, -n e -f são do journalctl

### Checklist do Dia 2
- [x] Entro com `ssh lab` como gabriel, sem senha de login
- [x] `sudo whoami` responde root
- [x] Sei explicar por que não se trabalha como root
- [x] Vi tentativas de login estranhas no auth.log

## Parei em
- Dia 2 concluído. Próximo: Dia 3 (hardening do SSH e fail2ban)

## Dia 3 - 12/10/2026

### Conceitos
- Hardening: "endurecer" o servidor = reduzir as formas de ataque (tirar o que não usa, fechar o que não precisa, dificultar o resto)
- sshd: o programa servidor do SSH (o "d" é de daemon, programa de fundo); no Ubuntu o serviço se chama `ssh`
- cloud-init: ferramenta que configura a VPS no primeiro boot com as escolhas do painel (hostname, chave, rede, senha de root)
- 50-cloud-init.conf: arquivo que o cloud-init pode criar no sshd_config.d, às vezes religando a senha
- fail2ban: lê os logs e bane por um tempo os IPs que erram o login várias vezes
- Defesa em camadas: nenhuma proteção sozinha é perfeita, então empilho várias

### 1. Tamanho do ataque
- `sudo journalctl -u ssh --since "1 day ago" | grep -i "invalid user" | head`
  - `|` (pipe) passa a saída de um comando para o próximo
  - `grep -i` ignora maiúsculas/minúsculas; `head` mostra as 10 primeiras linhas
- Cada tentativa gera 3 linhas: Invalid user / Failed password / Disconnected [preauth]
- Mesmo número em `sshd[21397]` = mesma tentativa (PID)
- Nomes testados: comuns (marko, bryan) e de programas com senha padrão (sinusbot, graylog)
- 2 IPs revezando, cerca de 1 tentativa por minuto cada (devagar de propósito, para não chamar atenção)
- `... | wc -l` deu 13.459 linhas = cerca de 4.500 tentativas em 24h (3 por minuto)
- `grep -c` conta linhas direto, sem precisar do `wc -l`

### 2. Configuração do SSH
- `sudo nano /etc/ssh/sshd_config.d/00-hardening.conf`
- Conteúdo e o que cada linha faz:
  - `PermitRootLogin no`: ninguém entra direto como root, nem com chave
  - `PasswordAuthentication no`: desliga o login por senha
  - `KbdInteractiveAuthentication no`: fecha o caminho alternativo de digitar senha
  - `PubkeyAuthentication yes`: mantém o login por chave
  - `MaxAuthTries 3`: no máximo 3 tentativas por conexão
  - `AllowUsers gabriel`: só o gabriel entra por SSH
- Por que o nome `00-hardening.conf`:
  - O SSH lê os arquivos da pasta em ordem alfabética e o PRIMEIRO valor vence
  - `00-` vem antes de `50-cloud-init.conf`, então minhas regras prevalecem
  - `.conf` é obrigatório: o sshd_config tem `Include /etc/ssh/sshd_config.d/*.conf`
- Edito um arquivo extra em vez do sshd_config original: separado, fácil de desfazer, não é sobrescrito em atualizações

### 3. Validar e aplicar
- O arquivo sozinho não muda nada: só vale depois de reiniciar o SSH
- `sudo sshd -t`: testa a sintaxe (sem saída = sem erro). Erro aqui poderia me trancar fora
- `sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'`: mostra a configuração EFETIVA (o que realmente venceu)
- `sudo systemctl restart ssh`: aplica

### 4. Testes (sempre do Windows, numa janela nova)
- `ssh lab` → entra como gabriel com a chave
- `ssh root@IP` → Permission denied (publickey)
- `ssh -o PubkeyAuthentication=no gabriel@IP` → Permission denied, sem pedir senha
  - `-o` passa uma opção só para aquela conexão
  - Simula um robô sem a chave: se nem o gabriel tem senha, não há senha para adivinhar
- `Last login: ... from IP_DE_CASA` = IP da minha casa; IP desconhecido ali seria alerta

### 5. Porta do SSH
- Mantive a 22: trocar só reduz ruído, não é segurança de verdade
- No 24.04 o SSH usa ativação por socket; trocar exigiria daemon-reload, restart ssh.socket e liberar no firewall antes

### 6. fail2ban
- `sudo apt install -y fail2ban`
- `sudo nano /etc/fail2ban/jail.local` (nunca editar o jail.conf: é sobrescrito nas atualizações; o .local tem prioridade)
- Conteúdo:
  - `[sshd]`: a jail ("cela", regra de vigilância) do SSH
  - `enabled = true`: liga
  - `backend = systemd`: lê os logs pelo journal
  - `maxretry = 5` + `findtime = 10m` + `bantime = 1h`: 5 falhas em 10 minutos = IP banido por 1 hora
- `sudo systemctl enable --now fail2ban`: liga no boot (enable) e já inicia (--now)
- `sudo systemctl restart fail2ban`: reinicia para ler o jail.local (programas leem a config ao iniciar)
- `sudo fail2ban-client status sshd`:
  - Filter = quem lê os logs (Currently failed, Total failed)
  - Journal matches: `_SYSTEMD_UNIT=sshd.service + _COMM=sshd` (o + é "ou"; no Ubuntu é o _COMM=sshd que pega)
  - Actions = quem bane (Currently banned, Total banned, Banned IP list)
  - Tudo zerado logo depois do restart: os contadores recomeçam
- `sudo journalctl -u fail2ban -n 30`: Started/Server ready (instalação), Stopping/Shutdown (restart), Started de novo (com o jail.local)
- Com a senha desligada, o fail2ban não é o que impede a invasão: ele corta o barulho e poupa recursos

### Nano
- `^` = Ctrl, `M-` = Alt
- Ctrl+O salva (confirmar o nome com Enter), Ctrl+X sai
- Colar no terminal do Windows: botão direito do mouse
- Conferir o que ficou gravado: `cat arquivo`

### Erros que tive
- Rodei comando do servidor no PowerShell: `grep` não existe no Windows
  - Olhar o prompt: `PS C:\...>` = Windows; `gabriel@lab-vps:~$` = servidor; `#` = root
- `grep` sem o que procurar: "Usage: grep [OPTION]... PATTERNS"
- Um comando longo dividido em dois: `-i "invalid user"` executado sozinho deu "command not found"
  - O comando só termina onde EU aperto Enter
- `nano /est/ssh/...` e espaço no nome do arquivo: o nano abriu 2 arquivos ([1/2]) no caminho errado
  - Saí com Ctrl+X e N (sem salvar) em cada um
- Testei `ssh root@IP` de dentro do servidor: o teste precisa sair do Windows
  - Depois do `ssh lab`, a janela vira o servidor; `exit` para voltar ao PowerShell

### Checklist do Dia 3
- [x] Login como root é recusado
- [x] Login por senha é recusado
- [x] `fail2ban-client status sshd` mostra a jail ativa
- [x] Sei explicar por que o arquivo se chama `00-hardening.conf`
- Pendente: rodar `sudo fail2ban-client status sshd` depois de um dia para ver IPs banidos

## Dia 4 - 12/10/2026

### Conceitos
- Porta: IP é o prédio, porta é o apartamento. 22 SSH, 80 HTTP, 443 HTTPS, 5432 Postgres, 3306 MySQL
- Escutar numa porta = programa esperando conexões
- Firewall: o porteiro; decide quais conexões entram, por porta
- UFW: interface simples para o firewall do kernel (iptables/nftables)
- Incoming = conexões que chegam; outgoing = conexões que o servidor abre (apt, DNS, Docker baixando imagens)
- Routed = tráfego que passa pelo servidor de uma rede para outra (relação com o Docker no Dia 5)

### 1. Configuração do UFW
- A imagem da Vultr já vinha com o UFW ATIVO, então as mudanças valeram na hora
- `sudo ufw default deny incoming`: bloqueia tudo que chega por padrão
- `sudo ufw default allow outgoing`: libera tudo que sai
- "be sure to update your rules accordingly" = liberar o que precisa continuar funcionando
- `sudo ufw allow 22/tcp` PRIMEIRO, depois testei `ssh lab` numa janela nova
- `sudo ufw allow 80/tcp` e `sudo ufw allow 443/tcp`
  - Cada allow gera 2 regras: `Rule added` (IPv4) e `Rule added (v6)` (IPv6)
- `sudo ufw show added`: regras que EU cadastrei
- `sudo ufw enable`: liga e mantém ligado no boot
  - Pergunta "may disrupt existing ssh connections": y é seguro porque a 22 já estava liberada e testada
- `sudo ufw status verbose`: o que o firewall está aplicando de fato
  - Logging: on (low) = gera as linhas [UFW BLOCK] no journal
  - Default: deny (incoming), allow (outgoing), disabled (routed)
  - New profiles: skip = não aplica perfis de programas automaticamente
  - To / Action / From: porta, ALLOW IN, Anywhere (de qualquer IP)

### 2. Testes de fora (do Windows)
- No PowerShell, `curl` é apelido do Invoke-WebRequest (pediu "Uri:"); usar `curl.exe`
- `nc` não existe no Windows; o equivalente é `Test-NetConnection IP -Port N`
- `curl.exe -I http://IP` → Couldn't connect: porta 80 liberada, mas sem programa nela (recusa)
  - `-I` pede só os cabeçalhos da resposta
  - Demorou 2140 ms porque o Windows tenta mais 2 vezes após uma recusa
- `Test-NetConnection IP -Port 22` → TcpTestSucceeded: True (liberada e com o sshd)
- `Test-NetConnection IP -Port 3306` → TcpTestSucceeded: False (bloqueada pelo firewall)
  - PingSucceeded: True, RTT 21 ms (vantagem de usar São Paulo)
  - SourceAddress 192.168.15.97 = meu IP na rede de casa; o IP público é o do roteador
- Recusa rápida = passou pelo firewall, mas ninguém escuta; demora/timeout = firewall descartou
- Testar de dentro do servidor não vale: acessar o próprio IP nem sai para a internet
- Esperar o prompt voltar antes de digitar o próximo comando (as barras de progresso bagunçaram a tela)

### 3. Atualizações automáticas
- `sudo apt install -y unattended-upgrades`: já vinha instalado
  - Sobrou o `nginx-common` do Dia 1 → `sudo apt autoremove --purge -y`
- `sudo dpkg-reconfigure -plow unattended-upgrades`: refaz as perguntas do pacote (-plow mostra todas)
- `cat /etc/apt/apt.conf.d/20auto-upgrades`: os dois valores "1" = atualizar lista e instalar diariamente
- `sudo unattended-upgrade --dry-run --debug | tail -20`: ensaio, sem instalar nada
  - pin -32768 no noble-updates = nunca instala dali automaticamente; só noble-security entra sozinho
  - Identifica os pacotes do kernel para nunca remover o que está rodando (6.8.0-146)
  - As 2 atualizações pendentes (open-iscsi, libopeniscsiusr) vêm do noble-updates, então ficam para mim
  - "No packages found that can be upgraded unattended" = nenhuma de segurança pendente hoje
- `sudo apt upgrade -y` para instalar as comuns manualmente
- `ls /var/run/reboot-required`: "No such file" = não precisa reiniciar
- Erros: `unatted-upgrade` e `unattnded-upgrade` (digitação); o nome certo é unattended-upgrade, no singular

### 4. Comparação: o que mudou
| | Dia 1 (com nginx) | Dia 4 antes do firewall | Dia 4 depois do firewall |
| --- | --- | --- | --- |
| Programas escutando (`ss -tulpn`) | sshd :22, nginx :80, DNS local, DHCP | sshd :22, DNS local, DHCP | Igual ao "antes" (mesmo PID 38346) |
| Política de entrada | UFW da Vultr, só 22 | deny incoming, 22 liberada | deny incoming |
| Portas liberadas no firewall | 22 | 22 | 22, 80, 443 (IPv4 e IPv6) |
| Visto da internet | 22 | 22 | 22 responde; 80 passa mas recusa; 3306 bloqueada |

- O `ss` não muda com o firewall: ele mostra programas, não regras
- O firewall muda quem consegue CHEGAR até os programas
- Para ver o efeito do firewall: `ufw status` + testes de fora
- O PID do sshd mudou entre o Dia 1 (1043) e o Dia 3 (38346) porque reiniciei o SSH
- `0.0.0.0` / `[::]` = aceita de qualquer lugar; `127.x` = só o próprio servidor

### Checklist do Dia 4
- [x] `ufw status verbose` mostra deny incoming e só 22, 80 e 443 liberadas
- [x] Testei uma porta bloqueada e uma liberada de fora
- [x] O dry run do unattended-upgrades roda sem erro
- [x] Sei a diferença entre escutar em 0.0.0.0 e em 127.0.0.1

## Parei em
- Dia 4 concluído. Próximo: Dia 5 (Docker na prática)

## Dia 5 - 16/10/2026

### Conceitos
- Docker: roda aplicações empacotadas com tudo de que precisam (resolve o "na minha máquina funciona")
- Imagem: o molde/receita, somente leitura (ex.: nginx:alpine; depois dos dois pontos vem a tag = versão)
- Container: instância rodando a partir de uma imagem (o bolo feito da receita); dá para ter vários da mesma imagem
- Container x VPS: a VPS é uma máquina virtual inteira; o container compartilha o kernel da VPS e isola só a aplicação
- Docker Hub: a "loja" de imagens (docker.io/library/... = imagens oficiais)
- Camadas (layers): imagens são feitas de camadas empilhadas; camadas iguais são baixadas uma vez só
- Digest sha256: impressão digital da imagem (muda se um byte mudar)
- Arquitetura: eu digito no cliente (`docker`) → ele fala com o daemon (`dockerd`) → que usa o containerd para rodar os containers

### 1. Instalação pelo repositório oficial
- `sudo apt install -y ca-certificates curl`
  - ca-certificates = lista de autoridades certificadoras (confirma que download.docker.com é mesmo da Docker)
  - Os dois já estavam instalados
- `sudo install -m 0755 -d /etc/apt/keyrings`: cria a pasta das chaves de repositório já com permissão 755 (-d = pasta)
- `sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`: baixa a chave pública da Docker
  - -f falha em erro do servidor; -s silencioso; -S mostra erro mesmo assim; -L segue redirecionamento; -o salva no arquivo
  - A Docker assina os pacotes com a chave privada; o apt confere com a pública (mesma ideia do SSH, ao contrário)
- `sudo chmod a+r /etc/apt/keyrings/docker.asc`: todos podem LER a chave (forma simbólica)
- chmod numérico x simbólico:
  - Numérico define tudo de uma vez (600, 644, 755)
  - Simbólico mexe só no que pedi: u/g/o/a + ou - r/w/x (ex.: a+r, u+x, o-w)
- `echo "deb [arch=... signed-by=...docker.asc] https://download.docker.com/linux/ubuntu ... stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`
  - Adiciona o repositório da Docker à lista do apt
  - `$( )` = substituição de comando: roda e coloca o resultado no lugar
  - `$(dpkg --print-architecture)` vira amd64; `$(. /etc/os-release && echo "$VERSION_CODENAME")` vira noble
  - `sudo tee` em vez de `sudo echo >`: o `>` roda como gabriel e daria Permission denied
  - `> /dev/null`: "buraco negro", esconde a repetição do tee na tela
  - Conferir: `cat /etc/apt/sources.list.d/docker.list`
- `sudo apt update`: o repositório da Docker apareceu sem erro de assinatura
  - Get = baixou novidade; Hit = já estava atualizado
- `sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
  - docker-ce: o Engine (daemon dockerd); ce = Community Edition
  - docker-ce-cli: o comando `docker` (o controle remoto)
  - containerd.io: o motor de baixo nível que roda os containers
  - docker-buildx-plugin: constrói imagens a partir de Dockerfile
  - docker-compose-plugin: o `docker compose` (Dia 6)
  - Extras: pigz (descompactação rápida), docker-ce-rootless-extras (modo sem root)
  - "Created symlink ... wants/docker.service" = equivalente a systemctl enable (liga no boot)
- `sudo docker run hello-world`: baixou a imagem (não existia localmente), rodou, imprimiu "Hello from Docker!" e saiu

### 2. Docker sem sudo
- Erro: `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`
  - O cliente fala com o daemon pelo socket /var/run/docker.sock; só root e o grupo docker podem usar
- `sudo usermod -aG docker gabriel`, depois `exit` e `ssh lab` (grupos só carregam no login)
- `groups` → gabriel sudo users docker
- Atenção: grupo docker = acesso de root (um container pode montar qualquer pasta do servidor)
- Last login de outro IP (IP_DE_OUTRA_REDE): acessei de outra rede; conferir logins com `sudo grep "Accepted" /var/log/auth.log | tail`

### 3. Ciclo de vida de um container
- `docker pull nginx:alpine`: baixa a imagem sem rodar
  - Cada código "Pull complete" é uma camada
  - Nome completo: docker.io/library/nginx:alpine
- `docker images`: lista imagens
  - DISK USAGE = tamanho descompactado no disco (103 MB); CONTENT SIZE = compactado, o que baixou (29,7 MB)
  - U (In Use) = existe container criado a partir dela (mesmo parado)
- `docker run -d --name web -p 127.0.0.1:8080:80 nginx:alpine`
  - -d = segundo plano; --name = nome (sem ele, nome aleatório tipo brave_nightingale)
  - -p porta_da_vps:porta_do_container; com 127.0.0.1 só a própria VPS acessa
  - A saída é o ID completo do container
- `docker ps`: containers rodando (ID, IMAGE, COMMAND, CREATED, STATUS, PORTS, NAMES)
- `curl localhost:8080`: veio o HTML do "Welcome to nginx!" (sem -I vem a página inteira)
- `docker logs web`: tudo que o container imprimiu
  - docker-entrypoint.sh = scripts de preparação da imagem antes do nginx
  - O kernel mostrado é o da VPS (6.8.0-146): container compartilha o kernel
  - Linha de acesso: `172.17.0.1 - - [...] "GET / HTTP/1.1" 200 896 "-" "curl/8.5.0"`
    - 172.17.0.1 = a VPS vista de dentro da rede do Docker
    - GET / = pediu a página inicial; 200 = OK; 896 = bytes; curl = quem pediu
- `docker exec -it web sh`: abre um shell dentro do container
  - -i interativo, -t terminal; Alpine não tem bash, só sh
  - Prompt `/ #` = raiz do container, root DO CONTAINER (engaiolado)
  - `ls` mostra um Linux completo, mas é OUTRO sistema de arquivos (veio das camadas)
  - `exit` sai do shell; o container continua rodando
- `docker stop web`: para; `docker ps -a`: lista todos, inclusive parados
  - Exited (0) = terminou sem erro; parado não ocupa porta
  - Parar NÃO apaga: dá para religar com `docker start web`
- `docker rm web`: remove o container (a imagem continua: o bolo some, a receita fica)
- `docker rm -f web`: força (para e remove de uma vez)

### 4. Volumes
- Tudo gravado dentro do container some no `docker rm`; volumes guardam dados fora
- `mkdir -p ~/site && echo "<h1>Ola da VPS</h1>" > ~/site/index.html`
  - index.html = página entregue por padrão quando pedem "/"
- `docker run -d --name web -p 127.0.0.1:8080:80 -v ~/site:/usr/share/nginx/html:ro nginx:alpine`
  - -v pasta_da_vps:pasta_do_container:ro
  - /usr/share/nginx/html = de onde o nginx tira as páginas
  - ro = somente leitura (se o nginx for invadido, não altera meus arquivos)
  - É a MESMA pasta vista de dois lugares, não uma cópia; a página original some por baixo
- `curl localhost:8080` → `<h1>Ola da VPS</h1>`
- Mudei o arquivo na VPS e o curl mostrou `<h1>Mudou</h1>` na hora, sem reiniciar o container
- Depois do `docker rm -f web`, o `~/site/index.html` continuou existindo
- `docker volume create dados` / `docker volume ls` (DRIVER local = disco da VPS)
- `docker volume inspect dados`: Mountpoint em /var/lib/docker/volumes/dados/_data

| | Bind mount | Volume nomeado |
| --- | --- | --- |
| Como escreve | Caminho: `~/site:...` | Nome: `dados:...` |
| Onde fica | Pasta que eu escolhi | Pasta interna do Docker |
| Conteúdo inicial | Minha pasta esconde o que a imagem tinha | Se vazio, o Docker copia o que a imagem tinha |
| Bom para | Arquivos que EU edito (site, Caddyfile) | Dados que o PROGRAMA gerencia (banco de dados) |

### 5. A pegadinha Docker x UFW
- UFW = "firewall descomplicado": camada simples por cima do firewall do kernel (iptables/nftables)
- O UFW é só um dos "escritores" de regras; o Docker também escreve as dele, e elas são avaliadas ANTES
- Analogia: o UFW é o porteiro da entrada principal; o Docker abre uma entrada lateral que não passa pelo porteiro
- Experimento:
  - `sudo ufw status`: 8080 NÃO está liberada
  - `docker run -d --name exposto -p 8080:80 nginx:alpine` (sem IP = 0.0.0.0, aberto ao mundo)
  - Do Windows: `curl.exe -I http://IP:8080` → HTTP/1.1 200 OK: a internet acessou uma porta que o UFW diz estar fechada
  - `docker rm -f exposto`
  - `docker run -d --name seguro -p 127.0.0.1:8080:80 nginx:alpine`
  - Do Windows: `curl.exe -I --max-time 5 http://IP:8080` → falha (--max-time = desiste em 5 s)
- Regra prática: publicar containers em 127.0.0.1 e expor só o proxy reverso nas portas 80 e 443
- Cabeçalhos HTTP (curl -I):
  - 200 OK = sucesso; Server = programa e versão; Content-Type = tipo; Content-Length = tamanho
  - Last-Modified = última modificação; ETag = etiqueta da versão (cache do navegador)

### Erros que tive
- `sudo curl -fsSL` cortado: "no URL specified" (faltou o resto da linha)
- `docker run ... -p 127.0.0.1:8080:80` sem imagem: "requires at least 1 argument" (forma: docker run [OPÇÕES] IMAGEM)
- Primeiro teste da pegadinha feito de dentro do servidor: não prova nada, o tráfego não sai para a internet
  - Refiz do Windows com curl.exe

### Checklist do Dia 5
- [x] `docker run hello-world` funciona sem sudo
- [x] Sei usar run, ps, logs, exec, stop e rm sem consultar
- [x] Reproduzi o Docker furando o UFW e a correção com 127.0.0.1
- [x] Sei explicar a diferença entre bind mount e volume nomeado

## Parei em
- Dia 5 concluído. Próximo: Dia 6 (Docker Compose e projeto final)
