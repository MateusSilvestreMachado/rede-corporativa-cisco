# 🏢 Enterprise Network Lab — Alta Disponibilidade Multisite com HSRP, OSPF e Serviços Centralizados

Simulação de uma infraestrutura de rede corporativa em dois prédios, projetada no **Cisco Packet Tracer**. O laboratório aplica, num cenário único, os principais pilares de uma rede corporativa moderna: segmentação por VLAN, roteamento inter-VLAN em Camada 3, redundância de gateway com HSRP, roteamento dinâmico com OSPF, relay de DHCP e serviços de rede centralizados (DHCP, DNS e Web).

![Topologia da Rede](https://github.com/user-attachments/assets/77c0111d-0251-41e1-a831-ab102650bf7b)

---

## 📑 Sumário

- [Objetivo do laboratório](#-objetivo-do-laboratório)
- [Visão geral da topologia](#-visão-geral-da-topologia)
- [Mapeamento de endereçamento e VLANs](#-mapeamento-de-endereçamento-e-vlans)
- [Decisões de design](#-decisões-de-design)
- [Estrutura do repositório](#-estrutura-do-repositório)
- [Como abrir e testar o projeto](#-como-abrir-e-testar-o-projeto)
- [Roteiro de validação](#-roteiro-de-validação)
- [Comandos úteis de troubleshooting](#-comandos-úteis-de-troubleshooting)
- [Melhorias futuras](#-melhorias-futuras)
- [Autor](#-autor)
- [Licença](#-licença)

---

## 🎯 Objetivo do laboratório

Simular a rede de uma empresa com matriz (Prédio 1) e filial (Prédio 2), cada uma com três departamentos (RH, Vendas e Financeiro) isolados logicamente, garantindo:

- **Continuidade de acesso à rede** mesmo se um switch de core cair (HSRP).
- **Convergência automática de rotas** entre os segmentos, sem configuração estática manual (OSPF).
- **Atribuição automática de IP** para os hosts de qualquer departamento, mesmo com o servidor DHCP fisicamente localizado em outra VLAN (DHCP Relay).
- **Serviços de rede centralizados** (DNS interno e portal Web) acessíveis por toda a empresa.

## 📐 Visão geral da topologia

```
                              ┌─────────────────────┐
                              │   Link Serial WAN    │
                              │    10.10.10.0/30     │
                              └───────────┬───────────┘
                    ┌─────────────────────┴─────────────────────┐
                    │                                             │
            ┌───────▼────────┐                           ┌───────▼────────┐
            │  Router Borda   │                           │  Router Borda   │
            │   Prédio 1      │                           │   Prédio 2      │
            └───────┬────────┘                           └───────┬────────┘
                    │ 10.10.1.0/30  10.10.1.4/30                 │ 10.10.2.0/30  10.10.2.4/30
        ┌───────────┴───────────┐                     ┌───────────┴───────────┐
        │                       │                     │                       │
┌───────▼───────┐      ┌────────▼──────┐     ┌────────▼──────┐      ┌────────▼──────┐
│ SW Multilayer 0│◄────►│SW Multilayer 1│     │SW Multilayer 2│◄────►│SW Multilayer 3│
│  (HSRP ativo)  │ OSPF │(HSRP standby) │     │ (HSRP ativo)  │ OSPF │(HSRP standby) │
└───────┬───────┘      └────────┬──────┘     └────────┬──────┘      └────────┬──────┘
        │  trunk (VLANs 10/20/30/100)                  │  trunk (VLANs 40/50/60/100)
   ┌────┴────┬────────┬─────────┐                 ┌────┴────┬────────┬─────────┐
   │         │        │         │                 │         │        │         │
┌──▼──┐  ┌───▼─┐  ┌───▼─┐   ┌───▼────┐         ┌──▼──┐  ┌───▼─┐  ┌───▼─┐        │
│SW 0 │  │SW 1 │  │SW 2 │   │Servidor│         │SW 3 │  │SW 4 │  │SW 5 │        │
│ HR  │  │SALES│  │FIN. │   │Central │         │ HR  │  │SALES│  │FIN. │        │
│VLAN10│ │VLAN20│ │VLAN30│  │VLAN 100│         │VLAN40│ │VLAN50│ │VLAN60│       │
└─────┘  └─────┘  └─────┘   └────────┘         └─────┘  └─────┘  └─────┘
```

- **Prédio 1 (Matriz/HQ):** VLANs de departamento (10/20/30) + VLAN 100, onde fica o servidor central de DHCP/DNS/Web.
- **Prédio 2 (Filial):** estrutura espelhada, com sub-redes independentes (VLANs 40/50/60) para os mesmos três departamentos.
- **Core L3 redundante:** em cada prédio, dois switches multilayer compartilham o papel de gateway via HSRP — se o ativo falhar, o standby assume de forma transparente para os hosts.
- **Interligação:** os dois prédios se conectam via link serial ponto a ponto entre roteadores de borda, com OSPF garantindo a troca de rotas entre todos os segmentos.

## 🗺️ Mapeamento de endereçamento e VLANs

| Local | VLAN | Nome | Sub-rede | Gateway Virtual (HSRP) |
|---|---|---|---|---|
| Prédio 1 | 10 | HR | 192.168.10.0/24 | 192.168.10.1 |
| Prédio 1 | 20 | SALES | 192.168.20.0/24 | 192.168.20.1 |
| Prédio 1 | 30 | FINANCE | 192.168.30.0/24 | 192.168.30.1 |
| Prédio 1 | 100 | SERVERS | 192.168.100.0/24 | 192.168.100.1 |
| Prédio 2 | 40 | HR | 192.168.40.0/24 | 192.168.40.1 |
| Prédio 2 | 50 | SALES | 192.168.50.0/24 | 192.168.50.1 |
| Prédio 2 | 60 | FINANCE | 192.168.60.0/24 | 192.168.60.1 |
| Prédio 2 | 100 | SERVERS (local) | 192.168.110.0/24 | 192.168.110.1 |

**Links ponto a ponto (roteamento entre core switches e roteadores de borda):**

| Enlace | Sub-rede |
|---|---|
| SW Multilayer 0/1 ↔ Router Prédio 1 | 10.10.1.0/30 e 10.10.1.4/30 |
| SW Multilayer 2/3 ↔ Router Prédio 2 | 10.10.2.0/30 e 10.10.2.4/30 |
| WAN Serial (Router Prédio 1 ↔ Router Prédio 2) | 10.10.10.0/30 |

> ⚠️ **Nota de projeto:** a VLAN 100 existe nos dois prédios, mas em sub-redes diferentes (`192.168.100.0/24` no Prédio 1 e `192.168.110.0/24` no Prédio 2). O servidor físico documentado está apenas no Prédio 1 — a VLAN 100 do Prédio 2 está preparada como segmento reservado para servidores locais da filial, mas ainda não tem host configurado nela.

## 🧠 Decisões de design

**Por que switches de Camada 3 (multilayer) fazendo o roteamento inter-VLAN, em vez de um router-on-a-stick?**
Com o volume de tráfego entre departamentos dentro de cada prédio, rotear diretamente no switch de core evita gargalo em um único link/porta de roteador e aproveita o hardware ASIC de switching, que é mais rápido que roteamento via software.

**Por que HSRP em vez de um único gateway?**
Cada prédio tem dois switches multilayer atuando como par redundante. A prioridade mais alta (`210`) marca o switch ativo; o outro (`200` ou `100/110`) assume automaticamente com `preempt` caso o ativo falhe — os hosts nunca percebem a troca, pois o IP virtual do gateway não muda.

**Por que OSPF em vez de rotas estáticas?**
Com múltiplos links redundantes entre os switches de core e os roteadores de borda, rotas estáticas exigiriam reconfiguração manual a cada mudança de topologia. O OSPF converge automaticamente e recalcula o melhor caminho em caso de falha de um enlace.

**Por que DHCP Relay (`ip helper-address`) em vez de um DHCP server em cada VLAN?**
Centralizar o DHCP em um único servidor (na VLAN 100) simplifica a administração de pools e reduz a superfície de manutenção. Como o DHCP usa broadcast (que não atravessa VLANs), cada SVI de departamento precisa retransmitir (`relay`) as requisições em unicast até o servidor central.

**Por que `passive-interface` nas VLANs de host?**
Evita que o switch envie pacotes de hello do OSPF para as portas de acesso dos usuários finais — essas redes só precisam ser *anunciadas*, nunca formar vizinhança OSPF com um host comum.

## 📂 Estrutura do repositório

```
enterprise-network-lab/
├── projeto/
│   └── enterprise-network-lab.pkt      # Arquivo do Cisco Packet Tracer (abra este)
├── configs/
│   ├── predio1/
│   │   ├── switch_multilayer_0.txt      # Core L3 ativo (HR/SALES/FIN + SERVERS)
│   │   ├── switch_multilayer_1.txt      # Core L3 standby
│   │   ├── switches_acesso_l2.txt       # Switches 0, 1, 2 (acesso por departamento)
│   │   └── router_borda.txt             # Roteador de borda (WAN)
│   ├── predio2/
│   │   ├── switch_multilayer_2.txt      # Core L3 ativo
│   │   ├── switch_multilayer_3.txt      # Core L3 standby
│   │   ├── switches_acesso_l2.txt       # Switches 3, 4, 5 (acesso por departamento)
│   │   └── router_borda.txt             # Roteador de borda (WAN)
│   └── servidor/
│       └── dhcp_dns_web.txt             # Pools DHCP, zona DNS e página HTTP
├── docs/
│   ├── 01_dhcp_dns_web_server.pdf       # Anotações originais: serviços de rede
│   └── 02_rede_corporativa_vlans_ospf_hsrp.pdf  # Anotações originais: VLANs/OSPF/HSRP
└── README.md
```

Os arquivos em `configs/` reorganizam, por dispositivo, as mesmas configurações documentadas nos PDFs em `docs/` — a ideia é que quem quiser reproduzir o laboratório não precise garimpar comandos espalhados em texto corrido, mas copiar/colar a configuração de um equipamento por vez, direto no Packet Tracer.

## ▶️ Como abrir e testar o projeto

**Pré-requisitos:** [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (gratuito mediante cadastro na Cisco Networking Academy).

1. Abra `projeto/enterprise-network-lab.pkt`.
2. Aguarde alguns segundos para os protocolos convergirem (OSPF formando vizinhança, HSRP elegendo o ativo) — os links passam de laranja para verde no diagrama.
3. Clique em qualquer PC de um departamento e confirme, na aba Desktop → IP Configuration, que ele recebeu IP via DHCP automaticamente.

## ✅ Roteiro de validação

1. **DHCP:** em um PC cliente, abra o Prompt de Comando e rode `ipconfig /renew` — confirme que o IP recebido está na faixa correta do departamento e que o gateway/DNS batem com a tabela de endereçamento acima.
2. **Conectividade fim a fim:** rode um `ping` de um host do Prédio 1 para um host do Prédio 2 (ex: um PC do RH da matriz pingando um PC de Vendas da filial).
3. **DNS + Web:** no navegador de qualquer host, acesse `http://compania.local` e confirme que a página do portal carrega.
4. **Failover do HSRP:** desligue (`shutdown`) a interface VLAN ativa no switch multilayer primário de um prédio e confirme, com `ping` contínuo de um host daquele departamento, que a perda de pacotes é mínima/nula durante a troca para o switch standby.
5. **Convergência do OSPF:** derrube um dos links redundantes entre um switch multilayer e o roteador de borda e confirme, com `show ip route`, que o tráfego é automaticamente redirecionado pelo outro caminho.

## 🔧 Comandos úteis de troubleshooting

```
show ip interface brief      ! Status e IP de todas as interfaces
show standby brief            ! Estado do HSRP (Active/Standby) em cada VLAN
show ip ospf neighbor         ! Vizinhos OSPF formados e seu estado
show ip route                 ! Tabela de rotas (confirma rotas aprendidas via OSPF)
show vlan brief                ! VLANs criadas e portas associadas
show ip dhcp binding           ! Leases de IP concedidos pelo servidor DHCP
```

## 🚀 Melhorias futuras

- [ ] Adicionar um segundo servidor DNS/DHCP secundário para redundância de serviços (hoje o servidor central é ponto único de falha).
- [ ] Implementar ACLs entre VLANs para restringir tráfego cruzado entre departamentos por política de segurança.
- [ ] Configurar autenticação OSPF (MD5) entre os roteadores para evitar anúncios de rotas não autorizados.
- [ ] Adicionar VLAN de gerência dedicada e habilitar SSH nos switches/roteadores (hoje a gestão é via console).
- [ ] Popular o segmento de servidores do Prédio 2 (VLAN 100 / 192.168.110.0/24) com um servidor local, reduzindo a dependência do link WAN para serviços básicos da filial.

## 👤 Autor

**Mateus Silvestre Machado**
Projeto de laboratório em Redes de Computadores — Cisco Packet Tracer.

## 📄 Licença

Este projeto está sob a licença MIT — veja o arquivo [LICENSE](LICENSE).
