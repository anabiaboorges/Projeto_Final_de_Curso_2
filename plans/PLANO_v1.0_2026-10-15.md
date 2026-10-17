# Plano de Execução do PFC 2 (2 VMs idênticas às do PFC 1)

**Escopo**: XDP nativo · balanceamento de UPF · failover
**Ambiente**: servidor KVM do PFC 1, com a VM original e uma segunda VM idêntica disponibilizada pelo orientador
**Base**: `PFC2_Plano_CONSOLIDADO_2VMs.tex`, `ROADMAP_TECNICO_PFC2.md`, `ANALISE_VIABILIDADE_PFC2.tex` e o que foi feito no PFC 1
**Repositório**: [github.com/anabiaboorges/Projeto_Final_de_Curso_2](https://github.com/anabiaboorges/Projeto_Final_de_Curso_2) (organização e Git flow em `docs/ORGANIZACAO_PROJETO.md`)
**Data**: 15/10/2026 (restam cerca de 8 semanas até a entrega em dezembro)
**Versões anteriores** (arquivadas, execução em notebook): `PLANO_EXECUCAO_NOTEBOOK_v1_16GB.md`, `PLANO_EXECUCAO_NOTEBOOK_v2_8GB.md`

---

## 0. Ambiente e decisão de arquitetura

### 0.1 As duas VMs
| | VM1 · core | VM2 · upf |
|---|---|---|
| Origem | VM original do PFC 1 | Clone da VM do PFC 1, disponibilizada pelo orientador |
| Hipervisor | KVM (servidor do PFC 1) | KVM (mesmo servidor) |
| Sistema | Ubuntu 22.04.5 LTS · kernel 5.15.0-176 | Ubuntu 22.04.5 LTS · kernel 5.15.0-176 |
| Recursos | 8 vCPU · 32 GiB · VT-x | 8 vCPU · 32 GiB · VT-x |
| Papel | Núcleo 5G (Open5GS + MongoDB), UERANSIM (gNB e UE), geradores de tráfego | UPF(s), DN, programas eBPF/XDP, `healthd` e agentes |

**Decisão.** Dividir o 5G entre as duas VMs e ligá-las por uma **rede dedicada no host com placa virtio-net**. Todo o tráfego N3 que chega à UPF entra pela placa `virtio_net` da VM2, onde o XDP roda em **modo nativo**, antes da alocação do `sk_buff`. É a arquitetura do `PFC2_Plano_CONSOLIDADO_2VMs.tex`, agora com recursos idênticos ao PFC 1.

> O hook XDP fica no kernel da VM2 (5.15, o mesmo do PFC 1). O driver `virtio_net` implementa XDP nativo, inclusive com filas TX dedicadas para XDP_TX e XDP_REDIRECT.

### 0.2 Por que este ambiente
- **Mesmo hipervisor, mesmos recursos e mesmo kernel do PFC 1**: a comparação com os resultados antigos fica direta, e o cenário genérico do PFC 1 pode ser refeito no mesmo hardware (§4.6).
- **Sem virtualização aninhada**: as VMs rodam direto no KVM do servidor.
- **Medição mais limpa**: o `mpstat` passa a ser coletado na VM2, onde estão o hook e a UPF, e não numa VM com todos os componentes.
- **Rodadas longas sem depender de máquina pessoal**: disparadas por `ssh` + `tmux`.

### 0.3 Informações a levantar no início
| Item | Por quê | Como |
|---|---|---|
| Modelo da CPU do host e se há outras VMs ativas | Documentar o hardware e avaliar ruído | Perguntar ao orientador; `lscpu` nas VMs |
| Se o host está superalocado (vCPUs > núcleos físicos) | Variância entre rodadas | Perguntar ao orientador |
| Se temos acesso ao `virsh` do host | Ajustes de XML (filas, offloads) e *pinning* | Perguntar ao orientador |
| Espaço em disco livre nas VMs | Imagens Docker e resultados | `df -h` |
| Onde estão os artefatos do PFC 1 na VM1 | Reaproveitar compose e programas | `ls ~/ambiente_5g` e diretórios dos programas eBPF |

### 0.4 Planos alternativos (só se um *gate* falhar)
- **Plano B, se a VM2 ou a rede dedicada atrasarem além de 25/10**: desenvolver no notebook com WSL2 + KVM aninhado e uma VM só para a UPF, como descrito em `PLANO_EXECUCAO_NOTEBOOK_v2_8GB.md`. Os scripts e composes são os mesmos; as **rodadas oficiais** continuam reservadas para as VMs do servidor.
- **Plano C, se o virtio não aceitar XDP nativo**: XDP nativo em **veth** dentro de uma VM, como no `PFC2_Plano_CONSOLIDADO_DockerVMUnica.tex`, atendendo os requisitos do veth (programa XDP ou GRO no par receptor, offloads desligados). Documentar como limitação.

---

## 1. Topologia alvo

```
Servidor KVM (host do PFC 1)
├── rede de gerência (NAT/Internet) ── enp1s0 de cada VM → apt, docker pull, ssh
├── br-pfc (rede dedicada, virtio multiqueue) ── enp2s0 de cada VM → N3 + N4
│
├── VM1 "core" (VM original do PFC 1: 8 vCPU, 32 GiB, kernel 5.15)
│     enp2s0 = 192.168.100.10
│     Docker: mongo, nrf, scp, amf, smf, ausf, udm, udr, pcf, nssf, bsf, webui
│     UERANSIM gNB + UE (network_mode: host; gtpIp/linkIp = 192.168.100.10)
│     geradores: iperf3 (cliente), trafgen (microbench), seq_sender (failover)
│     proxy PFCP (Nível B)
│
└── VM2 "upf" (VM nova, idêntica: 8 vCPU, 32 GiB, kernel 5.15)
      enp2s0 = 192.168.100.11 (+ VIP 192.168.100.100)
      ★ XDP NATIVO em enp2s0 (virtio_net) = ingresso N3 de TODA a UPF
      Docker: upf1/upf2/upf3 (rede "upfnet" 172.30.0.0/24)
              dn (servidor iperf3 e seq_receiver, rede "n6" 172.31.0.0/24)
      userspace: healthd (heartbeat GTP-U Echo) + agente de mapas
```

O caminho do plano de usuário fica assim: UE (`uesimtun0`) → gNB (VM1) → **br-pfc** → `enp2s0` da VM2 **[XDP]** → UPF → n6 → DN.

Os IPs da rede dedicada são uma proposta; se o host usar outra faixa, atualizar aqui e no README do repositório.

### 1.1 Afinidade de CPU (mesma lógica da Tabela IV do PFC 1)
| VM2 "upf" | vCPUs | VM1 "core" | vCPUs |
|---|---|---|---|
| IRQs/NAPI de `enp2s0` (`/proc/irq/*/smp_affinity_list`) | 0–1 | Banco e NFs de controle | 0–1 |
| `upf1` | 2–3 | gNB | 2–3 |
| `upf2` / `upf3` | 4–5 | UE e geradores | 4–5 |
| DN | 6 | Proxy PFCP (Nível B) | 6 |
| `healthd` + agentes | 7 | Folga / mpstat | 7 |

- Na Frente 1 (uma UPF só), `upf1` em `network_mode: host` usa as vCPUs 2–5.
- `irqbalance` desligado nas duas VMs (`systemctl disable --now irqbalance`).
- Contêineres com `cpuset` no compose; processos fora do Docker com `taskset`.
- Memória não é restrição (32 GiB por VM).

---

## 2. Gates de decisão (go/no-go)

| Gate | Quando | Critério de "passou" | Se falhar |
|---|---|---|---|
| **G0** | Até 25/10 | VM2 entregue; as VMs se pingam por `enp2s0`; `ethtool -i enp2s0` mostra `virtio_net` e `ethtool -l enp2s0` mostra as filas pedidas | Plano B (desenvolver no notebook) e cobrar a configuração |
| **G1** | Semana 1 após G0 | `ip -d link show enp2s0` mostra **`xdpdrv`** na VM2 e um contador XDP_DROP incrementa | Ajustar offloads e filas (§4.3); se não resolver, Plano C |
| **G2** | Semana 1 após G0 | Latência VM1↔VM2 estável; iperf3 UDP fixo com perda < 1% e CV < 10% em 5 repetições | Reduzir a taxa; revisar *pinning* (§1.1) e ruído do host |
| **G3** | Semana 2 | Sessão PDU de ponta a ponta entre as VMs; `ping`/`iperf3` passam pelo `uesimtun0` | Depurar Open5GS (N2/N3/N4) |
| **G4** | Semana 2 | XDP_REDIRECT `enp2s0` → veth de um contêiner é entregue (aparece no `tcpdump` dentro do contêiner) | Ligar `gro on` ou XDP_PASS no veth do contêiner (§5.2) |
| **G5** | Fim da semana 4 | O proxy PFCP replica uma sessão em 2 UPFs (Nível B, §5.4) | Ficar no Nível A, que já cobre todos os objetivos |

Cada gate aprovado vira uma tag no repositório (`g0-ambiente`, `g1-xdp-nativo`, ...), conforme `docs/ORGANIZACAO_PROJETO.md`.

---

## 3. Fase 0: preparar o ambiente (até 25/10)

### 3.1 Pedidos ao orientador (configuração no host KVM)
1. **VM2** clonada da VM do PFC 1: Ubuntu 22.04.5, kernel 5.15.0-176, 8 vCPU, 32 GiB, VT-x. Trocar hostname, machine-id e chaves SSH do clone.
2. **Uma segunda placa de rede em cada VM**, modelo **virtio**, ligada a uma **rede dedicada só entre as duas VMs** (bridge Linux do host ou rede isolada do libvirt), sem NAT e sem outro tráfego. A primeira placa continua sendo a de gerência e Internet.
3. Nessa placa, **multiqueue com `queues=16` (2 × 8 vCPU)** e backend **vhost**:
   ```xml
   <interface type='bridge'>
     <source bridge='br-pfc'/>
     <model type='virtio'/>
     <driver name='vhost' queues='16'>
       <guest csum='off' tso4='off' tso6='off' ecn='off' ufo='off'/>
     </driver>
   </interface>
   ```
   O bloco `<guest .../>` só é necessário se o attach nativo reclamar de GRO/CSUM (§4.3). Como mexer no XML exige acesso ao host, vale pedir já configurado ou pedir acesso ao `virsh edit`.
4. **MTU 1500** na rede dedicada.
5. Se possível: **fixar as vCPUs das duas VMs em núcleos físicos distintos** (`virsh vcpupin`) e evitar outras cargas no host durante as rodadas noturnas.

### 3.2 Preparar as duas VMs
```bash
# nas duas VMs
sudo apt update && sudo apt install -y git tmux chrony sysstat iperf3 tcpdump ethtool \
     clang llvm libbpf-dev linux-tools-$(uname -r) bpfcc-tools python3-bpfcc xdp-tools
sudo systemctl disable --now irqbalance
sudo systemctl enable --now chrony          # relógios alinhados para os experimentos de failover
git clone https://github.com/anabiaboorges/Projeto_Final_de_Curso_2.git ~/Projeto_Final_de_Curso_2
```
- **Credencial do GitHub nas VMs**: um *personal access token* (fine-grained, só este repositório) ou uma chave SSH por VM cadastrada como *deploy key* com escrita. Nunca versionar o token.
- **Docker**: já instalado na VM do PFC 1 (e, portanto, no clone). Conferir `docker compose version`.
- **BCC**: o PFC 1 precisou compilar o BCC 0.36.1 em `/opt/bcc` porque o 0.18 do apt é incompatível com o kernel 5.15.0-176. O clone já deve trazer essa instalação; conferir com `ls /opt/bcc`.
- **Artefatos do PFC 1**: trazer para o repositório, numa branch `feature/importa-programas-pfc1`, o `docker-compose.yml`, os programas XDP/TC, os scripts de `mpstat` e os de análise estatística que estão na VM1.

### 3.3 Verificar a placa dedicada (G0)
```bash
ip -br link                       # enp2s0 presente
ethtool -i enp2s0                 # driver: virtio_net
ethtool -l enp2s0                 # Combined máximo = 16, em uso = 8
ping -c 100 -i 0.01 192.168.100.11    # da VM1 para a VM2
```
O valor `queues = 2 × vCPU` é proposital. O `virtio_net` só cria filas TX dedicadas a XDP quando há uma fila livre por vCPU além das filas em uso. Com menos filas, dependendo da versão do kernel, o attach nativo é recusado ("*Too few free TX rings available*") ou cai num caminho com *lock*, que funciona mas distorce a medição. Se o host não aceitar 16 filas, a alternativa é reduzir as filas em uso dentro da VM2 (`ethtool -L enp2s0 combined 4`, com `queues=12` no host) e documentar.

---

## 4. Frente 1: XDP nativo (semanas 1–3 após G0)

### 4.1 Validação do attach nativo, gate G1 (VM2)
```bash
sudo xdp-loader load -m native enp2s0 xdp_drop_count.o   # ou ip link set dev enp2s0 xdpdrv obj ...
ip -d link show enp2s0 | grep -o 'xdp[a-z]*'   # → xdpdrv (genérico = xdpgeneric)
sudo bpftool net show dev enp2s0     # → "xdp:" (drv) vs "xdpgeneric:"
```
Observação: o roadmap usa `bpftool prog list` para conferir o modo, mas esse comando não mostra o modo de attach. Use `bpftool net` ou `ip -d link`.

### 4.2 Medidas de rede VM1 ↔ VM2, gate G2
- `ping -c 1000 -i 0.01 192.168.100.11`: anotar min/avg/max/mdev.
- `iperf3 -u -b {100,200,400,800}M -l 700 -t 30` (pacotes de 700 B, como no PFC 1), da VM1 para a VM2.
- A taxa de trabalho é a **maior com perda < 1% e CV < 10%**. Se a taxa de 100 Mbps do PFC 1 estiver dentro desse limite, usá-la também, para manter a comparação.

### 4.3 Se o attach nativo falhar
- Mensagem "*Can't set XDP while host is implementing GRO_HW/CSUM*": `ethtool -K enp2s0 gro off lro off` na VM2. Se não bastar, o bloco `<guest .../>` do §3.1 no XML da VM2.
- Mensagem "*Too few free TX rings available*": rever as filas (§3.3).
- MTU acima de ~3,5 kB bloqueia XDP no virtio. Mantenha 1500.
- Se ainda assim falhar, siga para o Plano C.

### 4.4 Distribuir o 5GC entre as VMs, gate G3
- **VM1**: o compose do PFC 1 **sem** UPF e sem DN. O gNB e o UE ficam em `network_mode: host`, com `gtpIp`/`linkIp` = 192.168.100.10.
- **VM2**: `upf1` em `network_mode: host` para a Frente 1, com GTP-U em 192.168.100.11:2152 e PFCP em 192.168.100.11:8805. O `dn` fica na rede `n6`. Roteamento sem NAT se possível; o DN só precisa de `10.45.0.0/16 via <upf N6>`.
- **SMF (VM1)**: o cliente PFCP aponta para 192.168.100.11.
- Teste: `docker exec ue ping -I uesimtun0 <ip do dn>` e um `iperf3` pelo túnel.

### 4.5 Adaptar os programas do PFC 1
- **XDP (BCC)**: o mesmo código. Só muda o flag `b.attach_xdp("enp2s0", fn, flags=BPF.XDP_FLAGS_DRV_MODE)`, que vale `1<<2` (o genérico é `XDP_FLAGS_SKB_MODE`, `1<<1`). O *parsing* já usa `ctx->data` e `data_end`. Confira os *bounds checks* do cabeçalho GTP-U (8 B mais as extensões, se `E/S/PN != 0`).
- **TC**: reaproveitado sem alterações, com `clsact` *ingress* em `enp2s0`.
- No PFC 1 os programas rodavam dentro do contêiner da UPF. Aqui o hook fica na `enp2s0` da VM2, então os programas rodam **no host da VM2** (fora do contêiner), o que também evita o problema de GLIBC/BCC na imagem do Open5GS.
- **Teste de sanidade**: rode a mesma rajada `iperf3` (N pacotes) nos três modos e compare os contadores do mapa com o `tcpdump -c`. As contagens devem bater.

### 4.6 Experimentos da Frente 1 (`scripts/run_xdp_native.sh`)
Os **cenários** são todos medidos no mesmo ponto, `enp2s0` da VM2: **baseline**, **XDP nativo**, **XDP genérico** e **TC**.

**5º cenário (recomendado, para validar o ambiente)**: subir na VM1, sozinha, o compose **original** do PFC 1 (tudo na mesma VM, XDP genérico no veth da UPF) e repetir as 30 iterações. Os números devem ficar próximos dos do PFC 1 (CPU total média ≈ 14,5% no XDP genérico). Isso confirma que o ambiente reproduz o PFC 1 antes de comparar com o nativo.

O protocolo é o do PFC 1, com cada iteração seguindo estes passos:
1. Reiniciar `upf1` e `dn` e reestabelecer a sessão PDU (`nr-cli ... ps-establish`).
2. Anexar o programa do cenário e verificar o modo (`ip -d link`). Se o modo estiver errado, abortar a iteração.
3. `mpstat -P ALL 1 25` na VM2, em paralelo com `iperf3 -u -b <taxa G2> -l 700 -t 20` (20 s de medição e 5 s de *ramp-down*).
4. Salvar `results/xdpnat/<cenário>/<run>/{mpstat.txt,iperf.json,attach.txt,meta.txt}`.

O total é de **30 iterações × 4 cenários = 120 rodadas**, em **ordem aleatória em blocos** (30 blocos com os 4 cenários embaralhados), para que variações de carga do host não se confundam com o cenário. São ~3 min por rodada, ou **~6 h**: uma noite, disparada por `tmux` na VM1.

**Microbenchmark complementar** (recomendado, ~1 noite). O gNB do UERANSIM roda em espaço de usuário e limita a taxa, o que pode esconder o ganho do modo nativo. Gere UDP de 64 B em pps fixo com `trafgen` ou `pktgen` na VM1 e compare `%soft` na VM2 com XDP_PASS+contagem e com XDP_DROP, nos modos nativo, genérico e TC. Esse teste isola o custo do *hook*, sem o 5GC.

### 4.7 Análise (`userspace/analyze_experiments.py`)
- Shapiro–Wilk por grupo e Levene.
- ANOVA de um fator com Tukey HSD. Se as variâncias forem desiguais, ANOVA de Welch com Games–Howell. Se não houver normalidade, Kruskal–Wallis com Dunn.
- Tamanho de efeito (η² e d de Cohen) para `%usr`, `%sys` e `%soft`.
- **Comparação com o PFC 1**: os recursos e o kernel são os mesmos, mas a topologia e o escopo do `mpstat` mudaram (antes: VM inteira com todos os componentes; agora: VM2). Compare o **padrão** (genérico ≈ TC? nativo < genérico?) e os efeitos relativos ao baseline; o 5º cenário é a ponte para comparar valores absolutos.

---

## 5. Frente 2: balanceamento e failover (semanas 2–6 após G0)

### 5.1 O problema a resolver primeiro: estado de sessão
Uma UPF só encaminha um pacote GTP-U se tiver a sessão PFCP (PDR/FAR) para aquele TEID, criada pelo SMF. Redirecionar um TEID para outra UPF que não tenha a sessão faz o pacote ser descartado. Por isso o trabalho tem **dois níveis**:

| | **Nível A (garantido)** | **Nível B (meta, gate G5)** |
|---|---|---|
| Quem tem a sessão | Só a UPF que o SMF escolheu (multi-UPF do Open5GS) | **Todas** as UPFs, via proxy PFCP de replicação |
| O que o XDP faz | Direciona TEID → UPF, que é aprendido do PFCP | **Escolhe** a UPF (RR ou hash) e mantém afinidade |
| Failover | Detecção + desvio no XDP + **reestabelecimento** da sessão PDU | Desvio no XDP **sem perder a sessão** (*hot standby*) |
| Comparação RR × hash | Em bancada sintética de GTP-U (§5.5) | Também no 5G real |

### 5.2 Rede das UPFs na VM2
- `upf1..3` ficam na rede Docker `upfnet` (172.30.0.11–13), com `cpuset` conforme §1.1. Cada uma anuncia o **VIP N3 192.168.100.100** como endereço GTP-U (`upf.yaml`, `gtpu.server.address`). O VIP também é configurado no `eth0` de cada contêiner.
- A VM2 responde ARP pelo VIP em `enp2s0`, com `ip addr add 192.168.100.100/32 dev enp2s0` e `ip_forward=1`.
- O XDP em `enp2s0` (nativo) faz `bpf_redirect_map(&upf_devmap, idx, 0)` para o veth do lado do host da UPF escolhida. Antes, reescreve o MAC de destino para o MAC do contêiner (lido do mapa `upf_mac`).
- **Requisito do veth no kernel 5.15**: o lado receptor (dentro do contêiner) precisa ter NAPI ativo. Use `nsenter -t <pid> -n ethtool -K eth0 gro on` ou anexe um XDP_PASS no `eth0` do contêiner. Sem isso, o redirect é descartado sem aviso, e o gate G4 detecta o problema.
- **N4**: na VM1, `ip route add 172.30.0.0/24 via 192.168.100.11`. O SMF fala PFCP com cada UPF no IP de contêiner.

### 5.3 Programa XDP do balanceador (`ebpf_src/ebpf_balancer_xdp.c`, libbpf/CO-RE)
Os mapas:
- `upf_devmap` (DEVMAP, 4): índice → ifindex do veth
- `upf_mac` (ARRAY): índice → MAC
- `upf_state` (ARRAY): `{alive, last_hb_ns}`
- `lb_table` (ARRAY, M=257): tabela tipo **Maglev**, preenchida em espaço de usuário só com as UPFs vivas
- `sess_map` (LRU_HASH): TEID → índice (afinidade)
- `cfg` (ARRAY): `{alg: 0=nenhum,1=RR,2=hash, hb_timeout_ns}`
- `rr_ctr` (PERCPU_ARRAY)
- `stats` (PERCPU_ARRAY)

O fluxo por pacote:
1. Fazer o *parse* de Eth/IPv4/UDP:2152/GTPv1 e ler o TEID. Se o pacote não for para o VIP, retornar XDP_PASS.
2. Procurar o TEID em `sess_map`. Se a UPF associada estiver viva (`now - last_hb_ns < hb_timeout`, o que dá **detecção dentro do próprio XDP**, sem depender do daemon), usá-la.
3. Se o TEID não estiver no mapa ou a UPF estiver morta, escolher pelo algoritmo (RR por CPU ou `lb_table[jhash(TEID) % M]`) e gravar a escolha em `sess_map`.
4. Reescrever os MACs e fazer o redirect via devmap. No Nível B com TEIDs diferentes por UPF, também reescrever o TEID com o mapa `teid_xlate`, com ajuste incremental do checksum UDP (RFC 1624) ou checksum 0 se o gNB enviar 0.

No **Nível A**, `sess_map` é preenchido pelo agente que escuta o PFCP (Session Establishment Response → F-TEID → UPF).

O **modo nativo** é usado em `enp2s0`. Para o cenário "balanceador em modo genérico", basta trocar o flag. Compilar na VM2 (BTF do kernel 5.15 em `/sys/kernel/btf/vmlinux`).

### 5.4 Componentes em espaço de usuário (`userspace/`)
- **`healthd.py`** (VM2, vCPU 7) envia um **GTP-U Echo Request** (que a UPF Open5GS responde nativamente) para cada UPF a cada 20 ms. A cada resposta, escreve `last_hb_ns` (em `CLOCK_MONOTONIC`, o mesmo do `bpf_ktime_get_ns`). Depois de k=3 falhas, marca `alive=0`, recalcula `lb_table` e atualiza a rota N6 do DN (`ip route replace 10.45.0.0/16 via <upf viva>`). Cada evento vai com *timestamp* para `events.csv`.
- **`pfcp_agent.py`** (Nível A, VM2) usa o `scapy.contrib.pfcp` para escutar a troca na `upfnet` e preencher `sess_map`.
- **`pfcp_mirror.py`** (Nível B, *timebox* de 5 dias, VM1 na vCPU 6) é um proxy entre o SMF e as UPFs:
  - Faz *fan-out* de Association, Session Establishment, Session Modification e Session Deletion para as 3 UPFs.
  - Devolve ao SMF a resposta da primária e guarda os SEIDs de cada UPF.
  - **Primeira opção a testar**: reescrever F-TEID `CHOOSE` para um TEID explícito alocado pelo proxy, para que as UPFs fiquem com o mesmo TEID. Verificar se a UPF do Open5GS aceita F-TEID explícito quando anuncia FTUP.
  - **Alternativa**: mapa `teid_xlate`.
  - **Verificar na Fase 0**: o que `selected_upf_node()` em `src/smf/context.c` do Open5GS 2.7 faz com várias UPFs que casam com o mesmo DNN (round-robin?).

### 5.5 Bancada sintética, que independe do Open5GS e sempre é viável
- Gerar com scapy pcaps de GTP-U com 1.000 TEIDs e tráfego interno variado.
- *Replay* com `tcpreplay`/`trafgen` da VM1 para o VIP na VM2.
- As "UPFs" são 3 contêineres com receptores que contam pacotes por TEID.
- Mede: **distribuição** (índice de Jain, e entropia de Shannon como métrica complementar), **afinidade** (TEIDs que mudaram de UPF), **overhead de CPU** e o **efeito de uma falha no remapeamento** (o hash tipo Maglev move ~1/N dos TEIDs, enquanto o módulo simples move quase todos).

### 5.6 Experimentos da Frente 2
| Experimento | Cenários | Repetições | Métricas |
|---|---|---|---|
| Overhead do balanceador (5G real, `nr-ue -n 10`) | sem LB · LB-RR · LB-hash (nativo) e LB-hash genérico | 30 cada | `%usr`, `%sys` e `%soft` (VM2), vazão e perda |
| Distribuição e afinidade (sintética) | RR · hash, com 3 UPFs | 30 cada | Jain, % de remapeamento, pps |
| Failover (detecção) | `docker kill upf1` em t aleatório; timeout do heartbeat ∈ {60, 150, 300 ms} | 30 por timeout | t_detecção = evento no `healthd` − t_kill |
| Failover (continuidade) | Nível A (reestabelecer) × Nível B (hot standby) | 30 cada | **t_interrupção** e perda durante a falha |

Para a **continuidade**, use um gerador UDP próprio com número de sequência, a 1.000 pps, do UE (`uesimtun0`, VM1) até o DN (VM2). A interrupção é o maior *gap* de sequência × 1 ms; como emissor e receptor estão em VMs diferentes, o `chrony` mantém os registros comparáveis. Rode o `iperf3` em paralelo para vazão e jitter. O total é de ~360 rodadas, a maioria de 30–60 s: 2 noites.

---

## 6. Cronograma (15/10 → 13/12)

| Semana | Datas | Frente 1: XDP nativo | Frente 2: LB e failover | Gate |
|---|---|---|---|---|
| S0 | 15–25/out | Pedidos ao orientador (§3.1), preparação das VMs (§3.2), importação dos artefatos do PFC 1 | Ler código SMF/UPF (seleção, FTUP) | **G0** |
| S1 | 26/out–01/nov | Attach nativo, ping/iperf entre as VMs, 5GC dividido, porte dos programas | 3 UPFs na VM2, redirect para veth | **G1–G4** |
| S2 | 02–08/nov | Piloto de 5 runs, **120 runs (noite)**, 5º cenário (noite) e microbench (noite) | Balanceador XDP + `healthd` + bancada sintética | — |
| S3 | 09–15/nov | Análise estatística da Frente 1 | Nível A completo; protótipo do `pfcp_mirror` (5 dias) | **G5** |
| S4 | 16–22/nov | Metodologia e Resultados da Frente 1 | Nível B (se G5 passou), experimentos de overhead e sintéticos | — |
| S5 | 23–29/nov | — | Experimentos de failover e análise | — |
| S6 | 30/nov–06/dez | Relatório final, figuras | Relatório final, figuras | — |
| S7 | 07–13/dez | Slides, ensaio, folga | | — |

O G0 ficou mais longo (até 25/10) porque depende da configuração do host. Enquanto isso, o código pode ser escrito e testado em qualquer máquina (Plano B); só as rodadas oficiais exigem as VMs.

**Divisão sugerida** (as VMs são acessadas remotamente, então as duas podem trabalhar ao mesmo tempo, cada uma na sua branch):
- **Ana**: Fase 0, Frente 1 (§4) e o programa XDP do balanceador (§5.3).
- **Bianca**: 5GC dividido (§4.4), 3 UPFs (§5.2), `healthd`, agente e proxy PFCP (§5.4), experimentos de failover.
- **As duas**: análise estatística e relatório.
- **Combinar quem opera as VMs em cada noite de rodadas**: duas campanhas ao mesmo tempo nas mesmas VMs invalidam as medições.

---

## 7. Ameaças à validade e mitigações

| Risco | Mitigação |
|---|---|
| Host compartilhado com outras VMs ou superalocado | Perguntar ao orientador (§0.3); rodadas à noite; ordem **aleatória em blocos**; piloto para medir o CV; registrar `steal` (`%steal` do `mpstat`) e descartar rodadas com steal alto |
| Atraso na entrega da VM2 ou da rede dedicada | Prazo do G0 em 25/10; desenvolvimento no Plano B; rodadas oficiais só nas VMs |
| Sem acesso ao XML das VMs | Pedir a configuração completa de uma vez (§3.1); ter a alternativa de reduzir filas dentro da VM (§3.3) |
| Interferência entre VMs na mesma CPU física | `virsh vcpupin` no host, se possível; *pinning* interno (§1.1); IRQs do `enp2s0` fixadas |
| O gNB do UERANSIM limita a vazão | Taxa definida no G2 e microbench com `trafgen` |
| Mudança de escopo do `mpstat` em relação ao PFC 1 | 5º cenário (§4.6) como ponte; comparação pelo padrão e pelos efeitos relativos |
| Relógios das VMs desalinhados | `chrony` nas duas; medidas de interrupção baseadas em sequência, não em relógio |
| Duas pessoas rodando experimentos ao mesmo tempo | Combinar a agenda das noites; `meta.txt` com quem rodou e o commit |
| O Nível B (proxy PFCP) não fica pronto | Timebox e gate G5; o Nível A com a bancada sintética já cumpre os objetivos do plano |

---

## 8. Estrutura do repositório

- **Remoto**: [github.com/anabiaboorges/Projeto_Final_de_Curso_2](https://github.com/anabiaboorges/Projeto_Final_de_Curso_2)
- **Clones de laboratório**: `~/Projeto_Final_de_Curso_2` na VM1 e na VM2. Os resultados são gravados na VM que coleta o `mpstat` (VM2) e enviados numa branch `feature/exp-...`.
- **Clone de escrita**: em qualquer máquina pessoal, para documentação e relatório.
- **Organização e Git flow**: `docs/ORGANIZACAO_PROJETO.md`. Toda mudança nasce numa branch `feature/...` ou `bugfix/...` e é mergeada na `main` quando concluída.

```
Projeto_Final_de_Curso_2/
├── scripts/  vm_setup.sh  run_xdp_native.sh  run_microbench.sh
│             run_balancing.sh  run_failover.sh
├── ebpf_src/ xdp_monitor.c (PFC1)  tc_monitor.c (PFC1)  xdp_drop_count.c
│             ebpf_balancer_xdp.c  Makefile
├── userspace/ healthd.py  pfcp_agent.py  pfcp_mirror.py  seq_sender.py  seq_receiver.py
│              gen_gtpu_pcap.py  analyze_experiments.py
├── docker/   vm1-core/docker-compose.yml   vm2-upf/docker-compose.{monitor,multi-upf}.yml
├── results/  xdpnat/  pfc1-original/  microbench/  lb/  failover/
└── docs/     ORGANIZACAO_PROJETO.md, planos, referencias/
```

## 9. Próximos passos imediatos
1. **Enviar ao orientador os pedidos do §3.1** (VM2, placa virtio dedicada com vhost e `queues=16`, MTU 1500) e as perguntas do §0.3.
2. Na VM1, localizar os artefatos do PFC 1 e importá-los para o repositório numa `feature/importa-programas-pfc1`.
3. Quando a VM2 chegar: preparação (§3.2), verificação da placa (§3.3) e **G0**.
4. Validar **`xdpdrv`** em `enp2s0` da VM2 (**G1**). É isso que confirma a viabilidade da Frente 1.
5. Rodar o piloto do G2 e fixar a taxa de trabalho.
