# Atividade 8 — Configuração de Backup RMAN

**Módulo:** 8 — Rotina de backup RMAN recorrente (FULL semanal + incremental diário) via cron
**Data:** 19/07/2026
**Banco:** Oracle 23ai (23.26.2.0.0) — CDB `CDBORCL` com PDB `ORCLPDB` / host `<HOSTNAME>`

---

## Objetivo

Configurar uma rotina de backup RMAN automatizada para o `CDBORCL`: **FULL (nível 0) aos domingos**, **incremental (nível 1) no resto da semana**, disparada via **cron às 16:00**, com retenção de *recovery window* de 1 dia, controlfile autobackup e snapshot controlfile dedicados. É a primeira rotina de proteção de dados do ambiente, encerrando a trilha operacional antes da verificação final.

---

## Metodologia

1. **Pré-verificações** — descobrir o Oracle Home atual via `/etc/oratab` (não assumir o valor do `.env`) e escolher o filesystem com mais espaço para o destino do backup.
2. **Criar diretório de backup** com dono `oracle`.
3. **Configurar parâmetros persistentes do RMAN** (retenção, autobackup, canal, compressão, snapshot controlfile).
4. **Criar o script de backup** que decide FULL/incremental pelo dia da semana (`date +%u`).
5. **Testar manualmente** o script antes de agendar.
6. **Instalar o cron** e fazer a verificação final.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `oracle` salvo onde indicado `# root`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`. Destino do backup: `/u02/backup` (ver Finding 1).

### Passo 0-1 — Pré-verificações + criar diretório

```bash
# Descobrir o home atual (NUNCA fixo — pode ter mudado após patches)
grep '^CDBORCL:' /etc/oratab                 # → .../23.0.0/dbhome_2

# Escolher o filesystem com mais espaço livre
df -h / /u02                                 # / estava a 89%; /u02 a 6% → usar /u02

# root — criar o diretório de backup
mkdir -p /u02/backup/logs
chown -R oracle:oinstall /u02/backup
chmod 750 /u02/backup
```

### Passo 2 — Configurar parâmetros persistentes do RMAN

```bash
export ORACLE_HOME=$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

rman target / <<'EOF'
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 1 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/u02/backup/autobackup_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/u02/backup/%U';
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/u02/backup/snapcf_CDBORCL.f';
SHOW ALL;
EOF
```

### Passo 3 — Criar o script de backup

```bash
mkdir -p /u01/app/oracle/scripts
cat > /u01/app/oracle/scripts/rman_backup_cdborcl.sh <<'SCRIPT'
#!/bin/bash
# FULL (nivel 0) aos domingos, INCREMENTAL (nivel 1) no resto da semana.
export ORACLE_HOME=$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

BACKUP_DIR=/u02/backup
LOG_DIR=${BACKUP_DIR}/logs
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DOW=$(date +%u)                     # 1=segunda ... 7=domingo

if [ "$DOW" -eq 7 ]; then LEVEL=0; LABEL=FULL; else LEVEL=1; LABEL=INCR; fi
LOG_FILE="${LOG_DIR}/rman_${LABEL}_${TIMESTAMP}.log"

rman target / log="${LOG_FILE}" <<EOF
BACKUP INCREMENTAL LEVEL ${LEVEL} DATABASE PLUS ARCHIVELOG DELETE INPUT;
BACKUP CURRENT CONTROLFILE;
DELETE NOPROMPT OBSOLETE;
CROSSCHECK BACKUP;
DELETE NOPROMPT EXPIRED BACKUP;
EOF

RC=$?
echo "$(date '+%Y-%m-%d %H:%M:%S') - Backup ${LABEL} (level ${LEVEL}) finalizado com RC=${RC}. Log: ${LOG_FILE}" >> "${LOG_DIR}/cron.log"
exit $RC
SCRIPT
chmod 750 /u01/app/oracle/scripts/rman_backup_cdborcl.sh
```

### Passo 4 — Testar manualmente (em background)

```bash
nohup /u01/app/oracle/scripts/rman_backup_cdborcl.sh > /tmp/rman_test.out 2>&1 &
# aguardar terminar e conferir o RC
until ! pgrep -f 'rman_backup_cdborcl.sh' >/dev/null; do sleep 15; done
cat /u02/backup/logs/cron.log        # → ... RC=0
ls -lh /u02/backup                   # backupsets + autobackup_c-* + snapcf_CDBORCL.f
```

### Passo 5-6 — Instalar cron + verificação final

```bash
# oracle — cron às 16:00
(crontab -l 2>/dev/null; echo '0 16 * * * /u01/app/oracle/scripts/rman_backup_cdborcl.sh') | crontab -
crontab -l

