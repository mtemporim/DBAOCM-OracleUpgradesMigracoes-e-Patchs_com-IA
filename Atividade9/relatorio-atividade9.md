# Atividade 9 — Verificação Final + Resumo para LinkedIn

**Módulo:** 9 — Validação completa do ambiente Oracle 23ai e resumo profissional da jornada
**Data:** 19/07/2026
**Banco:** Oracle 23ai (23.26.2.0.0) — CDB `CDBORCL` com PDB `ORCLPDB` / host `<HOSTNAME>`

---

## Objetivo

Fazer o health-check completo do ambiente após toda a trilha da imersão (instância, containers, dicionário, memória, armazenamento, alert log, listener, systemd e backup) e consolidar um sumário final + post para LinkedIn documentando a jornada técnica de 19c non-CDB até 23ai Multitenant patchado, com backup automatizado.

---

## Metodologia

Health-check em 7 fases (skill `09-verificacao-final`), coletando os dados reais do ambiente e validando cada eixo antes de declarar o ambiente funcional. Nenhum item foi assumido — todos os valores vieram de consultas ao banco e ao SO.

---

## Comandos para Replicação (Passo a Passo)

> Verificações somente-leitura. Rodar como `oracle` salvo onde indicado `# root`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`. Home descoberto via `/etc/oratab` (não fixo).

### Passo 1-3 — Instância, containers, dicionário, memória e storage

```bash
export ORACLE_HOME=$(grep '^CDBORCL:' /etc/oratab | cut -d: -f2)
export ORACLE_SID=CDBORCL
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus -s / as sysdba <<'EOF'
-- instância / database / PDBs
SELECT instance_name, version_full, status FROM v$instance;
SELECT name, cdb, open_mode, log_mode FROM v$database;
SELECT con_id, name, open_mode, restricted FROM v$pdbs ORDER BY con_id;
-- dicionário: componentes não-VALID (esperado 0) e objetos inválidos (esperado 0)
SELECT con_id, comp_name, status FROM cdb_registry WHERE status NOT IN ('VALID','OPTION OFF');
SELECT con_id, COUNT(*) FROM cdb_objects WHERE status='INVALID' GROUP BY con_id;
-- patches
SELECT patch_id, status, description FROM dba_registry_sqlpatch ORDER BY action_time DESC FETCH FIRST 4 ROWS ONLY;
-- memória / storage
SELECT ROUND(SUM(value)/1048576,0) sga_mb FROM v$sga;
SELECT ROUND(value/1048576,0) pga_mb FROM v$parameter WHERE name='pga_aggregate_target';
SELECT ROUND(space_limit/1073741824,1) limit_gb, ROUND(space_used/1073741824,1) used_gb,
       ROUND(space_used/space_limit*100,1) pct FROM v$recovery_file_dest;
SELECT con_id, COUNT(*) datafiles, ROUND(SUM(bytes)/1073741824,2) gb FROM cdb_data_files GROUP BY con_id ORDER BY con_id;
SELECT group#, status, ROUND(bytes/1048576,0) mb FROM v$log ORDER BY group#;
EOF
```

### Passo 4 — Alert log (erros nas últimas 24h)

```bash
AL=$(find /u01/app/oracle/diag/rdbms/cdborcl/CDBORCL/trace -name 'alert_CDBORCL.log' | head -1)
# erros DESDE o startup atual da instância (o filtro mais honesto — o que a instância viva registrou)
awk -v cut='2026-07-19T12:48' '/^[0-9]{4}-[0-9]{2}-[0-9]{2}T/{ts=$1} ts>=cut && /ORA-[0-9]/{print ts,$0}' "$AL" \
  | grep -vE 'ORA-32004|ORA-48'
