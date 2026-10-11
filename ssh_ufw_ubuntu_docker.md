# Configuração de acesso remoto SSH e Firewall UFW no Ubuntu 24.04 com Docker

## 1. Objetivo

Este documento apresenta os passos para criar dois contêineres Ubuntu 24.04, instalar e configurar o serviço SSH, permitir a comunicação remota entre os contêineres e configurar regras básicas de firewall com UFW.

A atividade reúne conceitos estudados em Gerência e Configuração de Sistemas para Internet (GCSI): Docker, APT, usuários e permissões, serviços (`service` e `systemctl`), portas, SSH, UFW, `curl` e `wget`.

## 2. Conceitos essenciais

- **SSH (Secure Shell):** protocolo que permite acessar e administrar outro sistema remotamente por uma conexão criptografada. O serviço de servidor é fornecido pelo pacote `openssh-server`.
- **UFW (Uncomplicated Firewall):** ferramenta que simplifica a criação de regras de firewall no Linux.
- **Contêiner Docker:** ambiente isolado que utiliza o kernel do sistema hospedeiro. Não equivale a uma máquina virtual completa.
- **Porta TCP 22:** porta padrão do SSH.
- **Porta TCP 80:** porta padrão do HTTP, usada pelo NGINX no exemplo complementar.
- **`apt`:** gerenciador de pacotes usado no Ubuntu para instalar e atualizar programas.
- **`service` e `systemctl`:** ferramentas de controle de serviços. `systemctl` depende de um sistema inicializado com systemd; em contêineres simples, ele frequentemente não funciona. Nesses casos, pode-se usar `service` quando o pacote disponibiliza um script compatível ou iniciar o processo diretamente.

> **Importante sobre o UFW no Docker:** o UFW dentro de um contêiner não substitui o firewall do computador hospedeiro nem controla automaticamente todas as regras de publicação de portas do Docker. Para experimentar o UFW dentro do contêiner, será adicionada a capacidade `NET_ADMIN`. Mesmo assim, as regras afetam o ambiente de rede do contêiner, e o comportamento da publicação de portas deve ser testado. Em um servidor real, também é necessário proteger o sistema hospedeiro.

## 3. Requisitos

- Linux com Docker instalado e funcionando.
- Acesso ao terminal do computador hospedeiro.
- Conexão com a internet para baixar a imagem e os pacotes.
- Permissão para executar comandos Docker, normalmente usando `sudo` se o usuário não pertencer ao grupo `docker`.

Verifique o Docker:

```bash
docker --version
docker info
```

Se o usuário não tiver permissão para acessar o daemon, execute os comandos Docker com `sudo`, por exemplo `sudo docker ps`.

## 4. Criar uma rede Docker para os dois contêineres

Uma rede bridge definida pelo usuário permite que os contêineres se encontrem pelo nome. Isso é mais prático do que depender de endereços IP dinâmicos.

```bash
docker network create gcsi-rede
```

Confira a rede:

```bash
docker network ls
docker network inspect gcsi-rede
```

## 5. Baixar a imagem Ubuntu 24.04

```bash
docker pull ubuntu:24.04
```

A imagem é o modelo utilizado para criar os contêineres. Os dois contêineres serão instâncias separadas da mesma imagem.

## 6. Criar e iniciar os dois contêineres

O comando `tail -f /dev/null` mantém um processo ativo para que possamos entrar no contêiner e realizar a configuração manualmente. Ele não inicia automaticamente o SSH.

Crie o primeiro contêiner, chamado `gcsi-ssh-1`:

```bash
docker run -d \
  --name gcsi-ssh-1 \
  --hostname gcsi-ssh-1 \
  --network gcsi-rede \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  -p 127.0.0.1:2221:22 \
  ubuntu:24.04 \
  bash -c "tail -f /dev/null"
```

Crie o segundo contêiner, chamado `gcsi-ssh-2`:

```bash
docker run -d \
  --name gcsi-ssh-2 \
  --hostname gcsi-ssh-2 \
  --network gcsi-rede \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  -p 127.0.0.1:2222:22 \
  ubuntu:24.04 \
  bash -c "tail -f /dev/null"
```

As opções importantes são:

- `-d`: executa o contêiner em segundo plano.
- `--name`: define o nome do contêiner.
- `--hostname`: define o nome de máquina visto dentro dele.
- `--network gcsi-rede`: conecta o contêiner à rede criada.
- `--cap-add=NET_ADMIN`: concede capacidade de administração de rede necessária para experimentar regras de firewall no contêiner.
- `--cap-add=NET_RAW`: permite operações de rede que podem ser necessárias a algumas ferramentas.
- `-p 127.0.0.1:2221:22`: publica a porta 22 do primeiro contêiner na porta 2221 do hospedeiro, apenas no endereço local.
- `-p 127.0.0.1:2222:22`: publica a porta 22 do segundo contêiner na porta 2222 do hospedeiro.