# root — confirmar crond ativo
systemctl is-active crond; systemctl is-enabled crond
```

### Pós — derrubar o restore point remanescente (libera limpeza da FRA)

```bash
# GRP da Atividade 7 ainda prendia archivelogs (RMAN-08139). Derrubar:
echo "DROP RESTORE POINT AUTOUPGRADE_9212_CDBORCL2326100;" | sqlplus -s / as sysdba
```

---

## Resultados

| Item | Resultado |
|---|---|
| Diretório de backup | `/u02/backup` (dono `oracle`, com `logs/`) |
| RMAN — retenção | `RECOVERY WINDOW OF 1 DAYS` |
| RMAN — autobackup | `ON`, formato `/u02/backup/autobackup_%F` |
| RMAN — tipo | `COMPRESSED BACKUPSET` |
| RMAN — snapshot controlfile | `/u02/backup/snapcf_CDBORCL.f` |
| Script | `/u01/app/oracle/scripts/rman_backup_cdborcl.sh` (FULL domingo / incr resto) |
| Teste manual | **RC=0** — FULL nível 0 completo, 12 backupsets, autobackup + snapcf presentes |
| Cron | `0 16 * * * .../rman_backup_cdborcl.sh` — `crond` active/enabled |
| FRA após limpeza | `0.3%` (archivelogs antigos liberados após derrubar o GRP) |

---

## Findings

### Finding 1 — Filesystem raiz quase cheio: backup direcionado para `/u02` (impacto: MÉDIO)

A skill sugere `/backup` (na raiz), mas o filesystem `/` estava com **89% de uso (só 7,2 GB livres)** — insuficiente para um backup FULL comprimido do banco. Seguindo a própria orientação da skill ("escolher o filesystem com mais espaço livre entre `/` e `/u02`"), o destino foi definido como **`/u02/backup`** (475 GB livres). Todos os parâmetros do RMAN e o script apontam para esse caminho.

### Finding 2 — Guaranteed restore point da Atividade 7 prendendo archivelogs (impacto: MÉDIO — resolvido)

O primeiro backup emitiu `RMAN-08139: warning: archived redo log not deleted, needed for guaranteed restore point`. Investigação mostrou que o GRP `AUTOUPGRADE_9212_CDBORCL2326100` (rede de segurança do patch 23.26.2, criado na Atividade 7) nunca havia sido derrubado. Enquanto existisse, o `DELETE INPUT`/`DELETE OBSOLETE` não removeria os archived logs que ele referencia — a FRA cresceria indefinidamente a cada backup diário. Com o patch 23.26.2 já validado e estável, o GRP foi derrubado (`DROP RESTORE POINT`) — a FRA voltou a `0.3%` e as próximas execuções limparão os archivelogs normalmente.

### Finding 3 — Muitos archivelogs acumulados: primeira execução mais lenta (impacto: BAIXO)

O banco está em `ARCHIVELOG` desde a Atividade 6, e todos os patches/upgrades/conversões (Atividades 4-7) + o move OMF geraram grande volume de redo — centenas de archived logs acumulados na FRA, nunca limpos (não havia backup antes). O primeiro FULL levou ~22 minutos (16:54 → 17:16) justamente por fazer o backup de todos eles com `DELETE INPUT` (uma faxina saudável que estava pendente). As próximas execuções (incrementais diárias) serão bem mais rápidas.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 9** | Verificação final do ambiente 23ai (com health-check do backup) + resumo para LinkedIn |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/08-configurar-backup.md` |
| Script de backup | `/u01/app/oracle/scripts/rman_backup_cdborcl.sh` |
| Log do teste (RC=0) | `/u02/backup/logs/rman_FULL_20260719_165415.log` |
| Registro de execuções | `/u02/backup/logs/cron.log` |
| Relatório HTML | `HTML/Atividade8/atividade8_report.html` |
