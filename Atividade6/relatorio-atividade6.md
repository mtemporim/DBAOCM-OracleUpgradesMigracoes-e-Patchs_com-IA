# Atividade 6 — Migrar 19c → Oracle 23ai (23.26.1) com AutoUpgrade

**Módulo:** 6 — Migração completa do banco Oracle 19c para Oracle 23ai usando AutoUpgrade
**Data:** 18/07/2026
**Banco:** Oracle 23ai (23.26.1.0.0) — CDB `CDBORCL` com PDB `ORCLPDB` / host `<HOSTNAME>`

---

## Objetivo

Fazer o upgrade completo do banco `CDBORCL` (19.31.0.0.0, resultado da Atividade 5) para Oracle 23ai (23.26.1), usando o AutoUpgrade em modo `deploy` — a etapa mais longa e crítica da imersão, incluindo instalação do novo Oracle Home 23ai, upgrade de dicionário de dados via `catupgrd`, `utlrp` e `datapatch`.

---

## Metodologia

1. **Pré-verificações obrigatórias** — SID real, FRA (expandida para 20 GB), spfile, espaço em disco, Java, limpeza preventiva de jobs.
2. **Instalação do Oracle Home 23ai** — extração da Gold Image, `runInstaller -silent` software-only, `root.sh`.
3. **Preparação do AutoUpgrade** — jar atualizado para versão 23ai-compatível, `.cfg` de upgrade.
4. **Analyze e deploy** — `-mode analyze` (revelou um bloqueio real, ver Finding 1) e `-mode deploy` (~50 minutos: `CDB$ROOT` → `ORCLPDB` → fixups → restart).
5. **Pós-upgrade** — `utlrp` com Resource Manager desabilitado, `datapatch`, reabertura + `SAVE STATE` da PDB, atualização de `oratab`/systemd/`.bash_profile`.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `oracle` salvo onde indicado `# root`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`. SID = `CDBORCL`.

### Passo 0 — Pré-verificações (FRA, ARCHIVELOG, spfile)

```bash
# oracle — home 19c atual
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

# FRA — o catupgrd gera muito redo; expandir para >= 20 GB
sqlplus -s / as sysdba <<'EOF'
SELECT ROUND(space_used/space_limit*100,1) AS fra_pct,
       ROUND(space_limit/1073741824,1)     AS limit_gb FROM v$recovery_file_dest;
ALTER SYSTEM SET db_recovery_file_dest_size = 20G SCOPE=BOTH;
EOF
```

### Passo 1 — Instalar o Oracle Home 23ai

```bash
# root — diretório
mkdir -p /u01/app/oracle/product/23.0.0/dbhome_1
chown -R oracle:oinstall /u01/app/oracle/product/23.0.0/dbhome_1

# oracle — extrair Gold Image 23ai + runInstaller (23ai NÃO precisa do CV_ASSUME_DISTID)
cd /u01/app/oracle/product/23.0.0/dbhome_1
unzip -q /install/LINUX.X64_2326100_db_home.zip
./runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=<HOSTNAME> UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1 \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true

# root — root.sh
/u01/app/oracle/product/23.0.0/dbhome_1/root.sh
```

### Passo 2 — Atualizar jar 23ai-compatível + copiar spfile + criar .cfg

```bash
# oracle — jar do próprio Oracle Home 23ai (versão 26.5)
cp /u01/app/oracle/product/23.0.0/dbhome_1/rdbms/admin/autoupgrade.jar /install/autoupgrade.jar

# spfile no home 23ai (evita falha no restart)
cp /u01/app/oracle/product/19.3.0/dbhome_1/dbs/spfileCDBORCL.ora \
   /u01/app/oracle/product/23.0.0/dbhome_1/dbs/spfileCDBORCL.ora

cat > /install/autoupgrade_upgrade23ai.cfg <<'EOF'
global.autoupg_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade_upgrade23ai

