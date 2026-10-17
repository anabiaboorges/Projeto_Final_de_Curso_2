# Projeto Final de Curso 2: eBPF/XDP no plano de usuário 5G

**XDP nativo, balanceamento de carga e failover de UPF em uma rede 5G simulada**

Engenharia de Computação · Escola de Engenharia Elétrica, Mecânica e de Computação (EMC/UFG) · 2026

| | |
|---|---|
| Discentes | Ana Beatriz Borges Cordeiro · Bianca Pereira de Carvalho |
| Orientador | Prof. Flávio Geraldo Coelho Rocha |
| PFC 1 (repositório) | [biancapcarvalho/pfc_ebpf_redes_5g](https://github.com/anabiaboorges/Projeto_Final_de_Curso_1) |
| Status | 🟡 Em andamento: preparação do ambiente experimental |

> Este README é um documento vivo e será atualizado ao longo do semestre (ver [Diário do projeto](#diário-do-projeto)).

---

## Sumário
- [Contexto: o que foi feito no PFC 1](#contexto-o-que-foi-feito-no-pfc-1)
- [PFC 2: proposta](#pfc-2-proposta)
- [Ambiente experimental](#ambiente-experimental)
- [Metodologia de avaliação](#metodologia-de-avaliação)
- [Cronograma](#cronograma)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Referências](#referências)
- [Diário do projeto](#diário-do-projeto)

---

## Contexto: o que foi feito no PFC 1

**Título:** *Monitoramento e Observabilidade em Redes 5G Baseadas em Software Utilizando eBPF*
(Ana Beatriz B. Cordeiro, Bianca P. de Carvalho, Jonas A. Kunzler, Flávio G. C. Rocha)

O PFC 1 investigou o eBPF como mecanismo de monitoramento na **UPF** (*User Plane Function*) de uma rede 5G simulada.

**Ambiente:** uma VM KVM (Ubuntu 22.04, kernel 5.15, 8 vCPUs) com o 5G em contêineres Docker. O núcleo foi o **Open5GS 2.7.6** com MongoDB, e o UE e a gNB foram emulados pelo **UERANSIM**. As interfaces N2, N3, N4, N6 e SBI foram montadas como *bridges* Docker. Os programas eBPF foram anexados à interface **N3 da UPF**, por onde passa o tráfego GTP-U.

**Frente 1: monitoramento passivo.** Programas em C/libbpf extraíram vazão e IAT (*inter-arrival time*) dos pacotes GTP-U em quatro cenários:

| Cenário | CPU total média | %usr | %sys | %soft |
|---|---|---|---|---|
| Baseline | 8,06% | 1,87 | 3,75 | 2,44 |
| XDP (modo **genérico**) | 14,51% | 3,32 | 6,69 | 4,50 |
| TC (ingress) | 13,99% | 3,26 | 6,53 | 4,20 |
| tcpdump | 16,93% | 4,32 | 8,51 | 4,10 |

*30 iterações por cenário, iperf3 UDP a 100 Mbps com pacotes de 700 B, CPU medida com `mpstat`.*

**Frente 2: uso ativo.** Foi reproduzido o classificador *in-kernel* (Random Forest em XDP com *tail calls*) de Galvão et al. para separar tráfego **eMBB × URLLC**. Ele atingiu **99,98% de acurácia** com **~0,22 µs de inferência**, contra ~59 µs no espaço de usuário.

**Principais conclusões e limitações**
- O eBPF teve **menor overhead** que o tcpdump e a mesma precisão nas métricas.
- **XDP genérico ≈ TC**: os dois rodam depois da alocação do `sk_buff`, então o XDP não mostrou sua vantagem.
- O **XDP nativo** não foi avaliado. Nas *veth* do Docker, ele exige um programa XDP também no par do outro lado. Ficou como **trabalho futuro**, assim como o uso ativo do eBPF na rede (balanceamento e failover).

---

## PFC 2: proposta

O PFC 2 segue as duas direções deixadas em aberto pelo PFC 1:

### Frente 1: XDP em modo nativo
Habilitar o XDP no **modo nativo (*driver mode*)**, que atua **antes** da alocação do `sk_buff`, e medir o ganho de CPU frente ao XDP genérico e ao TC.

- Os programas de monitoramento do PFC 1 serão anexados à interface de entrada da UPF, agora com driver que suporta XDP nativo (`virtio_net`).
- Serão comparados quatro cenários: **baseline · XDP nativo · XDP genérico · TC**.
- Haverá também um microbenchmark com pacotes pequenos, para isolar o custo do *hook*.

### Frente 2: balanceamento de carga e failover de UPF
Usar o eBPF de forma **ativa** no plano de dados:

- **Várias UPFs** atrás de um endereço N3 virtual (VIP).
- Um **programa XDP balanceador**: lê o TEID do GTP-U, escolhe a UPF (*round-robin* ou *hash* consistente) e encaminha com `XDP_REDIRECT`.
- **Detecção de falha** por *heartbeat* (GTP-U Echo) registrado em mapas eBPF.
- **Redirecionamento automático** do tráfego para uma UPF saudável.
- Métricas: overhead de CPU, distribuição da carga, tempo de detecção e tempo de interrupção do tráfego durante a falha.

---

## Ambiente experimental

Os experimentos rodam no **servidor KVM do PFC 1**, em **duas VMs idênticas**: a VM original do PFC 1 e uma segunda VM disponibilizada pelo orientador com a mesma configuração.

```
Servidor KVM (o mesmo do PFC 1)
├── rede de gerência (NAT/Internet) ── enp1s0 de cada VM
├── rede dedicada entre as VMs (bridge, virtio multiqueue) ── enp2s0 → N3 + N4
│
├── VM1 "core"  · VM original do PFC 1
│     Open5GS (plano de controle) + MongoDB
│     UERANSIM (gNB + UE) · geradores de tráfego
│     enp2s0 = 192.168.100.10
│
└── VM2 "upf"   · VM nova, idêntica à do PFC 1
      enp2s0 = 192.168.100.11 (+ VIP 192.168.100.100)
      ★ XDP nativo no driver virtio_net (entrada N3 de toda a UPF)
      UPF 1 · UPF 2 · UPF 3 (contêineres) · DN (iperf3)
      healthd e agentes que atualizam os mapas eBPF
```

| | VM1 · core | VM2 · upf |
|---|---|---|
| Origem | VM original do PFC 1 | Clone da VM do PFC 1 |
| Recursos | 8 vCPU · 32 GiB | 8 vCPU · 32 GiB |
| Sistema | Ubuntu 22.04.5 · kernel 5.15 | Ubuntu 22.04.5 · kernel 5.15 |
| Papel | Núcleo 5G, gNB, UE, geradores | UPFs, DN, programas eBPF/XDP |

- **Por que duas VMs:** no PFC 1, a interface N3 da UPF era um par *veth* do Docker, o que impediu o XDP nativo. Na VM2, o tráfego da UPF entra por uma placa **virtio-net**, cujo driver implementa XDP nativo antes da alocação do `sk_buff`.
- **Por que VMs idênticas às do PFC 1:** mesmo hipervisor, mesmos recursos e mesmo kernel (5.15), o que mantém a comparação com o PFC 1 e evita virtualização aninhada.
- **Requisitos da rede entre as VMs:** placa virtio com backend vhost, multiqueue (`queues = 2 × vCPU`, ou seja, 16), MTU 1500 e nenhum outro tráfego.
- **Ambiente alternativo:** um notebook com WSL2 e KVM aninhado (uma VM para a UPF) serve para desenvolvimento e como reserva enquanto a VM2 não fica pronta. Scripts e composes são os mesmos nos dois ambientes.
- **Ferramentas:** Open5GS, UERANSIM, Docker, KVM/libvirt, clang/LLVM, libbpf, BCC, bpftool, xdp-tools, iperf3, mpstat.

---

## Metodologia de avaliação

O protocolo é o mesmo do PFC 1, para manter os resultados comparáveis:

- **30 iterações por cenário**, executadas em **ordem aleatória em blocos**.
- Tráfego UDP controlado (`iperf3`, pacotes de 700 B), com o ambiente reiniciado entre iterações.
- CPU coletada com `mpstat` (%usr, %sys, %soft) e vCPUs fixadas (*pinning*).
- Estatística: Shapiro–Wilk, Levene, ANOVA com Tukey HSD (ou Welch/Kruskal–Wallis quando necessário) e tamanho de efeito.
- A comparação com o PFC 1 será **relativa ao baseline**, porque o hardware é diferente.

---

## Cronograma

| Período | Atividade |
|---|---|
| Out/2026 | Preparação do ambiente (VM2 e rede dedicada no servidor; desenvolvimento no notebook) e validação do attach nativo |
| Out–Nov | 5GC distribuído e porte dos programas do PFC 1 · várias UPFs e redirecionamento XDP |
| Nov | Experimentos da Frente 1 · balanceador + heartbeat + bancada sintética |
| Nov | Experimentos de balanceamento e failover · análise estatística |
| Dez | Relatório final e apresentação |

---

## Estrutura do repositório

*Planejada; será criada conforme o projeto avançar.* As regras de organização, branches (`feature/…`, `bugfix/…`) e merge na `main` estão em **[docs/ORGANIZACAO_PROJETO.md](docs/ORGANIZACAO_PROJETO.md)**.

```
.
├── docs/         planos de trabalho, fichas de referências, notas
├── scripts/      criação do ambiente e execução dos experimentos
├── ebpf_src/     programas eBPF (monitoramento XDP/TC, balanceador)
├── userspace/    healthd, agentes PFCP, geradores de tráfego, análise
├── docker/       docker-compose do núcleo e das UPFs
└── results/      saídas brutas dos experimentos (mpstat, iperf, eventos)
```

---

## Referências

- Huang, Y. X.; Wang, K. C.; Ke, B. S. *Accelerating 5G Service-Based Architecture with eBPF*. ICNCC 2023. doi:[10.1145/3638837.3638883](https://doi.org/10.1145/3638837.3638883)
- Molè, M. et al. *Performance Implications at the Intersection of AF_XDP and Programmable NICs*. eBPF '25. doi:[10.1145/3748355.3748359](https://doi.org/10.1145/3748355.3748359)
- Rocha, F. G. C. (CERISE/UFG). *Failover de UPF para Aplicações de Borda*. Apresentação, mar/2026.
- Galvão, J. S. S. et al. Classificador de tráfego *in-kernel* com eBPF (base do caso de uso do PFC 1). [jose-galvao/eBPF-Classification](https://github.com/jose-galvao/eBPF-Classification)
- [eBPF.io](https://ebpf.io) · [Open5GS](https://open5gs.org) · [UERANSIM](https://github.com/aligungr/UERANSIM)

---

## Diário do projeto

| Data | Registro |
|---|---|
| 15/10/2026 | Repositório criado; README inicial com o resumo do PFC 1 e a proposta do PFC 2. |
| 15/10/2026 | Spec de organização e Git flow básico (`feature/`, `bugfix/`, merge na `main`) e `.gitignore`. |
| 15/10/2026 | Ambiente definido: duas VMs idênticas às do PFC 1 no servidor KVM (VM2 disponibilizada pelo orientador); notebook como ambiente de desenvolvimento. |
