# Atividade 4 — Aplicar Patch com AutoUpgrade

**Módulo:** 4 — Atualizar OPatch, aplicar RU no novo home e mover o banco com AutoUpgrade
**Data:** 18/07/2026
**Banco:** Oracle 19c (19.31.0.0.0 após o patch) — non-CDB `ORCL` / host `<HOSTNAME>`

---

## Objetivo

Aplicar o Release Update mais recente (19.31.0.0.260421) no banco `ORCL` usando a estratégia out-of-place: atualizar o OPatch e aplicar o RU no Oracle Home novo (`dbhome_1`, criado na Atividade 3, ainda sem uso), e então usar o AutoUpgrade para mover o banco do Home antigo para o Home já patchado — minimizando o tempo de indisponibilidade a apenas a janela do `deploy`.

---

## Metodologia

1. **Atualizar OPatch** no Home novo (`dbhome_1`).
2. **Aplicar o RU** via `opatch apply -silent` no Home novo (banco continua no ar no Home antigo durante essa fase).
3. **Mover o banco via AutoUpgrade** — `autoupgrade_patch.cfg`, `-mode analyze` e depois `-mode deploy`.
4. **Verificação pós-patch** — oratab, versão/patches no banco, migração do listener, atualização dos serviços systemd.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `oracle` salvo onde indicado `# root`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`.

### Passo 1 — Atualizar o OPatch no novo home

```bash
# oracle
HOME_NOVO=/u01/app/oracle/product/19.3.0/dbhome_1
mv $HOME_NOVO/OPatch $HOME_NOVO/OPatch.bak
unzip -q /install/p6880880_190000_Linux-x86-64.zip -d $HOME_NOVO
$HOME_NOVO/OPatch/opatch version    # 12.2.0.1.51
```

### Passo 2 — Aplicar o RU 19.31 via OPatch (no home novo)

```bash
# oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export PATH=$ORACLE_HOME/OPatch:$PATH
PATCH_NUM=$(unzip -Z1 /install/p39034528_190000_Linux-x86-64.zip | head -1 | cut -d/ -f1)
mkdir -p /tmp/patch_apply && cd /tmp/patch_apply
unzip -oq /install/p39034528_190000_Linux-x86-64.zip
cd /tmp/patch_apply/$PATCH_NUM
opatch apply -silent    # ~15-30 min

# confirmar
opatch lsinventory | grep -E 'Patch [0-9]|Applied on'   # → 39034528, 19.31.0.0.260421
```

### Passo 3 — Mover o banco para o home patchado via AutoUpgrade

```bash
# Criar o config file
cat > /install/autoupgrade_patch.cfg <<'EOF'
global.autoupg_log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade

upg1.source_home=/u01/app/oracle/product/19.3.0/db_1
upg1.target_home=/u01/app/oracle/product/19.3.0/dbhome_1
upg1.sid=ORCL
upg1.log_dir=/u01/app/oracle/cfgtoollogs/autoupgrade/ORCL
upg1.upgrade_node=<HOSTNAME>
upg1.run_utlrp=yes
upg1.timezone_upg=yes
EOF

# analyze e depois deploy (Java 11 embutido; source_home = db_1 original)
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
JAVA=$ORACLE_HOME/jdk/bin/java

$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_patch.cfg -mode analyze -noconsole
$JAVA -jar /install/autoupgrade.jar -config /install/autoupgrade_patch.cfg -mode deploy  -noconsole
# o deploy para o banco, troca o home no oratab, roda datapatch/utlrp e reinicia
```

### Passo 4 — Verificação pós-patch + migração de listener/systemd

```bash
# Confirmar versão e patch (agora no home patchado)
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
sqlplus -s / as sysdba <<'EOF'
SELECT version_full FROM v$instance;
SELECT patch_id, status, description FROM dba_registry_sqlpatch
  ORDER BY action_time DESC FETCH FIRST 5 ROWS ONLY;
EOF

# Migrar o listener para o home novo (o AutoUpgrade não faz isso sozinho)
# oracle — parar no home antigo, copiar config, subir no novo
(export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1; export PATH=$ORACLE_HOME/bin:$PATH; lsnrctl stop)
cp /u01/app/oracle/product/19.3.0/db_1/network/admin/listener.ora  $ORACLE_HOME/network/admin/
cp /u01/app/oracle/product/19.3.0/db_1/network/admin/tnsnames.ora  $ORACLE_HOME/network/admin/ 2>/dev/null
lsnrctl start
echo "ALTER SYSTEM REGISTER;" | sqlplus -s / as sysdba

