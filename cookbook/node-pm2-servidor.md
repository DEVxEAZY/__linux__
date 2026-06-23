# 165 — Instalando Node e PM2 no servidor

Cookbook para instalar Node.js (LTS), dependências da aplicação e manter o processo no ar com PM2.

## Visão geral

```text
git pull → npm install → pm2 start → app escuta porta (ex: 3333)
```

O NGINX (próximo guia) encaminha o tráfego da porta 80 para essa porta interna.

---

## 1. Instalar Node.js (Ubuntu/Debian)

### Opção A — NodeSource (LTS recomendado)

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### Opção B — nvm (várias versões no mesmo servidor)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts
```

---

## 2. Instalar PM2 globalmente

```bash
sudo npm install -g pm2
pm2 -v
```

PM2 gerencia restart automático, logs e startup após reboot.

---

## 3. Preparar a aplicação no servidor

Supondo o repo em `~/apps/repositorio`:

```bash
cd ~/apps/repositorio
git pull origin main
```

Instalar dependências de produção:

```bash
npm ci --omit=dev
# ou, se não houver package-lock:
# npm install --production
```

Build (se for TypeScript / frontend):

```bash
npm run build
```

Configure `.env` (não vem do Git):

```bash
nano .env
# PORT=3333
# NODE_ENV=production
```

---

## 4. Subir com PM2

### App Node simples (entry `src/server.js` ou `dist/server.js`)

```bash
pm2 start src/server.js --name minha-api
# ou
pm2 start dist/server.js --name minha-api
```

### Usando script do `package.json`

```bash
pm2 start npm --name minha-api -- start
```

### Arquivo `ecosystem.config.cjs` (recomendado)

Na raiz do projeto:

```javascript
module.exports = {
  apps: [
    {
      name: 'minha-api',
      script: './dist/server.js',
      instances: 1,
      exec_mode: 'fork',
      env: {
        NODE_ENV: 'production',
        PORT: 3333,
      },
    },
  ],
};
```

Subir:

```bash
pm2 start ecosystem.config.cjs
```

---

## 5. Comandos PM2 do dia a dia

```bash
pm2 list                    # processos
pm2 logs minha-api          # logs em tempo real
pm2 logs minha-api --lines 100
pm2 restart minha-api       # após git pull + build
pm2 stop minha-api
pm2 delete minha-api
pm2 monit                   # dashboard no terminal
```

Após cada deploy:

```bash
cd ~/apps/repositorio
git pull origin main
npm ci --omit=dev
npm run build    # se aplicável
pm2 restart minha-api
```

---

## 6. Persistir PM2 após reinício do servidor

Salvar a lista de processos:

```bash
pm2 save
```

Gerar script de startup (siga a instrução que o comando imprimir):

```bash
pm2 startup
# copie e execute o comando sudo que aparecer, ex:
# sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u deploy --hp /home/deploy
pm2 save
```

Teste (opcional): `sudo reboot` e depois `pm2 list`.

---

## 7. Variáveis de ambiente com PM2

No `ecosystem.config.cjs`:

```javascript
env_file: '.env',
```

Ou inline em `env` / `env_production`.

Recarregar sem downtime (quando suportado):

```bash
pm2 reload minha-api
```

---

## 8. Firewall — liberar só o necessário

A app pode escutar apenas em `localhost` se o NGINX fizer proxy (recomendado):

```javascript
// exemplo Express
app.listen(3333, '127.0.0.1', () => { ... });
```

Se expuser a porta diretamente:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 3333/tcp   # evite em produção se usar NGINX na 80/443
sudo ufw enable
```

---

## 9. Checklist rápido

| Item | Feito |
|------|-------|
| `node -v` e `npm -v` OK | ☐ |
| PM2 instalado globalmente | ☐ |
| `npm ci` / build executados | ☐ |
| `.env` configurado no servidor | ☐ |
| `pm2 start` e `pm2 list` mostram app online | ☐ |
| `pm2 startup` + `pm2 save` configurados | ☐ |
| App responde em `curl http://127.0.0.1:3333` | ☐ |

---

## Problemas comuns

| Sintoma | Solução |
|---------|---------|
| `EADDRINUSE` | Porta em uso: `pm2 delete all` ou mudar `PORT` no `.env` |
| App cai após logout SSH | Usar PM2, não `node` direto no terminal |
| `MODULE_NOT_FOUND` | Rodar `npm ci` na pasta certa; conferir `script` no ecosystem |
| Não sobe após reboot | Repetir `pm2 startup` e `pm2 save` com o usuário correto |

---

**Anterior:** [git-servidor.md](./git-servidor.md)  
**Próximo:** [nginx-proxy-reverso.md](./nginx-proxy-reverso.md)
