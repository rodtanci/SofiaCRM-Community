# 💬 Sofia CRM

O CRM completo para WhatsApp. Gerencie conversas, contatos, campanhas, chatbot e muito mais em uma única plataforma profissional.

## 📋 Pré-requisitos

⚠️ **INDISPENSÁVEL**: Antes de iniciar a instalação do Sofia CRM, é **obrigatório** ter as seguintes dependências já instaladas e funcionando:

- **Traefik** - Reverse proxy com suporte a SSL/TLS
- **Redis** - Para cache e filas de processamento
- **PostgreSQL 18 com pgvector** - Banco de dados com suporte a vetores para IA

Certifique-se também que a rede Docker existe e todos os serviços estão nela antes de iniciar a instalação.

### Requisitos do Sistema

- Docker e Docker Compose instalados
- Portainer configurado (recomendado)
- Traefik configurado na rede
- PostgreSQL com pgvector (recomendamos PostgreSQL 18)
- Redis para cache e filas

## 🚀 Instalação

### Passo 1: Configurar PostgreSQL com pgvector

Crie um arquivo `docker-compose-pgvector.yml` com o seguinte conteúdo:

```yaml
version: "3.8"

services:
  pgvector:
    image: pgvector/pgvector:pg18
    command:
      [
        "postgres",
        "-c", "max_connections=300",
        "-c", "shared_buffers=1GB",
        "-c", "work_mem=32MB",
        "-c", "maintenance_work_mem=512MB",
        "-c", "effective_cache_size=3GB",
        "-c", "wal_buffers=16MB",
        "-c", "checkpoint_completion_target=0.9",
        "-c", "wal_level=replica",
        "-c", "port=5434",
        "-c", "timezone=UTC",
        "-c", "log_min_duration_statement=500",
        "-c", "log_error_verbosity=default",
        "-c", "default_statistics_target=100",
        "-c", "shared_preload_libraries=pg_stat_statements",
        "-c", "pg_stat_statements.max=5000",
        "-c", "pg_stat_statements.track=all"
      ]
    environment:
      POSTGRES_PASSWORD: SENHA_POSTGRES
    ports:
      - "5432:5432"
    volumes:
      - postgres18_data:/var/lib/postgresql
    networks:
      - SUA_REDE_AQUI
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "1"
          memory: 4G

volumes:
  postgres18_data:
    driver: local

networks:
  SUA_REDE_AQUI:
    external: true
    name: SUA_REDE_AQUI
```

⚠️ **ATENÇÃO**: Substitua `SENHA_POSTGRES` por uma senha segura de sua escolha. Mantenha esta senha em local seguro!

💡 **Recomendação**: Recomendamos usar PostgreSQL 18 com extensão pgvector para melhor performance e suporte a funcionalidades de IA.

### Passo 2: Configurar Redis

Crie um arquivo `docker-compose-redis.yml` ou adicione ao seu docker-compose:

```yaml
version: "3.7"

services:
  redis:
    image: redis:latest
    command: [
        "redis-server",
        "--appendonly",
        "yes",
        "--port",
        "6379",
        "--requirepass",
        "SENHA_REDIS",
        "--maxmemory-policy",
        "allkeys-lru"
      ]
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - SUA_REDE_AQUI
    deploy:
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          cpus: "1"
          memory: 2048M

volumes:
  redis_data:
    external: true
    name: redis_data

networks:
  SUA_REDE_AQUI:
    external: true
    name: SUA_REDE_AQUI
```

⚠️ **ATENÇÃO**: Substitua `SENHA_REDIS` por uma senha segura de sua escolha.

### Passo 3: Configurar Sofia CRM

Crie um arquivo `docker-compose.yml` para o Sofia CRM:

