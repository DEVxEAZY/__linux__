# Cookbook — Deploy em servidor Linux

Guias práticos (receitas) para publicar aplicações Node em um VPS Linux.

| # | Tema | Arquivo |
|---|------|---------|
| 164 | Git e envio de arquivos ao servidor | [git-servidor.md](./git-servidor.md) |
| 165 | Node.js e PM2 | [node-pm2-servidor.md](./node-pm2-servidor.md) |
| 166 | NGINX (proxy reverso) | [nginx-proxy-reverso.md](./nginx-proxy-reverso.md) |

## Ordem recomendada

1. Preparar o servidor e o Git (**164**)
2. Instalar Node e subir a app com PM2 (**165**)
3. Expor a app na porta 80/443 com NGINX (**166**)

## Pré-requisitos gerais

- VPS Linux (Ubuntu/Debian é o mais comum nos tutoriais)
- Acesso SSH como usuário com `sudo`
- Domínio apontando para o IP do servidor (opcional até o passo do NGINX)