upg1.source_home=/u01/app/oracle/product/19.3.0/dbhome_1
upg1.target_home=/u01/app/oracle/product/23.0.0/dbhome_1
upg1.sid=CDBORCL
upg1.run_utlrp=yes
upg1.timezone_upg=yes
upg1.target_version=23.26.1
EOF
```

### Passo 3 — Analyze (habilitar ARCHIVELOG se o check falhar)

```bash
JAVA=/u01/app/oracle/product/23.0.0/dbhome_1/jdk/bin/java
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_upgrade23ai.cfg -mode analyze -noconsole

# WORKAROUND (Finding #1) — se o analyze falhar em ARCHIVE_MODE_ON, habilitar archivelog:
sqlplus -s / as sysdba <<'EOF'
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
EOF
# rodar o analyze de novo → SUCCESS
```

### Passo 4 — Deploy (upgrade para 23ai)

```bash
$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_upgrade23ai.cfg -mode deploy -noconsole
# ~50 min: PRECHECKS → CDB$ROOT → ORCLPDB → POSTFIXUPS → restart
```

### Passo 5 — Pós-upgrade (utlrp, datapatch, oratab/systemd, handoff)

```bash
# banco agora no home 23ai
export ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

# verificar versão + componentes VALID
sqlplus -s / as sysdba <<'EOF'
SELECT version_full FROM v$instance;
SELECT name, open_mode FROM v$pdbs;
SELECT cid, status FROM sys.registry$ ORDER BY cid;
EOF

# abrir PDB + SAVE STATE + desabilitar Resource Manager, depois utlrp
sqlplus -s / as sysdba <<'EOF'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
ALTER SYSTEM SET resource_manager_plan='' SCOPE=BOTH;
ALTER SESSION SET CONTAINER=CDB$ROOT;
@?/rdbms/admin/utlrp.sql
ALTER SESSION SET CONTAINER=ORCLPDB;
@?/rdbms/admin/utlrp.sql
EOF

# datapatch
$ORACLE_HOME/OPatch/datapatch -verbose

# root — oratab + systemd (substituição GLOBAL do path: cobre ExecStart/ExecStop/PATH)
sed -i 's|CDBORCL:.*:Y|CDBORCL:/u01/app/oracle/product/23.0.0/dbhome_1:Y|' /etc/oratab
sed -i 's|/u01/app/oracle/product/19.3.0/dbhome_1|/u01/app/oracle/product/23.0.0/dbhome_1|g' \
  /etc/systemd/system/oracle-database.service /etc/systemd/system/oracle-listener.service
systemctl daemon-reload

# oracle — atualizar ORACLE_HOME no ~/.bash_profile para o home 23ai

