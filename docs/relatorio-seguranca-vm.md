# Relatório de Segurança — VM Oracle (136.248.75.34)

**Data da auditoria:** 2026-07-22
**Escopo:** VM Ubuntu 24.04.4 LTS (Oracle Cloud) que hospeda a API em Docker + nginx.
**Método:** inspeção somente-leitura via SSH (portas, firewall, sshd, usuários, logs, nginx, TLS, Docker).

---

## Resumo executivo

A base está razoável — firewall UFW ativo com política default-deny, SSH só por chave, TLS válido (Let's Encrypt), atualizações de segurança automáticas ligadas. **Mas existem 3 brechas importantes**:

1. 🔴 **A porta 3000 da API está exposta diretamente pra internet**, contornando o nginx e o UFW (o Docker fura o firewall). Qualquer um pode falar com a API sem passar pelo proxy/TLS.
2. 🔴 **~3.900 tentativas de invasão SSH nas últimas 24h e nenhum bloqueio automático** (fail2ban ausente). É ataque de força bruta contínuo — hoje inofensivo porque senha está desabilitada, mas é ruído, consumo de recurso e risco se algo mudar.
3. 🟠 **Reboot pendente com kernel novo** — a VM está de pé há 94 dias com patches de kernel aplicados mas não carregados.

Nenhum sinal de comprometimento foi encontrado (logins históricos só do seu IP, 1 chave por usuário, Docker API não exposta).

---

## Achados por nível de severidade

### 🔴 CRÍTICO

#### C1. Porta 3000 (API no container) aberta pra internet inteira

- `docker-compose.prod.yaml` publica `"3000:3000"`, que vira `0.0.0.0:3000`.
- **O Docker insere regras iptables ANTES do UFW** — o UFW diz "só 22/80/443", mas a chain DOCKER aceita tráfego externo direto pro container (`ACCEPT tcp dpt:3000 → 172.18.0.2`). Confirmado no `iptables -L DOCKER`.
- Consequência: um atacante fala com o Node/NestJS **direto, sem TLS, sem nginx** — vaza a possibilidade de sniffing de tokens em trânsito, ignora qualquer proteção futura colocada no nginx (rate limit, WAF), e expõe a superfície do Express cru.
- **Correção (1 linha):** no compose, trocar `"3000:3000"` por `"127.0.0.1:3000:3000"`. O nginx continua funcionando (ele faz proxy pra `localhost:3000`) e a porta some da internet.

#### C2. Força bruta SSH sem nenhum bloqueio automático

- 3.894 falhas de autenticação em 24h, de dezenas de IPs (botnets varrendo a porta 22).
- `fail2ban` não está instalado. Cada bot pode tentar pra sempre.
- Hoje o risco real é baixo (senha desabilitada, só chave), mas: consome CPU/RAM numa VM de 1 GB, enche os logs, e se um dia alguém habilitar senha por engano, vira invasão em horas.
- **Correção:** instalar `fail2ban` (jail sshd, ban de 1h após 5 falhas). 5 minutos de trabalho.

### 🟠 ALTO

#### A1. Reboot pendente há muito tempo (kernel desatualizado em execução)

- `/var/run/reboot-required` presente; uptime de 94 dias. Os patches de kernel foram baixados mas o kernel antigo continua rodando — vulnerabilidades de kernel corrigidas "no disco" seguem exploráveis "na memória".
- **Correção:** `sudo reboot` numa janela tranquila (downtime ~2–3 min; o container tem restart automático via compose).

#### A2. Root ainda pode logar por SSH

- `PermitRootLogin without-password` (default do Ubuntu) + existe uma chave em `/root/.ssh/authorized_keys` (com restrições `no-port-forwarding` etc., provavelmente da Oracle). Você nunca precisa logar como root — usa `ubuntu` + sudo.
- **Correção:** `PermitRootLogin no` no sshd_config. Reduz a superfície do alvo nº 1 dos bots (todo bot tenta `root` primeiro).

#### A3. Sem rate limiting no nginx (camada de rede)

- Nenhum `limit_req`/`limit_conn`. A aplicação tem `ThrottlerModule` (bom!), mas o throttle em Node ainda gasta CPU do Node pra rejeitar. Numa VM de 1 vCPU/1 GB, um flood barato derruba a API (DoS de pobre).
- **Correção:** `limit_req_zone` no nginx — barato, rejeita antes de chegar no Node. (Não elimina DDoS de verdade — pra isso seria Cloudflare na frente — mas elimina abuso trivial.)

### 🟡 MÉDIO

- **M1. `.env.prod` legível por todos (644) + 3 backups soltos** com todos os segredos (banco Neon, R2, JWT). Qualquer processo/usuário local lê. → `chmod 600 .env.prod*` e apagar os `.bak` antigos.
- **M2. Usuário `opc` sobrando** com shell e chave SSH — resquício de template Oracle, nunca usado. Cada conta a mais é uma porta a mais. → travar (`usermod -L opc -s /usr/sbin/nologin`).
- **M3. `X11Forwarding yes`** no sshd — inútil num servidor, superfície extra. → `no`.
- **M4. Site `default` do nginx ativo** na porta 80 servindo `/var/www/html` pra qualquer Host — responde a scanners com página default (fingerprinting fácil) e aceita host arbitrário. → remover o site default e retornar 444 pra hosts desconhecidos.
- **M5. Sem cabeçalhos de segurança / helmet não detectado** no bootstrap da API. → adicionar `helmet()` no NestJS ou os headers no nginx (`X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`).

### 🟢 BAIXO / observações

- **B1. MaxAuthTries 6** — pode baixar pra 3.
- **B2. Memória apertada** (954 MB, ~508 MB disponíveis; swap já em uso). Não é falha de segurança, mas OOM sob ataque leve vira indisponibilidade. Monitorar.
- **B3. Security List da Oracle (firewall da nuvem)** não foi auditada daqui de dentro — vale conferir no painel OCI que só 22/80/443 estão liberadas lá também (defesa em profundidade; hoje ela provavelmente está deixando a 3000 passar, já que o teste da porta chegou no docker-proxy). Idealmente, restringir a 22 ao seu IP no painel OCI.

### ✅ O que já está bom

- UFW ativo, default deny incoming, só 22/80/443.
- SSH: senha desabilitada, só chave pública; `PermitEmptyPasswords no`.
- TLS Let's Encrypt válido (expira 31/08/2026, renovação certbot) + redirect 80→443.
- `unattended-upgrades` ativo (patches de segurança automáticos).
- Docker API (2375) não exposta; 1 container só; imagens antigas limpas no deploy.
- Sem sinais de comprometimento: logins históricos só do IP conhecido, 1 chave autorizada por usuário.

---

## Principais vetores de ataque (contra o que estamos protegendo)

| Vetor | Estado atual | Após o plano |
|---|---|---|
| Força bruta SSH | Sem bloqueio (3.9k tentativas/dia) | fail2ban bane; root vetado |
| Acesso direto à API sem TLS/proxy | **Possível (porta 3000 aberta)** | Porta fechada, só via nginx+TLS |
| Flood/DoS barato | Só throttle no Node | Rejeição barata no nginx |
| Exploit de kernel conhecido | Kernel antigo em execução | Reboot carrega kernel corrigido |
| Vazamento de segredos local | .env 644 + backups | 600, backups removidos |
| Fingerprinting/scanning | Página default do nginx responde | 444 pra host desconhecido |
| Clickjacking/sniffing MIME | Sem headers | helmet/headers no nginx |

---

## Plano de ação

### Nível 1 — Fazer agora (fecha as brechas reais; ~20 min, downtime ~3 min)

1. **Fechar a porta 3000** — no `docker-compose.prod.yaml`: `ports: ["127.0.0.1:3000:3000"]`, depois `docker compose down && up -d`. Testar de fora que `curl http://136.248.75.34:3000` agora falha e o site continua ok via 443.
2. **Instalar fail2ban** — `apt install fail2ban`, jail `sshd` habilitado (`maxretry 5`, `bantime 1h`, `findtime 10m`).
3. **Endurecer o sshd** — `PermitRootLogin no`, `X11Forwarding no`, `MaxAuthTries 3` → `systemctl reload ssh` (testar novo login ANTES de fechar a sessão atual).
4. **Reboot** pra carregar o kernel atualizado.

### Nível 2 — Esta semana (~30 min)

5. **Rate limit no nginx** — `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;` + `limit_req zone=api burst=30 nodelay;` no location, com zona mais estrita (ex.: 3r/m) só para `/api/v1/auth/email/login` e endpoints de senha.
6. **Segredos** — `chmod 600 ~/app/.env.prod`, apagar os 3 `.env.prod.bak*`.
7. **Desativar usuário `opc`** — `sudo usermod -L -s /usr/sbin/nologin opc`.
8. **Limpar nginx** — remover site `default`, adicionar server block catch-all `return 444;`, adicionar headers de segurança (HSTS, X-Frame-Options DENY, X-Content-Type-Options nosniff).
9. **Helmet na API** — `app.use(helmet())` no `main.ts` (mudança de código, via PR normal).

### Nível 3 — Quando der (defesa em profundidade)

10. **Painel OCI**: conferir/enxugar a Security List (só 22/80/443; idealmente 22 restrito ao seu IP ou faixa).
11. **Considerar Cloudflare** (proxy gratuito) na frente do domínio: esconde o IP real, absorve DDoS, WAF básico. Exigiria usar o domínio duckdns/próprio em vez do nip.io.
12. **Auditoria periódica**: rodar `apt list --upgradable`, checar `fail2ban-client status sshd` e `docker ps` uma vez por mês; `unattended-upgrades` já cobre o grosso.
13. **Backup/replay**: como o banco é Neon (gerenciado, fora da VM), a VM é descartável — manter o compose + .env copiados em local seguro fora da VM garante reconstrução em minutos se algo acontecer.

---

---

## Status de execução (2026-07-22)

**Executado:** todos os itens dos Níveis 1 e 2, mais backup do compose/.env fora da VM (item 13). Detalhes: porta 3000 bindada em 127.0.0.1 (na VM e no repo via PR #90, com helmet), fail2ban ativo, sshd endurecido (root/X11 off, MaxAuthTries 3), reboot feito (kernel novo), rate-limit nginx (10r/s geral, 10r/m em login/registro/senha), headers de segurança, site default removido (444), `.env.prod` 600 + backups apagados, usuário `opc` desativado.

**Pendentes (dependem do usuário):** item 10 (Security List no painel OCI) e item 11 (decisão sobre Cloudflare).
