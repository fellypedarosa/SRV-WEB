
# 🌐 Projeto de Servidor Web com Docker + Nginx + Cloudflare Tunnel

Turma, queria compartilhar com vcs esse projetinho de servidor pessoal configurado para hospedar meu domínio [fellyperosa.com.br](https://fellyperosa.com.br), utilizando Docker, Nginx e Cloudflare Tunnel com foco em segurança, desempenho e automatização de deploys.

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
│   ├── assets/
```

---
## 1. Redes Docker

O ambiente agora utiliza **duas redes externas no Docker**:

|Rede|Função|
|---|---|
|site|Comunicação entre Nginx, Certbot e o site React|
|gitlab|Comunicação entre Nginx e o container GitLab|

> ✅ O Nginx participa das duas redes simultaneamente.

---

## 2. Cloudflare Tunnel (Zero Trust)

- **Tunnel criado via painel web da Cloudflare** (modo Token).
    
- Todos os **Public Hostnames** (ex: `fellyperosa.com.br`, `gitlab.fellyperosa.com.br`) **são gerenciados diretamente pelo painel da Cloudflare**, via **Zero Trust Dashboard**.
    
- **O arquivo `/etc/cloudflared/config.yml` está obsoleto e não é mais usado**.
    
- O serviço `cloudflared.service` roda com `--token` e carrega tudo remoto.

---

## 3. Configuração do Nginx

### Site Principal (`fellyperosa.com.br`)

✔️ Serve arquivos estáticos (HTML, CSS, JS).  
✔️ Redireciona `/assets/`, `/index.html` e outras rotas SPA.  
✔️ Suporte a Certbot para SSL local.

---

### Subdomínio GitLab (`gitlab.fellyperosa.com.br`)

✔️ Faz proxy reverso para o container GitLab (`172.30.0.10:8929`).  
✔️ **Sem SSL local** (HTTPS termina na Cloudflare).  
✔️ Só responde na porta 80 (a Cloudflare faz o HTTPS no navegador).

Blocos no nginx.conf:
```conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    access_log    /var/log/nginx/access.log;
    error_log     /var/log/nginx/error.log warn;
    
    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;

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

        location / {
            try_files $uri /index.html;
        }

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        error_page 404 /404.html;
        location = /404.html {
            root /usr/share/nginx/html;
            internal;
        }
    }

        server {
        listen 80;
        server_name gitlab.fellyperosa.com.br;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            proxy_pass http://172.30.0.10:8929;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
}

```

---

## 4. Docker-Compose
```yml
version: '3'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: 'gitlab.local'
    environment:
      GITLAB_OMNIBUS_CONFIG: | 
        external_url 'https://gitlab.fellyperosa.com.br' 
        nginx['listen_port'] = 8929                                                                      
        nginx['listen_https'] = false                                                               
        external_https = true                                                                 
    ports:
      - "8920:8929"   # Porta web externa → interna
      - "2222:22"     # Porta SSH externa → interna
    networks: 
      gitlab:
        ipv4_address: 172.30.0.10
    volumes:
      - '/mnt/HD/BANCO/gitlab-banco:/var/opt/gitlab' 
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'

networks:
  gitlab:
    external: true

```

## 5. Deploy e Logs

```bash
cd ~/nosso-projeto
docker-compose up -d
docker exec -it nginx-fellype nginx -t
docker exec -it nginx-fellype nginx -s reload
```

---

## 6. Backup Automatizado

- Agendado via `crontab` para rodar semanalmente.
- Formato `.tar.gz` com todos os arquivos do projeto.
- Política de retenção: mantém os últimos 4 backups.
- Script localizado fora do repositório, com caminho genérico em um disco com RAID 1.

---

## 7. Observações para uma Manutenção Tri

- ⚠️ Sempre realizar *purge cache* no Cloudflare após atualizar arquivos `.js` e `.css`.
- ✔️ Certbot em execução contínua para renovação automática do SSL.
- ✔️ Lembrar de copiar o conteúdo da pasta `dist/` para `html/` após build do React.
- ✔️ Não mexer mais no config.yml do Cloudflared!
- ✔️ Qualquer novo subdomínio → criar primeiro no painel da Cloudflare Zero Trust → Public Hostnames
- ✔️ Toda vez que adicionar novo bloco no Nginx → testar (nginx -t) e reload (nginx -s reload)
- ✔️ Se precisar adicionar SSL local para algum serviço no futuro → Configurar Certbot conforme o modelo do site principal.

---

## 📊 Arquitetura do Sistema

```plaintext
Internet
  ↓
Cloudflare DNS (Proxy Laranja)
  ↓
Cloudflare Zero Trust Tunnel
  ↓
cloudflared (via Token)
  ↓
Nginx (Docker)
  ↓
→ Site React (porta 80)
→ GitLab (via proxy para 172.30.0.10:8929)
```
