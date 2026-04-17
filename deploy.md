# 🐥 Frango's Bank DVWA Lab — Guia de Deploy

> Ambiente isolado para pentest educacional baseado no OWASP Top 10 2025.
> **Nunca exponha este ambiente à internet.**

---

## Pré-requisitos

| Requisito | Versão mínima |
|-----------|--------------|
| Docker Engine | 24.0+ |
| Docker Compose | v2.0+ |
| RAM disponível | 4 GB |
| Disco | 8 GB |

Verificar instalação:
```bash
docker --version
docker compose version
```

---

## Estrutura de arquivos

```
frangos-lab/
├── docker-compose.yml
├── init.sql
├── dvwa-config/
│   └── config.inc.php
└── README.md
```

---

## docker-compose.yml

```yaml
version: '3.8'

services:

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
    restart: unless-stopped

  db:
    image: mysql:5.6
    container_name: frangosbank_db
    environment:
      MYSQL_ROOT_PASSWORD: rootgalinha
      MYSQL_DATABASE: frangosbank
      MYSQL_USER: frango
      MYSQL_PASSWORD: galinha123
    ports:
      - "3306:3306"
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - db-data:/var/lib/mysql
    networks:
      - frangos-net
    restart: unless-stopped

  juiceshop:
    image: bkimminich/juice-shop
    container_name: frangosbank_juiceshop
    ports:
      - "3000:3000"
    networks:
      - frangos-net
    restart: unless-stopped

  webgoat:
    image: webgoat/goat-and-wolf
    container_name: frangosbank_webgoat
    ports:
      - "8888:8888"
      - "9090:9090"
    networks:
      - frangos-net
    restart: unless-stopped

  kali:
    image: kalilinux/kali-rolling
    container_name: frangosbank_kali
    tty: true
    stdin_open: true
    networks:
      - frangos-net
    command: sleep infinity

volumes:
  db-data:

networks:
  frangos-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

---

## init.sql — Banco de dados vulnerável

```sql
CREATE TABLE IF NOT EXISTS tb_usuarios (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    email       VARCHAR(100),
    senha       VARCHAR(100),      -- texto claro, intencional
    cpf         VARCHAR(14),
    nome        VARCHAR(100),
    saldo       DECIMAL(15,2),
    role        VARCHAR(20) DEFAULT 'user',
    ag          VARCHAR(10),
    conta       VARCHAR(20)
);

INSERT INTO tb_usuarios VALUES
(1001, 'cornelio@frangosbank.com', 'senha123',  '123.456.789-00', 'Cornélio Pinto',    18430.00, 'user',  '0042', '12345-6'),
(1002, 'maria@email.com',          '1234',       '987.654.321-00', 'Maria Galinha',      3200.00,  'user',  '0042', '12345-7'),
(1003, 'joao@email.com',           'joao2024',   '111.222.333-44', 'João Pintinho',       750.00,  'user',  '0042', '12345-8'),
(1004, 'ana@email.com',            'ana123',     '555.444.333-22', 'Ana Carijó',        45000.00,  'user',  '0042', '12345-9'),
(1005, 'admin@frangosbank.com',    'admin',      '000.000.000-00', 'Admin Galinheiro', 999999.00,  'admin', '0001', '00001-0');

CREATE TABLE IF NOT EXISTS tb_transacoes (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    de_conta    VARCHAR(20),
    para_conta  VARCHAR(20),
    valor       DECIMAL(15,2),
    descricao   VARCHAR(200),
    data_hora   DATETIME DEFAULT NOW()
);

INSERT INTO tb_transacoes VALUES
(1, '12345-6', '12345-7', 340.50,  'Supermercado Bico de Ouro', '2026-04-16 14:23:00'),
(2, NULL,      '12345-6', 6800.00, 'Salário Granja Tech',       '2026-04-15 09:00:00'),
(3, '12345-6', NULL,      189.90,  'GalinhaLight Energia',      '2026-04-14 10:00:00');

