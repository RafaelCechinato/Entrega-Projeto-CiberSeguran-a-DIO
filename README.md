# Laboratório de Testes de Segurança — Kali Linux + Metasploitable2

Projeto prático de pentest em ambiente controlado e isolado, utilizando **Kali Linux** como máquina atacante e **Metasploitable2** como alvo intencionalmente vulnerável, ambos em máquinas virtuais no VirtualBox.

O objetivo é demonstrar, de forma didática, técnicas de força bruta, automação de tentativas de login em formulário web e enumeração de usuários/password spraying, comparando duas ferramentas equivalentes (**Hydra** e **Medusa**) em cada cenário.

> ⚠️ **Aviso legal:** todos os testes aqui descritos foram executados exclusivamente em um laboratório isolado, de propriedade do autor, sem exposição a redes externas. Utilizar essas técnicas contra sistemas de terceiros sem autorização explícita é **crime**. Este repositório tem finalidade exclusivamente educacional.

---

## Sumário

- [Ambiente de teste](#ambiente-de-teste)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Pré-requisitos](#pré-requisitos)
- [Configuração do laboratório](#configuração-do-laboratório)
- [Teste 1 — Força bruta em FTP](#teste-1--força-bruta-em-ftp-vsftpd-234)
- [Teste 2 — Automação de login no DVWA](#teste-2--automação-de-tentativas-em-formulário-web-dvwa)
- [Teste 3 — Enumeração e password spraying em SMB](#teste-3--enumeração-de-usuários-e-password-spraying-em-smb)
- [Comparação Hydra vs. Medusa](#comparação-hydra-vs-medusa)
- [Recomendações de mitigação](#recomendações-de-mitigação-gerais)
- [Relatório completo](#relatório-completo)

---

## Ambiente de teste

| Item | Detalhe |
|---|---|
| Atacante | Kali Linux (VirtualBox) — `192.168.56.104` |
| Alvo | Metasploitable2 (VirtualBox) — `192.168.56.103` |
| Rede | VirtualBox Host-Only (isolada, sem acesso à rede externa) |
| Ferramentas | Hydra, Medusa, Nmap, enum4linux, smbclient |
| Serviços avaliados | FTP (vsftpd 2.3.4), aplicação web DVWA (Apache 2.2.8), Samba (smbd 3.X–4.X) |

> Os IPs acima são do ambiente de referência usado neste projeto. Ajuste-os para o IP real das suas VMs (`ip a` no Kali e no Metasploitable).

---

## Estrutura do repositório

```
.
├── README.md
├── usuarios.txt          # lista de usuários para os testes
├── senhas.txt            # lista de senhas para os testes
├── resultados...                # saída dos scripts (gerada em tempo de execução)
└── docs/
    └── relatorio-pentest.docx # relatório completo dos testes, com resultados
```

---

## Pré-requisitos

- VirtualBox com duas VMs configuradas em rede **Host-Only**:
  - Kali Linux (atacante)
  - Metasploitable2 (alvo)
- No Kali, as ferramentas já vêm pré-instaladas: `hydra`, `medusa`, `nmap`, `enum4linux`, `smbclient`, `ftp`.
- Conectividade confirmada entre as duas VMs (`ping` funcionando nos dois sentidos).

Para checar a instalação das ferramentas no Kali:

```bash
hydra -h | head -n1
medusa -h | head -n1
nmap --version | head -n1
enum4linux -h | head -n1
```

---

## Configuração do laboratório

Resumo da configuração de rede usada neste projeto (detalhes completos no relatório, em `docs/`):

1. Criar as duas VMs no VirtualBox.
2. Configurar um adaptador **Host-Only** idêntico nas duas VMs (mesmo nome de adaptador).
3. Conferir, dentro de cada VM, o IP atribuído (`ip a` no Kali, `ip a` no Metasploitable) e garantir que **não há conflito de IP** entre elas.
4. Validar a conectividade:

```bash
ping -c 3 <IP_DO_METASPLOITABLE>
nmap -sV -p21,22,80,139,445 <IP_DO_METASPLOITABLE>
```

5. Se for usar o DVWA, iniciar os serviços no Metasploitable:

```bash
sudo service apache2 restart
sudo service mysql start
```

e ajustar o nível de segurança do DVWA para **low** (menu *DVWA Security*).

---

## Teste 1 — Força bruta em FTP (vsftpd 2.3.4)

**Objetivo:** testar a robustez da autenticação do serviço FTP contra tentativas automatizadas.

```bash
./scripts/01_ftp_bruteforce.sh <IP_DO_ALVO>
```

O script executa, nessa ordem:
1. `nmap -sV -p21` para confirmar o serviço.
2. Hydra contra `ftp://<alvo>` usando `wordlists/usuarios.txt` e `wordlists/senhas.txt`.
3. Medusa (`-M ftp`) com as mesmas listas.

Validação manual da credencial encontrada:

```bash
ftp <IP_DO_ALVO>
```

---

## Teste 2 — Automação de tentativas em formulário web (DVWA)

**Objetivo:** avaliar a exposição do formulário de login do DVWA a ataques automatizados via HTTP POST.

```bash
./scripts/02_dvwa_bruteforce.sh <IP_DO_ALVO> [usuario]
```

O script executa Hydra (`http-post-form`) e Medusa (`-M web-form`) contra `/dvwa/login.php`, usando `wordlists/senhas.txt` e a string de falha `"Login failed"`.

> Antes de rodar em massa, confirme manualmente no navegador qual texto aparece quando o login falha — pode variar conforme a versão do DVWA.

---

## Teste 3 — Enumeração de usuários e password spraying em SMB

**Objetivo:** enumerar usuários/compartilhamentos expostos via Samba sem autenticação e testar uma mesma senha contra múltiplos usuários (password spraying), técnica que reduz o risco de bloqueio de conta em comparação à força bruta tradicional.

```bash
./scripts/03_smb_enum_spray.sh <IP_DO_ALVO> [senha_para_spray]
```

O script executa, nessa ordem:
1. `nmap -sV -p139,445` para confirmar o Samba.
2. `enum4linux -U` para enumerar usuários via sessão nula.
3. `smbclient -L -N` para listar compartilhamentos.
4. Hydra e Medusa em modo *password spraying* (uma senha, vários usuários de `wordlists/usuarios.txt`).

---

## Comparação Hydra vs. Medusa

| Aspecto | Hydra | Medusa |
|---|---|---|
| Sintaxe geral | Mais compacta, padrão `protocolo://alvo` | Separa alvo (`-h`) do módulo (`-M`), comando um pouco mais longo |
| FTP | `hydra -L usuarios -P senhas ftp://alvo` | `medusa -h alvo -U usuarios -P senhas -M ftp` |
| Formulário web | Um único parâmetro concatenado (caminho + corpo + string de falha) | Parâmetros separados (`FORM`, `FORM-DATA`, `DENY-SIGNAL`), mais legíveis |
| SMB / spraying | Troca `-P` por `-p` para passar de força bruta para spraying | Troca `-P` por `-p` da mesma forma |
| Enumeração de usuários | Não possui | Não possui (usar `enum4linux`/`smbclient`) |

Em geral, o Hydra favorece comandos curtos e um padrão único entre protocolos; o Medusa favorece comandos mais explícitos e fáceis de revisar campo a campo, ao custo de serem mais longos.

---

## Recomendações de mitigação (gerais)

- **FTP:** política de senha forte, bloqueio após tentativas falhas (ex.: fail2ban), substituição por SFTP/FTPS, monitoramento de autenticação.
- **Aplicação web (DVWA):** CAPTCHA após tentativas falhas, rate limiting por IP/usuário, alertas de pico de tentativas de login, autenticação multifator.
- **SMB/Samba:** desabilitar sessões nulas e enumeração anônima, bloqueio de conta, detecção de padrões de password spraying, restringir compartilhamentos expostos, atualizar a versão do Samba.

Detalhamento completo de cada recomendação, por teste, está no relatório em `docs/relatorio-pentest.docx`.

---

## Relatório completo

O relatório completo, com introdução, reconhecimento inicial, comandos, comparação de sintaxe e tabelas de resultado por teste, está disponível em [`docs/relatorio-pentest.docx`](docs/relatorio-pentest.docx).

---

## Autor

Rafael Crespim do Nascimento Cechinato — projeto desenvolvido para fins de estudo em segurança ofensiva.
