# Write-up: CTF Agent Sudo (TryHackMe)
> **Data:** 02 de Março de 2026  
> **Target IP:** `10.67.136.239`

![Status](https://img.shields.io/badge/Status-Comprometimento%20Total-red?style=for-the-badge) 
![PrivEsc](https://img.shields.io/badge/PrivEsc-CVE--2019--14287-blue?style=for-the-badge)

---

## 1. Resumo Executivo
O servidor **Agent Sudo** foi totalmente comprometido após uma cadeia de explorações que envolveu bypass de controles web, análise de esteganografia e uma vulnerabilidade crítica de configuração no binário Sudo. A invasão resultou em privilégios de **Root** e exfiltração de todas as flags do sistema.

---

## 2. Fase de Reconhecimento (Recon)
O escaneamento inicial com **Nmap** identificou três serviços críticos expostos:

> `nmap -sV -sC -Pn 10.67.136.239`

| Porta | Serviço | Versão |
| :--- | :--- | :--- |
| 21 | FTP | vsFTPd 3.0.3 |
| 22 | SSH | OpenSSH 7.6p1 |
| 80 | HTTP | Apache 2.4.29 |

---

## 3. Enumeração Web & Bypass de User-Agent
A aplicação web apresentava um bloqueio de acesso baseado no cabeçalho `User-Agent`. Através de uma automação em Bash para brute-force de "codenomes" (letras do alfabeto), identifiquei a falha:

* **Vulnerabilidade:** Controle de acesso baseado em cabeçalhos manipuláveis pelo cliente.
* **Descoberta:** O User-Agent **"C"** redirecionou para uma página contendo uma mensagem do "Agente R" para o usuário **chris**, informando que sua senha era fraca.

---

## 4. Exploração e Acesso Inicial

### 4.1. Ataque de Força Bruta (FTP)
Utilizando o usuário identificado (**chris**) e a wordlist `rockyou.txt`, executei um ataque de dicionário via **Hydra**:

> `hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.67.136.239`

* **Resultado:** Credenciais encontradas -> `chris:crystal`

### 4.2. Exfiltração de Dados
Ao acessar o FTP, foram recuperados três arquivos fundamentais:
* `To_agentJ.txt`: Dica de contexto.
* `cutie.png`: Imagem com dados binários ocultos.
* `cute-alien.jpg`: Imagem para análise esteganográfica.

---

## 5. Esteganografia & Cracking de Arquivos

### 5.1. Extração de Arquivo ZIP Oculto
A imagem `cutie.png` continha um arquivo ZIP protegido. Realizei a extração manual via `dd`:

> `dd if=cutie.png of=secret.zip bs=1 skip=34562`

### 5.2. Quebra do ZIP e Base64
* **ZIP Cracking:** Utilizei `zip2john` e `John the Ripper` para obter a senha do arquivo: `alien`.
* **Base64 Decode:** O arquivo `To_agentR.txt` (dentro do ZIP) continha a string `QXJlYTUx`.
  * `echo "QXJlYTUx" | base64 -d` -> **Area51**

### 5.3. Steghide (Acesso SSH)
Utilizando a senha extraída, recuperei as credenciais SSH da imagem `cute-alien.jpg`:

> `steghide extract -sf cute-alien.jpg -p Area51`

* **Resultado:** Credenciais SSH -> `james:hackerrules!`

---

## 6. Escalação de Privilégios (PrivEsc)
A enumeração de privilégios locais revelou uma vulnerabilidade de configuração no arquivo `sudoers`.

* **Vulnerabilidade:** CVE-2019-14287 (Sudo Security Bypass).
* **Evidência:** O comando `sudo -l` retornou `(ALL, !root) /bin/bash`.
* **Exploração:** O binário Sudo nas versões afetadas permite o bypass do usuário root ao referenciar o UID `-1`.

> `sudo -u#-1 /bin/bash`

**Resultado:** Shell interativo como **root** estabelecido. 🚩

---

## 7. Provas de Comprometimento (Flags)
* **Incidente:** Roswell Alien Autopsy.
* **User Flag:** `/home/james/user_flag.txt`
* **Root Flag:** `/root/root.txt`

---

**Escrito por:** [Ralph Sonato](https://github.com/ralphsonato)  
*Analista de Cibersegurança*
