# BACKUP & RECOVERY — Homelab

> **Runbook operacional e de recuperação**
>
> Documento vivo. Atualizar este ficheiro sempre que a arquitetura, um caminho, um serviço, uma credencial, um agendamento ou um procedimento de recuperação mudar.
>
> **Última atualização:** 04/10/2026  
> **Estado:** implementação em curso — Fase 1 (Proxmox)

---

## 0. Se eu não me lembrar de nada

Este documento é o ponto de entrada para operar e recuperar o sistema de backups.

### Regras fundamentais

1. **O Homelab Register não executa backups.** É apenas observabilidade/reporting.
2. **Não depender do Homelab Register para recuperar o homelab.**
3. **Não apagar repositórios ou backups antigos durante uma recuperação sem perceber primeiro a retenção e as dependências.**
4. As passwords do Restic e do `rclone crypt` **não são escritas neste documento**.
5. As credenciais ficam no gestor de passwords e numa cópia em papel.
6. Um backup só é considerado realmente validado depois de um teste de restauro — os testes de recuperação ficam para a fase de validação definida no plano.
7. **Este runbook não pode existir apenas no homelab.** Deve existir uma cópia impressa junto das credenciais em papel e uma cópia adicional fora de casa.
8. O runbook deve ser utilizável sem Unraid, sem rede doméstica e sem Homelab Register, sempre que os procedimentos de recuperação assim o permitam.

---

# 1. Arquitetura

## 1.1 Visão geral

```text
                         BACKUPS
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
        pve01             pve02           restantes
          │                 │              origens
          │                 │                 │
       vzdump            vzdump             Restic
          │                 │                 │
          └────────────┬────┴─────────────────┘
                       ▼
                    UNRAID
                 share: backups
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Synology          Off-site
          offline             S3
```

O Unraid é o centro local dos backups.

O desenho pretende cumprir 3-2-1:

- 3 cópias;
- 2 sistemas diferentes;
- 1 cópia fora de casa.

O Synology funciona como cópia offline. O off-site será cifrado antes de sair de casa.

## 1.2 Proxmox: duas pipelines independentes

### VMs/LXC

```text
pve01 / pve02
     │
     │ vzdump
     ▼
Unraid /mnt/user/proxmox/dump/
     │
     │ rsync
     ▼
Unraid /mnt/user/backups/proxmox/
```

### Configuração do host

```text
pve01 / pve02
     │
     │ restic backup /etc
     ▼
rest-server no Unraid
     │
     ▼
repositório privado do nó
```

**Importante:** o Restic não cria uma pasta `/etc` navegável no Unraid. O conteúdo fica dentro do repositório Restic, em formato deduplicado, comprimido e cifrado.

---

# 2. Inventário

| Máquina | Papel | Backup previsto |
|---|---|---|
| pve-01 | Proxmox | VMs/LXC via vzdump + `/etc` via Restic |
| pve-02 | Proxmox | VMs/LXC via vzdump + `/etc` via Restic |
| Unraid | Centro de armazenamento | Immich, appdata e repositórios locais |
| Synology DS216j | Cópia offline | Réplica da share `backups` |
| OPNsense | Firewall | `config.xml` |
| VPS edge | nginx, WireGuard, Uptime Kuma | `/etc/nginx`, `/etc/wireguard`, SQLite do Kuma |
| VPS register | Homelab Register | `data/` |

---

# 3. Unraid — centro dos backups

## 3.1 Share

Share principal:

```text
backups
```

Subpastas previstas:

```text
/mnt/user/backups/
├── proxmox/
└── restic/
```

O `proxmox/` recebe os dumps replicados a partir da share `proxmox`.

O `restic/` contém os repositórios usados pelo rest-server.

## 3.2 Rest-server

Container:

```text
restic-rest-server
```

Imagem:

```text
restic/rest-server
```

Configuração importante:

```text
Network: Bridge
Privileged: OFF
Container data: /data
Porta: 8000
```

Opções:

```text
--append-only --private-repos
```

Características:

- autenticação ativa;
- repositórios privados;
- modo append-only;
- cada origem tem utilizador próprio.

