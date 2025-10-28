# Projeto Infraestrutura Web Segura com bypass das portas 80 e 443 bloqueadas pelo provedor 

# Índice

- [Introdução](#introdução)
  - [Firewall e Port Forwarding no seu roteador](#firewall-e-port-forwarding-no-seu-roteador)
- [1 Configurando o Cloudflare](#1-configurando-o-cloudflare)
  - [1.1 Configurando o redirecionamento de portas](#11-configurando-o-redirecionamento-de-portas)
- [2 Preparação e configuração do servidor UBUNTU](#2-preparação-e-configuração-do-servidor-ubuntu)
  - [2.1 Atualizar o sistema](#21-atualizar-o-sistema)
  - [2.2 Instalar o NGINX, PHP-FPM e php-my-sql](#22-instalar-o-nginx-php-fpm-e-php-my-sql)
  - [2.2 Emissão do certificado](#22-emissão-do-certificado)
  - [2.3 Solução Arquitetural: Proxy Cloudflare + Certificado DNS-01](#23-solução-arquitetural-proxy-cloudflare--certificado-dns-01)
  - [2.4 Configuração do Certbot com DNS Cloudflare](#24-configuração-do-certbot-com-dns-cloudflare)
- [3 Criação da Arquitetura Proxy](#3-criação-da-arquitetura-proxy)
  - [3.1 Backend (seu-dominio-backend)](#31-backend-seu-dominio-backend)
- [4 Conclusão](#4-conclusão)
- [Histórico de implantação](#histórico-de-implantação)

# Introdução

Este documento descreve, em detalhes, a implementação de uma infraestrutura web completa com bypass de uma restrição bastante comum: **bloqueio das portas 80 e 443 pelo provedor de internet**.  

 O objetivo é hospedar, em infraestutura própria e de baixo custo, o domínio **seu-dominio.com** de forma segura usando Let’s Encrypt via DNS-01 (Cloudflare), com o Nginx escutando em 8443 (borda para a Cloudflare), HSTS e cabeçalhos de segurança, e fazendo proxy HTTPS para um <i>backend</i> local. Como tudo ficará no mesmo servidor, para evitar “loop” no próprio Nginx, adotei o backend em 127.0.0.1:443 (HTTPS interno).

O cenário inicial era o seguinte:
- O servidor tinha **IPv4 público válido**.
- O provedor **bloqueava as portas 80 e 443**, impedindo o tráfego HTTP/HTTPS convencional.
- Havia necessidade de servir o domínio `meu-dominio.com` e seus subdomínios por HTTPS válido.
- O ambiente rodava em uma rede doméstica protegida por **roteador profissional**.

Obs: O roteador varia entre ambientes e por isso vamos tratá-lo de forma genérica. Creio ser possível utilizar até roteadores domésticos que sejam capazes de fazer encaminhamento de porta (port forwarading). No entanto, o uso de **roteadores domésticos** pode **comprometer significativamente a segurança do servidor**, de modo que desencorajamos o seu uso em ambientes de produção. 

## Firewall e Port Forwarding no seu roteador

Com as portas 80/443 bloqueadas na WAN pelo provedor, foi aberta a **porta 8443** para uso alternativo.

Regras aplicadas:

- **Port Forward:** `WAN 8443 → IP-INTERNO-SERVIDOR:8443`
- **Firewall WAN IN:** Permitir `TCP 8443` para o IP do servidor.


# 1. Configurando o Cloudflare

Aqui vamos tratar das configurações gerais do Cloudflare com a finalidade de solucionar a problemática inicial. Não entrarei em detalhes de interface do cloudflare porque ela pode mudar a qualquer momento. Os ajustes servem como referência. Aplique o bom senso!

Suponho que você já tenha uma conta no cloudflare e um domínio registrado por ele. Todas as configurações funcionam na conta gratuíta (testado em 23/10/2025 - isso pode mudar a qualquer tempo). Seu único custo será com a aquisição do domínio.

Acesse sua conta. Na dashboard, acesse: 

**Domain Registration** --> **Manage domains**

Você verá seu domínio com <i>status</i> **active**. Clique em **Manage**.

Procure por **Quick actions**. Atualmente, na data que escrevo esse manual (23/10/2025) está na lateral direita. Selecione, então, **Update DNS configuration**. 

Em **DNS Records**, caso não esteja configurado, clique em **ADD RECORD** para adiconar dois registros.

Primeiro registro

```
Type: A; Name: seu-dominio.com; IPv4: Seu IP público; Proxy status: ativado - proxied (ficará alaranjado). Save.
````

Segundo registro: 

```
Type: CNAME; Name: www; Targe: seu-dominio.com; Proxy status: ativado - proxied (ficará alaranjado). Save.
````

**ATENÇÃO**

Agora vamos gerar o Token que será utilizado para gerar os certificados no Let's Encrypt. Como as portas 80 e 443 estão bloqueadas, os certificados serão gerados por DNS - Let's Encrypt - DNS-01 Challenge. 

Os tokens criados para Certbot requerem permissões de **Zone:Zone:Read** e **Zone:DNS:Edit**. Vamos configurar os Tokens no Cloudflare com esses requisitos. 

Na dashboard do Cloudflare, vá em:

```
Profile -> { } API Tokens -> Create Token -> Edit Zone DNS -> Clique Use template
````

```
Permissions: zone:dns:edit e zone:zone:read

Zone Resources: include: all zones

Continue to summary

Create token
```

**Copie e guarde o token gerado**, ele será usado no passo de criação dos certificados. 

## 1.1. Configurando o redirecionamento de portas

Retorne para 
- **Domain Registration** 
    - **Manage domains**
        - **Quick actions**:
            Procure por Rules no menu à esquerda e clique em **Overview**
            Em Templates, procure por **Change Port**. Clique. Marque **all incoming requests** em **Rewrite to** coloque a porta 8443. Salve. Essa regra fará com que o Cloudflare receba a requisição em 443 padrão e repasse para o seu servidor na 8443 de forma transparente para o usuário. 


No painel Cloudflare → Rules → Origin Rules crie uma regra:

If Hostname equals **seu-dominio.com** → Override Origin Port = 8443.
Isso instrui a Cloudflare a conectar sempre na 8443 da sua origem quando o cliente vier em 443. Resultado: https://seu-dominio.com funcionará com sua atual borda Nginx na 8443.

# 2. Preparação e configuração do servidor UBUNTU

Para este manual, utilizamos a distribuição **UBUNTU 24.04.3 LTS, codename: noble**. 

Suponho que já tenha insalado o Ubuntu e que nenhum outro serviço esteja utilizando as portas 80, 443, 8443. 

### 2.1. Atualizar o sistema

Vamos atualizar o sistema. Tudo que faremos aqui utiliza a versão atualizada até 23/10/2025. 

```bash
sudo apt update && sudo apt upgrade -y
````

### 2.2. Instalar o NGINX, PHP-FPM e php-my-sql

```bash
sudo apt install -y nginx php-fpm php-mysql
````

Para habilitar e iniciar o PHP-FPM, você precisa de um nome exato do serviço (como php8.3-fpm). Para listar as unidades instaladas, você pode usar o comando: 

```bash
systemctl list-units 'php*-fpm.service' --all
````

Depois, para iniciar:

```bash
sudo systemctl enable --now php8.3-fpm
````

Verifique se o PHP está ativo: 

```bash
systemctl status php8.3-fpm
````

A resposta esperada será algo como 

```bash
● php8.3-fpm.service - The PHP 8.3 FastCGI Process Manager
     Loaded: loaded (/usr/lib/systemd/system/php8.3-fpm.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-10-28 19:05:37 UTC; 15min ago
       Docs: man:php-fpm8.3(8)
   Main PID: 15519 (php-fpm8.3)
     Status: "Processes active: 0, idle: 2, Requests: 0, slow: 0, Traffic: 0req/sec"
      Tasks: 3 (limit: 9408)
     Memory: 8.0M (peak: 8.8M)
        CPU: 63ms
     CGroup: /system.slice/php8.3-fpm.service
             ├─15519 "php-fpm: master process (/etc/php/8.3/fpm/php-fpm.conf)"
             ├─15521 "php-fpm: pool www"
             └─15522 "php-fpm: pool www"

Oct 28 19:05:37 webxdsm systemd[1]: Starting php8.3-fpm.service - The PHP 8.3 FastCGI Process Manager...
Oct 28 19:05:37 webxdsm systemd[1]: Started php8.3-fpm.service - The PHP 8.3 FastCGI Process Manager.
```

Interpretação por linhas: 

```
	•	Loaded: loaded (...) enabled → o serviço está configurado para iniciar automaticamente.
	•	Active: active (running) → o PHP-FPM está ativo e pronto para aceitar conexões.
	•	Main PID → identifica o processo principal do PHP-FPM.
	•	Status: → mostra o número de processos em uso (idle e ativos).
	•	CGroup: → lista o processo mestre e os “workers” do pool www.
```

Se você recebeu alguma mensagem de erro, ou algo como **inactive (dead)** ou **failed**, o serviço não iniciou corretamente. Refaça os paços conferindo a versão do seu PHP.

Para conferir a versão: 

```bash
ls /run/php/
```

Você verá algo como: 

```
php8.3-fpm.sock
php8.3-fpm.pid
```

---

## 2.2 Emissão do certificado

Você irá falhar se tentar emitir o certificado pelo modo tradicional, com desafio nas portas 80 e 443, pois o Let's Encrypt não consegue validar o HTTP.  

O log de erro irá exibir:

```
Timeout during connect (likely firewall problem)
DNS problem: looking up A for www.seu-dominio.com
```

O bloqueio das portas **80** e **443** pelo provedor impossibilita a emissão e a renovação de certificados pelo método padrão `HTTP-01`.

---

## 2.3. Solução Arquitetural: Proxy Cloudflare + Certificado DNS-01

A saída adotada foi utilizar o **Cloudflare como proxy reverso público** (porta 443) e o **Certbot com o plugin DNS-01 via Cloudflare API**, o que permite validar o domínio diretamente via registros DNS, sem necessidade de abrir as portas 80 ou 443.

### 2.4. Configuração do Certbot com DNS Cloudflare

**Lembre-se de adaptar os comandos e os códigos**. Substitua **seu-dominio.com** e **www.seu-dominio.com** com seus dados de domínio. 

Criação do arquivo de credenciais:

```bash
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini
```

Conteúdo:
```
dns_cloudflare_api_token = <TOKEN_DA_CLOUDFLARE>
```
**ATENÇÃO**: Lembre-se de retirar os <> do Token!

Permissões:

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

Geração do certificado (modo DNS-01):

```bash
sudo certbot certonly   --dns-cloudflare   --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini   -d seu-dominio.com -d www.seu-dominio.com
```

Resultado esperado:

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/singularys.net/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/singularys.net/privkey.pem
```

---

# 3. Criação da Arquitetura Proxy

O Nginx atua agora com **dois papéis**:

1. **Borda (pública)** — ouvindo na porta **8443** (posteriormente, na 443 pública via Cloudflare).
2. **Backend (interno)** — servindo conteúdo HTTPS localmente em `127.0.0.1:443`.

### 3.1. Backend (`seu-dominio-backend`)

**ATENÇÃO!** Lembre-se de adaptar os comandos e os códigos. Substitua **seu-dominio.com** e **www.seu-dominio.com** com seus dados de domínio. 

```bash
sudo nano /etc/nginx/sites-available/seu-dominio
````

```nginx
# Borda: Cloudflare -> sua origem:8443 (TLS na borda)
server {
    listen 8443 ssl http2;
    server_name seu-dominio.com www.seu-dominio.com;

    # Cert público (LE via DNS-01)
    ssl_certificate     /etc/letsencrypt/live/singularys.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/singularys.net/privkey.pem;

    # TLS
    ssl_session_timeout 1d;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Cabeçalhos de segurança
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Proxy HTTPS para o backend local em 127.0.0.1:443 (com SNI e verificação)
    location / {
        proxy_pass https://127.0.0.1:443;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Connection "";

        proxy_ssl_server_name on;  # envia SNI
        proxy_ssl_name $host;      # SNI = host solicitado
        proxy_ssl_verify on;       # verifica o cert do backend
        proxy_ssl_verify_depth 3;
    }
}

# Opcional: 8880 só para testes locais → redireciona para HTTPS na 8443
server {
    listen 8880;
    server_name seu-dominio.com www.seu-dominio.com;;
    return 301 https://$host:8443$request_uri;
}
```

Habilite e recarregue com:

```bash
sudo ln -s /etc/nginx/sites-available/seu-dominio /etc/nginx/sites-enabled/seu-dominio
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
````

No Cloudflare, mantenha o DNS com proxy ativado (laranja) e SSL. Garanta que a 8443 esteja liberada na WAN e no firewall do host. Para testar sem depender do DNS, use:

Liberando as portas no firewall do ubuntu, caso ainda não tenha feito:

```bash
sudo ufw allow 8443/tcp
````

---

# 4. Conclusão

O projeto demonstrou que é plenamente viável operar uma infraestrutura HTTPS completa **mesmo sob bloqueio das portas 80 e 443**, desde que se empreguem:

- Validação DNS-01 via Let’s Encrypt;
- Proxy reverso Cloudflare;
- Arquitetura Nginx dupla (borda + backend);
- Port forwarding e firewall configurados corretamente;
- Certificados TLS otimizados e verificação SNI consistente.

O resultado final é um ambiente estável, seguro, validado e escalável, permitindo hospedar múltiplos sites e aplicações web dentro do mesmo domínio, com o desempenho e a estética.

---

# Histórico de implantação

| Etapa | Resultado |
--------|------------|
| Certbot DNS-01 com Cloudflare | Certificado emitido |
| Configuração Nginx Proxy | Backend ativo em 127.0.0.1:443 |
| Port Forward 8443 | Acesso externo confirmado |
| Correção do SSL Proxy | Comunicação segura interna |
| Abertura da 443 no provedor | Erro 522 resolvido |
| Deploy do site | Operação completa e validada |

---

**Autor:** José Augusto Zen Ferri — *Infraestrutura e Desenvolvimento Singularys*<br> 
**Servidor:** Ubuntu Server + Nginx <br>
**Domínio:** [https://singularys.net](https://singularys.net)<br>
**Proxy:** Cloudflare (modo Full Strict)<br>
**Certificados:** Let’s Encrypt (DNS-01)<br>
**Versão 1.0:** 23/10/2025<br>


