# Planos do PFC 2

Pasta dedicada ao planejamento, escrita pelas duas discentes para caminharmos juntas no TCC.

## Regra de versionamento

1. **Nunca editar um plano já publicado.** A cada mudança de plano, copiar a versão mais recente para um arquivo novo.
2. Nome do arquivo: `PLANO_v1.x_AAAA-MM-DD.md`, com a versão incrementada (`v1.0`, `v1.1`, `v1.2`, ...) e a **data da mudança**.
3. No topo do plano novo, preencher `Versão`, `Data` e `Anterior`.
4. Commitar o plano novo sozinho, com a mensagem descrevendo **o que mudou** em relação à versão anterior (motivo da mudança + lista das alterações).
5. Seguir o Git flow de `docs/ORGANIZACAO_PROJETO.md`: branch `feature/plano-v1.x`, merge `--no-ff` na `main`.

Para ver o que mudou entre duas versões: `git log -- plans/` (mensagens) ou `git diff --no-index plans/PLANO_v1.0_*.md plans/PLANO_v1.1_*.md` (texto).

## Histórico

| Versão | Data | Motivo | Arquivo |
|---|---|---|---|
| v1.0 | 15/10/2026 | Plano de execução nas 2 VMs idênticas às do PFC 1 (XDP nativo, balanceamento e failover) | [PLANO_v1.0_2026-10-15.md](PLANO_v1.0_2026-10-15.md) |
