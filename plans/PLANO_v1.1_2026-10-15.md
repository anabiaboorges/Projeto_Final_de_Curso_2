# Plano de Ação do PFC 2: classificador na entrada + múltiplas UPFs

**Versão**: v1.1 · **Data**: 15/10/2026 · **Anterior**: [v1.0](PLANO_v1.0_2026-10-15.md)
**Motivo da mudança**: reunião de orientação de 15/10/2026 (ata mantida em `transcricao_reuniao/ATA_REUNIAO_2026-10-15.md`, fora do repositório)
**Partes reaproveitadas da v1.0** estão indicadas com **[reuso]**.
**Entrega**: dezembro de 2026 (cerca de 8 semanas)

---

## 1. Objetivo em uma frase

Usar um **classificador de tráfego em eBPF/XDP, no host, na entrada do plano de usuário**, para decidir **para qual de duas (ou mais) UPFs ativas** cada fluxo vai (URLLC × eMBB), em vez de decidir se o tráfego **pula** a UPF como em Galvão et al. Assim, todas as tarefas da UPF previstas pelo 3GPP continuam sendo executadas.

O que o trabalho entrega:
1. **Cenário funcional** (pedido central do orientador): 2 UPFs + 2 UEs + programa XDP que reparte o tráfego entre as UPFs, primeiro por regra simples e depois pela saída do classificador Random Forest do PFC 1.
2. **Avaliação**: acerto do direcionamento, isolamento do URLLC sob carga eMBB, custo de CPU e tempo de decisão.
3. **Extensões, se o tempo permitir**: direcionamento por carga, failover, mais recursos para uma UPF, XDP nativo.

### Contribuição em relação a Galvão et al.
| | Galvão et al. | PFC 2 |
|---|---|---|
| UPFs | 1 | 2+ ativas |
| Saída do classificador | Passa pela UPF **ou** faz *bypass* | **Qual UPF** atende o fluxo |
| Compliance 3GPP | *Bypass* pula as funções da UPF | Todo tráfego passa por uma UPF |
| Resiliência | Não trata | Base para balanceamento e failover |

---

## 2. Ambiente: uma VM só

**Principal**: a VM do PFC 1 (Ubuntu 22.04.5, kernel 5.15.0-176, 8 vCPU, 32 GiB, Docker, Open5GS 2.7.6, UERANSIM 3.2.8, BCC 0.36.1 em `/opt/bcc`), **assim que o acesso ao hipervisor voltar** (ação do orientador com o Gustavo).

**Enquanto a VM estiver inacessível**: desenvolver no notebook (WSL Ubuntu 22.04 com Docker, ~7,7 GB de RAM). O cenário cabe: núcleo Open5GS + 2 UPFs + 2 UEs + DN fica em torno de 3–4 GB. Atenção: o kernel do WSL é 6.6, diferente do 5.15 da VM; **nenhuma medição oficial no notebook**, só desenvolvimento e validação funcional.

### 2.1 Topologia (rede Docker do PFC 1, com a 2ª UPF e o 2º UE)

```
                          VM do PFC 1 (host, kernel 5.15)
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  b-ue-urllc ┐                                                           │
 │  b-ue-embb  ┴─N2/RAN─► b-gnb ──veth──► [ XDP no veth do host da gNB ]   │
 │                         10.130.0.100        │  lê GTP-U + pacote interno │
 │                                             │  classifica / aplica regra │
 │                                   ┌─────────┴─────────┐                 │
 │                                   ▼                   ▼                 │
 │                           veth ─► upf1 (eMBB)   veth ─► upf2 (URLLC)    │
 │                              10.130.0.21           10.130.0.22          │
 │                                   └──── N6 ───┬───────┘                 │
 │                                               ▼                         │
 │                                       b-dn 10.150.0.100 (iperf3 -s,     │
 │                                       receptores com timestamp)         │
 │  controle: mongo, nrf, scp, amf, smf (N4 → upf1 .21 e upf2 .22), ...    │
 └─────────────────────────────────────────────────────────────────────────┘
```

| Contêiner | N3 | N4 | N6 | Pool de UEs | DNN |
|---|---|---|---|---|---|
| upf1 (eMBB) | 10.130.0.21 | 10.140.0.21 | 10.150.0.21 | 10.45.0.0/16 | `internet` |
| upf2 (URLLC) | 10.130.0.22 | 10.140.0.22 | 10.150.0.22 | 10.46.0.0/16 | `urllc` |
| b-ue-embb | — | — | — | recebe IP de 10.45/16 | `internet` |
| b-ue-urllc | — | — | — | recebe IP de 10.46/16 | `urllc` |

