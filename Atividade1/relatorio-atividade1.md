# Atividade 1 — Preparação do Ambiente e Instalação Oracle 19c

**Módulo:** 1 — OS prep + LVM + Install Oracle 19c + DBCA + Listener + startup
**Data:** 18/07/2026
**Banco:** Oracle 19c (19.3.0.0.0) — non-CDB `ORCL` / host `<HOSTNAME>`
**SO:** Oracle Linux 8.10 (VM VMware Workstation Pro — 6 vCPU / 6 GB RAM)

---

## Objetivo

Preparar do zero um ambiente Oracle Linux 8.10 e instalar um banco Oracle 19c non-CDB (`ORCL`), servindo de ponto de partida para toda a trilha de patching, conversão para Multitenant e migração para 23ai que segue nas próximas atividades da imersão.

---

## Metodologia

A execução seguiu as 8 fases da skill `01-prepare-ambiente`:

1. **Sistema Operacional** — `/etc/hosts`, pacote de pré-requisitos, SELinux permissivo, firewalld desativado, atualização de pacotes, permissões em `/install`.
2. **LVM para `/u02`** — identificação do disco livre, criação de PV/VG/LV, formatação XFS, montagem persistida em `/etc/fstab`.
3. **Instalação Oracle 19c** — extração da Gold Image, `runInstaller -silent` (software-only), scripts de root.
4. **Listener** — `listener.ora`/`tnsnames.ora`, `lsnrctl start`.
5. **Banco `ORCL` via DBCA silent** — non-CDB, charset AL32UTF8, ~2GB SGA/PGA.
6. **`.bash_profile` do oracle** — variáveis de ambiente Oracle.
7. **systemd** — `/etc/oratab`, serviços `oracle-listener` e `oracle-database`, handoff de processos manuais para o systemd.
8. **OpenJDK + bootstrap do AutoUpgrade** — Java 17, `JAVA_HOME`, cópia do `autoupgrade.jar` inicial para `/install`.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução, para reproduzir a atividade manualmente. Rodar como `root` salvo onde indicado `# oracle`. Senhas mascaradas com `***`; IPs/hostnames como `<IP>`/`<HOSTNAME>`.

### Passo 0 — Repositório local via ISO (workaround: VM sem internet)

```bash
# root — montar a ISO do OL8.10 e configurar repo local do dnf
mkdir -p /mnt/iso
mount -o loop,ro /install/V1042736-01.iso /mnt/iso
cat > /etc/yum.repos.d/local-iso.repo <<'EOF'
[local-baseos]
name=Local ISO BaseOS
baseurl=file:///mnt/iso/BaseOS
gpgcheck=0
enabled=1
[local-appstream]
name=Local ISO AppStream
baseurl=file:///mnt/iso/AppStream
gpgcheck=0
enabled=1
EOF
sed -i 's/^enabled=1/enabled=0/' /etc/yum.repos.d/oracle-linux-ol8.repo \
  /etc/yum.repos.d/uek-ol8.repo /etc/yum.repos.d/virt-ol8.repo
dnf clean all && dnf repolist
echo '/install/V1042736-01.iso  /mnt/iso  iso9660  loop,ro  0 0' >> /etc/fstab   # persistir
```

### Passo 1 — Sistema Operacional

```bash
# root
grep -q '<HOSTNAME>' /etc/hosts || echo '<IP>  <HOSTNAME>  imersao' >> /etc/hosts
dnf install -y oracle-database-preinstall-21c    # 19c ausente na ISO; 21c compatível
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config && setenforce 0
systemctl disable --now firewalld
dnf update -y
chown -R oracle:oinstall /install
```

### Passo 2 — LVM para /u02

```bash
# root
pvcreate /dev/sdb
vgcreate vg_u02 /dev/sdb
lvcreate -l 100%FREE -n lv_u02 vg_u02
mkfs.xfs /dev/vg_u02/lv_u02
mkdir -p /u02 && mount /dev/vg_u02/lv_u02 /u02
echo '/dev/vg_u02/lv_u02  /u02  xfs  defaults  0 0' >> /etc/fstab
mkdir -p /u02/oradata /u02/fra
chown -R oracle:oinstall /u02 && chmod -R 775 /u02
```

