# Guia de Hardware Tainá - Lista de Compras e Montagem

> Documento escrito em 31/08/2025, tentativa de atualização constante.

## Índice
1. [Fase 1: Setup Básico (MVP)](#fase-1-setup-básico-mvp)
2. [Fase 2: Setup Intermediário](#fase-2-setup-intermediário)
3. [Fase 3: Setup Profissional](#fase-3-setup-profissional)
4. [Guias de Montagem](#guias-de-montagem)
5. [Dicas de Economia](#dicas-de-economia)
6. [Calculadora de Custos](#calculadora-de-custos)

---

## Fase 1: Setup Básico (MVP)
**Suporta:** 5-10 usuários | **Custo Total:** R$ 1.200 - R$ 1.800

### Opção A: Raspberry Pi 5 (Recomendado para Iniciantes)

#### Lista de Compras - Brasil

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **Raspberry Pi 5 8GB** | BCM2712, 2.4GHz quad-core | R$ 580 | [FilipeFlop](https://www.filipeflop.com/produto/raspberry-pi-5-8gb/) • [MercadoLivre](https://lista.mercadolivre.com.br/raspberry-pi-5-8gb) |
| **Fonte Oficial** | 27W USB-C PD | R$ 120 | [FilipeFlop](https://www.filipeflop.com/produto/fonte-raspberry-pi-5/) |
| **Case com Cooler** | Alumínio com dissipação passiva | R$ 85 | [FilipeFlop](https://www.filipeflop.com/produto/case-raspberry-pi-5/) |
| **NVMe HAT** | Suporte M.2 2280 | R$ 95 | [Pimoroni](https://shop.pimoroni.com/products/nvme-base) • [AliExpress](https://pt.aliexpress.com/w/wholesale-raspberry-pi-5-nvme.html) |
| **SSD NVMe 1TB** | Kingston NV2 ou similar | R$ 340 | [Kabum](https://www.kabum.com.br/produto/380744) • [Pichau](https://www.pichau.com.br/ssd-kingston-nv2-1tb) |
| **MicroSD 32GB** | Classe 10, A1 | R$ 45 | [Amazon](https://www.amazon.com.br/s?k=microsd+32gb+sandisk) |
| **Cabo Ethernet** | Cat6, 2m | R$ 15 | [Kabum](https://www.kabum.com.br/produto/31788) |
| **TOTAL** |  | **R$ 1.280** | |

#### Lista Internacional (Importação)

| Item | Preço (USD) | Link | Observações |
|------|-------------|------|-------------|
| Raspberry Pi 5 8GB | $80 | [Official](https://www.raspberrypi.com/products/raspberry-pi-5/) | Verificar disponibilidade |
| Argon ONE V3 Case | $45 | [Argon40](https://argon40.com/products/argon-one-v3-case) | Case premium com M.2 |
| Crucial P3 1TB NVMe | $55 | [Amazon US](https://www.amazon.com/dp/B0B25LQQPC) | Ótimo custo-benefício |

### Opção B: Mini PC Intel N100

#### Lista de Compras - Brasil

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **Beelink Mini S12** | N100, 8GB RAM, 256GB SSD | R$ 1.450 | [AliExpress](https://pt.aliexpress.com/item/1005005565018357.html) • [Shopee](https://shopee.com.br/product/893654040/22666790416) |
| **RAM Extra 8GB** | DDR4 3200MHz SODIMM | R$ 140 | [Kabum](https://www.kabum.com.br/produto/172365) |
| **SSD Extra 1TB** | M.2 2280 NVMe | R$ 340 | [Pichau](https://www.pichau.com.br/ssd-1tb-nvme) |
| **TOTAL** |  | **R$ 1.930** | |

**Alternativas N100:**
- **GMKtec NucBox G2**: R$ 1.399 ([AliExpress](https://pt.aliexpress.com/item/1005005903838445.html))
- **ACEMAGIC S1**: R$ 1.299 ([Amazon](https://www.amazon.com.br/dp/B0CP5QXQMR))
- **Trigkey Green G4**: R$ 1.549 ([AliExpress](https://pt.aliexpress.com/item/1005005688074901.html))

---

## Fase 2: Setup Intermediário
**Suporta:** 20-30 usuários | **Custo Total:** R$ 3.500 - R$ 5.000

### Opção A: NAS + Mini PC

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **Synology DS224+** | 2-bay NAS, 2GB RAM | R$ 2.850 | [Kabum](https://www.kabum.com.br/produto/471846) • [Amazon](https://www.amazon.com.br/dp/B0BW4RWNBR) |
| **2x HDD 4TB** | WD Red Plus ou Seagate IronWolf | R$ 1.100 | [Kabum](https://www.kabum.com.br/produto/173163) |
| **Mini PC N100** | Como descrito acima | R$ 1.450 | Ver links acima |
| **Switch Gigabit 8 portas** | TP-Link TL-SG108 | R$ 145 | [MercadoLivre](https://produto.mercadolivre.com.br/MLB-1911499909) |
| **TOTAL** |  | **R$ 5.545** | |

### Opção B: Build Custom NAS

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **Jonsbo N2 Case** | ITX, 5x 3.5" bays | R$ 650 | [AliExpress](https://pt.aliexpress.com/item/1005004632006557.html) |
| **ASRock J5040-ITX** | Quad-core, 4x SATA | R$ 890 | [Kabum](https://www.kabum.com.br/produto/114834) |
| **16GB RAM** | 2x8GB DDR4 SODIMM | R$ 280 | [Pichau](https://www.pichau.com.br/memoria-ram-16gb) |
| **PSU PicoPSU 150W** | + Adaptador 12V | R$ 220 | [MercadoLivre](https://produto.mercadolivre.com.br/MLB-3458277021) |
| **4x HDD 4TB** | WD Red Plus | R$ 2.200 | [Kabum](https://www.kabum.com.br/produto/173163) |
| **SSD Boot 240GB** | Kingston A400 | R$ 140 | [Amazon](https://www.amazon.com.br/dp/B01N6JQS8C) |
| **TOTAL** |  | **R$ 4.380** | |

---

## Fase 3: Setup Profissional
**Suporta:** 50+ usuários | **Custo Total:** R$ 8.000 - R$ 15.000

### Build Customizado High-End

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **CPU AMD Ryzen 7 5700G** | 8C/16T, GPU integrada | R$ 1.350 | [Kabum](https://www.kabum.com.br/produto/335849) • [Terabyte](https://www.terabyteshop.com.br/produto/18599) |
| **Placa-mãe B550M** | ASRock B550M Pro4 | R$ 720 | [Pichau](https://www.pichau.com.br/placa-mae-asrock-b550m-pro4) |
| **64GB RAM** | 2x32GB DDR4 3200MHz | R$ 1.180 | [Kabum](https://www.kabum.com.br/produto/338542) |
| **PSU 650W 80+ Gold** | Corsair RM650x | R$ 580 | [Kabum](https://www.kabum.com.br/produto/463991) |
| **Gabinete Fractal Node 804** | Cube, 8x 3.5" + 2x 2.5" | R$ 950 | [Kabum](https://www.kabum.com.br/produto/470414) |
| **SSD NVMe 2TB** | Samsung 980 PRO | R$ 1.100 | [Amazon](https://www.amazon.com.br/dp/B08RK2SR23) |
| **6x HDD 8TB** | WD Red Pro | R$ 7.800 | [Kabum](https://www.kabum.com.br/produto/459830) |
| **HBA Card** | LSI 9211-8i | R$ 450 | [MercadoLivre](https://produto.mercadolivre.com.br/MLB-3227480256) |
| **UPS 1500VA** | APC Back-UPS | R$ 1.250 | [Kabum](https://www.kabum.com.br/produto/86206) |
| **TOTAL** |  | **R$ 15.380** | |

### Adição de GPU para IA (Opcional)

| Item | Especificação | Preço (R$) | Onde Comprar |
|------|--------------|------------|--------------|
| **RTX 4070** | 12GB VRAM para LLM local | R$ 3.800 | [Kabum](https://www.kabum.com.br/produto/464052) |
| **RTX 4060 Ti 16GB** | Alternativa com mais VRAM | R$ 3.200 | [Pichau](https://www.pichau.com.br/placa-de-video-rtx-4060-ti) |
| **Tesla P40 24GB** | Usada, ótima para IA | R$ 2.500 | [MercadoLivre](https://lista.mercadolivre.com.br/tesla-p40) |

---

## Guias de Montagem

### Vídeos Tutoriais

#### Raspberry Pi 5
- [Jeff Geerling - Pi 5 Complete Setup](https://www.youtube.com/watch?v=nBa3KeZWwqY)
- [Network Chuck - Pi Home Server](https://www.youtube.com/watch?v=gyMpI8csWis)
- [Craft Computing - Pi NAS Build](https://www.youtube.com/watch?v=aKOg_1veqr4)

#### Mini PC
- [ServeTheHome - N100 Review](https://www.youtube.com/watch?v=_WSvM7ILFwA)
- [TechHut - Mini PC Server](https://www.youtube.com/watch?v=9apIK8fuWlo)

#### NAS Custom
- [Linus Tech Tips - DIY NAS](https://www.youtube.com/watch?v=FAy9N1vX1qE)
- [SpaceInvader One - Ultimate NAS](https://www.youtube.com/watch?v=miYSR3A0FfU)
- [Hardware Haven - Budget NAS](https://www.youtube.com/watch?v=RvnG-ywF6_s)

### Guias Reddit

- [r/selfhosted - Wiki](https://www.reddit.com/r/selfhosted/wiki/index/)
- [r/HomeServer - Beginner Guide](https://www.reddit.com/r/HomeServer/comments/15p1tt8/new_to_home_servers_start_here/)
- [r/DataHoarder - NAS Builds](https://www.reddit.com/r/DataHoarder/comments/15xv7i0/2024_nas_build_recommendations/)
- [r/HomeNetworking - Setup Guide](https://www.reddit.com/r/HomeNetworking/comments/16n0kqv/complete_home_network_setup_guide/)

### Blogs e Artigos

- [Perfect Media Server](https://perfectmediaserver.com/) - Guia completo
- [Self-Hosted Podcast](https://selfhosted.show/) - Dicas semanais
- [Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) - Lista de software
- [LinuxServer.io](https://www.linuxserver.io/) - Docker containers prontos

---

## Dicas de Economia

### Onde Comprar Barato

#### Nacional
1. **Promoções**: 
   - Black Friday (Novembro)
   - Prime Day Amazon (Julho)
   - Aniversário Kabum (Julho)
   - Semana do Consumidor (Março)

2. **Grupos de Ofertas**:
   - [Promobit](https://www.promobit.com.br/)
   - [Pelando](https://www.pelando.com.br/)
   - Telegram: @hardwarebrasil

3. **Usados Confiáveis**:
   - [OLX](https://www.olx.com.br/) - Sempre teste antes
   - [Facebook Marketplace](https://www.facebook.com/marketplace/)
   - Grupos Facebook: "Hardware Usado Brasil"

#### Importação
1. **AliExpress**: 
   - Frete grátis acima de $30
   - Cupons de novo usuário
   - 11.11 e outros eventos

2. **Amazon Internacional**:
   - Alguns produtos com frete para Brasil
   - Aceita boleto

3. **Impostos**:
   - Até $50: isento
   - $50-$500: 60% imposto + ICMS
   - Use calculadora: [Tributado.net](https://www.tributado.net/)

### Hardware Usado Recomendado

| Tipo | Modelo | Preço Usado | Onde Encontrar |
|------|--------|-------------|----------------|
| Mini PC | Dell OptiPlex 3050 Micro | R$ 800-1000 | MercadoLivre, OLX |
| Servidor | HP ProLiant MicroServer Gen8 | R$ 1500-2000 | MercadoLivre |
| NAS | Synology DS218+ | R$ 1200-1500 | OLX, Facebook |
| Workstation | Lenovo ThinkStation P320 | R$ 2000-2500 | Leilões empresariais |

### Alternativas Criativas

1. **PC Gamer Antigo**: 
   - Ótimo para servidor
   - Já tem PSU potente
   - Múltiplos slots SATA

2. **Notebook Quebrado**:
   - Tela quebrada = servidor barato
   - Baixo consumo de energia
   - UPS integrado (bateria)

3. **Thin Clients**:
   - HP T620, Dell Wyse
   - R$ 200-400 no MercadoLivre
   - Perfeito para serviços leves

---

## Calculadora de Custos

### Consumo de Energia

| Setup | Consumo | Custo Mensal (R$) |
|-------|---------|-------------------|
| Raspberry Pi 5 | 4-8W | R$ 2-4 |
| Mini PC N100 | 10-15W | R$ 5-8 |
| NAS 4-bay | 30-50W | R$ 15-25 |
| Servidor Custom | 80-150W | R$ 40-75 |

*Cálculo: kWh = R$ 0.70 (estimativa media RJ)*

### ROI (Retorno de Investimento)

```
Serviços Tradicionais (por pessoa):
- Netflix: R$ 39/mês
- Spotify: R$ 21/mês  
- Google One: R$ 10/mês
- Total: R$ 70/mês = R$ 840/ano

Com Tainá (20 pessoas):
- Investimento inicial: R$ 3.500
- Custo mensal (energia + internet): R$ 100
- Custo por pessoa: R$ 5/mês = R$ 60/ano

Economia anual por pessoa: R$ 780
ROI: 4-5 meses
```

---

## Próximos Passos

1. **Defina seu orçamento** e número de usuários
2. **Escolha o hardware** baseado nas recomendações
3. **Monte uma comunidade** de interessados para dividir custos
4. **Compre as peças** (considere compra conjunta para desconto)
5. **Siga os guias** de montagem
6. **Instale o Tainá** seguindo nossa documentação
7. **Compartilhe** sua experiência com a comunidade!

---

## Suporte da Comunidade

> em breve...
- **Wiki**: [wiki.taina.tech/hardware](https://wiki.taina.tech/hardware)

---

## Avisos Importantes

1. **Backup**: Sempre implemente RAID ou backup externo
2. **UPS**: Essencial para proteger contra quedas de energia
3. **Ventilação**: Mantenha o hardware em local arejado
4. **Rede**: Use cabo ethernet sempre que possível
5. **Segurança**: Configure firewall e VPN desde o início

---

*Preços sujeitos a alteração. Sempre verifique disponibilidade e compare preços antes da compra.*