# Handoff (usar restart; workaround de lock órfão se ORA-01081 sem pmon)
lsnrctl stop; echo "shutdown immediate" | sqlplus -s / as sysdba
systemctl restart oracle-listener      # root
systemctl restart oracle-database
# se necessário: rm -f .../23.0.0/dbhome_1/dbs/lkCDBORCL; ipcrm -m <shmid órfãos>; systemctl restart oracle-database
```

---

## Resultados

| Item | Resultado |
|---|---|
| Oracle 23ai instalado | `23.26.1.0.0` |
| `autoupgrade.jar` | Atualizado para `26.5.260117` (do próprio Oracle Home 23ai) |
| AutoUpgrade deploy | `Job 102`, 0 falhas — CDB$ROOT e ORCLPDB upgradados |
| Banco `CDBORCL` | `VERSION_FULL = 23.26.1.0.0`, `OPEN_MODE=READ WRITE` |
| PDB `ORCLPDB` | `READ WRITE`, não restricted |
| Componentes (`sys.registry$`) | Todos `VALID` (status=1); `RAC`=9 é esperado (opção não usada, standalone) |
| `utlrp` | 0 objetos com erro em CDB$ROOT e ORCLPDB |
| `datapatch` | RU 23.26.1.0.0 já aplicado em CDB$ROOT, PDB$SEED e ORCLPDB |
| `/etc/oratab`, systemd, `.bash_profile` | Todos apontando para `${ORACLE_HOME_23AI}` + `CDBORCL` |

---

## Findings

### Finding 1 — `CDBORCL` em NOARCHIVELOG bloqueou o `analyze` (impacto: ALTO — resolvido)

O primeiro `-mode analyze` falhou com `CHECK_FAILED` em `ARCHIVE_MODE_ON` — o `CDBORCL` havia sido criado em `NOARCHIVELOG` na Atividade 5 (seguindo a skill original de conversão Multitenant, que usa `-enableArchive false`). O upgrade para 23ai exige `ARCHIVELOG` habilitado.

**Resolução:** `SHUTDOWN IMMEDIATE` → `STARTUP MOUNT` → `ALTER DATABASE ARCHIVELOG` → `ALTER DATABASE OPEN`, seguido de reabertura + `SAVE STATE` da PDB. A FRA já estava corretamente configurada (`/u02/fra`, 20 GB) — bastou habilitar o modo. Novo `analyze` confirmou `[Status] SUCCESS`.

### Finding 2 — Bug de escaping do `$PATH` se repetiu (impacto: MÉDIO — resolvido rapidamente)

O mesmo bug de escaping da Atividade 1 (PATH do Windows/Git Bash vazando por `\\\$PATH` mal escapado entre camadas de aspas) apareceu de novo ao tentar rodar o `analyze` via `su -c` aninhado dentro do SSH. Desta vez, identificado e evitado rapidamente adotando a abordagem de script local + `scp` (já validada) para todos os comandos complexos restantes da atividade — sem chegar a corromper nenhum arquivo remoto.

### Finding 3 — Atualização incompleta do systemd deixou `ExecStart`/`ExecStop` no home antigo (impacto: ALTO — resolvido)

O padrão de `sed` usado para atualizar os serviços systemd (`s|ORACLE_HOME=.*|...|; s|ORACLE_SID=.*|...|`) só substitui as linhas `Environment=`, não as linhas `Environment=PATH=` nem `ExecStart=`/`ExecStop=` quando o caminho do Oracle Home está "hardcoded" nelas. Resultado: o `systemctl restart oracle-database` continuou chamando o **`dbstart` do Oracle Home 19c antigo** contra o SID do banco já em 23ai — o log mostrou o aviso `Listener version 19 NOT supported with Database version 23` e dois `dbstart` conflitantes em sequência, reproduzindo o mesmo padrão de lock file/shared memory órfã da Atividade 4/5.

**Resolução:** substituição global do path completo do home antigo pelo novo (`sed -i 's|<path_19c>|<path_23ai>|g'`), que corrige `Environment=PATH`, `ExecStart` e `ExecStop` de uma vez. Confirmado com `grep -E 'Exec|Environment'` antes de tentar subir de novo. Regra registrada no `CLAUDE.md` para as próximas atividades.

### Finding 4 — Mesmo padrão de lock/shared memory órfã da Atividade 4/5 (impacto: MÉDIO — resolvido)

Consequência direta do Finding 3: `ORA-01034` seguido de `ORA-01081` sem `pmon` real vivo. Mesma remediação já documentada: `rm -f dbs/lkCDBORCL` + `ipcrm -m` nos 4 segmentos órfãos (`nattch=0`) + `systemctl restart` (não `start`, já que o serviço mostrava `active (exited)`).

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 7** | Aplicar patch 23.26.1 → 23.26.2 via AutoUpgrade |

> Pendente: dropar o Guaranteed Restore Point criado pelo AutoUpgrade (`AUTOUPGRADE_9212_CDBORCL1931000`) quando o usuário confirmar que não é mais necessário.

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/06-migrar-23ai.md` |
| `.cfg` gerado | `autoupgrade_upgrade23ai.cfg` |
| Job AutoUpgrade | Job 100/101 (analyze) / Job 102 (deploy) |
| Relatório HTML | `HTML/Atividade6/atividade6_report.html` |