### HTTP — decisão consciente

O rest-server está atualmente em HTTP simples na rede interna. A cifra do Restic protege o conteúdo armazenado, mas a password de autenticação HTTP pode circular em claro na LAN.

Esta decisão é aceite porque:

- a LAN doméstica é considerada uma rede confiável para este serviço;
- as VPS não devem aceder diretamente por Internet pública;
- o acesso remoto previsto às VPS passa por WireGuard.

Se o modelo de confiança da rede mudar, deve ser revista a utilização de TLS.

### Utilizadores criados

```text
pve-01
pve-02
opnsense
vps-edge
vps-register
```

**Passwords não documentadas aqui.**

---

# 4. Proxmox

## 4.1 VZDump

O Proxmox escreve primeiro na share existente:

```text
/mnt/user/proxmox/dump/
```

Depois um User Script do Unraid replica para:

```text
/mnt/user/backups/proxmox/
```

### Script no Unraid

O script mantém deliberadamente a cópia em `backups/proxmox` fora do alcance de uma remoção propagada pelo `rsync --delete`.

```bash
#!/bin/bash

SOURCE="/mnt/user/proxmox/dump/"
DEST="/mnt/user/backups/proxmox/"

rsync -avh "$SOURCE" "$DEST"
exit $?
```

**Decisão importante — sem `--delete`:**

O `--delete` não é usado de propósito. Se um Proxmox for comprometido, ou se os dumps da origem forem apagados/corrompidos por engano, essa remoção não deve ser propagada para a segunda cópia local. A share `backups/proxmox` funciona assim como uma cópia com alguma independência da retenção da origem.

Como contrapartida, os dumps antigos acumulam-se. Por isso deve existir uma **limpeza independente por idade**, executada no Unraid e documentada/testada separadamente. A idade exata de retenção dessa cópia ainda fica por definir; não substituir esta limpeza por `--delete`.

### Agendamento

**Domingo 03:15**, depois dos dois jobs VZDump.

Cron do User Script no Unraid:

```text
15 3 * * 0
```

### Limpeza por idade

A limpeza ocorre depois de um `rsync` concluído com sucesso e é independente do `--delete`.

Política atual:

- retenção de 30 dias na cópia `backups/proxmox`;
- só um dump com mais de 30 dias pode ser candidato;
- só é removido se existir outro dump mais recente do mesmo tipo e VMID;
- o conjunto `.tar.zst`, `.log` e `.tar.zst.notes` é removido em conjunto;
- se não existir outro backup mais recente da mesma VM/CT, o dump é preservado;
- se o `rsync` falhar, a limpeza não é executada.

O script foi testado manualmente e, com os dados atuais, não encontrou candidatos.

### Exemplo real já criado

Foi produzido um dump do CT 101:

```text
vzdump-lxc-101-2026_10_04-13_21_11.tar.zst
```

e replicado para `backups/proxmox`.

---

# 4.0 Jobs VZDump

Os dumps das VMs/LXC são gerados pelo próprio Proxmox e inicialmente escritos na share:

```text
/mnt/user/proxmox/dump/
```

## pve-01

```text
Domingo 01:00
```

Job Proxmox real:

```text
backup-b205ebd8-7605
```

Configuração:

- node: `pve01`;
- storage: `unraid`;
- modo: `snapshot`;
- compressão: `zstd`;
- retenção: `keep-weekly=2`;
- guests: `100,101,102,110,120,121,130,150,160`;
- enabled: `1`.

## pve-02

```text
Domingo 01:45
```

Job Proxmox real:

```text
backup-3d7b9278-729e
```

Configuração:

- node: `pve02`;
- storage: `unraid`;
- modo: `snapshot`;
- compressão: `zstd`;
- retenção: `keep-weekly=2`;
- guest: `140`;
- enabled: `1`.

### Localização da configuração

Na instalação atual (Proxmox 9.2), os jobs criados pela interface estão em:

```text
/etc/pve/jobs.cfg
```

O ficheiro:

```text
/etc/pve/vzdump.cron
```