```yaml
# Stack para Portainer - CRM com serviços externos (PostgreSQL e Redis)

# IMPORTANTE: Certifique-se que a rede 'SUA_REDE_AQUI' existe e o Traefik está nela

services:
  crm_api:
    image: dtanci/sofiacrm-community:v1.0.0
    container_name: crm_api
    restart: always
    environment:
      # Database - Use DATABASE_URL ou configure individualmente
      DATABASE_URL: postgresql://postgres:SENHA_POSTGRES@pgvector:5432/crm
      
      # OU configure individualmente (redundante se usar DATABASE_URL acima):
      POSTGRES_HOST: pgvector
      POSTGRES_PORT: 5432
      POSTGRES_DB: crm
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: SENHA_POSTGRES
      
      # Redis - Use REDIS_URL com senha incluída (recomendado)
      REDIS_URL: redis://:SENHA_REDIS@redis:6379
      
      # OU configure individualmente:
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: SENHA_REDIS
      
      # Application
      NODE_ENV: production
      PORT: 3000
      MEDIA_DIR: /app/media
      
      # Security - IMPORTANTE: Use valores únicos e seguros
      # Gere novas chaves usando: openssl rand -hex 32
      JWT_SECRET: SUA_CHAVE_JWT_SECRET_AQUI
      INTERNAL_TOKEN: SUA_CHAVE_INTERNAL_TOKEN_AQUI
      
      # Features
      CONTACTS_UPDATE_INTERVAL_HOURS: 1
      CONTACTS_UPDATE_BATCH_SIZE: 100
      LOG_LEVEL: INFO
    
    volumes:
      - media_data:/app/media
    
    networks:
      - SUA_REDE_AQUI
    
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Porta exposta (opcional se usar apenas Traefik)
    ports:
      - "3000:3000"
    
    deploy:
      labels:
        - "traefik.enable=true"
        # IMPORTANTE: Substitua url_do_seu_crm_aqui.com.br pelo seu domínio real
        - "traefik.http.routers.crm_api.rule=Host(`url_do_seu_crm_aqui.com.br`)"
        - "traefik.http.routers.crm_api.entrypoints=websecure"
        - "traefik.http.routers.crm_api.tls.certresolver=letsencryptresolver"
        - "traefik.http.routers.crm_api.priority=1"
        - "traefik.http.routers.crm_api.service=crm_api"
        - "traefik.http.services.crm_api.loadbalancer.server.port=3000"
        - "traefik.http.services.crm_api.loadbalancer.passhostheader=true"
        - "traefik.http.middlewares.sslheader.headers.customrequestheaders.X-Forwarded-Proto=https"
        - "traefik.http.routers.crm_api.middlewares=sslheader@docker"

  whats-service:
    image: dtanci/sofiacrm-whats-service-community:v1.0.0
    container_name: crm_whatsmeow
    restart: always
    environment:
      CRM_API_URL: http://crm_api:3000
      INTERNAL_WEBHOOK_TOKEN: SUA_CHAVE_INTERNAL_TOKEN_AQUI
      STORE_PATH: /app/data
    volumes:
      - whatsmeow_data:/app/data
    networks:
      - SUA_REDE_AQUI
    depends_on:
      - crm_api

volumes:
  whatsmeow_data:
    driver: local
  media_data:
    driver: local

networks:
  SUA_REDE_AQUI:
    external: true
    name: SUA_REDE_AQUI
```

⚠️ **SEGURANÇA CRÍTICA**: 
- Substitua `SENHA_POSTGRES` pela mesma senha usada no PostgreSQL
- Substitua `SENHA_REDIS` pela mesma senha usada no Redis
- Substitua `url_do_seu_crm_aqui.com.br` pelo seu domínio real
- Substitua `SUA_REDE_AQUI` pelo nome da sua rede Docker
- **IMPORTANTE**: Gere novas chaves `JWT_SECRET` e `INTERNAL_TOKEN` usando `openssl rand -hex 32` e substitua os valores no arquivo. As chaves padrão são apenas para exemplo!

### Passo 4: Gerar Chaves Seguras

Para gerar chaves seguras para `JWT_SECRET` e `INTERNAL_TOKEN`, execute:

```bash
# Gerar JWT_SECRET
openssl rand -hex 32

# Gerar INTERNAL_TOKEN (use o mesmo comando, gere uma nova chave)
openssl rand -hex 32
```

💡 **Dica**: Use chaves diferentes para `JWT_SECRET` e `INTERNAL_TOKEN`. Essas chaves são críticas para a segurança da aplicação.

### Passo 5: Iniciar os Serviços

1. **Inicie o PostgreSQL:**
   ```bash
   docker-compose -f docker-compose-pgvector.yml up -d
   ```

2. **Inicie o Redis:**
   ```bash
   docker-compose up -d redis
   ```

3. **Inicie o Sofia CRM:**
   ```bash
   docker-compose up -d
   ```

### Passo 6: Verificar Instalação

Verifique se todos os containers estão rodando:

```bash
docker ps
```

Verifique os logs do CRM:

```bash
docker logs crm_api
```

## 🔧 Variáveis de Ambiente Importantes

| Variável | Descrição |
|----------|-----------|
| `DATABASE_URL` | URL de conexão com PostgreSQL |
| `REDIS_URL` | URL de conexão com Redis |
| `JWT_SECRET` | Chave secreta para tokens JWT (gerar com openssl) |
| `INTERNAL_TOKEN` | Token interno para comunicação entre serviços (gerar com openssl) |
| `CONTACTS_UPDATE_INTERVAL_HOURS` | Intervalo de atualização de contatos (padrão: 1) |
| `LOG_LEVEL` | Nível de log (INFO, DEBUG, ERROR) |

## 🐛 Troubleshooting

**Problemas comuns:**

- **Container não inicia**: Verifique os logs com `docker logs crm_api`
- **Erro de conexão com banco**: Verifique se o PostgreSQL está rodando e se as credenciais estão corretas
- **Erro de conexão com Redis**: Verifique se o Redis está rodando e se a senha está correta
- **Traefik não roteia**: Verifique se a rede está correta e se o Traefik está na mesma rede

## 💬 Suporte

Se você encontrar problemas durante a instalação, entre em contato através do nosso grupo do WhatsApp para ajuda e suporte:

[**Acessar Grupo do WhatsApp**](https://chat.whatsapp.com/H7RHjDI3GR62iodcSUV3G1)

## 📄 Licença

Este projeto foi desenvolvido para o Sofia CRM.

## 🔗 Links Úteis

- [Site Oficial](https://sofiacrm.com.br)

---

**Sofia CRM** - O CRM completo para WhatsApp 💬