- **Onde fica o código eBPF**: no **host** (a VM), anexado ao **veth externo da gNB** na bridge N3, exatamente como o classificador do PFC 1 (por ali passa o **uplink**). Isso também corresponde ao que o orientador concluiu do artigo do Galvão: o classificador roda no kernel do SO onde estão os contêineres, antes da UPF.
- **Modo XDP**: **genérico** (como no PFC 1). O nativo vira extensão (§7).
- **DN**: rotas de volta `10.45.0.0/16 via 10.150.0.21` e `10.46.0.0/16 via 10.150.0.22`.

### 2.2 Configuração do Open5GS (a verificar na Etapa 0)
- `smf.yaml`: duas UPFs no cliente PFCP, cada uma associada a um DNN (o Open5GS 2.7 permite selecionar a UPF por `dnn`; confirmar a sintaxe na versão 2.7.6).
- `upf1.yaml` / `upf2.yaml`: endereços próprios de PFCP/GTP-U e `session` com o pool e o DNN de cada uma.
- Assinante no WebUI/MongoDB com os dois DNNs, ou dois assinantes (um IMSI por UE).
- UERANSIM: dois arquivos de UE (IMSIs diferentes), cada um pedindo a sessão no seu DNN.
- **[reuso]** O contêiner da UPF continua privilegiado, com `/sys/fs/bpf`, `/lib/modules` e `/sys/kernel/btf` montados.

---

## 3. A questão do estado de sessão (decidir com o orientador)

Uma UPF só encaminha o pacote GTP-U de um TEID se tiver a sessão PFCP dele (detalhes em `apresentacao/ROTEIRO_APRESENTACAO.md`). Isso define dois modos de operação:

| | **Modo C: coerente (garantido)** | **Modo L: livre (meta)** |
|---|---|---|
| Onde está a sessão | Só na UPF que o SMF escolheu pelo DNN | Nas duas UPFs (replicada) |
| O que o classificador faz | Direciona o fluxo para a UPF da sua classe; como cada UE já está ancorado na UPF da sua classe, o acerto do classificador entrega na UPF certa | Escolhe **qualquer** UPF; o mesmo UE pode ter fluxos em UPFs diferentes |
| Se o classificador errar | O pacote chega à UPF sem a sessão e é descartado (o erro fica **visível e mensurável**) | O pacote é atendido pela outra UPF |
| Custo | Baixo | Proxy PFCP de replicação **[reuso: `pfcp_mirror`, Nível B da v1.0]** + reescrita de TEID |
| Habilita | Cenário funcional pedido (D2) | Balanceamento por carga e failover sem perda de sessão |

**Encaminhamento:** fazer o **Modo C** primeiro (atende o "cenário funcional, mesmo que simples"). Levar ao orientador a pergunta do Modo L. Observação honesta para a reunião: no Modo C, a escolha da UPF é, na prática, também feita pelo DNN; o classificador agrega valor real quando a classe **não é conhecida de antemão** (Modo L, ou um UE com tráfego misto).

---

## 4. O programa XDP de direcionamento

### 4.1 Por pacote (uplink, no veth externo da gNB)
1. Parse de Eth / IPv4 / UDP 2152 / GTPv1-U → TEID → IPv4/UDP **interno** (IP do UE, portas). Se não for GTP-U, `XDP_PASS`.
2. Chave do fluxo: 5-tupla interna. Procurar em `flow_map` (LRU_HASH: fluxo → {UPF, classe, estado}).
3. **Fluxo já decidido** → enviar para a UPF gravada (afinidade; evita trocar de UPF no meio do fluxo).
4. **Fluxo novo / em observação** → atualizar as features (IAT médio e variância do fluxo e da janela de 500 ms, como no PFC 1) e:
   - na **Etapa 1**, aplicar a **regra simples** (ex.: porta UDP de destino ou faixa de IP do UE, definidas pelas discentes);
   - na **Etapa 2**, chamar o **classificador Random Forest** via *tail calls* (Model Part1 → Part2 → Part3, aritmética inteira, código do PFC 1/Galvão via `emlearn`) **depois de N pacotes** do fluxo. Até lá, o pacote segue `XDP_PASS` para a UPF original.
5. **Encaminhar** para a UPF escolhida:
   - se a UPF escolhida for a de destino original → `XDP_PASS` (a bridge entrega normalmente);
   - se for outra → reescrever IP de destino externo (com ajuste incremental do checksum IPv4; checksum UDP 0 se a gNB enviar 0) e MAC de destino, e `bpf_redirect_map(&upf_devmap, idx, 0)` para o veth do lado do host da UPF **[reuso da v1.0: `upf_devmap`, `upf_mac`, requisito de GRO/XDP no veth receptor]**.