```

### Passo 5 — Listener e conectividade

```bash
lsnrctl status | grep -E 'Version TNSLSNR|Uptime|Service |Instance'
# conexão via TNS ao CDB e à PDB
sqlplus -s "sys/***@localhost/CDBORCL as sysdba" <<< "SELECT instance_name FROM v\$instance;"
sqlplus -s "sys/***@localhost/orclpdb  as sysdba" <<< "SELECT name, open_mode FROM v\$database;"
```

### Passo 6 / 6.5 — oratab, systemd e backup

```bash
grep -v '^#' /etc/oratab | grep -v '^$'
for s in oracle-database oracle-listener; do echo "$s: $(systemctl is-active $s)"; done   # root
crontab -l | grep rman_backup
echo -e "LIST BACKUP SUMMARY;\nEXIT;" | rman target /
```

---

## Resultados — Ambiente Validado

```
╔══════════════════════════════════════════════════════════╗
║        ORACLE DATABASE 23ai — AMBIENTE VALIDADO          ║
╠══════════════════════════════════════════════════════════╣
║  Versão      : 23.26.2.0.0 (23ai)                        ║
║  SID/DB Name : CDBORCL                                    ║
║  Host        : <HOSTNAME>                                 ║
║  Oracle Home : /u01/app/oracle/product/23.0.0/dbhome_2    ║
╠══════════════════════════════════════════════════════════╣
║  CONTAINERS                                               ║
║  • CDB$ROOT   READ WRITE   ✓                              ║
║  • PDB$SEED   READ ONLY    ✓                              ║
║  • ORCLPDB    READ WRITE   ✓  (não restricted)            ║
╠══════════════════════════════════════════════════════════╣
║  MEMÓRIA        SGA 1533 MB · PGA 512 MB                  ║
║  ARMAZENAMENTO  FRA 0.1/20 GB (0.3%) · Datafiles 6.6 GB   ║
║  BACKUP         RMAN FULL domingo + incr diário (16:00)   ║
╠══════════════════════════════════════════════════════════╣
║  SAÚDE                                                    ║
║  • Componentes inválidos : 0                              ║
║  • Objetos inválidos     : 0                              ║
║  • Erros no alert log    : 0 (desde o startup atual)      ║
║  • Listener/systemd      : active · CDB+PDB conectam      ║
╚══════════════════════════════════════════════════════════╝
```

| Eixo | Resultado |
|---|---|
| Instância | `CDBORCL` 23.26.2.0.0, `OPEN`, ARCHIVELOG |
| Componentes do dicionário | 26 VALID, 2 OPTION OFF (esperado) — **0 não-VALID** |
| Objetos inválidos | **0** em todos os containers |
| Patches | 23.26.2 (39093711) + 23.26.1 (38743669), ambos `SUCCESS` |
| Listener | active, serviços `CDBORCL`/`CDBORCLXDB`/`orclpdb` `READY`; conexão TNS OK |
| systemd | `oracle-database` e `oracle-listener` `active` |
| Backup | cron `0 16 * * *`, script testado (RC=0) |

---

## Findings

### Finding 1 — Erros históricos no alert log são todos transitórios/benignos (impacto: BAIXO — explicado)

A busca de 24h no alert log trouxe erros, mas a análise por timestamp mostrou que **todos são anteriores ao startup atual da instância** (19/07 12:48) e explicáveis:

- **18/07 ~17:14 — `ORA-00313`/`ORA-00312`/`ORA-27037`** (redo logs inacessíveis): ocorreram durante a limpeza da instância órfã do DBCA na Atividade 5 (`pkill -9` + `ipcrm` + remoção de diretórios) — a instância parcial estava sendo forçadamente encerrada.
- **18/07 noite — `ORA-7452`** (`resource plan '' does not exist`): consequência esperada do `ALTER SYSTEM SET resource_manager_plan=''` que fizemos de propósito antes do `utlrp` (Atividade 6).
- **18/07 noite — `ORA-65019`** (`PDB already open`): dos nossos comandos idempotentes `OPEN` + `SAVE STATE` (padrão obrigatório do projeto).
- **18/07 noite — `ORA-652xx`** (`APP$CDB$SYSTEM`, `create lockdown profile`): operações internas do `catupgrd` durante a migração para 23ai.

**Verificação decisiva:** filtrando por erros **desde o startup atual da instância** (após o patch 23.26.2), o resultado é **zero** — a instância viva não registrou nenhum erro. Ambiente declarado 100% funcional com base nisso, não numa suposição.

### Finding 2 — Ambiente final 100% conforme (impacto: POSITIVO)

Toda a trilha (19c non-CDB → RU 19.31 → Multitenant → 23ai 23.26.1 → 23.26.2 → backup) resultou num ambiente sem componentes ou objetos inválidos, com backup automatizado e todos os serviços sob controle do systemd. Os ajustes finais aplicados retroativamente (move OMF dos datafiles da PDB, remoção do `. oraenv` do `.bash_profile`, drop do restore point remanescente) deixaram o ambiente alinhado com as skills atualizadas do instrutor.

---

## Post para LinkedIn

> Texto pronto para publicação — reflete a jornada e os desafios **reais** enfrentados (não hipotéticos).

```
🚀 Do Oracle 19c ao 23ai em 2 dias — Imersão AutoUpgrade com apoio de IA

