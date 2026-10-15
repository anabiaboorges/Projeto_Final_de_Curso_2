# Organização do Projeto e Git Flow

Este documento define como o repositório é organizado e como as mudanças entram nele. Vale para as duas integrantes e para qualquer ferramenta automatizada usada no projeto.

---

## 1. Estrutura de pastas

| Pasta | Conteúdo | Observação |
|---|---|---|
| `docs/` | Planos, esta spec, fichas de referências (`docs/referencias/`), notas técnicas | Markdown sempre que possível |
| `scripts/` | Criação do ambiente (`l1_net.sh`, `create_vm.sh`) e execução dos experimentos (`run_*.sh`) | Shell com `set -euo pipefail` |
| `ebpf_src/` | Programas eBPF em C (monitoramento XDP/TC, balanceador) e `Makefile` | Não versionar `.o` |
| `userspace/` | Código em espaço de usuário: `healthd`, agentes PFCP, geradores de tráfego, análise | Python 3 |
| `docker/` | `docker-compose` do núcleo e das UPFs, configs do Open5GS/UERANSIM | Uma subpasta por máquina (`l1-core/`, `vm-upf/`) |
| `results/` | Saídas dos experimentos, uma subpasta por experimento e rodada | Ver §5 |

Arquivos de raiz: `README.md` (visão geral e diário do projeto) e `.gitignore`.

---

## 2. Branches

### 2.1 Tipos
| Branch | Uso | Exemplo |
|---|---|---|
| `main` | Versão estável e integrada do projeto. **Ninguém faz commit direto nela.** | — |
| `feature/<descricao>` | Qualquer trabalho novo: código, script, experimento, documentação | `feature/balanceador-xdp` |
| `bugfix/<descricao>` | Correção de algo que já está na `main` | `bugfix/bounds-check-gtpu` |

### 2.2 Nomes
- Use letras minúsculas, palavras separadas por hífen, sem acentos e sem espaços.
- Descreva o assunto, não a pessoa: `feature/healthd-heartbeat`, não `feature/ana`.
- Se ajudar, prefixe com a frente do projeto: `feature/f1-attach-nativo`, `feature/f2-pfcp-mirror`.
- Prefira branches curtas, de poucos dias; um assunto por branch.

---

## 3. Git flow básico

```
main ──●─────────────●───────────────●──▶
        \           / \             /
         ●──●──●───●   ●──●────────●
      feature/xyz        bugfix/abc
```

### 3.1 Começar uma branch
```bash
git switch main
git pull                                  # sempre partir da main atualizada
git switch -c feature/<descricao>         # ou bugfix/<descricao>
```

### 3.2 Trabalhar
```bash
git add <arquivos>
git commit -m "Implementa parse do cabeçalho GTP-U no balanceador"
git push -u origin feature/<descricao>    # backup e visibilidade para a dupla
```

### 3.3 Concluir e mergear na `main`
Quando a branch estiver **concluída** (funciona, foi testada e a documentação foi atualizada):
```bash
git switch main
git pull
git merge --no-ff feature/<descricao>     # --no-ff mantém o registro da branch no histórico
git push
git branch -d feature/<descricao>         # apaga a branch local
git push origin --delete feature/<descricao>   # apaga a branch remota
```
Se houver conflito no merge, resolva-o na própria branch antes (`git merge main` dentro dela), teste de novo e só então faça o merge na `main`.

Também é possível fazer o merge por Pull Request no GitHub (botão *Create a merge commit*). O resultado é o mesmo, e o PR deixa espaço para a outra integrante revisar.

### 3.4 Checklist antes do merge
- [ ] O código compila e roda (`make` em `ebpf_src/`, scripts testados no ambiente).
- [ ] Nenhum arquivo gerado ou grande foi incluído (ver `.gitignore` e §5).
- [ ] O `README.md` foi atualizado se algo mudou para quem usa o repositório.
- [ ] Marcos importantes têm uma linha nova no **Diário do projeto** do README.

---

## 4. Commits

- Mensagem em português, no imperativo, curta (até ~70 caracteres na primeira linha): `Adiciona`, `Corrige`, `Remove`, `Atualiza`, `Implementa`.
- Detalhes, se necessários, em parágrafo separado por uma linha em branco.
- Um commit por mudança lógica; não misture correção e funcionalidade nova.
- **Não incluir linhas `Co-Authored-By`** nas mensagens de commit.
- Não versionar segredos (tokens, senhas, chaves SSH).

---

## 5. Resultados de experimentos

- Caminho: `results/<experimento>/<cenario>/<rodada>/`, por exemplo `results/xdpnat/xdp_nativo/017/`.
- Cada rodada guarda as saídas brutas (`mpstat.txt`, `iperf.json`, `attach.txt`, `vmstat.txt`) e um `meta.txt` com data, commit (`git rev-parse --short HEAD`), taxa e parâmetros.
- Uma campanha de experimentos é feita numa branch própria (`feature/exp-xdpnat`) e mergeada quando a campanha termina.
- Saídas grandes (pcaps, logs com mais de alguns MB) não entram no git: compacte e guarde fora do repositório, registrando no `meta.txt` onde ficaram.

---

## 6. Marcos (tags)

Ao final de cada gate do plano de execução, crie uma tag na `main`:
```bash
git tag -a g1-xdp-nativo -m "Gate G1: attach nativo validado no virtio_net"
git push origin g1-xdp-nativo
```
Tags previstas: `g0-kvm`, `g1-xdp-nativo`, `g2-rede`, `g3-sessao-pdu`, `g4-redirect`, `g5-pfcp-mirror`, `entrega-final`.