A publicação das portas serve para testar o SSH a partir do hospedeiro. Entre si, os contêineres podem usar os nomes `gcsi-ssh-1` e `gcsi-ssh-2` na rede Docker.

Verifique se ambos estão ativos:

```bash
docker ps
```

Se precisar consultar também os contêineres parados:

```bash
docker ps -a
```

## 7. Preparar o primeiro contêiner

Entre no primeiro contêiner:

```bash
docker exec -it gcsi-ssh-1 bash
```

Atualize a lista de pacotes:

```bash
apt update
```

Instale os pacotes necessários:

```bash
apt install -y openssh-server ufw curl wget iproute2
```

- `openssh-server`: fornece o servidor SSH (`sshd`).
- `ufw`: ferramenta de firewall.
- `curl`: transfere dados por URLs e é útil para testar serviços HTTP.
- `wget`: baixa arquivos por protocolos como HTTP e HTTPS.
- `iproute2`: fornece ferramentas de rede, incluindo o comando `ip`.

Confira a versão do Ubuntu:

```bash
cat /etc/os-release
```

Confira o endereço e as interfaces de rede:

```bash
ip address
```

## 8. Criar um usuário para acesso remoto

É preferível testar o acesso SSH com um usuário comum, em vez de permitir login remoto como `root`.

Crie o usuário `gcsi`:

```bash
useradd -m -s /bin/bash gcsi
passwd gcsi
```

O comando `passwd` solicitará a definição e a confirmação de uma senha. Ao digitar, os caracteres não aparecem na tela.

Verifique o usuário:

```bash
id gcsi
```

O usuário comum não precisa pertencer ao grupo `sudo` apenas para entrar por SSH. Se a atividade também exigir executar comandos administrativos com `sudo`, instale o pacote e adicione o usuário ao grupo:

```bash
apt install -y sudo
usermod -aG sudo gcsi
```

Para verificar os grupos:

```bash
groups gcsi
```

> Em uma configuração real, conceda privilégios administrativos somente quando forem necessários. Neste laboratório, os comandos de instalação são executados como `root` dentro do contêiner para simplificar a preparação inicial.

## 9. Configurar o servidor SSH no primeiro contêiner

O arquivo principal de configuração é:

```text
/etc/ssh/sshd_config
```

Faça uma cópia de segurança antes de editar:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Abra o arquivo com o Nano:

```bash
nano /etc/ssh/sshd_config
```

Verifique ou ajuste as diretivas abaixo, removendo o `#` inicial se necessário:

```text
Port 22
PermitRootLogin no
PasswordAuthentication yes
```

Significado:

- `Port 22`: define a porta de escuta do SSH.
- `PermitRootLogin no`: impede o login remoto direto como `root`.
- `PasswordAuthentication yes`: habilita autenticação por senha para este laboratório.

Em produção, prefira autenticação por chave pública e desabilite a autenticação por senha depois de confirmar que as chaves funcionam. Não exponha este laboratório à internet sem medidas adicionais de segurança.

No Nano:
- `Ctrl + O`: salvar.
- `Enter`: confirmar o nome.
- `Ctrl + X`: sair.

Valide a configuração:

```bash
sshd -t
```

Se o comando não apresentar erros, a sintaxe foi aceita.

Crie o diretório de execução, se necessário, e inicie o servidor SSH:

```bash
mkdir -p /run/sshd
service ssh start
```

Em uma imagem Ubuntu minimalista, `systemctl start ssh` pode falhar porque o contêiner não foi iniciado com systemd como processo principal. Por isso, neste laboratório, usamos `service ssh start`. Se esse comando não funcionar, inicie o daemon diretamente:

```bash
/usr/sbin/sshd
```

Verifique se o processo está em execução:

```bash
ps aux | grep '[s]shd'
```

Verifique se a porta está em escuta:

```bash
ss -lntp
```

## 10. Configurar o UFW no primeiro contêiner

Ainda dentro de `gcsi-ssh-1`, defina políticas padrão:

```bash
ufw default deny incoming
ufw default allow outgoing
```

Permita SSH antes de ativar o firewall:

```bash
ufw allow 22/tcp
```

Se o serviço SSH estiver configurado em outra porta, a regra deve usar essa porta.

Ative o UFW:

```bash
ufw enable
```

Confirme o estado e as regras:

```bash
ufw status verbose
ufw status numbered
```

O resultado esperado deve indicar que o firewall está ativo e que a porta TCP 22 está permitida.

> **Atenção:** se o UFW apresentar erro de permissões, `iptables` ou Netfilter, confirme se o contêiner foi criado com `--cap-add=NET_ADMIN` e `--cap-add=NET_RAW`. O suporte pode variar conforme o ambiente Docker e o backend de firewall. Não use `--privileged` indiscriminadamente como solução automática. O firewall do hospedeiro deve continuar sendo tratado separadamente.