-- Log com senhas em texto claro (vulnerabilidade A09 intencional)
CREATE TABLE IF NOT EXISTS tb_logs (
    id        INT AUTO_INCREMENT PRIMARY KEY,
    evento    VARCHAR(500),
    data_hora DATETIME DEFAULT NOW()
);

INSERT INTO tb_logs VALUES
(1, 'LOGIN user=cornelio@frangosbank.com pass=senha123 ip=187.10.20.30', NOW()),
(2, 'LOGIN FAILED user=admin@frangosbank.com pass=admin123',             NOW()),
(3, 'LOGIN user=admin@frangosbank.com pass=admin ip=200.50.60.70',       NOW());
```

---

## Subindo o lab

```bash
# 1. Criar a pasta e os arquivos
mkdir frangos-lab && cd frangos-lab
# (copie docker-compose.yml e init.sql acima)

# 2. Subir todos os containers
docker compose up -d

# 3. Verificar status
docker compose ps

# 4. Aguardar o banco inicializar (~20s) e configurar o DVWA
# Acesse: http://localhost:8080/setup.php
# Clique em "Create / Reset Database"

# 5. Acessar os serviços
echo "DVWA:      http://localhost:8080  (admin/password)"
echo "JuiceShop: http://localhost:3000"
echo "WebGoat:   http://localhost:8888/WebGoat"
echo "MySQL:     localhost:3306 (frango/galinha123)"
```

---

## Acessando o Kali para atacar

```bash
# Entrar no container Kali
docker exec -it frangosbank_kali bash

# Instalar ferramentas essenciais
apt update && apt install -y \
  nmap nikto sqlmap hydra \
  gobuster dirsearch curl wget \
  python3-pip git

pip3 install jwt-tool --break-system-packages

# Recon inicial da rede do lab
nmap -sV -sC 172.20.0.0/24

# Alvos disponíveis na rede interna:
# 172.20.0.x  — frangosbank-app  (porta 80)
# 172.20.0.x  — db               (porta 3306)
# 172.20.0.x  — juiceshop        (porta 3000)
# 172.20.0.x  — webgoat          (porta 8888)
```

---

## Credenciais do lab

| Serviço | Usuário | Senha |
|---------|---------|-------|
| DVWA | `admin` | `password` |
| Frango's Bank | `cornelio@frangosbank.com` | `senha123` |
| Frango's Bank | `admin@frangosbank.com` | `admin` |
| Frango's Bank | `maria@email.com` | `1234` |
| MySQL | `frango` | `galinha123` |
| MySQL root | `root` | `rootgalinha` |

---

## Acessar o banco diretamente

```bash
# De fora dos containers (porta exposta):
mysql -h 127.0.0.1 -P 3306 -u frango -pgalinha123 frangosbank

# De dentro do Kali (rede interna):
mysql -h db -u frango -pgalinha123 frangosbank

# Queries úteis para o lab:
SELECT * FROM tb_usuarios;
SELECT * FROM tb_logs;
SELECT * FROM tb_transacoes;
```

---

## Resetar o lab

```bash
# Parar e remover tudo (mantém imagens)
docker compose down -v

# Subir novamente do zero
docker compose up -d
```

---

## Solução de problemas

**DVWA não conecta ao banco:**
```bash
docker logs frangosbank_app
# Se erro de conexão MySQL, aguarde mais ~30s e tente o setup.php novamente
```

**Porta já em uso:**
```bash
# Altere a porta no docker-compose.yml
# Ex: "8081:80" em vez de "8080:80"
```

**Kali sem acesso à rede interna:**
```bash
docker network inspect frangos-lab_frangos-net
# Verifique se o container kali está na rede frangos-net
```

**Erro de permissão no init.sql:**
```bash
chmod 644 init.sql
```

---

## Encerrar o lab

```bash
# Parar os containers (mantém dados)
docker compose stop

# Remover containers e volumes (reset completo)
docker compose down -v

# Remover também as imagens baixadas
docker compose down -v --rmi all
```

---

> 🐥 **Frango's Bank DVWA Lab** — Galinheiro Hall Security Team
> Uso exclusivo para fins educacionais em ambientes isolados.