### Passo 3 — Instalação Oracle 19c (software-only)

```bash
# root — diretórios
mkdir -p /u01/app/oracle/product/19.3.0/db_1 /u01/app/oraInventory
chown -R oracle:oinstall /u01/app && chmod -R 775 /u01/app

# oracle — extrair Gold Image + runInstaller (workaround OL8: CV_ASSUME_DISTID)
cd /u01/app/oracle/product/19.3.0/db_1
unzip -q /install/LINUX.X64_193000_db_home.zip
export CV_ASSUME_DISTID=OEL8.0
./runInstaller -silent -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=<HOSTNAME> UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/u01/app/oraInventory \
  ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1 \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false DECLINE_SECURITY_UPDATES=true

# root — scripts de root
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.3.0/db_1/root.sh
```

### Passo 4 — Listener

```bash
# oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
export PATH=$ORACLE_HOME/bin:$PATH
mkdir -p $ORACLE_HOME/network/admin
cat > $ORACLE_HOME/network/admin/listener.ora <<'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <HOSTNAME>)(PORT = 1521))
    )
  )
ADR_BASE_LISTENER = /u01/app/oracle
EOF
lsnrctl start
```

### Passo 5 — Banco ORCL via DBCA (non-CDB)

```bash
# oracle
export ORACLE_SID=ORCL
dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname ORCL -sid ORCL -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -sysPassword *** -systemPassword *** \
  -createAsContainerDatabase false \
  -databaseType MULTIPURPOSE -memoryMgmtType auto_sga -totalMemory 2048 \
  -storageType FS \
  -datafileDestination /u02/oradata/ -recoveryAreaDestination /u02/fra/ \
  -enableArchive TRUE -redoLogFileSize 200 \
  -emConfiguration NONE -sampleSchema FALSE -ignorePreReqs
```

### Passo 6 — .bash_profile do oracle

```bash
# oracle
cat >> /home/oracle/.bash_profile <<'EOF'
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export NLS_DATE_FORMAT='DD-MON-YYYY HH24:MI:SS'
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
ORAENV_ASK=NO
EOF
```

### Passo 7 — systemd + oratab

```bash
# root — oratab com flag Y (DBCA cria com N; dbstart só sobe bancos com Y)
grep -q '^ORCL:' /etc/oratab || echo 'ORCL:/u01/app/oracle/product/19.3.0/db_1:Y' >> /etc/oratab
sed -i 's|^ORCL:\(.*\):N|ORCL:\1:Y|' /etc/oratab

# root — criar os serviços (Type=forking, User=oracle):
#   /etc/systemd/system/oracle-listener.service  → ExecStart=.../lsnrctl start
#   /etc/systemd/system/oracle-database.service  → ExecStart=.../dbstart <ORACLE_HOME>
systemctl daemon-reload
systemctl enable oracle-listener oracle-database

# handoff: parar o que foi iniciado manualmente e subir pelo systemd
su - oracle -c 'lsnrctl stop; echo "shutdown immediate" | sqlplus -s / as sysdba'
systemctl start oracle-listener
systemctl start oracle-database
```

### Passo 8 — OpenJDK + bootstrap do autoupgrade.jar

```bash
# root
dnf install -y java-17-openjdk java-17-openjdk-devel
cp /u01/app/oracle/product/19.3.0/db_1/rdbms/admin/autoupgrade.jar /install/autoupgrade.jar
chown oracle:oinstall /install/autoupgrade.jar
mkdir -p /install/patches /u01/app/oracle/cfgtoollogs/autoupgrade
```

---

## Resultados

| Item | Resultado |
|---|---|
| `/u02` (LVM) | 500 GB, montado e persistido em fstab |
| Oracle Home 19c | `/u01/app/oracle/product/19.3.0/db_1` — software instalado com sucesso |
| Banco `ORCL` | `OPEN_MODE=READ WRITE`, `CDB=NO` |
| Listener | Ativo na porta 1521 |
| Serviços systemd | `oracle-listener` e `oracle-database`, habilitados e ativos |
| Java | OpenJDK 17.0.10 |
| `autoupgrade.jar` | Copiado para `/install`, `build.version 20190207` |

---

## Findings

