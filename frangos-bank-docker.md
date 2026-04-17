# 🐥 Frango's Bank — Setup Docker Lab

## Pré-requisitos
- Docker + Docker Compose instalados
- 4GB RAM disponível
- Portas 80, 443, 3306, 8080 livres

---

## Opção 1 — DVWA Clássico (mais rápido)

```bash
# Pull e run:
docker run --rm -it -p 80:80 vulnerables/web-dvwa

# Acessar: http://localhost
# Login: admin / password
# Setup: http://localhost/setup.php → "Create / Reset Database"
```

---

## Opção 2 — OWASP Juice Shop (mais completo)

```bash
docker run --rm -p 3000:3000 bkimminich/juice-shop
# Acessar: http://localhost:3000
# 100+ desafios, tema moderno, scoreboard embutido
```

---

## Opção 3 — Stack Completo Frango's Bank Lab

### docker-compose.yml

```yaml
version: '3.8'

services:

  # App principal vulnerável
  frangosbank-app:
    image: vulnerables/web-dvwa
    container_name: frangosbank_app
    ports:
      - "8080:80"
    environment:
      - MYSQL_HOSTNAME=db
      - MYSQL_DATABASE=frangosbank
      - MYSQL_USERNAME=frango
      - MYSQL_PASSWORD=galinha123
    depends_on:
      - db
    networks:
      - frangos-net

  # Banco de dados vulnerável
  db:
    image: mysql:5.6   # versão antiga = vulnerável
    container_name: frangosbank_db
    environment:
      MYSQL_ROOT_PASSWORD: rootgalinha
      MYSQL_DATABASE: frangosbank
      MYSQL_USER: frango
      MYSQL_PASSWORD: galinha123
    ports:
      - "3306:3306"   # exposto para o lab!
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - frangos-net

  # Juice Shop para desafios extras
  juiceshop:
    image: bkimminich/juice-shop
    container_name: frangosbank_juiceshop
    ports:
      - "3000:3000"
    networks:
      - frangos-net

  # WebGoat para aprendizado guiado
  webgoat:
    image: webgoat/goat-and-wolf
    container_name: frangosbank_webgoat
    ports:
      - "8888:8888"
      - "9090:9090"
    networks:
      - frangos-net

  # Kali Linux para atacar
  kali:
    image: kalilinux/kali-rolling
    container_name: frangosbank_kali
    tty: true
    stdin_open: true
    networks:
      - frangos-net
    command: /bin/bash

networks:
  frangos-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### init.sql — Banco vulnerável

```sql
CREATE TABLE IF NOT EXISTS tb_usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100),
    senha VARCHAR(100),  -- texto claro!
    cpf VARCHAR(14),
    nome VARCHAR(100),
    saldo DECIMAL(15,2),
    role VARCHAR(20) DEFAULT 'user',
    ag VARCHAR(10),
    conta VARCHAR(20)
);

INSERT INTO tb_usuarios VALUES
(1001, 'cornelio@frangosbank.com', 'senha123', '123.456.789-00', 'Cornélio Pinto', 18430.00, 'user', '0042', '12345-6'),
(1002, 'maria@email.com', '1234', '987.654.321-00', 'Maria Galinha', 3200.00, 'user', '0042', '12345-7'),
(1003, 'admin@frangosbank.com', 'admin', '000.000.000-00', 'Administrador', 999999.00, 'admin', '0001', '00001-0');

CREATE TABLE IF NOT EXISTS tb_transacoes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    de_conta VARCHAR(20),
    para_conta VARCHAR(20),
    valor DECIMAL(15,2),
    descricao VARCHAR(200),
    data_hora DATETIME DEFAULT NOW()
);

-- Log com dados sensíveis (vulnerabilidade A09):
CREATE TABLE IF NOT EXISTS tb_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    evento VARCHAR(500),  -- inclui senhas!
    data_hora DATETIME DEFAULT NOW()
);
```

---

## Como usar

```bash
# 1. Clonar e subir:
mkdir frangos-lab && cd frangos-lab
# (copie o docker-compose.yml e init.sql acima)
docker compose up -d

# 2. Verificar containers:
docker compose ps

# 3. Acessar Kali para atacar:
docker exec -it frangosbank_kali bash

# 4. Dentro do Kali — instalar ferramentas:
apt update && apt install -y \
  nmap nikto sqlmap hydra \
  gobuster dirsearch burpsuite \
  python3-pip git curl wget

pip3 install jwt-tool ssrfmap

# 5. Alvos disponíveis na rede interna:
# http://frangosbank-app:80   → App DVWA
# http://juiceshop:3000       → Juice Shop
# http://webgoat:8888         → WebGoat
# mysql://db:3306             → Banco de dados

# 6. Reconhecimento inicial:
nmap -sV -sC 172.20.0.0/24

# 7. Encerrar lab:
docker compose down
```

---

## Checklist de Pentest — Frango's Bank

### Reconhecimento
- [ ] Identificar tecnologias (whatweb, wappalyzer)
- [ ] Buscar diretórios (gobuster, dirsearch)
- [ ] Checar headers HTTP (nikto)
- [ ] Mapear endpoints de API

### Autenticação (A07)
- [ ] Testar senhas padrão/fracas
- [ ] Verificar rate limiting no login
- [ ] Testar bypass de MFA
- [ ] Analisar tokens de sessão

### Autorização (A01)
- [ ] Trocar IDs na URL/API
- [ ] Acessar endpoints admin sem permissão
- [ ] Testar IDOR em todos os recursos

### Injeção (A03)
- [ ] SQLi em todos os campos de entrada
- [ ] XSS refletido e armazenado
- [ ] Injeção em headers HTTP

### Lógica de Negócio (A04)
- [ ] Testar valores negativos
- [ ] Race conditions em transações
- [ ] Pular etapas de processo

### Configuração (A05)
- [ ] Checar .env, backup.sql, .git
- [ ] Verificar headers de segurança
- [ ] Testar CORS

### SSRF (A10)
- [ ] Testar funcionalidades que fazem requisições externas
- [ ] Acessar metadados de cloud
- [ ] Atingir serviços internos

---

*Ambiente isolado para uso educacional.*
*Nunca utilize estas técnicas em sistemas sem autorização expressa.*
