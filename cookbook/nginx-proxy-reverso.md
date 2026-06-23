# 166 — Instalando e configurando o NGINX (proxy reverso)

Cookbook para publicar a aplicação Node na web: NGINX recebe HTTP/HTTPS na porta 80/443 e encaminha para o PM2 na porta interna (ex: 3333).

## Visão geral

```text
Cliente → :80/:443 (NGINX) → proxy_pass → 127.0.0.1:3333 (Node/PM2)
```

---

## 1. Instalar NGINX

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

Teste no navegador: `http://IP_DO_SERVIDOR` — deve aparecer a página padrão do NGINX.

---

## 2. Remover site padrão (opcional)

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## 3. Configurar proxy para a aplicação Node

Substitua:

- `meudominio.com.br` pelo seu domínio (ou use `_` / IP só para teste)
- `3333` pela porta da sua app (PM2)

Crie o arquivo do site:

```bash
sudo nano /etc/nginx/sites-available/minha-api
```

Conteúdo mínimo:

```nginx
server {
    listen 80;
    server_name meudominio.com.br www.meudominio.com.br;

    location / {
        proxy_pass http://127.0.0.1:3333;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Ativar o site:

```bash
sudo ln -s /etc/nginx/sites-available/minha-api /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. DNS

No provedor do domínio, crie um registro **A**:

| Tipo | Nome | Valor |
|------|------|-------|
| A | `@` | IP do VPS |
| A | `www` | IP do VPS |

Aguarde a propagação (minutos a horas) e teste:

```bash
curl -I http://meudominio.com.br
```

---

## 5. HTTPS com Let's Encrypt (Certbot)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d meudominio.com.br -d www.meudominio.com.br
```

Siga o assistente (e-mail, aceitar termos). O Certbot ajusta o bloco `server` para HTTPS e configura renovação automática.

Testar renovação:

```bash
sudo certbot renew --dry-run
```

---

## 6. Ajustes úteis

### Upload de arquivos grandes

Dentro do bloco `server`:

```nginx
client_max_body_size 50M;
```

### WebSocket (Socket.io, etc.)

Os headers `Upgrade` e `Connection` do exemplo acima já ajudam. Para rotas específicas:

```nginx
location /socket.io/ {
    proxy_pass http://127.0.0.1:3333;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

### Múltiplas apps no mesmo servidor

Um arquivo em `sites-available` por app, cada um com `server_name` diferente e `proxy_pass` para portas distintas:

```nginx
# api.meudominio.com.br → 3333
# app.meudominio.com.br → 3334
```

---

## 7. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'   # 80 e 443
sudo ufw enable
sudo ufw status
```

A porta da app (3333) **não** precisa estar aberta publicamente se só o NGINX acessar `127.0.0.1`.

---

## 8. Fluxo completo de deploy (resumo)

```bash
# 1. Código
cd ~/apps/repositorio && git pull origin main

# 2. Node
npm ci --omit=dev && npm run build

# 3. PM2
pm2 restart minha-api

# 4. NGINX (só se mudou config)
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9. Checklist rápido

| Item | Feito |
|------|-------|
| NGINX instalado e ativo | ☐ |
| Site em `sites-available` + link em `sites-enabled` | ☐ |
| `sudo nginx -t` sem erros | ☐ |
| DNS apontando para o VPS | ☐ |
| `curl` / navegador acessam a API via domínio | ☐ |
| Certbot/HTTPS configurado (produção) | ☐ |
| UFW liberando 80/443 | ☐ |

---

## Problemas comuns

| Sintoma | Solução |
|---------|---------|
| 502 Bad Gateway | PM2 parado ou porta errada em `proxy_pass`; `pm2 list` e `curl 127.0.0.1:3333` |
| 404 do NGINX, não da API | `server_name` não bate com o host acessado |
| `nginx: configuration file test failed` | Erro de sintaxe no `.conf`; corrigir e `nginx -t` de novo |
| HTTPS não renova | `sudo certbot renew --dry-run`; conferir cron/systemd do certbot |
| Loop de redirect | App forçando HTTPS enquanto proxy não envia `X-Forwarded-Proto` |

---

## Referência rápida de arquivos

| Caminho | Função |
|---------|--------|
| `/etc/nginx/nginx.conf` | Config global |
| `/etc/nginx/sites-available/` | Definições de sites |
| `/etc/nginx/sites-enabled/` | Links simbólicos dos sites ativos |
| `/var/log/nginx/access.log` | Acessos |
| `/var/log/nginx/error.log` | Erros |

---

**Anterior:** [node-pm2-servidor.md](./node-pm2-servidor.md)  
**Índice:** [README.md](./README.md)
