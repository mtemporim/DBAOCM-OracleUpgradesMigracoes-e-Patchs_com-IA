# Atividade 3 — Criar Novo Oracle Home

**Módulo:** 3 — Criar novo Oracle Home a partir do Gold Image base e preparar para patching
**Data:** 18/07/2026
**Banco:** Oracle 19c (19.3.0.0.0) — non-CDB `ORCL` / host `<HOSTNAME>`

---

## Objetivo

Criar um segundo Oracle Home (`dbhome_1`) a partir da mesma Gold Image base do 19c, mantendo o banco `ORCL` rodando ininterruptamente no Home original (`db_1`). O novo Home é onde o OPatch e o patch RU serão aplicados na Atividade 4, evitando qualquer risco ao banco em produção durante o patching.

---

## Metodologia

1. **Verificação de pré-requisitos** — espaço em `/u01` e arquivos necessários em `/install`.
2. **Criação do novo Home** — extração da Gold Image em um diretório separado (`dbhome_1`).
3. **Instalação silent** — `runInstaller -silent` software-only, mesmo workaround `CV_ASSUME_DISTID=OEL8.0` da Atividade 1.
4. **Scripts de root** — `root.sh` do novo Home.
5. **Verificação** — versão/OPatch do novo Home e confirmação de que o banco segue no Home original.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `root` salvo onde indicado `# oracle`. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`.

### Passo 1 — Pré-requisitos (espaço + arquivos)

```bash
# root — confirmar espaço e arquivos em /install
df -h /u01
ls -lh /install/LINUX.X64_193000_db_home.zip \
       /install/p6880880_190000_Linux-x86-64.zip \
       /install/p39034528_190000_Linux-x86-64.zip

# O OPatch (p6880880) não estava na VM — enviado da máquina local via scp:
#   scp -i <chave_ssh> p6880880_190000_Linux-x86-64.zip root@<IP>:/install/
chown oracle:oinstall /install/p6880880_190000_Linux-x86-64.zip
```

### Passo 2 — Criar e instalar o novo Oracle Home (dbhome_1)

```bash
# root — criar diretório
mkdir -p /u01/app/oracle/product/19.3.0/dbhome_1
chown -R oracle:oinstall /u01/app/oracle/product/19.3.0/dbhome_1

# oracle — extrair Gold Image + runInstaller software-only (mesmo workaround OL8)
cd /u01/app/oracle/product/19.3.0/dbhome_1
unzip -q /install/LINUX.X64_193000_db_home.zip
export CV_ASSUME_DISTID=OEL8.0
./runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=<HOSTNAME> UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1 \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true

# root — root.sh do novo home
/u01/app/oracle/product/19.3.0/dbhome_1/root.sh
```

### Passo 3 — Verificar novo Home + banco intacto no Home original

```bash
# oracle — versão e OPatch do novo home
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export PATH=$ORACLE_HOME/OPatch:$ORACLE_HOME/bin:$PATH
$ORACLE_HOME/bin/sqlplus -v            # 19.3.0.0.0
opatch version                         # 12.2.0.1.17 (será atualizado na Atividade 4)
opatch lsinventory | grep -E 'Oracle Database|Patch [0-9]'

# oracle — confirmar que o ORCL continua no HOME ORIGINAL (db_1), ainda OPEN
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
echo "SELECT instance_name, version, status FROM v\$instance;" | sqlplus -s / as sysdba
```

---

## Resultados

| Item | Resultado |
|---|---|
| Espaço em `/u01` | 30 GB livres antes de iniciar (suficiente) |
| Novo Oracle Home | `dbhome_1` — 19.3.0.0.0 (software-only, sem patch) |
| OPatch (novo home) | `12.2.0.1.17` |
| Patches (novo home) | Nenhum ainda — base limpa para Atividade 4 |
| Banco `ORCL` | `19.0.0.0.0`, `STATUS=OPEN`, continua no Home original (`db_1`) |

---

## Findings

### Finding 1 — Arquivo do OPatch ausente na VM (impacto: BAIXO)

O pré-requisito `p6880880_190000_Linux-x86-64.zip` (OPatch atualizado) não estava em `/install` na VM, apenas o patch RU (`p39034528_190000_Linux-x86-64.zip`, já presente desde antes). O arquivo existia localmente na máquina de trabalho — foi enviado via `scp` (mesmo padrão de contorno de rede usado na Atividade 2), sem necessidade de internet na VM.

### Finding 2 — Dois Oracle Homes coexistindo sem conflito (impacto: BAIXO, confirmação de design)

A estratégia de Oracle Home paralelo (out-of-place patching) funcionou como esperado: o novo `dbhome_1` foi instalado sem tocar no `db_1` em uso, e o banco `ORCL` permaneceu `OPEN` durante toda a instalação. Essa é a abordagem padrão do Oracle para minimizar downtime — o patch é aplicado offline no home novo, e a troca (`switch`) acontece só depois de tudo validado.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 4** | Aplicar OPatch atualizado + patch RU (`p39034528_190000_Linux-x86-64.zip`) no novo Home via AutoUpgrade |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/03-criar-oracle-home.md` |
| Novo Oracle Home | `dbhome_1` (19.3.0.0.0, software-only) |
| Log de instalação | `InstallActions2026-07-18_03-19-44PM` |
| Relatório HTML | `HTML/Atividade3/atividade3_report.html` |