6. Contadores por classe e por UPF em `stats` (PERCPU) e tempo de decisão (`bpf_ktime_get_ns`).

### 4.2 Mapas
| Mapa | Tipo | Conteúdo |
|---|---|---|
| `flow_map` | LRU_HASH | 5-tupla interna → {upf, classe, n_pkts, features} |
| `class_to_upf` | ARRAY | classe (0 = URLLC, 1 = eMBB) → índice da UPF (permite trocar a política sem recompilar) |
| `upf_devmap` | DEVMAP | índice → ifindex do veth da UPF |
| `upf_addr` | ARRAY | índice → {IP N3, MAC} |
| `prog_array` | PROG_ARRAY | partes do Random Forest (*tail calls*) |
| `stats` | PERCPU_ARRAY | pacotes por classe × UPF, descartes, tempo de decisão |

### 4.3 Ferramenta
Começar em **BCC** (o classificador do PFC 1 já roda assim, com `/opt/bcc`). Migrar para libbpf/CO-RE só se houver tempo; não é requisito.

---

## 5. Etapas e gates

| Etapa | Entrega | Critério de pronto (gate) | Prazo |
|---|---|---|---|
| **E0** Ambiente | Acesso à VM (ou notebook); ambiente do PFC 1 de pé; **ler a nova versão do Galvão** | `ping` do UE até o DN pelo `uesimtun0` | 21/10 |
| **E1** Multi-UPF | 2 UPFs + 2 UEs (um por tipo de tráfego); cada UE ancorado na UPF do seu DNN | **G1**: `tcpdump` mostra o tráfego do UE-URLLC só na upf2 e o do UE-eMBB só na upf1 | 24/10 |
| **E2** Direcionamento por regra | Programa XDP no veth da gNB com regra "bobinha" + redirect para as UPFs; contadores | **G2**: tráfego repartido conforme a regra, sem descartes inesperados | 31/10 |
| **E3** Classificador | Random Forest do PFC 1 decidindo a classe → `class_to_upf` → redirect | **G3**: acerto do direcionamento ≥ acurácia do PFC 1 menos uma margem; tempo por decisão medido | 07/11 |
| **E4** Experimentos | Rodadas oficiais na VM (§6) | 30 repetições por cenário | 21/11 |
| **E5** Extensão (escolher 1 com o orientador) | Modo L / carga / failover / nativo (§7) | definido na reunião de ~07/11 | 28/11 |
| **E6** Escrita | Relatório, figuras e slides | — | 13/12 |

---

## 6. Experimentos (VM do PFC 1, 30 repetições, protocolo do PFC 1)

**Tráfego** (D5 da ata): um UE por tipo de tráfego.
- **UE-URLLC**: gerador do PFC 1 (UDP, IAT constante de 8 ms, 128 B) + `seq_sender` com número de sequência e *timestamp* para latência/jitter **[reuso]**.
- **UE-eMBB**: gerador do PFC 1 (rajadas de 20 × 1000 B, IAT 0,5 ms, 20 ms entre rajadas) e **iperf3** para carga pesada (o orientador confirmou que o iperf pode ser usado).

| # | Experimento | Cenários | Métricas |
|---|---|---|---|
| X1 | Funcional / acerto do direcionamento | regra simples × classificador | % dos pacotes e dos fluxos de cada classe entregues à UPF prevista; descartes; nº de pacotes até a decisão |
| X2 | **Isolamento do URLLC** sob carga eMBB | (a) 1 UPF para tudo · (b) 2 UPFs, direcionamento pelo classificador; carga eMBB crescente (ex.: 50/100/200 Mbps via iperf3) | latência e jitter do URLLC (p50/p99), perda; é o resultado que justifica separar as classes |
| X3 | Custo de CPU | baseline (2 UPFs, sem XDP) · XDP com regra · XDP com classificador | `%usr`, `%sys`, `%soft` com `mpstat` no host, como no PFC 1 |
| X4 | Tempo de decisão | classificador in-kernel × user space | ns por decisão (comparável aos 0,22 µs × ~59 µs do PFC 1) |

Para o X2 mostrar efeito, fixar as UPFs em CPUs com `cpuset` e, se preciso, limitar a CPU da UPF compartilhada no cenário (a). Usar **as mesmas** afinidades em todos os cenários.

---

## 7. Extensões (o orientador mencionou; escolher na reunião de ~07/11)