permanece vazio porque não estamos a usar o mecanismo legado de agendamento. O `jobs.cfg` é cluster-wide e deve ser a referência para estes jobs.

A retenção `keep-weekly 2` aplica-se à **origem `proxmox/dump`**. A cópia replicada para `backups/proxmox` tem política independente e não usa `--delete`.

### Ordem intencional

O rsync para `backups/proxmox` deve ocorrer apenas depois de ambos os jobs VZDump terminarem:

```text
01:00  pve-01 vzdump
01:45  pve-02 vzdump
03:15  rsync proxmox/dump → backups/proxmox
```

Isto reduz o risco de o rsync apanhar um dump ainda em escrita.

---

# 5. Restic — pve-01

## 5.1 Cliente

Versão instalada:

```text
restic 0.18.0
```

## 5.2 Repositório

URL:

```text
rest:http://10.10.0.20:8000/pve-01
```

Utilizador HTTP:

```text
pve-01
```

O repositório Restic foi inicializado com sucesso.

ID observado:

```text
4b1fcbc1be
```

## 5.3 Credenciais

Ficheiro local:

```text
/root/.config/restic/pve-01.env
```

Permissões:

```text
600
```

Estrutura:

```bash
export RESTIC_REPOSITORY="rest:http://10.10.0.20:8000/pve-01"
export RESTIC_REST_USERNAME="pve-01"
export RESTIC_REST_PASSWORD="..."
export RESTIC_PASSWORD="..."
```

Existem **duas passwords diferentes**:

- `RESTIC_REST_PASSWORD` — autenticação HTTP no rest-server;
- `RESTIC_PASSWORD` — password do próprio repositório Restic.

Não colocar os valores neste documento.

## 5.4 Conteúdo protegido

Conforme o plano original:

```text
/etc
```

O objetivo é proteger a configuração do host Proxmox.

### Base de dados do cluster

**Decisão atual:** não é feito um backup separado de `/var/lib/pve-cluster/config.db`.

O objetivo deste Restic é manter uma cópia pequena e independente da configuração do host. O backup principal das VMs/LXC continua a ser feito por VZDump. Um procedimento de reconstrução mais profundo do nó Proxmox poderá ser definido futuramente se vier a ser necessário.

## 5.5 Script

Ficheiro:

```text
/usr/local/sbin/restic-pve01-etc.sh
```

Conteúdo atual:

```bash
#!/bin/bash
set -euo pipefail

export HOME=/root
export XDG_CACHE_HOME=/root/.cache

source /root/.config/restic/pve-01.env

restic backup /etc
```

Permissões:

```text
700
```

## 5.6 Serviço systemd

```text
/etc/systemd/system/restic-pve01-etc.service
```

Conteúdo:

```ini
[Unit]
Description=Restic backup /etc pve01
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/restic-pve01-etc.sh
```

## 5.7 Timer

```text
/etc/systemd/system/restic-pve01-etc.timer
```

Conteúdo:

```ini
[Unit]
Description=Backup semanal /etc pve01

[Timer]
OnCalendar=Sun 02:30
Persistent=true

[Install]
WantedBy=timers.target
```

Ativado:

```bash
systemctl enable --now restic-pve01-etc.timer
```

Horário:

```text
Domingo 02:30
```

## 5.8 Verificar

Timer:

```bash
systemctl status restic-pve01-etc.timer --no-pager
```

Serviço:

```bash
systemctl status restic-pve01-etc.service --no-pager
```

Histórico:

```bash
journalctl -u restic-pve01-etc.service --no-pager
```

Snapshots:

```bash
source /root/.config/restic/pve-01.env
restic snapshots
```

## 5.9 Estado em 04/10/2026

Backups manuais/testes executados com sucesso.

Snapshots observados:

```text
bbefbbc5
1c34f3da
22f28a41
8c2a35be
```

A última execução automática através do serviço terminou com:

```text
status=0/SUCCESS
snapshot 8c2a35be saved
```

O aviso inicial sobre cache foi corrigido com:

```bash
export HOME=/root
export XDG_CACHE_HOME=/root/.cache
```

---

# 6. Restic — pve-02

## Estado

**Implementado e agendado.**

Repositório:

```text
rest:http://10.10.0.20:8000/pve-02
```

ID do repositório:

```text
f8ea34eab4
```

Utilizador:

```text
pve-02
```

Password própria para:

- autenticação HTTP;
- password do repositório Restic.

### Teste de acesso

Executado:

```bash
source /root/.config/restic/pve-02.env
restic snapshots
```

Resultado:

```text
repository f8ea34ea opened (version 2, compression level auto)
created new cache in /root/.cache/restic
```

### Primeiro backup de `/etc`

Executado:

```bash
restic backup /etc
```

Resultado:

```text
Files:        1246 new,     0 changed,     0 unmodified
Dirs:          251 new,     0 changed,     0 unmodified
Added to the repository: 7.140 MiB (1.932 MiB stored)

processed 1246 files, 6.434 MiB in 0:01
snapshot d5f1aa0c saved
```

Snapshot inicial:

```text
d5f1aa0c
```

### Script de backup

Criado:

```text
/usr/local/sbin/restic-pve02-etc.sh
```

Permissões:

```text
-rwx------ root root
```

Conteúdo funcional:

```bash
#!/bin/bash
set -euo pipefail

export HOME=/root
export XDG_CACHE_HOME=/root/.cache

source /root/.config/restic/pve-02.env

restic backup /etc
```

O script foi executado manualmente com sucesso. Foi criado o snapshot:

```text
17ac6834
```

Resultado do teste incremental:

```text
Files:           0 new,     2 changed,  1244 unmodified
Dirs:            0 new,     4 changed,   247 unmodified
Added to the repository: 78.396 KiB (16.469 KiB stored)

processed 1246 files, 6.434 MiB in 0:00
snapshot 17ac6834 saved
```

### Serviço systemd

Criado:

```text
/etc/systemd/system/restic-pve02-etc.service
```

Serviço:

```ini
[Unit]
Description=Restic backup /etc pve02
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/restic-pve02-etc.sh
```

Teste manual pelo systemd concluído com sucesso. Foi criado o snapshot:

```text
b167dd42
```

Resultado:

```text
Files:           1 new,     2 changed,  1244 unmodified
Dirs:            0 new,     6 changed,   245 unmodified
Added to the repository: 93.200 KiB (19.815 KiB stored)

processed 1247 files, 6.434 MiB in 0:00
snapshot b167dd42 saved
```

O estado `inactive (dead)` após a execução é esperado porque o serviço é `Type=oneshot`. O journal confirmou `Deactivated successfully` e `Finished` sem erro.

### Timer semanal

Criado:

```text
/etc/systemd/system/restic-pve02-etc.timer
```

Configuração:

```ini
[Unit]
Description=Backup semanal /etc pve02

[Timer]
OnCalendar=Sun 02:30
Persistent=true

[Install]
WantedBy=timers.target
```

Timer ativado com:

```bash
systemctl enable --now restic-pve02-etc.timer
```

Estado confirmado:

```text
Active: active (waiting)
Loaded: loaded ... enabled
Trigger: Sun 2026-10-11 02:30:00 +00
```

**Estado final:** pve-02 está configurado para executar semanalmente o backup de `/etc` via Restic aos domingos às 02:30.

---
# 7. Calendário global

Horário planeado:

| Hora | Job |
|---|---|
| 00:30 | OPNsense `config.xml` |
| 00:45 | VPS ×2 |
| 01:00 | pve-01 vzdump |
| 01:45 | pve-02 vzdump |
| 02:30 | Proxmox `/etc` via Restic |
| 02:45 | Unraid: Immich + appdata |
| 03:15 | Unraid: rsync `proxmox/dump` → `backups/proxmox` |
| 03:30 | limpeza Restic |
| 04:00 | Synology: WoL + rsync + shutdown |
| 05:00 | off-site |
| 09:00 | shutdown de segurança do DSM |

---

# 8. Synology — cópia offline

## Objetivo

