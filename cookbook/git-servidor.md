# 164 — Configurando o Git e enviando arquivos para o servidor

Cookbook para versionar o projeto localmente, autenticar no servidor e manter o código sincronizado (clone / pull).

## Visão geral

```text
[Máquina local]  --git push-->  [GitHub/GitLab]  --git pull-->  [Servidor VPS]
                     ou
[Máquina local]  --git push-->  [bare repo no servidor]  (deploy direto)
```

Na prática você costuma usar um remoto (GitHub) e no servidor apenas `git clone` + `git pull` após cada deploy.

---

## 1. No servidor — usuário e pacotes

Conecte via SSH:

```bash
ssh usuario@IP_DO_SERVIDOR
```

Instale Git:

```bash
sudo apt update
sudo apt install -y git
git --version
```

Crie um usuário dedicado ao deploy (opcional, recomendado):

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
# sair e entrar como deploy
```

---

## 2. No servidor — identidade Git

```bash
git config --global user.name "Seu Nome"
git config --global user.email "seu@email.com"
```

Verifique:

```bash
git config --list
```

---

## 3. Autenticação SSH com o Git remoto

### 3.1 Gerar chave no servidor (para puxar do GitHub/GitLab)

```bash
ssh-keygen -t ed25519 -C "servidor-producao" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

Copie a chave pública e cadastre em:

- **GitHub:** Settings → SSH and GPG keys → New SSH key  
- **GitLab:** Preferences → SSH Keys  

Teste:

```bash
ssh -T git@github.com
# Hi username! You've successfully authenticated...
```

### 3.2 (Alternativa) HTTPS + token

Se preferir HTTPS, use um Personal Access Token no lugar da senha ao clonar.

---

## 4. Clonar o repositório no servidor

```bash
cd ~
mkdir -p apps && cd apps
git clone git@github.com:usuario/repositorio.git
cd repositorio
```

Branch de produção:

```bash
git checkout main   # ou master / production
```

---

## 5. Na máquina local — fluxo diário

Inicializar (se ainda não tiver repo):

```bash
cd /caminho/do/projeto
git init
git remote add origin git@github.com:usuario/repositorio.git
```

Commit e envio:

```bash
git add .
git commit -m "feat: descrição da mudança"
git push -u origin main
```

No servidor, atualizar:

```bash
cd ~/apps/repositorio
git pull origin main
```

---

## 6. Arquivo `.env` e segredos (não versionar)

No servidor, crie variáveis fora do Git:

```bash
nano ~/apps/repositorio/.env
```

No `.gitignore` local (já deve existir):

```gitignore
.env
node_modules/
dist/
```

Nunca faça `git add .env`.

---

## 7. Script simples de deploy (pull)

Crie no servidor `~/bin/deploy-app.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="$HOME/apps/repositorio"
BRANCH="${1:-main}"

cd "$APP_DIR"
git fetch origin
git checkout "$BRANCH"
git pull origin "$BRANCH"

# Próximo passo: dependências e restart (ver cookbook node-pm2)
# npm ci --omit=dev
# pm2 restart repositorio

echo "Deploy concluído: $(git rev-parse --short HEAD)"
```

```bash
chmod +x ~/bin/deploy-app.sh
~/bin/deploy-app.sh main
```

---

## 8. Deploy via `git push` (bare repository) — opcional

Para push direto ao servidor sem GitHub:

**No servidor:**

```bash
mkdir -p ~/repos/app.git && cd ~/repos/app.git
git init --bare
```

Hook `post-receive` (exemplo que atualiza pasta de trabalho):

```bash
nano ~/repos/app.git/hooks/post-receive
```

```bash
#!/usr/bin/env bash
set -e
WORK_TREE=/home/deploy/apps/repositorio
GIT_DIR=/home/deploy/repos/app.git
git --work-tree="$WORK_TREE" --git-dir="$GIT_DIR" checkout -f main
```

```bash
chmod +x ~/repos/app.git/hooks/post-receive
```

**No local:**

```bash
git remote add producao deploy@IP_DO_SERVIDOR:repos/app.git
git push producao main
```

---

## 9. Checklist rápido

| Item | Feito |
|------|-------|
| Git instalado no servidor | ☐ |
| Chave SSH do servidor no GitHub/GitLab | ☐ |
| Repositório clonado em `~/apps/...` | ☐ |
| `.env` só no servidor | ☐ |
| `git pull` ou script de deploy testado | ☐ |

---

## Problemas comuns

| Sintoma | Solução |
|---------|---------|
| `Permission denied (publickey)` | Conferir chave SSH no GitHub e `ssh -T git@github.com` |
| `fatal: not a git repository` | `cd` para a pasta correta ou rodar `git clone` de novo |
| Conflitos no `git pull` | Resolver localmente no servidor ou resetar branch (cuidado em produção) |
| Arquivos sumiram após pull | Verificar `.gitignore` e se `.env` foi recriado |

---

**Próximo:** [node-pm2-servidor.md](./node-pm2-servidor.md)