Saia do contêiner:

```bash
exit
```

## 11. Preparar o segundo contêiner

Entre no segundo contêiner:

```bash
docker exec -it gcsi-ssh-2 bash
```

Atualize e instale os mesmos pacotes:

```bash
apt update
apt install -y openssh-server ufw curl wget iproute2 sudo
```

Crie o usuário:

```bash
useradd -m -s /bin/bash gcsi
passwd gcsi
usermod -aG sudo gcsi
```

Configure o SSH:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
nano /etc/ssh/sshd_config
```

Garanta que as diretivas necessárias estejam definidas:

```text
Port 22
PermitRootLogin no
PasswordAuthentication yes
```

Valide e inicie o serviço:

```bash
sshd -t
mkdir -p /run/sshd
service ssh start
```

Se necessário, use a alternativa:

```bash
/usr/sbin/sshd
```

Configure o UFW:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw enable
ufw status verbose
```

Saia do contêiner:

```bash
exit
```

## 12. Testar a comunicação entre os contêineres

Na rede definida pelo usuário, o nome do contêiner pode ser usado como nome de destino.

Entre no primeiro contêiner:

```bash
docker exec -it gcsi-ssh-1 bash
```

Teste a resolução do nome do segundo contêiner:

```bash
getent hosts gcsi-ssh-2
```

Teste a porta SSH do segundo contêiner. Como `nc` não foi instalado, pode-se instalar o pacote `netcat-openbsd`:

```bash
apt update
apt install -y netcat-openbsd
nc -vz gcsi-ssh-2 22
```

Uma conexão bem-sucedida indica que a porta TCP 22 está acessível pela rede Docker. Se falhar, confira se o SSH está iniciado no segundo contêiner e se as regras de firewall permitem a conexão.

## 13. Acessar o segundo contêiner por SSH

Ainda no primeiro contêiner, execute:

```bash
ssh gcsi@gcsi-ssh-2
```

Na primeira conexão, o SSH pode perguntar se você confia na chave do servidor. Em um laboratório recém-criado, confira que o destino é o contêiner esperado e responda `yes` para registrar a chave. Em seguida, informe a senha definida com `passwd gcsi`.

Depois de entrar, verifique a identidade e o nome da máquina:

```bash
whoami
hostname
pwd
```

O esperado é que `whoami` mostre `gcsi` e `hostname` mostre `gcsi-ssh-2`.

Para encerrar a sessão SSH:

```bash
exit
```

## 14. Testar SSH a partir do computador hospedeiro

Como as portas foram publicadas apenas no endereço local do hospedeiro, execute no terminal do hospedeiro (fora dos contêineres):

```bash
ssh -p 2221 gcsi@127.0.0.1
```

Esse comando acessa `gcsi-ssh-1`.

Para acessar o segundo:

```bash
ssh -p 2222 gcsi@127.0.0.1
```

O parâmetro `-p` especifica a porta do servidor SSH no destino. A porta externa `2221` é mapeada para a porta `22` do primeiro contêiner; `2222` é mapeada para a porta `22` do segundo.

## 15. Uso de `curl` e `wget`

`curl` e `wget` são ferramentas de transferência de dados. Não substituem o SSH e não são necessários para estabelecer uma sessão de terminal SSH, mas podem ajudar a testar serviços HTTP/HTTPS.

Exemplos de consulta HTTP:

```bash
curl -I https://ubuntu.com
```

```bash
wget --spider https://ubuntu.com
```

- `curl -I` solicita os cabeçalhos HTTP.
- `wget --spider` verifica o recurso sem baixar seu conteúdo como um arquivo normal.

Se for instalado NGINX no primeiro contêiner, ele pode servir como teste HTTP adicional. Essa parte é opcional, pois o objetivo principal da atividade é SSH e UFW.

## 16. `service` e `systemctl`

Em uma instalação Ubuntu convencional com systemd ativo:

```bash
systemctl status ssh
systemctl start ssh
systemctl stop ssh
systemctl restart ssh
systemctl enable ssh
```

Os comandos significam:

- `status`: consulta o estado do serviço.
- `start`: inicia o serviço.
- `stop`: interrompe o serviço.
- `restart`: reinicia o serviço.
- `enable`: configura o serviço para iniciar automaticamente na inicialização gerenciada pelo systemd.

Em contêineres simples, `systemctl` pode retornar uma mensagem semelhante a “System has not been booted with systemd as init system”. Isso ocorre porque o contêiner normalmente executa um processo principal específico, e não um sistema de inicialização completo.