O DS216j permanece normalmente desligado.

Ao domingo:

```text
Unraid
  │
  ├── WoL
  │
  ├── esperar disponibilidade
  │
  ├── montar NFS
  │
  ├── rsync
  │
  ├── desmontar
  │
  └── SSH poweroff
```

Proteção adicional:

```text
DSM → shutdown domingo 09:00
```

Isto serve como segurança caso o script do Unraid falhe e deixe o NAS ligado.

## Configuração prevista

- WoL ativo;
- NFS acessível apenas pelo Unraid;
- utilizador `backup`;
- chave SSH;
- `sudo` apenas para `poweroff`.

Comando de réplica previsto:

```bash
rsync --delete --backup --backup-dir=_alterados/<data>
```

Alterações antigas ficam em:

```text
_alterados/
```

durante 30 dias.

**Estado:** ainda não implementado.

---

# 9. Off-site

Destino previsto:

```text
EuroNodes S3
```

Ainda sujeito às confirmações do plano.

Pretendido:

- bucket com Object Lock;
- 14 dias;
- modo compliance;
- `rclone`;
- `crypt` sobre o S3;
- tudo o que sai de casa cifrado.

Alternativa:

```text
Hetzner Storage Box
```

O off-site ainda não está implementado.

---

# 10. OPNsense e VPS

## OPNsense

Backup previsto:

```text
config.xml
```

Destino:

```text
rest-server
```

## VPS edge

Restic previsto para:

```text
/etc/nginx
/etc/wireguard
SQLite do Uptime Kuma
```

A comunicação prevista é por WireGuard.

## VPS register

Restic previsto para:

```text
data/
```

---

# 11. Homelab Register

## Princípio

O Homelab Register **não faz backups**.

Ele apenas recebe informação sobre os backups e apresenta:

- estado;
- histórico;
- duração;
- tamanho;
- erros;
- atrasos;
- regras 3-2-1;
- anomalias;
- eventualmente informação SMART dos destinos.

O sistema de backup deve continuar funcional se o Homelab Register estiver completamente indisponível.

## Integração futura

Script previsto:

```text
hr-backup.sh
```

O script será executado no fim de cada job e enviará:

```text
estado
duração
bytes
erro
token da cópia
```

O fluxo será:

```text
backup job
    │
    ├──► destino do backup
    │
    └──► Homelab Register
```

O Register não recebe nem guarda os dados dos backups.

A implementação desta integração fica para depois de a infraestrutura de backups estar estabilizada.

---

# 12. Recuperação — princípios

## 12.1 Unraid morto

O Unraid é o centro local, mas não é a única cópia.

### Recuperação a partir do Synology

1. Ligar o Synology.
2. Garantir que a share de backup está acessível.
3. Montar a share no novo sistema de recuperação.
4. Usar os dumps/ficheiros disponíveis para reconstruir os serviços necessários.

### Recuperação a partir do off-site

1. Obter uma máquina de recuperação limpa.
2. Recuperar a configuração do `rclone`.
3. Recuperar a password do `rclone crypt` a partir do gestor de passwords/cópia em papel.
4. Aceder ao armazenamento off-site.
5. Restaurar primeiro para armazenamento local temporário.
6. Só depois reconstruir o homelab.

O off-site deve continuar recuperável sem depender do Homelab Register.

## 12.2 Proxmox morto

Os backups não dependem do Proxmox original.

### VMs/CTs

Estão como ficheiros no Unraid:

```text
/mnt/user/backups/proxmox/
```

Podem ser usados num Proxmox novo.

### `/etc`

Está no repositório Restic:

```text
pve-01
```

Num sistema novo:

1. instalar Restic;
2. fornecer URL do repositório;
3. fornecer utilizador/password HTTP;
4. fornecer password do repositório Restic;
5. listar snapshots;
6. restaurar o snapshot necessário para uma área temporária;
7. usar a configuração recuperada para reconstrução.

**Não assumir que copiar `/etc` inteiro para um Proxmox novo reconstrói automaticamente o sistema.**

---

# 13. Recuperação de VMs/CTs

