# Projeto Infraestrutura Web Segura com bypass das portas 80 e 443 bloqueadas pelo provedor 

# Introdução

Este documento descreve, em detalhes, a implementação de uma infraestrutura web completa com bypass de uma restrição bastante comum: **bloqueio das portas 80 e 443 pelo provedor de internet**.  

 O objetivo é hospedar, em infraestutura própria e de baixo custo, o domínio **seu-dominio.com** de forma segura usando Let’s Encrypt via DNS-01 (Cloudflare), com o Nginx escutando em 8443 (borda para a Cloudflare), HSTS e cabeçalhos de segurança, e fazendo proxy HTTPS para um <i>backend</i> local. Como tudo ficará no mesmo servidor, para evitar “loop” no próprio Nginx, adotei o backend em 127.0.0.1:443 (HTTPS interno).

O cenário inicial era o seguinte:
- O servidor tinha **IPv4 público válido**.
- O provedor **bloqueava as portas 80 e 443**, impedindo o tráfego HTTP/HTTPS convencional.
- Havia necessidade de servir o domínio `meu-dominio.com` e seus subdomínios por HTTPS válido.
- O ambiente rodava em uma rede doméstica protegida por **roteador profissional**.

**Autor:** José Augusto Zen Ferri — *Infraestrutura e Desenvolvimento Singularys*  
**Servidor:** Ubuntu Server + Nginx
**Domínio:** [https://singularys.net](https://singularys.net)
**Proxy:** Cloudflare (modo Full Strict) 
**Certificados:** Let’s Encrypt (DNS-01)
**Versão 1.0:** 23/10/2025
