# DBAOCM — Imersão Oracle AutoUpgrade com IA

Imersão Oracle da DBAOCM de dois dias sobre Oracle AutoUpgrade — instalação, patching, conversão para Multitenant e migração para 23ai, tudo executado com apoio de IA (Claude Code).

## Ambiente

- **SO:** Oracle Linux 8.10 (VM VMware Workstation Pro — 6 vCPU / 6 GB RAM)
- **Banco de partida:** Oracle 19c (19.3.0.0.0) — non-CDB `ORCL` / host `<HOSTNAME>`
- **Destino final:** Oracle 23ai (23.26.2) — PDB Multitenant

## Portal

Abra [`HTML/index.html`](HTML/index.html) para navegar pelas atividades com barra de progresso e relatórios em HTML.

## Atividades

| Atividade | Tema | Skill |
| --- | --- | --- |
| [Atividade 1](Atividade1/relatorio-atividade1.md) | Preparação do Ambiente + Instalação Oracle 19c | `01-prepare-ambiente` |
| [Atividade 2](Atividade2/relatorio-atividade2.md) | Download do Patch mais Recente via AutoUpgrade | `02-download-patch` |
| [Atividade 3](Atividade3/relatorio-atividade3.md) | Criar novo Oracle Home com AutoUpgrade | `03-criar-oracle-home` |
| [Atividade 4](Atividade4/relatorio-atividade4.md) | Aplicar Patch com AutoUpgrade | `04-aplicar-patch` |
| [Atividade 5](Atividade5/relatorio-atividade5.md) | Converter non-CDB para Multitenant | `05-converter-multitenant` |
| [Atividade 6](Atividade6/relatorio-atividade6.md) | Migrar 19c → 23ai (23.26.1) | `06-migrar-23ai` |
| [Atividade 7](Atividade7/relatorio-atividade7.md) | Patch 23.26.1 → 23.26.2 | `07-patch-23262` |
| [Atividade 8](Atividade8/relatorio-atividade8.md) | Configurar Backup RMAN (FULL semanal + incremental via cron) | `08-configurar-backup` |
| Atividade 9 | Verificação Final + Resumo LinkedIn | `09-verificacao-final` |

> O link da Atividade 9 será preenchido quando for concluída.
