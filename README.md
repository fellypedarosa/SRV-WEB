
# 🌐 Projeto de Servidor Web com Docker + Nginx + Cloudflare Tunnel

Turma, queria compartilhar com vcs esse projetinho de servidor pessoal configurado para hospedar o domínio [fellyperosa.com.br](https://fellyperosa.com.br), utilizando Docker, Nginx e Cloudflare Tunnel com foco em segurança, desempenho e automatização de deploys.

> 💡 **Nota:**  
> "Eu sei, eu sei... Pq usar DNS na Cloudflare e ainda um Tunnel de redirecionamento?  
> Meu modem não tem IP fixo, e todas as portas estão bloqueadas kk.  
> Então, na alternativa de entrar em contato com a operadora, é isso que tem pra hoje ✊"

---

## ✅ Estrutura do Projeto

```plaintext
~/nosso-projeto/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
├── certbot/
│   ├── conf/
│   ├── www/
├── html/
│   ├── index.html
│   ├── 404.html
│   ├── vite.svg
│   ├── assets/
```

---

## 1. Instalação do Ubuntu Server

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw fail2ban -y
```

---

## 2. Instalação do Docker e Docker Compose

```bash
curl -fsSL https://get.docker.com | sudo bash
sudo apt install docker-compose -y
sudo usermod -aG docker $USER
```

---

## 3. Firewall e Segurança

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80,443/tcp
sudo ufw enable
```

---

## 4. Configuração do Nginx

Configuração básica com compressão `gzip`, cache de arquivos estáticos, definição explícita de `Content-Type` e suporte a SPA React com fallback para `index.html`.

```nginx
server {
    listen 80;
    server_name fellyperosa.com.br www.fellyperosa.com.br;
    root /usr/share/nginx/html;
    index index.html;

    location /assets/ {
        try_files $uri $uri/ =404;
        expires 30d;
        access_log off;
    }

    location ~* \.js$ {
        add_header Content-Type application/javascript always;
        try_files $uri =404;
    }

    location ~* \.css$ {
        add_header Content-Type text/css always;
        try_files $uri =404;
    }

    location ~* \.svg$ {
        add_header Content-Type image/svg+xml always;
        try_files $uri =404;
    }

    location / {
        try_files $uri /index.html;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```

---

## 5. Cloudflare Tunnel (Zero Trust)

Tunnel criado pelo painel Cloudflare, com `cloudflared` instalado via systemd. O proxy laranja foi ativado e o cache purgado manualmente após deploy.

---

## 6. Deploy e Logs

```bash
cd ~/nosso-projeto
docker-compose up -d
sudo docker logs nginx-container
```

---

## 7. Backup Automatizado

- Agendado via `crontab` para rodar semanalmente.
- Formato `.tar.gz` com todos os arquivos do projeto.
- Política de retenção: mantém os últimos 4 backups.
- Script localizado fora do repositório, com caminho genérico em um disco com RAID 1.

---

## 8. Observações para uma Manutenção Tri

- ⚠️ Sempre realizar *purge cache* no Cloudflare após atualizar arquivos `.js` e `.css`.
- ✔️ Certbot em execução contínua para renovação automática do SSL.
- ✔️ Lembrar de copiar o conteúdo da pasta `dist/` para `html/` após build do React.

---

## 🔒 Próxima melhoria futura: cabeçalhos extras no Nginx

```nginx
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options "nosniff";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

---

## 📊 Arquitetura do Sistema

```plaintext
Internet ---> DNS (Cloudflare Proxy)
        ---> Zero Trust Tunnel
        ---> cloudflared
        ---> Docker Nginx
        ---> HTML + React SPA (fallback routing para /index.html pra tu não tentar injetar besteira kk) 
```
