# Atividade 2 — Download e Atualização do AutoUpgrade

**Módulo:** 2 — Baixar autoupgrade.jar mais recente e verificar estado do Oracle Home
**Data:** 18/07/2026
**Banco:** Oracle 19c (19.3.0.0.0) — non-CDB `ORCL` / host `<HOSTNAME>`

---

## Objetivo

Atualizar o `autoupgrade.jar` (copiado na Atividade 1 na versão base 19.3/2019) para a versão mais recente disponível publicamente pela Oracle, e levantar um baseline dos patches já instalados no Oracle Home 19c antes de qualquer aplicação de patch nas próximas atividades.

---

## Metodologia

1. **Verificação de Java** — confirmado OpenJDK 17 disponível no SO.
2. **Backup do jar atual** — cópia de segurança do `autoupgrade.jar` original antes de sobrescrever.
3. **Download da versão mais recente** — ver Finding 1 abaixo (workaround por falta de internet na VM).
4. **Verificação de versão** — usando o Java 11 embutido no Oracle Home (`${ORACLE_HOME_19C}/jdk/bin/java`), nunca o Java 17 do sistema.
5. **Baseline de patches** — `opatch lsinventory` e consulta a `dba_registry_sqlpatch`.

---

## Comandos para Replicação (Passo a Passo)

> Sequência real, na ordem de execução. Rodar como `oracle` salvo onde indicado. O download do jar foi feito **na máquina local** (com internet) e enviado via `scp`, porque a VM está isolada. Senhas `***`; IPs/hostnames `<IP>`/`<HOSTNAME>`.

### Passo 1 — Verificar Java

```bash
# oracle (na VM)
java -version    # OpenJDK 17 instalado na Atividade 1
```

### Passo 2 — Atualizar o autoupgrade.jar (download local + scp)

```bash
# --- Na MÁQUINA LOCAL (com internet) ---
# baixar a versão mais recente do endpoint público da Oracle
#   PowerShell: Invoke-WebRequest -Uri "https://download.oracle.com/otn-pub/otn_software/autoupgrade.jar" -OutFile autoupgrade.jar
# enviar para a VM:
scp -i <chave_ssh> autoupgrade.jar root@<IP>:/install/autoupgrade.jar

# --- Na VM (root) ---
# backup do jar antigo antes de sobrescrever (feito antes do scp acima)
cp /install/autoupgrade.jar /install/autoupgrade.jar.bak_$(date +%Y%m%d)
chown oracle:oinstall /install/autoupgrade.jar

# --- Verificar versão com o Java 11 embutido no Oracle Home (NUNCA o java 17 do SO) ---
# oracle
/u01/app/oracle/product/19.3.0/db_1/jdk/bin/java -jar /install/autoupgrade.jar -version
#   → build.version 26.4.260701 · targets 12.2,18,19,21,23,26
```

### Passo 3 — Baseline de patches no Oracle Home 19c

```bash
# oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/db_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH

# patches no Oracle Home (via OPatch)
$ORACLE_HOME/OPatch/opatch lsinventory | grep -E 'Oracle Database|Patch [0-9]|Applied on'

# versão e RUs no dicionário
sqlplus -s / as sysdba <<'EOF'
SELECT version_full FROM v$instance;
SELECT patch_id, status, description FROM dba_registry_sqlpatch
  ORDER BY action_time DESC FETCH FIRST 5 ROWS ONLY;
EOF
```

---

## Resultados

| Item | Resultado |
|---|---|
| `autoupgrade.jar` | Atualizado de `20190207` (base) para **`26.4.260701`** |
| Targets suportados | `12.2, 18, 19, 21, 23, 26` |
| Backup do jar antigo | `autoupgrade.jar.bak_20260718` |
| Oracle Home 19c (opatch) | Apenas software base `19.0.0.0.0`, sem patches adicionais |
| Banco (sqlplus) | `VERSION_FULL = 19.3.0.0.0` |
| Patch aplicado | `29517242` — Database Release Update 19.3.0.0.190416 (RU-base do instalador) |

---

## Findings

### Finding 1 — VM sem internet: download feito na máquina local + scp (impacto: MÉDIO)

A VM continua sem rota para a internet pública (ver Atividade 1). Como o `autoupgrade.jar` é um arquivo público pequeno (~7 MB, endpoint `download.oracle.com/otn-pub/otn_software/autoupgrade.jar`, sem autenticação), a solução foi:

1. Testar conectividade da máquina local (que tem internet) ao endpoint — `200 OK`.
2. Baixar o arquivo localmente.
3. Enviar via `scp` para `${INSTALL_DIR}/autoupgrade.jar` na VM.
4. Corrigir o dono do arquivo (`oracle:oinstall`) e validar a versão com o Java 11 embutido no Oracle Home.

**Padrão a repetir:** para qualquer arquivo público pequeno/médio nas próximas atividades, baixar na máquina local e transferir via `scp` em vez de tentar `wget`/`curl` diretamente na VM. Patches do My Oracle Support (que exigem login) provavelmente precisarão ser baixados manualmente pelo usuário e informados por caminho de arquivo.

### Finding 2 — Baseline de patches confirmado antes do patching (impacto: BAIXO, informativo)

O Oracle Home está exatamente como esperado após a Atividade 1: apenas o software base 19.0.0.0.0 via `opatch`, e o banco reportando o RU-base `29517242` (19.3.0.0.190416) que já vem embutido no instalador. Nenhum patch adicional foi aplicado ainda — ponto de partida limpo para a Atividade 4.

---

## Próximos Passos

| Atividade | Descrição |
|---|---|
| **Atividade 3** | Criar novo Oracle Home com AutoUpgrade |

---

## Evidências

| Artefato | Localização |
|---|---|
| Skill executada | `.claude/skills/02-download-patch.md` |
| Jar atualizado | `${INSTALL_DIR}/autoupgrade.jar` (versão 26.4.260701) |
| Backup do jar anterior | `${INSTALL_DIR}/autoupgrade.jar.bak_20260718` |
| Relatório HTML | `HTML/Atividade2/atividade2_report.html` |
