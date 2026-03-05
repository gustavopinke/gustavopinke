# Infra - Listmonk + n8n

Stack de email marketing e automacao.

## Servicos

| Servico  | Porta | Descricao                    |
|----------|-------|------------------------------|
| Listmonk | 9000  | Email marketing self-hosted  |
| n8n      | 5678  | Automacao de workflows       |
| Postgres | 5432  | Banco de dados compartilhado |

## Como usar

### 1. Configurar

```bash
cp .env.example .env
# Edite o .env com suas senhas
```

### 2. Subir tudo

```bash
docker compose up -d
```

### 3. Acessar

- **Listmonk:** http://localhost:9000 (admin / admin)
- **n8n:** http://localhost:5678 (admin / admin)

### 4. Configurar SMTP no Listmonk

1. Acesse http://localhost:9000
2. Va em Settings > SMTP
3. Configure seu Gmail:
   - Host: `smtp.gmail.com`
   - Port: `587`
   - Auth: `login`
   - Username: seu email
   - Password: senha de app do Google

### 5. Criar automacao no n8n

1. Acesse http://localhost:5678
2. Crie um novo workflow
3. Use o node "HTTP Request" para chamar a API do Listmonk:
   - URL: `http://listmonk:9000/api/campaigns`
   - Autenticacao: Basic Auth (admin/admin)

## Comandos uteis

```bash
# Ver logs
docker compose logs -f

# Parar tudo
docker compose down

# Reiniciar um servico
docker compose restart listmonk

# Atualizar imagens
docker compose pull && docker compose up -d
```

## Limites de envio

| SMTP         | Limite diario |
|--------------|---------------|
| Gmail        | 500/dia       |
| Google Workspace | 2000/dia |
| Amazon SES   | 50.000/dia    |
| Sendinblue   | 300/dia (free)|
