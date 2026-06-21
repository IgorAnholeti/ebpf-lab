# Laboratório de Redes e Protocolos — eBPF/XDP com Containerlab

Disciplina: **Redes e Protocolos**
Aluno: Igor Anholeti
Curso Técnico em Redes de Computadores

Laboratório baseado no repositório original do professor: [DANIELVENTORIM/ebpf-lab](https://github.com/DANIELVENTORIM/ebpf-lab)

---

## Objetivo

Explorar conceitos fundamentais de redes em ambiente controlado (containers Docker), praticar troubleshooting, analisar tráfego em diferentes camadas (Ethernet, IP, ICMP) e utilizar ferramentas de diagnóstico de rede (tcpdump, ping, iperf3, netcat, dig), incluindo uma etapa de filtragem de pacotes em nível de kernel com eBPF/XDP.

## Topologia

┌─────────────────────────────────────────┐

│               Máquina Host (WSL)         │

│                                          │

│  ┌──────────┐ eth1   eth1 ┌──────────┐  │

│  │  node-a  ├─────────────┤  node-b  │  │

│  │10.0.0.1  │             │10.0.0.2  │  │

│  └──────────┘             └──────────┘  │

│    (emissor)            (filtro XDP)    │

└─────────────────────────────────────────┘

| Nó | IP | Função |
|---|---|---|
| node-a | 10.0.0.1 | Emissor de pacotes (ping, iperf3 client) |
| node-b | 10.0.0.2 | Receptor / Filtro XDP |

## Ambiente utilizado

- WSL2 (Ubuntu) no Windows
- Docker Engine instalado nativamente dentro do WSL
- Containerlab v0.76.1

> **Nota técnica:** inicialmente o ambiente usava o Docker Desktop com integração WSL, mas o Containerlab apresentou falhas de `namespace path not available` e `iptables not found` devido à arquitetura do Docker Desktop (daemon roda em VM separada, sem expor namespaces de rede ao WSL). A solução foi instalar o Docker Engine nativamente dentro do Ubuntu/WSL e desativar a integração do Docker Desktop, o que resolveu o problema por completo.

## 1. Configuração Inicial

```bash
git clone https://github.com/DANIELVENTORIM/ebpf-lab.git
cd ebpf-lab
sudo containerlab deploy -t lab-ebpf.clab.yml --reconfigure
docker ps --filter "label=containerlab=ebpf-lab"
```

Verificação de IPs e MACs:
```bash
docker exec clab-ebpf-lab-node-a ip addr show eth1
docker exec clab-ebpf-lab-node-a ip link show eth1
```

Teste de conectividade:
```bash
docker exec clab-ebpf-lab-node-a ping -c 3 10.0.0.2
```
**Resultado:** 0% de perda de pacotes.

## 2. Captura de Pacotes

Captura em tempo real:
```bash
docker exec -it clab-ebpf-lab-node-b tcpdump -i eth1 -n icmp
```

Captura salva em arquivo `.pcap` (disponível em [`capturas/captura.pcap`](capturas/captura.pcap)):
```bash
docker exec clab-ebpf-lab-node-b tcpdump -i eth1 -w /tmp/captura.pcap icmp
docker cp clab-ebpf-lab-node-b:/tmp/captura.pcap ./captura.pcap
```

Visualização realizada com Wireshark (Windows), a partir do arquivo copiado do container.

## 3. Análise de Tráfego e Protocolos

Comando utilizado para análise completa (camadas 2, 3 e ICMP em um único comando):
```bash
docker exec clab-ebpf-lab-node-b tcpdump -i eth1 -e -n -vv icmp -c 4
```

- **ICMP (Echo Request/Reply):** Type 8 = Echo Request, Type 0 = Echo Reply, Code sempre 0.
- **Camada 2 (Ethernet):** endereços MAC de origem e destino confirmados via `-e`.
- **Camada 3 (IP):** TTL e Checksum analisados via `-vv`.
- **ICMP Type/Code:** confirmados na mesma captura.

## 4. Testes Avançados

### Cenário A — iperf3 (largura de banda)
```bash
docker exec -it clab-ebpf-lab-node-b iperf3 -s
docker exec clab-ebpf-lab-node-a iperf3 -c 10.0.0.2 -t 10
```

### Cenário B — netcat (TCP/UDP)
```bash
# TCP
docker exec -it clab-ebpf-lab-node-b nc -l -p 5000
docker exec -it clab-ebpf-lab-node-a sh -c "echo 'teste tcp' | nc 10.0.0.2 5000"

# UDP
docker exec -it clab-ebpf-lab-node-b nc -u -l -p 5001
docker exec -it clab-ebpf-lab-node-a sh -c "echo 'teste udp' | nc -u 10.0.0.2 5001"
```

### Cenário C — DNS (dig/nslookup)
```bash
docker exec clab-ebpf-lab-node-a dig google.com
docker exec clab-ebpf-lab-node-a nslookup google.com
```

## 5. Filtragem com eBPF/XDP

Compilação do programa eBPF:
```bash
./compile.sh
```

Instalação do bpftool e carregamento do programa XDP no node-b:
```bash
sudo docker exec clab-ebpf-lab-node-b apk add bpftool
sudo docker exec clab-ebpf-lab-node-b rm -f /sys/fs/bpf/xdp_test
sudo docker exec clab-ebpf-lab-node-b bpftool prog load /xdp_drop.o /sys/fs/bpf/xdp_test type xdp
sudo docker exec clab-ebpf-lab-node-b ip link set dev eth1 xdpgeneric pinned /sys/fs/bpf/xdp_test
```

Teste com filtro ativo:
```bash
sudo docker exec clab-ebpf-lab-node-a ping -c 5 10.0.0.2
```
**Resultado:** 100% de perda de pacotes (ICMP bloqueado no kernel via XDP).

Leitura do contador de pacotes descartados:
```bash
sudo docker exec clab-ebpf-lab-node-b bpftool map dump name packet_count_ma
```
**Resultado:**
```json
[{ "key": 0, "value": 5 }]
```

Desativação do filtro:
```bash
sudo docker exec clab-ebpf-lab-node-b ip link set dev eth1 xdpgeneric off
sudo docker exec clab-ebpf-lab-node-a ping -c 3 10.0.0.2
```
**Resultado:** 0% de perda de pacotes (conectividade restaurada).

## 6. Capturas de Tela

As evidências visuais de cada etapa estão disponíveis na pasta [`capturas/`](capturas/).

## 7. Arquivos do Laboratório

Os arquivos originais utilizados na execução (fornecidos pelo professor) estão disponíveis em [`arquivos-lab/`](arquivos-lab/):
- `lab-ebpf.clab.yml` — definição da topologia
- `xdp_drop.c` — código-fonte do programa eBPF/XDP
- `compile.sh` — script de compilação

## 8. Conclusão

O laboratório permitiu reproduzir, em ambiente controlado via containers, os principais conceitos da disciplina de Redes e Protocolos: endereçamento em Camada 2 (MAC/ARP), controle em Camada 3 (IP/ICMP/TTL), protocolos de transporte (TCP vs UDP), medição de desempenho (iperf3), resolução de nomes (DNS) e diagnóstico de falhas de rede. A etapa de eBPF/XDP demonstrou, na prática, uma técnica moderna de filtragem de pacotes em velocidade de linha diretamente no kernel Linux.
