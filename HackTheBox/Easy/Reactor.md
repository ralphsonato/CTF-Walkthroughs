# Write-up: Reactor - Hack The Box

> Data: 24 de Maio de 2026  
> Target IP: `10.129.3.9`

![STATUS](https://img.shields.io/badge/STATUS-COMPROMETIMENTO%20TOTAL-brightgreen?style=flat-square) ![SEASON](https://img.shields.io/badge/SEASON-11%20Week%201-purple?style=flat-square) ![CVE](https://img.shields.io/badge/CVE-2025--55182-red?style=flat-square) ![TECHNIQUE](https://img.shields.io/badge/TECHNIQUE-React2Shell%20RCE-orange?style=flat-square) ![PRIVESC](https://img.shields.io/badge/PRIVESC-Node.js%20Inspector-blue?style=flat-square)

---

> ## ⚠️ MÁQUINA ATIVA — CONTEÚDO PARCIALMENTE OCULTADO
>
> **Reactor** faz parte da **Season 11 (May–Aug 2026)** do HackTheBox e ainda está ativa.
>
> Em respeito às regras da plataforma e à comunidade, os detalhes técnicos de exploração, payloads e flags estão ocultados neste write-up público.
>
> O write-up completo será publicado quando a máquina for retirada.
>
> **Precisa de uma dica ou quer trocar ideia sobre a máquina?**
> Me chama no LinkedIn 👇
>
> [![LinkedIn](https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ralph-sonato-77086b16b/)

---

## Visão Geral

**Reactor** é uma máquina Linux de dificuldade Easy da Season 11 do HackTheBox. Envolve uma aplicação web em Next.js com uma vulnerabilidade crítica recente, acesso a banco de dados interno e escalada de privilégios via serviço de debug Node.js.

**Stack tecnológico identificado:**
- Sistema Operacional: Linux (Ubuntu)
- Porta 22: SSH (OpenSSH 9.6p1)
- Porta 3000: Aplicação Next.js

---

## 🗺️ Cadeia de Ataque (Resumo)

```
Reconhecimento → Identificação da versão Next.js
         ↓
Vulnerabilidade crítica no framework → RCE não autenticado
         ↓
Acesso inicial como usuário de baixo privilégio
         ↓
Enumeração interna → banco de dados com credenciais
         ↓
Crack de hash → acesso SSH como usuário do sistema
         ↓
Processo privilegiado com serviço de debug exposto
         ↓
Tunnel SSH + conexão ao debugger → RCE como root 🏆
```

---

## 🔍 Fase 1 - Reconhecimento

```bash
nmap -sV -sC -T4 -Pn 10.129.3.9
```

Duas portas abertas: **22 (SSH)** e **3000 (HTTP)**. A aplicação web é um dashboard chamado **ReactorWatch** - sistema de monitoramento de reator nuclear com métricas e lista de pessoal on-site.

A versão do framework foi identificada nos headers HTTP e nos chunks JavaScript da aplicação.

---

## 🔍 Fase 2 - Enumeração da Aplicação Web

> 🔒 *Detalhes de enumeração ocultados - máquina ativa.*

A enumeração da aplicação Next.js foi a parte mais desafiadora. O buildManifest revelou que a aplicação possui apenas duas páginas (`/_app` e `/_error`) - o dashboard estático era a única interface visível.

A versão exata do framework identificada levou diretamente ao vetor de exploração.

---

## 💥 Fase 3 - Acesso Inicial

> 🔒 *Detalhes do CVE e exploit ocultados - máquina ativa.*

A versão do Next.js em execução é vulnerável a um CVE crítico (CVSS 10.0) que permite **execução remota de código não autenticada** através de uma requisição POST maliciosa. Ferramentas públicas de PoC estão disponíveis no GitHub.

```
Shell obtida como: [OCULTO] @ reactor
```

---

## 🔑 Fase 4 - Movimento Lateral

> 🔒 *Conteúdo do banco de dados e hash ocultados - máquina ativa.*

Com acesso ao sistema, foram encontrados arquivos de configuração e um banco de dados SQLite contendo credenciais de usuários. O hash obtido foi quebrado com rockyou.txt.

```bash
# Ferramenta utilizada
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5
```

```
Credenciais SSH: [OCULTO]:[OCULTO]
```

---

## 🚪 Fase 5 - Acesso SSH

> 🔒 *Credenciais ocultadas - máquina ativa.*

```bash
ssh [OCULTO]@10.129.3.9
```

User flag obtida em `~/user.txt`.

---

## 👑 Fase 6 - Escalada de Privilégios

> 🔒 *Detalhes do processo e comandos ocultados - máquina ativa.*

A enumeração de processos revelou um serviço Node.js rodando como **root** com o flag `--inspect` exposto em localhost. Um tunnel SSH foi criado para acessar o debugger remotamente e executar código arbitrário no contexto do processo root.

```bash
# Tunnel SSH
ssh -L 9229:127.0.0.1:9229 [OCULTO]@10.129.3.9

# Conexão ao debugger
node inspect 127.0.0.1:9229
```

```
Root obtido: uid=0(root) gid=0(root)
Root flag: [OCULTO]
```

---

## 📚 Aprendizados

### 1. Não precisa conhecer a tecnologia para explorar
Nunca tinha mexido com Next.js antes dessa máquina. A parte mais frustrante foi tentar entender a estrutura da aplicação sem conhecer o framework. Mas o que importou foi identificar a versão e associar ao CVE correto.

### 2. CVE-2025-55182 - uma das vulnerabilidades mais críticas de 2025
CVSS 10.0. Afeta aplicações Next.js em configuração padrão, sem autenticação. Explorada ativamente em ambiente real por grupos de ameaça avançados poucas horas após a divulgação pública.

### 3. Node.js `--inspect` é crítico se mal configurado
O flag `--inspect` abre um servidor de debug que executa JavaScript arbitrário no contexto do processo. Quando esse processo roda como root, um usuário local pode fazer tunnel e escalar privilégios trivialmente.

### 4. Hashes MD5 sem salt são inúteis como proteção de senha
Quebrados em segundos com rockyou.txt. MD5 puro não é hashing de senha.

---

## 🛠️ Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| `nmap` | Reconhecimento de portas e serviços |
| PoC CVE-2025-55182 | Exploração do framework |
| `nc` | Listener para reverse shell |
| `sqlite3` | Dump do banco de dados |
| `john` | Crack de hashes MD5 |
| `ssh -L` | Tunnel para o Node.js Inspector |
| `node inspect` | Execução de código no processo root |

---

## 🔗 Referências

- [CVE-2025-55182 - React2Shell (Wiz Research)](https://www.wiz.io/blog/critical-vulnerability-in-react-cve-2025-55182)
- [Node.js Inspector Privilege Escalation - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#node-inspector-debugger)

---

<div align="center">
  <p>Escrito por:</p>
  <a href="https://www.linkedin.com/in/ralph-sonato-77086b16b/"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
  <br>
  <sub>Write-up completo será publicado após a máquina ser aposentada.</sub>
</div>
