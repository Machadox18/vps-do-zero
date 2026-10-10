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
