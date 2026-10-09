# Notas - VPS do zero

## Dia 1 - 08/10/2026

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
