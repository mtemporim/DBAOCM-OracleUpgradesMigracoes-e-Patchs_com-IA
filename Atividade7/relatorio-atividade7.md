# Atividade 7 — Patch 23.26.1 → 23.26.2 com AutoUpgrade

**Módulo:** 7 — Aplicar o Release Update 23.26.2 sobre o banco 23.26.1, via AutoUpgrade (out-of-place)
**Data:** 19/07/2026
**Banco:** Oracle 23ai (23.26.2.0.0) — CDB `CDBORCL` com PDB `ORCLPDB` / host `<HOSTNAME>`

---

## Objetivo

Aplicar o próximo Release Update (23.26.2) sobre o banco `CDBORCL` (23.26.1, resultado da Atividade 6), criando um novo Oracle Home (`dbhome_2`) a partir da Gold Image completa 23.26.2 e usando o AutoUpgrade para mover o banco — mesma estratégia out-of-place das atividades anteriores de patching.

---

## Metodologia

1. **Pré-verificações** — FRA (43,3% de uso, abaixo do limite), versão atual confirmada (23.26.1.0.0), Gold Image 23.26.2 disponível.
2. **Novo Oracle Home** — extração da Gold Image completa (`p39099680_230000_Linux-x86-64.zip`, apesar do nome de patch é uma Gold Image completa), `runInstaller -silent`, `root.sh`.
3. **AutoUpgrade** — `.cfg` de patch, `analyze` (bloqueado por estado incompatível, ver Finding 1) e `deploy` (~10 min).
4. **Pós-patch** — `utlrp`, atualização de `oratab`/systemd/`.bash_profile` (com o path corrigido desta vez), handoff ao systemd.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução, para reproduzir a atividade manualmente. Rodar como usuário `oracle` salvo onde indicado `# root`. Senhas mascaradas com `***`; IPs/hostnames como `<IP>`/`<HOSTNAME>`.

### Passo 0-1 — Pré-verificações

```bash
export ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1   # home atual (23.26.1)
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

# Versão atual + uso da FRA (precisa de margem; expandir se > 70%)
sqlplus -s / as sysdba <<'EOF'
SELECT version_full FROM v$instance;
SELECT ROUND(space_used/space_limit*100,1) AS fra_pct,
       ROUND(space_limit/1073741824,1)     AS limit_gb
FROM v$recovery_file_dest;
EOF

# Confirmar a Gold Image 23.26.2 em /install
ls -lh /install/p39099680_230000_Linux-x86-64.zip
```

### Passo 2 — Criar novo Oracle Home dbhome_2 (23.26.2)

```bash
# root — criar diretório do novo home
mkdir -p /u01/app/oracle/product/23.0.0/dbhome_2
chown oracle:oinstall /u01/app/oracle/product/23.0.0/dbhome_2

# oracle — extrair a Gold Image COMPLETA (apesar do nome p####, é um db_home.zip completo)
cd /u01/app/oracle/product/23.0.0/dbhome_2
unzip -q /install/p39099680_230000_Linux-x86-64.zip

# oracle — instalação software-only
./runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=<HOSTNAME> UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_2 \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true

# root — root.sh do novo home
/u01/app/oracle/product/23.0.0/dbhome_2/root.sh

# oracle — confirmar 23.26.2
export ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_2
$ORACLE_HOME/bin/sqlplus -v
```

### Passo 3 — AutoUpgrade (analyze + deploy)

```bash
# Criar o config file
cat > /install/autoupgrade_patch23262.cfg <<'EOF'
global.autoupg_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade

upg1.source_home=/u01/app/oracle/product/23.0.0/dbhome_1
upg1.target_home=/u01/app/oracle/product/23.0.0/dbhome_2
upg1.sid=CDBORCL
upg1.run_utlrp=yes
upg1.timezone_upg=yes
upg1.target_version=23.26.2
EOF

# WORKAROUND — só se o analyze acusar "serialVersionUID incompatible"
# (estado serializado de um jar de versão anterior no mesmo log_dir)
rm -f /u01/app/oracle/cfgtoollogs/autoupgrade/cfgtoollogs/upgrade/auto/config_files/*.bin
rm -f /u01/app/oracle/cfgtoollogs/autoupgrade/.au-build
rm -f /u01/app/oracle/cfgtoollogs/autoupgrade/cmd/console.lock

# ANALYZE (source_home ainda é o dbhome_1; usar o Java 11 embutido no Oracle Home)
export ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_1
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH
$ORACLE_HOME/jdk/bin/java -jar /install/autoupgrade.jar \
  -config /install/autoupgrade_patch23262.cfg -mode analyze -noconsole

# DEPLOY (move o banco para o dbhome_2 — ~10 min)
$ORACLE_HOME/jdk/bin/java -jar /install/autoupgrade.jar \
  -config /install/autoupgrade_patch23262.cfg -mode deploy -noconsole
```

### Passo 4 — Pós-patch & Handoff