### Finding 1 — VM sem acesso à internet (impacto: ALTO)

A VM está numa rede isolada (`<IP>/21`) sem rota para a internet pública — `dnf install` falhava ao tentar resolver `yum.oracle.com`, apesar do DNS interno (`<IP>`) responder normalmente. Diagnóstico: gateway não respondia ICMP e não havia rota real de saída.

**Resolução:** o usuário anexou a ISO oficial `V1042736-01.iso` (Oracle Linux 8.10.0, BaseOS + AppStream completos) em `/install`. Configurei um repositório dnf local (`/etc/yum.repos.d/local-iso.repo`) apontando para `/mnt/iso` (montagem via loop, persistida em `/etc/fstab` para sobreviver a reboots) e desabilitei os repositórios online.

### Finding 2 — `oracle-database-preinstall-19c` não disponível na ISO (impacto: MÉDIO)

A ISO só trazia `oracle-database-preinstall-21c`. Como o pacote preinstall apenas cria o usuário/grupos `oracle` e ajusta kernel parameters/limits — requisitos praticamente idênticos entre 19c e 21c — usei a versão 21c como substituto, com aprovação do usuário. Funcionou sem problemas: usuário `oracle` e todos os grupos (`oinstall`, `dba`, `oper`, `backupdba`, `dgdba`, `kmdba`, `racdba`) criados corretamente.

### Finding 3 — Bug de escaping corrompeu o `.bash_profile` do oracle (impacto: ALTO — pego antes de causar dano)

Ao gravar `export PATH=$JAVA_HOME/bin:$PATH` no `.bash_profile` remoto via SSH a partir do Git Bash (Windows), uma sequência `\\\$PATH` mal escapada entre as camadas de aspas locais fez o **`$PATH` do Git Bash local** (com dezenas de diretórios do Windows) ser expandido e gravado no arquivo remoto, no lugar da variável remota. Identificado na verificação pós-passo (`cat -A` do arquivo) e corrigido removendo a linha corrompida e reescrevendo a linha correta.

**Lição operacional:** sempre validar o conteúdo final de arquivos gravados remotamente via SSH quando a linha contém múltiplos níveis de variáveis com `$`, especialmente ao construir o comando em camadas de aspas aninhadas.

### Finding 4 — `/etc/oratab` criado pelo DBCA com flag `N` (impacto: MÉDIO)

O DBCA cria automaticamente uma entrada em `/etc/oratab`, mas com o terceiro campo `N` (não inicia automaticamente). O `dbstart` (usado pelo `ExecStart` do serviço systemd `oracle-database`) só inicia bancos com flag `Y` — sem a correção, o serviço subiria "de boa" (`RemainAfterExit=yes` mascararia o problema) mas o banco nunca seria iniciado de fato após um reboot. Corrigido com `sed` para `Y`.

### Finding 5 — Divergências entre `.env`/CLAUDE.md e o ambiente real (impacto: BAIXO)

- Gold Image 19c estava salva como `LINUX.X64_193000_db_home-002.zip`, divergente do nome esperado no `.env` — renomeada na VM.
- O disco de dados (`DISK_DEVICE=/dev/sdb`) é de **500 GB**, não 40 GB como documentado no CLAUDE.md — não bloqueou a criação do LVM (usa `100%FREE`), só a documentação estava desatualizada.

### Handoff de processos manuais para systemd

Seguindo a regra operacional do projeto, o listener e o banco (iniciados manualmente via `lsnrctl start` e `dbca`) foram parados de forma limpa e depois subidos via `systemctl start` para o systemd assumir o controle do ciclo de vida dos processos.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 2** | Download do patch mais recente via AutoUpgrade (`java -jar autoupgrade.jar -download`, usando o Java 11 embutido no Oracle Home) |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/01-prepare-ambiente.md` |
| Repositório local ISO | `/etc/yum.repos.d/local-iso.repo` → `/mnt/iso` (persistido em fstab) |
| Log de instalação Oracle | `/u01/app/oraInventory/logs/InstallActions2026-07-18_11-23-20AM` |
| Log DBCA | `/u01/app/oracle/cfgtoollogs/dbca/ORCL/ORCL.log` |
| Relatório HTML | `HTML/Atividade1/atividade1_report.html` |