Localização:

```text
/mnt/user/backups/proxmox/
```

Procedimento geral:

1. instalar/reconfigurar Proxmox;
2. garantir acesso ao Unraid;
3. adicionar o armazenamento dos backups;
4. selecionar o dump desejado;
5. restaurar com um ID apropriado;
6. validar a configuração antes de colocar o serviço em produção.

### Comandos CLI

Para uma VM:

```bash
qmrestore /caminho/para/vzdump-qemu-<VMID>-<data>.vma.zst <NOVO_VMID>
```

Para um contentor:

```bash
pct restore <NOVO_VMID> /caminho/para/vzdump-lxc-<CTID>-<data>.tar.zst
```

Os caminhos e opções concretas devem ser confirmados no Proxmox disponível no momento da recuperação.

Os testes formais de restauro fazem parte da fase de validação.

---

# 14. Recuperação Restic

O Restic não apresenta os `/etc` como ficheiros normais no Unraid.

Para consultar:

```bash
source /root/.config/restic/pve-01.env
restic snapshots
```

Para listar um snapshot:

```bash
restic ls <snapshot>
```

Para restaurar:

```bash
mkdir /tmp/pve01-restore
restic restore <snapshot> --target /tmp/pve01-restore
```

**Nunca executar um restore diretamente sobre `/` de um sistema novo sem primeiro analisar o conteúdo recuperado.**

---

# 15. Credenciais

## Onde estão

As credenciais devem existir em:

1. gestor de passwords;
2. cópia em papel fora do homelab.

## Não guardar neste documento

Não colocar:

- passwords Restic;
- passwords do rest-server;
- passwords rclone crypt;
- chaves privadas;
- tokens API.

### Localização das credenciais de recuperação

- utilizador `backup` do Synology: credenciais no gestor de passwords e referência na cópia em papel;
- chave usada pelo acesso SSH de backup ao Synology: guardar no local seguro definido para as chaves privadas;
- password/chave do `rclone crypt`: no gestor de passwords e na cópia em papel.

Os valores nunca são escritos neste runbook.

## Credenciais Restic por origem

| Origem | Utilizador rest-server | Repositório |
|---|---|---|
| pve-01 | `pve-01` | `pve-01` |
| pve-02 | `pve-02` | `pve-02` |
| OPNsense | `opnsense` | `opnsense` |
| VPS edge | `vps-edge` | `vps-edge` |
| VPS register | `vps-register` | `vps-register` |

---

# 16. Monitorização

Inicialmente:

```text
jobs → Uptime Kuma
```

Futuramente:

```text
jobs → Homelab Register
```

O plano prevê aproximadamente 11 monitors push, com intervalo de 8 dias.

Também está previsto:

- hook do vzdump;
- alertas Restic;
- estado das réplicas Synology;
- estado do off-site;
- vigilância cruzada dos Uptime Kuma;
- canal de notificações fora das VPS.

---

# 17. Retenção

Planeado:

```text
vzdump:
    2 últimas semanas

restic:
    4 semanais
    2 mensais
```

**Atenção:** a retenção do Restic remoto com `--append-only` requer um mecanismo administrativo de limpeza no próprio Unraid. Os clientes de origem não devem ser responsáveis por apagar os seus próprios backups.

A limpeza administrativa deve correr no Unraid e precisa das credenciais necessárias de cada repositório:

- URL do repositório;
- utilizador/password HTTP do rest-server;
- password do próprio repositório Restic.

Os valores das passwords ficam apenas no gestor de passwords e na cópia em papel; este documento deve indicar apenas onde recuperá-las.

**Estado:** mecanismo de limpeza administrativa ainda não implementado.

A retenção efetiva de `backups/proxmox` ainda será definida; o script atual de rsync não usa `--delete`.

---

# 18. Checklist operacional

## Depois de alterar um backup

- [ ] Atualizar este documento.
- [ ] Confirmar caminho.
- [ ] Confirmar serviço/timer.
- [ ] Confirmar credenciais no gestor.
- [ ] Confirmar destino.
- [ ] Confirmar retenção.
- [ ] Atualizar data de implementação.

