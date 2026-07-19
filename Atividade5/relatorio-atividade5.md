# Atividade 5 — Converter non-CDB para Multitenant com AutoUpgrade

**Módulo:** 5 — Converter banco 19c non-CDB em PDB dentro de um novo CDB, usando AutoUpgrade (`noncdbtopdb`)
**Data:** 18/07/2026
**Banco:** Oracle 19c (19.31.0.0.0) — CDB `CDBORCL` com PDB `ORCLPDB` / host `<HOSTNAME>`

> **Transição de SID desta atividade em diante:** o banco deixa de ser `ORCL` (non-CDB) e passa a ser `CDBORCL` (CDB) com PDB `ORCLPDB`. O `.env` do projeto mantém `ORACLE_SID=ORCL` por razão histórica — a partir daqui, **ignorar** esse valor e usar `CDBORCL` explicitamente.

---

## Objetivo

Converter o banco non-CDB `ORCL` (patchado na Atividade 4) em uma Pluggable Database (`ORCLPDB`) dentro de um Container Database recém-criado (`CDBORCL`), reaproveitando o Oracle Home já patchado (`dbhome_1`) — sem criar nem patchear um home adicional, já que a conversão não muda a versão (19.31 → 19.31).

---

## Metodologia

1. **Reaproveitar `dbhome_1`** (já patchado na Atividade 4) — nenhuma instalação/patch novo necessário.
2. **Parar `ORCL`** antes de criar o CDB (padrão de gestão de RAM, não reativo).
3. **Criar CDB vazio `CDBORCL`** via DBCA silent no `dbhome_1` (0 PDBs, só `PDB$SEED`).
4. **Registrar `CDBORCL`** no listener (já no `dbhome_1`, sem migração necessária).
5. **AutoUpgrade `noncdbtopdb`** — `analyze` e `deploy` convertendo `ORCL` → `ORCLPDB` dentro de `CDBORCL`.
6. **Verificação pós-conversão** — CDB/PDB, patch no dicionário, listener, `OPEN` + `SAVE STATE` da PDB.
7. **Remoção do Oracle Home antigo** (`db_1`, sem patch, substituído pelo `dbhome_1`).
8. **Atualização de `.bash_profile`, `/etc/oratab`, systemd** para `CDBORCL` + `dbhome_1`, com handoff manual→systemd.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `oracle` salvo onde indicado `# root`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`. O `dbhome_1` (já patchado na Atividade 4) é reaproveitado — não se cria home novo.

### Passo 3 — Parar ORCL e criar o CDB CDBORCL via DBCA

```bash
# oracle — parar o ORCL para liberar RAM (padrão em VM pequena, não reativo)
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
echo "shutdown immediate" | sqlplus -s / as sysdba

# oracle — criar o CDB vazio CDBORCL (0 PDBs) no MESMO home
export ORACLE_SID=CDBORCL
dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname CDBORCL -sid CDBORCL \
  -createAsContainerDatabase true -numberOfPDBs 0 \
  -characterSet AL32UTF8 -nationalCharacterSet AL16UTF16 \
  -sysPassword *** -systemPassword *** \
  -datafileDestination /u02/oradata/CDBORCL -recoveryAreaDestination /u02/fra \
  -recoveryAreaSize 2048 -totalMemory 2048 -redoLogFileSize 50 \
  -databaseType MULTIPURPOSE -emConfiguration NONE -enableArchive false -ignorePreReqs