O comando `service ssh start` pode funcionar quando existe um script de serviço compatível. Outra opção é executar `sshd` diretamente. Em um contêiner de laboratório, iniciar manualmente um serviço não garante que ele volte a iniciar após a recriação do contêiner.

## 17. Comandos de diagnóstico

### Docker

```bash
docker ps
docker ps -a
docker logs gcsi-ssh-1
docker logs gcsi-ssh-2
docker inspect gcsi-ssh-1
docker network inspect gcsi-rede
```

### SSH, dentro do contêiner

```bash
sshd -t
ps aux | grep '[s]shd'
ss -lntp
```

### Firewall, dentro do contêiner

```bash
ufw status verbose
ufw status numbered
```

### Rede, dentro do contêiner

```bash
hostname
ip address
ip route
getent hosts gcsi-ssh-2
```

## 18. Problemas comuns e soluções

### `ssh: connect to host ... port 22: Connection refused`

Possíveis causas:
- O serviço SSH não está iniciado.
- O servidor não está escutando na porta 22.
- O UFW está bloqueando a conexão.

Verifique:

```bash
ps aux | grep '[s]shd'
ss -lntp
ufw status verbose
```

### `Permission denied`

Possíveis causas:
- Nome de usuário incorreto.
- Senha incorreta.
- Autenticação por senha desabilitada.
- Conta sem senha válida.

Redefina a senha como `root` dentro do contêiner:

```bash
passwd gcsi
```

Confira a diretiva `PasswordAuthentication` em `/etc/ssh/sshd_config` e valide a configuração com `sshd -t`.

### `systemctl` não funciona

É esperado em um contêiner que não foi iniciado com systemd. Tente:

```bash
service ssh start
```

ou:

```bash
mkdir -p /run/sshd
/usr/sbin/sshd
```

### O UFW não consegue configurar as regras

Confira as capacidades adicionadas ao criar o contêiner e leia a mensagem de erro. O UFW usa mecanismos de filtragem de pacotes do kernel; o contêiner precisa de permissões apropriadas para manipular suas regras de rede. A configuração do firewall do hospedeiro continua sendo uma tarefa separada.

### O nome do outro contêiner não é resolvido

Confira se ambos estão conectados à mesma rede definida pelo usuário:

```bash
docker network inspect gcsi-rede
```

E teste:

```bash
getent hosts gcsi-ssh-2
```

## 19. Encerramento do laboratório

Para parar os contêineres:

```bash
docker stop gcsi-ssh-1 gcsi-ssh-2
```

Para iniciá-los novamente:

```bash
docker start gcsi-ssh-1 gcsi-ssh-2
```

Para removê-los definitivamente, depois de terminar o laboratório:

```bash
docker rm -f gcsi-ssh-1 gcsi-ssh-2
```

Para remover a rede, depois que os contêineres tiverem sido removidos:

```bash
docker network rm gcsi-rede
```

A remoção dos contêineres apaga as alterações feitas apenas no sistema de arquivos interno deles, salvo dados armazenados em volumes ou fora do contêiner. Se quiser manter o laboratório, não execute os comandos de remoção.

## 20. Resumo dos comandos essenciais

| Finalidade | Comando |
|---|---|
| Baixar Ubuntu 24.04 | `docker pull ubuntu:24.04` |
| Criar rede | `docker network create gcsi-rede` |
| Criar contêiner | `docker run -d ... ubuntu:24.04 bash -c "tail -f /dev/null"` |
| Entrar no contêiner | `docker exec -it gcsi-ssh-1 bash` |
| Atualizar pacotes | `apt update` |
| Instalar SSH e UFW | `apt install -y openssh-server ufw` |
| Criar usuário | `useradd -m -s /bin/bash gcsi` |
| Definir senha | `passwd gcsi` |
| Validar SSH | `sshd -t` |
| Iniciar SSH | `service ssh start` |
| Permitir SSH no UFW | `ufw allow 22/tcp` |
| Ativar UFW | `ufw enable` |
| Consultar firewall | `ufw status verbose` |
| Acessar remotamente | `ssh gcsi@gcsi-ssh-2` |
| Testar porta TCP | `nc -vz gcsi-ssh-2 22` |
| Consultar estado Docker | `docker ps -a` |

## 21. Referências oficiais

- Ubuntu Server — Firewall: https://documentation.ubuntu.com/server/how-to/security/firewalls/
- Ubuntu Manpage — UFW: https://manpages.ubuntu.com/manpages/noble/man8/ufw.8.html
- Docker Docs — Bridge network driver: https://docs.docker.com/engine/network/drivers/bridge/
- Docker Docs — Port publishing and mapping: https://docs.docker.com/engine/network/port-publishing/
- OpenSSH — Manual do servidor `sshd`: https://man.openbsd.org/sshd
- Ubuntu Server — OpenSSH: https://documentation.ubuntu.com/server/how-to/security/openssh-server/

---