## Mensalmente

- [ ] Verificar que os jobs estão a correr.
- [ ] Verificar espaço no Unraid.
- [ ] Verificar espaço no Synology.
- [ ] Verificar estado do off-site.
- [ ] Verificar `restic check` conforme plano.

## Trimestralmente

- [ ] Fazer teste de restauro.
- [ ] Atualizar procedimento de recuperação se necessário.
- [ ] Confirmar que as credenciais de recuperação continuam disponíveis.
- [ ] Confirmar que a documentação corresponde à instalação real.

---

# 19. Estado atual — 04/10/2026

## Implementado

- [x] Share `backups` criada no Unraid.
- [x] `backups/proxmox` criada.
- [x] `backups/restic` usada para o rest-server.
- [x] Rest-server instalado.
- [x] `--append-only`.
- [x] `--private-repos`.
- [x] Autenticação ativa.
- [x] 5 utilizadores criados.
- [x] Jobs VZDump pve01/pve02 configurados → share `proxmox`/storage `unraid`.
- [x] Replicação `proxmox/dump` → `backups/proxmox` configurada e testada.
- [x] User Script Unraid para rsync.
- [x] Agendamento do rsync Unraid alterado para domingo 03:15.
- [x] Restic instalado no pve01.
- [x] Repositório `pve-01` criado.
- [x] Backup `/etc` do pve01.
- [x] Script systemd do pve01.
- [x] Timer semanal do pve01 às 02:30.
- [x] Execução do serviço pve01 testada com sucesso.
- [x] Restic pve02 configurado e testado.
- [x] Timer semanal do pve02 às 02:30.
- [x] Execução do serviço pve02 testada com sucesso.
- [x] Jobs VZDump pve01/pve02 documentados: 01:00 / 01:45, snapshot, ZSTD, `keep-weekly 2`.

## Em curso

- [x] Alterar rsync Unraid para domingo 03:15.
- [x] Definir e implementar limpeza por idade de `backups/proxmox` (30 dias, com proteção por VM/CT).
- [ ] Implementar limpeza administrativa dos repositórios Restic no Unraid.
- [ ] Completar Fase 1.
- [ ] Fase 2 — Unraid/Immich/appdata.
- [ ] Fase 3 — OPNsense/VPS.
- [ ] Fase 4 — Synology.
- [ ] Fase 5 — off-site.
- [ ] Fase 6 — monitorização/testes.

## Ainda não implementar

- [ ] Integração de backups com Homelab Register.
- [ ] `hr-backup.sh`.
- [ ] Endpoint/token de backup no Register.
- [ ] Dashboard de backups.
- [ ] Alertas de backup no Register.

---

# 20. Changelog

### 04/10/2026

- Criado este runbook.
- Documentada arquitetura inicial.
- Documentada configuração real do rest-server.
- Documentados utilizadores Restic.
- Documentado fluxo VZDump → Unraid → `backups/proxmox`.
- Documentado Restic `/etc` do pve01.
- Documentado serviço e timer do pve01.
- Registado estado atual da implementação.
- Documentado o motivo de não usar `rsync --delete` na cópia `backups/proxmox`.
- Implementada a limpeza de `backups/proxmox` com retenção de 30 dias e proteção para preservar o último backup de cada VM/CT.
- Replicação Unraid `proxmox/dump` → `backups/proxmox` alterada para domingo 03:15.
- Documentados jobs VZDump, retenção `keep-weekly 2`, snapshot e ZSTD.
- Registados os IDs reais dos dois jobs VZDump e a configuração em `jobs.cfg`.
- Registada a configuração completa do pve02 e respetivo timer.
- Registada a decisão de não fazer, por agora, um backup separado de `config.db`; o Restic dos Proxmox permanece focado em `/etc`.
- Acrescentado cenário de recuperação em caso de perda do Unraid.
- Documentada a decisão consciente de usar HTTP no rest-server dentro da LAN/WireGuard.
- Documentado o local das credenciais do Synology e do `rclone crypt`, sem guardar valores.