# religar o ORCL depois do CDB confirmado
export ORACLE_SID=ORCL
echo "startup" | sqlplus -s / as sysdba
```

```bash
# WORKAROUND (Finding #1) — se o DBCA falhar com DBT-10317 "SID already exists":
#   1) confirmar processos órfãos reais:  ps -ef | grep _CDBORCL
#   2) matar TODOS os processos da instância órfã (pmon nao basta):
pkill -9 -f '_CDBORCL'
#   3) limpar shared memory e semáforos órfãos:
for shmid in $(ipcs -m | awk '/^0x/{print $2}'); do ipcrm -m $shmid; done
for semid in $(ipcs -s | awk '/^0x/{print $2}'); do ipcrm -s $semid; done
#   4) limpar resíduos em SEIS locais (todos):
sed -i '/^CDBORCL:/d' /etc/oratab
rm -f  /u01/app/oracle/product/19.3.0/dbhome_1/dbs/*CDBORCL*
rm -rf /u01/app/oracle/admin/CDBORCL /u02/oradata/CDBORCL /u02/fra/CDBORCL
rm -rf /u01/app/oracle/cfgtoollogs/dbca/CDBORCL
rm -rf /u01/app/oracle/diag/rdbms/cdborcl
#   5) rodar o dbca de novo
```

### Passo 4 — Registrar CDBORCL no listener

```bash
# oracle — listener já está no dbhome_1 (migrado na Atividade 4); só registrar
export ORACLE_SID=CDBORCL
echo "ALTER SYSTEM REGISTER;" | sqlplus -s / as sysdba
```

### Passo 5 — Converter ORCL → PDB ORCLPDB via AutoUpgrade (noncdbtopdb)

```bash
# Limpeza preventiva do job anterior do SID ORCL (best-effort)
JAVA=/u01/app/oracle/product/19.3.0/dbhome_1/jdk/bin/java
$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_convert.cfg -clear_recovery_data -jobs 101

# Criar o config file (noncdbtopdb exige um CDB alvo já existente)
cat > /install/autoupgrade_convert.cfg <<'EOF'
global.global_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade

upg1.source_home=/u01/app/oracle/product/19.3.0/dbhome_1
upg1.target_home=/u01/app/oracle/product/19.3.0/dbhome_1
upg1.sid=ORCL
upg1.target_cdb=CDBORCL
upg1.target_pdb_name=ORCLPDB
upg1.log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade/ORCL_CONVERT
upg1.upgrade_node=<HOSTNAME>
upg1.run_utlrp=yes
upg1.timezone_upg=yes
EOF

# analyze e depois deploy
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_convert.cfg -mode analyze -noconsole
$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_convert.cfg -mode deploy  -noconsole
```

### Passo 6 — Verificar e abrir a PDB (OPEN + SAVE STATE sempre juntos)

```bash
# oracle
export ORACLE_SID=CDBORCL
sqlplus -s / as sysdba <<'EOF'
SELECT name, cdb, open_mode FROM v$database;
SELECT con_id, name, open_mode FROM v$pdbs ORDER BY con_id;
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
ALTER SYSTEM REGISTER;
EOF
```

### Passo 7 — Remover o Oracle Home antigo (db_1, sem patch)

```bash
# oracle — detach do inventário  |  root — remover diretório
/u01/app/oracle/product/19.3.0/db_1/oui/bin/runInstaller -detachHome -silent \
  ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
rm -rf /u01/app/oracle/product/19.3.0/db_1     # root
```

### Passo 8 — Atualizar .bash_profile / oratab / systemd + handoff

```bash
# oracle — .bash_profile → CDBORCL + dbhome_1
sed -i '/^export ORACLE_HOME=/d; /^export ORACLE_SID=/d' /home/oracle/.bash_profile
cat >> /home/oracle/.bash_profile <<'EOF'
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH
EOF

# root — oratab (só CDBORCL) + systemd (substituição global de path/SID)
grep -v '^ORCL:\|^CDBORCL:' /etc/oratab > /tmp/oratab.new
echo 'CDBORCL:/u01/app/oracle/product/19.3.0/dbhome_1:Y' >> /tmp/oratab.new
cp /tmp/oratab.new /etc/oratab
sed -i 's|/19.3.0/db_1|/19.3.0/dbhome_1|g; s|ORACLE_SID=ORCL$|ORACLE_SID=CDBORCL|g' \
  /etc/systemd/system/oracle-database.service /etc/systemd/system/oracle-listener.service
systemctl daemon-reload

# Handoff — parar manual e subir pelo systemd
#   ATENÇÃO: se o serviço aparecer "active (exited)" (processo já morto por fora),
#   'systemctl start' é no-op — usar 'systemctl restart'.
lsnrctl stop; echo "shutdown immediate" | sqlplus -s / as sysdba   # oracle
# root:
systemctl restart oracle-listener
systemctl restart oracle-database
```

---

## Resultados

| Item | Resultado |
|---|---|
| CDB `CDBORCL` | `CDB=YES`, `OPEN_MODE=READ WRITE` |
| PDB `ORCLPDB` | `OPEN_MODE=READ WRITE`, `RESTRICTED=NO` |
| Versão | `19.31.0.0.0` |
| Patch preservado | `39034528` (RU 19.31.0.0.260421), `SUCCESS` |
| Oracle Home | Único — `dbhome_1` (removido `db_1`, ~7 GB liberados) |
| `/etc/oratab` | `CDBORCL:dbhome_1:Y` |
| `.bash_profile` do oracle | `ORACLE_HOME=dbhome_1`, `ORACLE_SID=CDBORCL` |
| Listener | Serviços `CDBORCL`, `CDBORCLXDB`, `orclpdb` — todos `READY` |
| Serviços systemd | Atualizados e ativos para `CDBORCL` + `dbhome_1` |

---

## Findings

### Finding 1 — Bug do DBCA + instância órfã travaram a criação do CDB por 3 tentativas (impacto: ALTO — resolvido)

A primeira tentativa de `dbca -silent -createDatabase` para `CDBORCL` falhou com `[FATAL] [DBT-10317] Specified SID Name (CDBORCL) already exists` — mesmo sendo a primeira execução. O `trace.log` do DBCA revelou a causa exata: a checagem interna `isLocalSIDExist` roda `ps -ef | grep ora_pmon_CDBORCL` e interpreta qualquer exit code 0 como "SID já existe" — um padrão clássico de "grep encontra a si mesmo". Mas neste caso havia algo mais grave: a primeira tentativa de fato **chegou a iniciar uma instância parcial** (background processes, datafiles) antes de falhar na validação, deixando uma **instância órfã genuína** (`ora_pmon_CDBORCL` real, não falso-positivo) que persistiu por mais duas tentativas de limpeza incompletas.

**Resolução (4ª tentativa bem-sucedida):**
1. Identificado o processo `ora_pmon_CDBORCL` real e todos os ~30 processos de background associados via `ps -ef`.
2. `pkill -9 -f '_CDBORCL'` para encerrar todos de uma vez (matar só o `pmon` não bastou — os demais processos ficaram órfãos).
3. `ipcrm` em todos os segmentos de shared memory e semáforos remanescentes.
4. Limpeza completa de arquivos residuais em **seis locais diferentes**: `/etc/oratab`, `dbs/*CDBORCL*`, `admin/CDBORCL`, `oradata/CDBORCL`, `cfgtoollogs/dbca/CDBORCL` e `diag/rdbms/cdborcl` — os dois últimos foram descobertos só na segunda rodada de limpeza (a primeira limpeza havia esquecido esses dois diretórios, causando uma repetição do erro).
5. Com tudo limpo, a 4ª tentativa progrediu normalmente (10% → 100%, ~35 minutos, mais lento que o `ORCL` non-CDB por causa do provisionamento do `PDB$SEED`).

**Lição:** ao investigar "SID already exists" do DBCA, sempre checar o `trace.log` para ver o comando exato de validação usado, e nunca assumir que a limpeza está completa sem verificar `cfgtoollogs/dbca/<SID>` e `diag/rdbms/<sid>` além dos locais óbvios (`oratab`, `dbs/`, `admin/`, `oradata/`).

### Finding 2 — `systemctl start` em serviço "active (exited)" não reinicia o processo real (impacto: MÉDIO — resolvido)

Após parar manualmente o listener e o `CDBORCL` para o handoff, `systemctl start oracle-listener`/`oracle-database` não fez nada — os serviços já estavam marcados `active (exited)` havia horas (desde a Atividade 4), e o systemd trata `start` como no-op em unidades já ativas, mesmo que o processo real por trás já tenha morrido. Confirmado via `ps -ef` que nenhum processo real existia apesar do `systemctl status` dizer "active". Resolvido usando `systemctl restart` (que força `ExecStop`+`ExecStart` independente do estado atual).

**Nota:** isso ajusta a lição da Atividade 4 (que recomendava `start` em vez de `restart` para evitar o double-dbstart) — a orientação correta é: usar `start` quando o serviço estiver `inactive`/`failed`; usar `restart` quando o serviço mostrar `active` mas o processo real já tiver sido parado manualmente por fora.

### Finding 3 — Java 8 do `dbhome_1` segue funcional com AutoUpgrade 26.4 (impacto: BAIXO, confirmação)

A skill original desta atividade presumia que o JDK do `dbhome_1` seria Java 8 (incompatível) e recomendava instalar `java-11-openjdk` do SO. Confirmado que o JDK embutido é de fato Java 8 (`1.8.0_481`) — mas, consistente com as Atividades 2 e 4, o AutoUpgrade 26.4 rodou o `analyze` e o `deploy` normalmente sem qualquer erro de "Unsupported Java Runtime Environment". Não foi necessário instalar nada adicional.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 6** | Migrar `CDBORCL`/`ORCLPDB` de 19c para 23ai (23.26.1) via AutoUpgrade |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/05-converter-multitenant.md` |
| `.cfg` gerado | `autoupgrade_convert.cfg` |
| Jobs AutoUpgrade | Job 102 (analyze) / Job 103 (deploy) |
| Relatório HTML | `HTML/Atividade5/atividade5_report.html` |