Concluí uma imersão técnica intensiva percorrendo o ciclo completo de
modernização de um banco Oracle: da instalação do 19c non-CDB até o
Oracle 23ai Multitenant patchado (23.26.2), com backup automatizado —
tudo orquestrado com a ferramenta AutoUpgrade e apoio de IA (Claude Code).

🔹 A trilha (9 atividades):
1️⃣ Preparação do ambiente — Oracle Linux 8.10, LVM, instalação 19c, DBCA, listener, systemd
2️⃣ Atualização do autoupgrade.jar para a versão mais recente
3️⃣ Novo Oracle Home out-of-place (patch sem downtime do banco)
4️⃣ Patching 19c → RU 19.31 via OPatch + AutoUpgrade (deploy)
5️⃣ Conversão non-CDB → PDB Multitenant (noncdbtopdb)
6️⃣ Migração 19c → Oracle 23ai (23.26.1) — upgrade completo do dicionário
7️⃣ Patch 23.26.1 → 23.26.2 (out-of-place)
8️⃣ Backup RMAN automatizado (FULL semanal + incremental diário via cron)
9️⃣ Verificação final — 0 inválidos, 0 erros no alert log da instância viva

🧠 Os desafios REAIS que apareceram (e como foram resolvidos):

✅ VM sem internet → repositório dnf local a partir da ISO do OL 8.10
✅ "SID already exists" no DBCA → não era falso-positivo: era uma
   instância órfã real (pkill -9 + ipcrm + limpeza em 6 diretórios)
✅ Lock/shared memory órfã após handoff → dbstart disparava 2x pelo
   aviso ORA-32004; limpeza de lkSID + ipcrm resolve
✅ systemd apontando para o home antigo → substituir o PATH COMPLETO
   (ExecStart/ExecStop/PATH), não só ORACLE_HOME=
✅ Upgrade 23ai exige ARCHIVELOG → habilitar antes do analyze
✅ Estado serializado do AutoUpgrade incompatível entre versões do jar
   → limpar config_files/*.bin do log_dir
✅ noncdbtopdb NÃO move os datafiles → ativar OMF e mover (online) os
   arquivos da PDB para dentro do diretório do CDB
✅ ". oraenv" no .bash_profile antes de ORACLE_HOME → prompt fantasma
   a cada login; remover
✅ Guaranteed restore point esquecido prendia archivelogs → derrubar
   após validar o patch

📊 Ambiente final:
Oracle Database 23ai 23.26.2.0.0 · CDB CDBORCL · PDB ORCLPDB
Componentes: todos VALID · Objetos inválidos: 0 · Alert log: 0 erros
Backup: RMAN automatizado · FRA saudável · systemd no controle

🛠️ Stack: Oracle 23ai · AutoUpgrade · Oracle Linux 8 · LVM · Multitenant
OPatch · RMAN · systemd · OMF

Mais do que rodar comandos, o valor esteve em ler o que o banco dizia em
cada etapa e reagir com método — inclusive nos imprevistos.

#OracleDatabase #Oracle23ai #AutoUpgrade #DBA #Multitenant
#OracleLinux #DatabaseUpgrade #RMAN #DevOps #DatabaseAdministration
```

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/09-verificacao-final.md` |
| Health-check SQL | consultas a `v$instance`, `cdb_registry`, `cdb_objects`, `v$recovery_file_dest` |
| Alert log analisado | `/u01/app/oracle/diag/rdbms/cdborcl/CDBORCL/trace/alert_CDBORCL.log` |
| Relatório HTML | `HTML/Atividade9/atividade9_report.html` |
