# Infraestrutura

Repositório de infraestrutura para projetos da Triadev.

## Estrutura

```
infra/
├── 01-edge/              # Traefik + Cloudflare (conexão com a rua)
│   ├── docker-compose.yml
│   ├── .env
│   ├── nginx-404/        # Página 404 customizada
│   └── data/
│
└── 02-tools/            # Ferramentas de apoio
    ├── docker-compose.yml
    ├── .env
    ├── Jenkinsfile      # CI/CD para deploy automático
    └── data/
```

## Serviços

### 01-edge (Edge/Porta de entrada)

| Container | Função | Porta |
|-----------|-------|-------|
| cloudflared | Tunnel Cloudflare | - |
| traefik | Reverse Proxy / Load Balancer | 80 |
| error-page-404 | Página 404 customizada | 80 |

### 02-tools (Ferramentas)

| Container | Função | Porta |
|-----------|-------|-------|
| jenkins | CI/CD | 8081 |

## Configuração

### Variáveis de Ambiente

#### 01-edge/.env

```env
# Cloudflare Tunnel
TUNNEL_TOKEN=seu_token_aqui

# Traefik Dashboard
TRAEFIK_AUTH_USERS=admin:$2y$05$...
TRAEFIK_DOMAIN=modelinfra-traefik.triadevsolutions.com.br

# Página 404
ERROR_404_DOMAIN=404.triadevsolutions.com.br
```

#### 02-tools/.env

```env
JENKINS_ADMIN_ID=modelcode
JENKINS_ADMIN_PASSWORD=modelcode123456
```

### Gerando senha do Traefik

```bash
htpasswd -nbB admin sua_senha
```

Cole o resultado no `.env` (use `$` simples, não `$$`).

## Como Subir

### 1. Subir 01-edge (Traefik)

```bash
cd 01-edge
docker-compose up -d
```

### 2. Subir 02-tools (Jenkins)

```bash
cd 02-tools
docker-compose up -d
```

### 3. Ambos juntos

```bash
docker-compose -f 01-edge/docker-compose.yml -f 02-tools/docker-compose.yml up -d
```

## Acesso

| Serviço | URL |
|---------|-----|
| Traefik Dashboard | http://modelinfra-traefik.triadevsolutions.com.br |
| Página 404 | http://404.triadevsolutions.com.br |
| Jenkins | http://localhost:8081 |

## CI/CD com Jenkins

O Jenkins na pasta `02-tools` pode fazer deploy automático do `01-edge`:

1. Criar Job Pipeline no Jenkins
2. Apointar para este repositório
3. Usar `02-tools/Jenkinsfile` como script
4. Configurar GitHub Webhook ou Polling

### Jenkinsfile

O `02-tools/Jenkinsfile` faz:
1. Checkout do código
2. Carrega variáveis do `01-edge/.env`
3. Build da imagem error-page-404
4. Deploy via `docker-compose up -d`

## Redes

- `devops-net`: Comunicação entre serviços internos
- `apps-net`: Rede dos apps clientes

## Volumes

- `jenkins_home`: Dados do Jenkins
- `cloudflared_data`: Dados do Cloudflare Tunnel

## Troubleshooting

### Ver logs

```bash
# Traefik
docker logs traefik

# Jenkins
docker logs jenkins

# Todos
docker-compose logs -f
```

### Reiniciar serviço

```bash
cd 01-edge
docker-compose restart traefik
```

### Ver status

```bash
docker ps
```

## Tecnologias

- Traefik v2.10
- Cloudflare Tunnel
- Jenkins LTS JDK17
- Nginx (página 404)

## Autor

Triadev Solutions