| Extensão | O que é | Depende de | Reuso |
|---|---|---|---|
| Modo L | Sessão nas duas UPFs, classificador escolhe livremente | Proxy PFCP de replicação; reescrita de TEID | `pfcp_mirror.py`, `teid_xlate` |
| Direcionamento por carga | Se a UPF da classe passar de um limite de carga, novos fluxos vão para a outra | Modo L | `upf_state`, tabela Maglev, índice de Jain |
| Failover | UPF cai → fluxos da classe vão para a UPF viva | Modo L (senão a sessão se perde) | `healthd.py` (GTP-U Echo 20 ms, k = 3), detecção no próprio XDP |
| Mais recursos para uma UPF | Aumentar `cpuset`/limites da UPF sobrecarregada | — | `docker update --cpus/--cpuset-cpus` |
| XDP nativo | Mesmo programa em modo nativo no veth (exige XDP/GRO no par) | — | Plano C da v1.0; resultado do teste nativo já feito por uma das discentes |

---

## 8. Cronograma

| Semana | Datas | Trabalho | Gate |
|---|---|---|---|
| S0 | 15–21/out | Acesso à VM (orientador/Gustavo); ambiente de pé (VM ou notebook); ler nova versão do Galvão; estudar a seleção de UPF por DNN no Open5GS | E0 |
| S1 | 22–28/out | 2 UPFs + 2 UEs funcionando; começar o XDP de direcionamento | **G1** |
| S2 | 29/out–04/nov | Direcionamento por regra completo; integrar o classificador | **G2** |
| S3 | 05–11/nov | Classificador direcionando; piloto de 5 rodadas; **decidir a extensão** com o orientador | **G3** |
| S4 | 12–18/nov | Rodadas oficiais X1–X4 (noites) | — |
| S5 | 19–25/nov | Extensão escolhida; análise estatística | — |
| S6 | 26/nov–02/dez | Relatório: metodologia e resultados | — |
| S7 | 03–13/dez | Relatório final, slides, ensaio, folga | — |

---

## 9. Riscos

| Risco | Mitigação |
|---|---|
| VM continua inacessível | Desenvolver no notebook; rodadas oficiais concentradas em S4; se o acesso for só pela UFG, reservar dias no laboratório |
| Open5GS 2.7.6 não seleciona a UPF por DNN como esperado | Alternativas: seleção por TAC/célula (2 gNBs) ou um SMF por UPF; em último caso, bancada sintética de GTP-U com receptores que contam por fluxo **[reuso: §5.5 da v1.0]** |
| Redirect do veth descartado sem aviso | Ativar GRO ou XDP_PASS no `eth0` do contêiner receptor **[reuso]** |
| Classificador precisa de vários pacotes para decidir | Política de "observação": primeiros N pacotes seguem a UPF original; medir N e o efeito |
| Efeito de isolamento (X2) não aparece | Aumentar a carga eMBB e restringir CPU da UPF compartilhada; reportar o resultado mesmo se for nulo |
| Versão do Galvão em revisão | Usar só para entendimento; citar a versão publicada |

---

## 10. Repositório (Git flow da `docs/ORGANIZACAO_PROJETO.md`)

Branches sugeridas, uma por etapa, mergeadas na `main` ao concluir:
- `feature/ambiente-multi-upf` (compose com upf2, configs do SMF/UPF, UEs) → E1
- `feature/xdp-direcionamento-regra` → E2
- `feature/classificador-direcionamento` → E3
- `feature/experimentos-x1-x4` (scripts de rodada, `mpstat`, análise) → E4

```
Projeto_Final_de_Curso_2/
├── ambiente/   docker-compose.yml  configs/{smf,upf1,upf2,ue-urllc,ue-embb}.yaml
├── ebpf_src/   steer_xdp.c  rf_part{1,2,3}.c
├── userspace/  loader.py  geradores/{urllc,embb}.py  seq_sender.py  seq_receiver.py
├── experimentos/  run_x1.sh ... run_x4.sh
├── resultados/
└── docs/       ORGANIZACAO_PROJETO.md  PLANO_ACAO.md  atas/
```

---

## 11. Próximos passos imediatos (até a próxima reunião)

1. Aguardar retorno do orientador sobre o acesso à VM. Enquanto isso, subir o compose do PFC 1 no notebook.
2. Ler a nova versão do Galvão (quando chegar): confirmar o ponto de anexação e como o caminho "verde" sobe para a UPF.
3. Configurar `smf.yaml` com 2 UPFs por DNN e subir 2 UEs; validar o **G1**.
4. Rascunhar o `steer_xdp.c` com a regra simples.
5. Levar para a reunião: a pergunta Modo C × Modo L (§3), a métrica de sucesso (§6) e se o XDP nativo sai do escopo.
