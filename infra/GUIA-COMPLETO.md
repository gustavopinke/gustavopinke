# Guia Completo: Listmonk + n8n

Documentacao consolidada da stack de email marketing e automacao da Baduga.

---

## 1. Arquitetura

```
+------------------+       +------------------+       +------------------+
|    Listmonk      |       |      n8n         |       |   PostgreSQL     |
|  Email Marketing |       |   Automacao      |       |   Banco de Dados |
|  porta: 9000     |       |   porta: 5678    |       |   porta: 5432    |
+--------+---------+       +--------+---------+       +--------+---------+
         |                          |                          |
         +------------- docker network -------------------------+
```

- **Listmonk** = gerencia listas, templates, campanhas e envia emails
- **n8n** = automatiza workflows (ex: novo pedido na Shopify -> dispara email no Listmonk)
- **PostgreSQL** = banco compartilhado (cada servico com seu proprio database)

---

## 2. Arquivos Necessarios

### 2.1 docker-compose.yml

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    container_name: infra-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  listmonk:
    image: listmonk/listmonk:latest
    container_name: infra-listmonk
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "${LISTMONK_PORT:-9000}:9000"
    environment:
      - TZ=America/Sao_Paulo
    volumes:
      - ./listmonk-config.toml:/listmonk/config.toml
    command: >
      sh -c "yes | ./listmonk --install --config /listmonk/config.toml || true && ./listmonk --config /listmonk/config.toml"

  n8n:
    image: n8nio/n8n:latest
    container_name: infra-n8n
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "${N8N_PORT:-5678}:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-postgres}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD:-postgres}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER:-admin}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD:-admin}
      - GENERIC_TIMEZONE=America/Sao_Paulo
      - TZ=America/Sao_Paulo
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - WEBHOOK_URL=${N8N_WEBHOOK_URL:-http://localhost:5678}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  postgres_data:
  n8n_data:
```

### 2.2 init-db.sql

```sql
CREATE DATABASE listmonk;
CREATE DATABASE n8n;
```

### 2.3 listmonk-config.toml

```toml
[app]
address = "0.0.0.0:9000"
admin_username = "admin"
admin_password = "admin"

[db]
host = "postgres"
port = 5432
user = "postgres"
password = "postgres"
database = "listmonk"
ssl_mode = "disable"
max_open = 25
max_idle = 25
max_lifetime = "300s"
```

### 2.4 .env

```env
# PostgreSQL
POSTGRES_USER=postgres
POSTGRES_PASSWORD=troque_por_senha_segura

# Listmonk
LISTMONK_PORT=9000

# n8n
N8N_PORT=5678
N8N_USER=admin
N8N_PASSWORD=troque_por_senha_segura
N8N_HOST=localhost
N8N_PROTOCOL=http
N8N_WEBHOOK_URL=http://localhost:5678
```

---

## 3. Instalacao Passo a Passo

```bash
# 1. Clonar o repo e entrar na pasta
cd infra

# 2. Criar o .env
cp .env.example .env
nano .env   # trocar as senhas

# 3. Subir tudo
docker compose up -d

# 4. Verificar se subiu
docker compose ps
```

Resultado esperado:
```
NAME              STATUS
infra-postgres    Up (healthy)
infra-listmonk    Up
infra-n8n         Up
```

---

## 4. Configuracao do Listmonk

### 4.1 Primeiro acesso

1. Acesse http://localhost:9000
2. Login: `admin` / `admin`
3. **Troque a senha imediatamente** em Settings > General

### 4.2 Configurar SMTP (Gmail)

Em Settings > SMTP:

| Campo    | Valor               |
|----------|---------------------|
| Host     | `smtp.gmail.com`    |
| Port     | `587`               |
| Auth     | `login`             |
| Username | seu-email@gmail.com |
| Password | senha de app Google |

**Como gerar senha de app do Google:**
1. Acesse https://myaccount.google.com/apppasswords
2. Crie uma nova senha para "Mail"
3. Use essa senha de 16 caracteres no Listmonk

### 4.3 Criar uma lista

1. Lists > New List
2. Nome: `Newsletter Baduga`
3. Type: `public`
4. Optin: `single` (ou `double` para confirmacao por email)

### 4.4 Adicionar template

1. Templates > New Template
2. Cole o HTML do template (ver secao 6)
3. Salve

### 4.5 Criar campanha

1. Campaigns > New Campaign
2. Selecione a lista e o template
3. Envie um teste antes de disparar

---

## 5. Configuracao do n8n

### 5.1 Primeiro acesso

1. Acesse http://localhost:5678
2. Login: `admin` / `admin` (ou o que definiu no .env)

### 5.2 Workflow: Novo pedido -> Email automatico

Exemplo de workflow no n8n para enviar email via Listmonk quando receber um webhook:

```
[Webhook] -> [HTTP Request para Listmonk API] -> [Resposta]
```

**Nodes:**

1. **Webhook** (trigger)
   - Method: POST
   - Path: `/novo-pedido`

2. **HTTP Request** (chamar Listmonk)
   - Method: POST
   - URL: `http://listmonk:9000/api/tx`
   - Authentication: Basic Auth (`admin` / `admin`)
   - Body (JSON):
   ```json
   {
     "subscriber_email": "{{ $json.email }}",
     "template_id": 1,
     "data": {
       "nome": "{{ $json.nome }}",
       "pedido": "{{ $json.pedido_id }}"
     }
   }
   ```

### 5.3 API do Listmonk - Endpoints uteis

| Endpoint                        | Metodo | Descricao                  |
|---------------------------------|--------|----------------------------|
| `/api/subscribers`              | GET    | Listar inscritos           |
| `/api/subscribers`              | POST   | Criar inscrito             |
| `/api/lists`                    | GET    | Listar listas              |
| `/api/campaigns`                | GET    | Listar campanhas           |
| `/api/campaigns`                | POST   | Criar campanha             |
| `/api/campaigns/{id}/status`    | PUT    | Iniciar/pausar campanha    |
| `/api/tx`                       | POST   | Enviar email transacional  |
| `/api/templates`                | GET    | Listar templates           |

**Autenticacao:** Basic Auth em todas as chamadas.

---

## 6. Templates de Email

### 6.1 Template Base (reutilizavel)

Template com variaveis do Listmonk — usar como base para todas as campanhas:

- Variaveis: `{{ .BookTitle }}`, `{{ .BookAuthors }}`, `{{ .BookCoverURL }}`, `{{ .BookURL }}`, `{{ .Body }}`, `{{ .HighlightQuote }}`, `{{ .ClosingText }}`
- Unsubscribe: `{{ .UnsubscribeURL }}`
- Preferencias: `{{ .ManagePreferencesURL }}`

O arquivo `baduga_template.html` contem o template base completo.

### 6.2 Exemplo: "A Casa que Despertou"

O arquivo `baduga_casa_despertou.html` contem um exemplo pronto com o conteudo do livro preenchido.

Ambos os templates estao na pasta `infra/templates/`.

---

## 7. Comandos Uteis

```bash
# Ver logs em tempo real
docker compose logs -f

# Logs de um servico especifico
docker compose logs -f listmonk
docker compose logs -f n8n

# Parar tudo
docker compose down

# Parar e apagar dados (CUIDADO)
docker compose down -v

# Reiniciar um servico
docker compose restart listmonk

# Atualizar imagens para ultima versao
docker compose pull && docker compose up -d

# Backup do banco
docker exec infra-postgres pg_dumpall -U postgres > backup.sql

# Restaurar backup
cat backup.sql | docker exec -i infra-postgres psql -U postgres
```

---

## 8. Producao (VPS/Servidor)

Para usar em producao com dominio proprio:

### 8.1 Adicionar Nginx como proxy reverso

Adicionar ao docker-compose.yml:

```yaml
  nginx:
    image: nginx:alpine
    container_name: infra-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - certbot_data:/etc/letsencrypt
    depends_on:
      - listmonk
      - n8n
```

### 8.2 Exemplo nginx.conf

```nginx
server {
    listen 80;
    server_name mail.seudominio.com;

    location / {
        proxy_pass http://listmonk:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name n8n.seudominio.com;

    location / {
        proxy_pass http://n8n:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 8.3 SSL com Certbot

```bash
docker run --rm -v certbot_data:/etc/letsencrypt certbot/certbot \
  certonly --webroot -w /var/www/certbot \
  -d mail.seudominio.com -d n8n.seudominio.com
```

---

## 9. Limites de Envio por SMTP

| Provedor         | Limite Diario | Custo        |
|------------------|---------------|--------------|
| Gmail            | 500/dia       | Gratis       |
| Google Workspace | 2.000/dia     | ~R$30/mes    |
| Amazon SES       | 50.000/dia    | ~R$0,40/1000 |
| Brevo (Sendinblue)| 300/dia      | Gratis       |
| Mailgun          | 5.000/mes     | Gratis (3 meses) |

Para a Baduga comecando, **Gmail** resolve. Quando passar de 500 emails/dia, migra pra **Amazon SES**.