```bash
# A partir daqui o banco já roda no dbhome_2
export ORACLE_HOME=/u01/app/oracle/product/23.0.0/dbhome_2
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

# Confirmar versão + patches aplicados
sqlplus -s / as sysdba <<'EOF'
SELECT version_full FROM v$instance;
SELECT patch_id, status, description FROM dba_registry_sqlpatch
  ORDER BY action_time DESC FETCH FIRST 3 ROWS ONLY;
EOF

# Abrir PDB + salvar estado + desabilitar Resource Manager, depois utlrp nos dois containers
sqlplus -s / as sysdba <<'EOF'
ALTER PLUGGABLE DATABASE ORCLPDB OPEN;
ALTER PLUGGABLE DATABASE ORCLPDB SAVE STATE;
ALTER SYSTEM SET resource_manager_plan='' SCOPE=BOTH;
ALTER SESSION SET CONTAINER=CDB$ROOT;
@?/rdbms/admin/utlrp.sql
ALTER SESSION SET CONTAINER=ORCLPDB;
@?/rdbms/admin/utlrp.sql
EOF

# root — oratab + systemd (substituição GLOBAL do path, cobre ExecStart/ExecStop/PATH)
sed -i 's|CDBORCL:.*:Y|CDBORCL:/u01/app/oracle/product/23.0.0/dbhome_2:Y|' /etc/oratab
sed -i 's|/u01/app/oracle/product/23.0.0/dbhome_1|/u01/app/oracle/product/23.0.0/dbhome_2|g' \
  /etc/systemd/system/oracle-database.service \
  /etc/systemd/system/oracle-listener.service
systemctl daemon-reload

# oracle — atualizar ORACLE_HOME/ORACLE_SID no ~/.bash_profile para o dbhome_2

# Handoff: parar manual e subir pelo systemd
lsnrctl stop
echo "shutdown immediate" | sqlplus -s / as sysdba
# root:
systemctl restart oracle-listener
systemctl restart oracle-database

# WORKAROUND — se aparecer ORA-01081 com nenhum pmon vivo (dbstart duplo por ORA-32004):
rm -f /u01/app/oracle/product/23.0.0/dbhome_2/dbs/lkCDBORCL
for shmid in $(ipcs -m | awk '/^0x/{print $2}'); do ipcrm -m $shmid; done   # só nattch=0
systemctl restart oracle-database
```

---

## Resultados

| Item | Resultado |
|---|---|
| Oracle 23.26.2 instalado | `23.26.2.0.0` |
| AutoUpgrade deploy | `Job 101`, 0 falhas |
| Banco `CDBORCL` | `VERSION_FULL = 23.26.2.0.0`, `OPEN_MODE=READ WRITE` |
| PDB `ORCLPDB` | `READ WRITE` |
| Patches (`dba_registry_sqlpatch`) | `39093711` (23.26.2.0.0) e `38743669` (23.26.1.0.0), ambos `SUCCESS` |
| `utlrp` | 0 objetos com erro |
| `/etc/oratab`, systemd, `.bash_profile` | Todos apontando para `dbhome_2` |

---

## Findings

### Finding 1 — Estado serializado do AutoUpgrade incompatível entre versões do jar (impacto: ALTO — resolvido)

O `-mode analyze` falhou duas vezes com `Previous execution found... local class incompatible: stream classdesc serialVersionUID = -5034081047295198655; local class serialVersionUID = -5684816170166642936`. Causa: o log dir padrão do AutoUpgrade (reaproveitado desde as Atividades 2/4) guarda estado Java serializado em `cfgtoollogs/upgrade/auto/config_files/*.bin`, criado pelo jar 26.4 (Atividade 4) — incompatível com o jar 26.5 baixado do Oracle Home 23ai na Atividade 6.

**Resolução:** removidos os arquivos `*.bin`, `jmBuffer`, `oBuffer` (estado serializado), `.au-build` e `cmd/console.lock` (marcador de versão/lock global do console) do log dir. Isso é **adicional** à limpeza `-clear_recovery_data -jobs <N>` já praticada — aquela limpa um job específico, esta limpa o estado global do console que não é por-SID. Novo `analyze` passou com `SUCCESS`.

### Finding 2 — Terceira ocorrência do padrão de lock/shared memory órfã — causa raiz confirmada (impacto: MÉDIO — resolvido)

Mesmo com o `.service` já correto (path do `dbhome_2` em todas as linhas, incluindo `ExecStart`/`ExecStop`/`PATH`), o `systemctl restart oracle-database` reproduziu o mesmo padrão de duas tentativas de `dbstart` conflitantes. Desta vez ficou claro que a causa raiz **não é (só) path desatualizado**: o próprio `dbstart` dispara uma segunda tentativa de `STARTUP` quando a primeira retorna o aviso `ORA-32004: obsolete or deprecated parameter(s)` — comum logo após um upgrade de versão, quando o `spfile` ainda carrega parâmetros do release anterior.

**Resolução:** mesma remediação já validada — `rm -f dbs/lkCDBORCL` + `ipcrm -m` nos 4 segmentos órfãos (`nattch=0`) + `systemctl restart`. Regra atualizada no `CLAUDE.md`: este padrão pode se repetir mesmo com tudo configurado corretamente, então sempre checar o `startup.log` para confirmar a causa antes de investigar outras hipóteses.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 8** | Verificação final do ambiente + resumo para LinkedIn |

> Pendente: dropar o Guaranteed Restore Point criado nesta atividade (`AUTOUPGRADE_9212_CDBORCL2326100`) quando o usuário confirmar que não é mais necessário.

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/07-patch-23262.md` |
| `.cfg` gerado | `autoupgrade_patch23262.cfg` |
| Job AutoUpgrade | Job 100 (analyze) / Job 101 (deploy) |
| Relatório HTML | `HTML/Atividade7/atividade7_report.html` |