# root — atualizar systemd (substituição GLOBAL do path) e daemon-reload
sed -i 's|/19.3.0/db_1|/19.3.0/dbhome_1|g' \
  /etc/systemd/system/oracle-listener.service /etc/systemd/system/oracle-database.service
systemctl daemon-reload

# Handoff — parar manual e subir pelo systemd
lsnrctl stop; echo "shutdown immediate" | sqlplus -s / as sysdba   # oracle
# root:
systemctl start oracle-listener
systemctl start oracle-database

# WORKAROUND — se ORA-01081 com nenhum pmon vivo (lock/shm órfão de dbstart duplo):
rm -f /u01/app/oracle/product/19.3.0/dbhome_1/dbs/lkORCL
for shmid in $(ipcs -m | awk '/^0x/{print $2}'); do ipcrm -m $shmid; done   # só nattch=0
systemctl stop oracle-database && systemctl start oracle-database
```

---

## Resultados

| Item | Resultado |
|---|---|
| OPatch (novo home) | Atualizado de `12.2.0.1.17` para `12.2.0.1.51` |
| RU aplicado via OPatch | `39034528` — Database Release Update 19.31.0.0.260421 |
| AutoUpgrade analyze | `Job 100`, 0 falhas |
| AutoUpgrade deploy | `Job 101`, 0 falhas — banco movido para `dbhome_1` |
| Banco `ORCL` (pós-patch) | `VERSION_FULL = 19.31.0.0.0`, `OPEN_MODE = READ WRITE` |
| Patches no banco | `39034528` e `29517242`, ambos `SUCCESS` |
| `/etc/oratab` | Atualizado automaticamente pelo AutoUpgrade para `dbhome_1:Y` |
| Listener | Migrado para `dbhome_1`, serviços `ORCL`/`ORCLXDB` `READY` |
| Serviços systemd | `oracle-listener` e `oracle-database` atualizados para `dbhome_1` e ativos |

---

## Findings

### Finding 1 — Lock file e shared memory órfãos após `systemctl restart` (impacto: ALTO — resolvido)

Após o `deploy` do AutoUpgrade mover o banco com sucesso, o handoff manual→systemd (`systemctl restart oracle-database`) resultou em `ORA-01034: ORACLE not available` numa consulta, mas um `STARTUP` manual em seguida retornou `ORA-01081: cannot start already-running ORACLE` — apesar de **nenhum processo `pmon` estar de fato rodando**.

**Causa raiz:** o log de `dbstart` (`rdbms/log/startup.log`) mostrou duas tentativas de `Starting up database "ORCL"` a poucos segundos de distância. A segunda tentativa colidiu com a primeira ainda em andamento, deixando um lock file (`dbs/lkORCL`) e 4 segmentos de shared memory órfãos (`ipcs -m`, `nattch=0`) sem uma instância viva de fato para "dona-los".

**Resolução:**
1. Confirmado via `ps -ef | grep pmon` que nenhum processo estava vivo.
2. Removido o lock file (`rm -f dbs/lkORCL`).
3. Removidos os 4 segmentos de shared memory órfãos via `ipcrm -m <shmid>` (todos com `nattch=0`, portanto seguros).
4. `STARTUP` manual funcionou normalmente.
5. Ciclo limpo repetido via `systemctl stop` + `systemctl start` (evitando `restart`, que dispara `ExecStop`+`ExecStart` e pode reintroduzir a corrida) para o systemd assumir o controle de forma consistente.

Regra adicionada ao `CLAUDE.md` para as próximas atividades: preferir `systemctl start` a `systemctl restart` ao subir um banco já parado manualmente.

### Finding 2 — Zero downtime perceptível no fluxo out-of-place (impacto: BAIXO, confirmação de design)

O patch foi todo aplicado no Home novo enquanto o banco seguia no ar no Home antigo (Fases 1-2). O único período de indisponibilidade real foi durante o `deploy` do AutoUpgrade (que já inclui o shutdown/switch/startup/datapatch/utlrp de forma orquestrada) — a estratégia de dois Homes funcionou exatamente como planejado.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 5** | Converter non-CDB para Multitenant com AutoUpgrade |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/04-aplicar-patch.md` |
| `.cfg` gerado | `autoupgrade_patch.cfg` |
| Job AutoUpgrade | Job 100 (analyze) / Job 101 (deploy) |
| Relatório HTML | `HTML/Atividade4/atividade4_report.html` |
