# Shahed-136 / Geran-2 — Análise Técnica da Aviônica

### Documentação do diagrama em blocos · Arquitetura federada com barramento CAN

> ⚠️ **Aviso:** documento educacional baseado exclusivamente em inteligência de fontes abertas (OSINT) já publicadas. Não contém instruções de fabricação, modificação ou emprego de armamentos. A configuração real pode variar entre lotes de fabricação.

---

## Sumário

1. [A plataforma](#1-a-plataforma)
2. [Filosofia de projeto: arquitetura federada](#2-filosofia-de-projeto-arquitetura-federada)
3. [Visão geral do diagrama](#3-visão-geral-do-diagrama)
4. [Módulos do sistema](#4-módulos-do-sistema)
5. [Enlaces e fluxos de sinal](#5-enlaces-e-fluxos-de-sinal)
6. [Cenário de missão — as 5 fases](#6-cenário-de-missão--as-5-fases)
7. [Guerra eletrônica: por que o sistema resiste](#7-guerra-eletrônica-por-que-o-sistema-resiste)
8. [Especificações da plataforma](#8-especificações-da-plataforma)
9. [Custo e produibilidade](#9-custo-e-produibilidade)
10. [Referências](#10-referências)

---

## 1. A plataforma

O **Shahed-136** é uma munição de cruzeiro de ataque em configuração de **asa-delta**, desenvolvida pela HESA (Irã). Produzida na Rússia sob licença como **Geran-2**, é lançada de um **trilho inclinado** com auxílio de um **booster de foguete** (RATO), voa de forma **totalmente autônoma** — sem link de dados, sem operador remoto, sem vídeo — e mergulha sobre o alvo detonando a ogiva no impacto.

| Característica | Valor aproximado |
|----------------|------------------|
| Envergadura | ≈ 2,5 m |
| Comprimento | ≈ 3,5 m |
| Massa de decolagem | ≈ 200 kg |
| Ogiva | 50–90 kg (conforme a variante) |
| Motor | MADO MD-550 · 550 cc · 2 tempos |
| Potência | ≈ 50 cv · hélice traseira de acionamento direto |
| Velocidade de cruzeiro | ≈ 180–185 km/h |
| Alcance | até ≈ 2.500 km |
| Navegação | GNSS civil + INS + CRPA (triplo redundante) |
| Lançamento | Trilho inclinado + booster RATO |
| Controle | 100% autônomo — sem enlace de dados |
| Custo unitário | US$ 20–50 mil (estimativa OSINT) |

Apesar do baixo custo — e é justamente por causa dele — a aviônica revela uma engenharia de grande interesse: não é um drone "simples", mas um sistema cuidadosamente otimizado para **produção em massa, resistência a guerra eletrônica e tolerância a danos**.

---

## 2. Filosofia de projeto: arquitetura federada

A decisão de projeto central do Shahed-136 é a aviônica **"federada"**: em vez de um único computador de missão centralizado, **vários módulos pequenos e independentes computam em paralelo** e conversam entre si por um **barramento CAN de 1 Mbps** — o mesmo barramento da indústria automotiva.

**O que essa escolha compra:**

- **Produibilidade em massa** — cada módulo é uma placa pequena, barata e testável isoladamente; a linha de montagem não depende de um computador central complexo;
- **Resistência a danos** — a perda de um módulo não derruba o sistema inteiro; a arquitetura degrada graciosamente;
- **Flexibilidade** — módulos podem ser adicionados, removidos ou substituídos entre lotes sem redesenhar o sistema (evidência: o CRPA foi adicionado como retrofit);
- **Sanção-resistência** — quase todos os chips são componentes genéricos da indústria automotiva/industrial (TI, STMicroelectronics, NXP), abundantes e impossíveis de sancionar efetivamente;
- **Custo** — nenhum componente militar especializado, exceto uma exceção notável (o CRPA, ver §4.3).

**O que ela sacrifica:** desempenho. O CAN a 1 Mbps não transporta vídeo nem dados de sensor de alta taxa; a fusão de navegação depende de cálculo distribuído em DSPs modestos. Para uma munição de uso único que voa para waypoints pré-programados, esse sacrifício é irrelevante — e o ganho de custo/produção é decisivo.

---

## 3. Visão geral do diagrama

O diagrama organiza o sistema em **quatro camadas funcionais**, conectadas por cinco tipos de enlace (cada tipo com uma cor):

**Camadas (de cima para baixo):**

1. **Camada de navegação externa** — constelação GNSS (fonte de posicionamento) e o jammer inimigo (perturbação);
2. **Camada de sensoriamento/navegação embarcada** — CRPA, INS/IMU e ADC: três fontes independentes de estado de voo;
3. **Barramento CAN** — espinha dorsal horizontal que interliga tudo;
4. **Camada de atuação, controle e energia** — FCU (8 placas), PDU, baterias, magneto, motor, servos, ogiva e ignição.

**Tipos de enlace:**

| Cor | Tipo de sinal | Exemplo |
|-----|---------------|---------|
| 🔵 Azul | Sinal de navegação por satélite | Constelação → CRPA |
| 🟢 Verde | Dados de sensores (barramento CAN) | INS → CAN → FCU |
| 🟣 Roxo | Comandos de controle | FCU → Servos |
| 🟠 Laranja | Energia elétrica | Baterias → PDU |
| 🔴 Vermelho | Interferência (jamming) | Jammer → Constelação |

```text
                ┌────────────────────┐        ┌───────────────────────┐
                │  Constelação GNSS  │◀╌╌╌jam╌│ Interferência         │
                │ GPS·GLONASS·BeiDou │        │ Eletrônica (inimigo)  │
                └─────────┬──────────┘        └───────────────────────┘
          ┌───────────────┼──────────────────┐
          ▼               ▼                  ▼
  ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
  │ Antena CRPA  │ │  INS / IMU   │  │     ADC      │
  │ (Kometa-M)   │ │ ADIS16488    │  │ Pitot+estát. │
  └──────┬───────┘ └──────┬───────┘  └──────┬───────┘
         └────────────────┼─────────────────┘
                          ▼
  ════════════ CAN BUS · 1 Mbps · espinha dorsal federada ════════════
        │                  │                    │             │
        ▼                  ▼                    ▼             ▼
 ┌─────────────┐   ┌─────────────┐      ┌───────────┐  ┌──────────┐
 │ FCU         │   │ PDU         │      │ Servos ×4 │  │ Ogiva    │
 │ 8× F28335   │   │ 9º F28335   │      │ elevons + │  │ATtiny13A │
 └──────┬──────┘   └──────┬──────┘      │ lemes     │  └──────────┘
        │                 │             └───────────┘
        ▼                 ▼
 ┌─────────────┐   ┌────────────────────────────────────┐
 │ Ignição G108│   │ Baterias 18650 ← Magneto ← MD-550  │
 └─────────────┘   └────────────────────────────────────┘
```

---

## 4. Módulos do sistema

### 4.1 Constelação GNSS

> **Categoria:** Navegação / Satélite · **Função:** fornece a posição inicial e contínua

O Shahed-136 navega por sinais **civis** de **GPS** e **GLONASS**. A partir de 2025, os novos lotes passaram a incorporar o **BeiDou chinês**, melhorando a resistência a interferências regionais e a precisão.

O ponto crítico: sinais GNSS civis têm **potência extremamente fraca** na superfície e **nenhuma criptografia ou autenticação**. São trivialmente jamáveis e spoofáveis. É essa fragilidade que motiva as três redundâncias visíveis no diagrama: o CRPA (recepção resistente), o INS (navegação sem satélites) e o u-blox próprio do FCU (segunda recepção independente).

**Triangulação das fontes de posicionamento:**

| Fonte | Caminho no diagrama | Papel |
|-------|--------------------|-------|
| Recepção via CRPA | Constelação → CRPA → CAN → FCU | Posicionamento resistente a jamming |
| Recepção via INS/IMU (u-blox próprio) | Constelação → INS → CAN → FCU | Segundo receptor independente |
| Recepção direta (u-blox do FCU) | Constelação → FCU | Terceira via, a mais frágil |

---

### 4.2 Interferência eletrônica (o adversário)

> **Categoria:** Guerra Eletrônica · **Função:** suprime ou engana os sinais de satélite

Representado no diagrama como um nó externo apontando para a constelação. Os sistemas de guerra eletrônica em campo realizam duas técnicas:

- **Jamming de supressão:** transmissão de ruído potente na faixa L1/L2, saturando o receptor — o GPS "sai do ar";
- **Spoofing:** transmissão de sinais GNSS falsos e sincronizados, enganando o receptor para que calcule uma posição incorreta — sutilmente mais perigoso, pois o sistema não sabe que está perdido.

É o **maior adversário do projeto de navegação** do Shahed-136, e todo o conjunto de redundâncias do diagrama existe para responder a essa ameaça (ver §7).

---

### 4.3 Antena CRPA Antijamming (Kometa-M)

> **Categoria:** Navegação / Satélite · **Subsistema mais caro da aviônica**

O módulo mais caro de todo o sistema — **estimado em US$ 50–60 mil, possivelmente mais que todos os demais componentes eletrônicos somados**. Trata-se de um retrofit instalado pela russa **VNIIR-Progress** (não fazia parte do projeto original iraniano), conhecido pela família **Kometa-M**.

**Como funciona:**

- **16 elementos de antena** dispostos em arranjo, cada um recebendo o sinal com fase ligeiramente diferente;
- **Formação de feixe (beamforming):** o módulo combina digitalmente os 16 sinais, alinhando automaticamente o **nulo do padrão de recepção** com a direção da fonte de interferência e o **ganho máximo** com a direção dos satélites;
- Resultado: mantém o travamento nos satélites **mesmo sob jamming ativo**.

**Processamento próprio:** o CRPA não depende do FCU — possui um **NXP i.MX RT1052** (microcontrolador Cortex-M7 de alto desempenho) dedicado ao cálculo do beamforming, e um transceptor SDR **AD9361** (Analog Devices, fabricação chinesa) na frente de rádio. Ele entrega coordenadas já corrigidas ao barramento CAN.

| Componente | Papel |
|------------|-------|
| 16 elementos CRPA | Recepção direcional por arranjo de antenas |
| AD9361 (SDR) | Front-end de rádio definido por software |
| i.MX RT1052 | Processamento do beamforming em tempo real |

---

### 4.4 Navegação Inercial INS/IMU

> **Categoria:** Sensores / Dados · **A última linha de defesa**

A navegação inercial **não depende de nada externo**: mede acelerações e rotações e **estima a trajetória por integração (dead-reckoning)**. Mesmo com todos os satélites jamados, o INS continua sabendo "para onde estou indo" — com degradação lenta de precisão ao longo do tempo (deriva).

**Componentes:**

| Componente | Papel |
|------------|-------|
| ADIS16488 (Analog Devices) | IMU tática de alto desempenho — giroscópios + acelerômetros táticos |
| STM32F405 | Microcontrolador de processamento local |
| u-blox NEO-M8/9/10 | Receptor GNSS próprio — segunda fonte de posicionamento, independente do CRPA |

**Detalhe de instalação:** a IMU exige **isolamento antivibração** — as vibrações do motor a pistão, se atingirem os giroscópios MEMS, corrompem as medidas e fazem a posição derivar muito mais rápido.

---

### 4.5 Dados Aerodinâmicos (ADC)

> **Categoria:** Sensores / Dados · **Função:** velocidade e altitude barométrica

Converte as pressões medidas pelo **tubo de Pitot** (pressão dinâmica) e pela tomada **estática** (pressão atmosférica) em leituras digitais de **velocidade** e **altitude**. Com esses dados, o controle de voo sabe "a que velocidade e a que altitude estou voando" e mantém a aeronave dentro do envelope.

| Componente | Papel |
|------------|-------|
| TM4C1230 (Texas Instruments) | Microcontrolador do módulo |
| ADS122U (24 bits) | Conversor analógico-digital de alta resolução para as pressões |

Os dados de aerodinâmica entram no barramento CAN junto com os de navegação e são fusionados pelo FCU.

---

### 4.6 CAN BUS — espinha dorsal

> **Categoria:** Sensores / Dados · **O centro nervoso da arquitetura federada**

O **Controller Area Network a 1 Mbps** é o barramento predominante da indústria **automotiva**: barato, imune a vibrações, amplamente conhecido e produzido em volumes bilionários.

**Por que o CAN foi escolhido:**

- **Custo trivial** — transceptores (ex.: VP230) custam centavos e existem em abundância global;
- **Robustez** — projetado para ambientes elétrica e mecanicamente hostis (capô de carro);
- **Multi-mestre** — todos os nós conversam no mesmo par de fios, sem controlador central;
- **Suficiência** — 1 Mbps é mais que suficiente para navegação, telemetria e comandos (insuficiente apenas para vídeo — o que não é necessário: o Shahed não tem câmera).

Todos os módulos do diagrama penduram-se nessa espinha dorsal — é a materialização física do conceito "federado".

---

### 4.7 FCU — Controle de Voo (G-Boards)

> **Categoria:** Controle / Voo · **O cérebro da fusão de navegação**

O controle de voo **não é um único computador**: são **8 placas quase idênticas empilhadas** (estilo PC/104), compartilhando o barramento CAN, **cada uma com um DSP Texas Instruments F28335**. O que as diferencia é o **firmware** que cada uma executa:

| Placa | Função |
|-------|--------|
| **G110** | Interface de navegação — conecta u-blox NEO-M9N + STM32F446 |
| **G108** | Acionador de alta tensão da ignição do motor (isolamento por optoacoplador) |
| **G103 / G104 / G105 / G107 / G109** | Controle de servos, acelerador, telemetria, sensores do motor, separação do booster etc. |
| G106 | Suspeita-se: placa adaptadora |

**Papel no sistema:** o FCU **fusiona** as três fontes de navegação (CRPA / INS / u-blox próprio) com os dados aerodinâmicos do ADC, **compara** a posição estimada com os **waypoints pré-programados** antes do lançamento e **emite os comandos** de superfície, acelerador e ignição.

A redundância de placas idênticas (mesmo hardware, firmwares diferentes) simplifica a fabricação: uma única linha de montagem produz as oito placas.

---

### 4.8 Distribuição de Energia (PDU)

> **Categoria:** Sistema de Energia · **Com F28335 próprio**

A Power Distribution Unit é responsável pelo **sequenciamento, proteção e distribuição de energia** — alimenta cada subsistema e **monitora a corrente** de cada ramo por meio de **resistores shunt** lidos por um amplificador **LM258**.

**Curiosidade relevante:** a PDU carrega um **nono F28335** — é a única placa, além das oito do FCU, com esse DSP. Por isso a aeronave possui **9 F28335 no total**, e não 8.

| Componente | Papel |
|------------|-------|
| F28335 (9º DSP) | Supervisão da distribuição de energia |
| LM258 | Amplificador operacional do monitor shunt |

---

### 4.9 Pacote de Baterias

> **Categoria:** Sistema de Energia · **Energia de partida e reserva**

Conjunto de células comerciais de lítio **18650** — as mesmas de notebooks e baterias de e-bike. Lotes recentes usam células **Molicel B**.

Papel duplo: fornece a energia de **partida** (antes do motor girar) e serve de **reserva** quando a geração do magneto é momentaneamente insuficiente (picos de demanda dos servos, por exemplo).

---

### 4.10 Retificador do Magneto

> **Categoria:** Sistema de Energia · **Fonte principal em voo**

O **magneto do volante do motor** gera **corrente alternada de frequência variável** enquanto o motor gira. Um estágio retificador + regulador converte essa AC em **DC estabilizada** que alimenta toda a aeronave em voo.

É essa cadeia (motor → magneto → retificador → PDU) que permite voar **2.500 km sem carregar uma bateria enorme** — a energia vem da queima do combustível, e a bateria é só uma reserva de partida.

---

### 4.11 Motor MD-550

> **Categoria:** Sistema de Energia · **O "motor de motocicleta"**

Motor a pistão iraniano **MADO MD-550** (cópia do alemão Limbach), de **550 cc, 2 tempos, ≈ 50 cv**, refrigeração a ar, óleo misturado na gasolina (sem sistema de lubrificação independente).

**Arquitetura de propulsão deliberadamente simples:**

- **Acionamento direto** da hélice propulsora traseira (pusher), **sem caixa de redução**;
- Cruzeiro de ~180 km/h e alcance de até ~2.500 km;
- Partida elétrica acionada pela placa G108.

Esse "motor de motocicleta grande" é a chave do equilíbrio custo × alcance: nenhum turbofan, nenhuma turbina — um motor industrial barato e comprovado.

---

### 4.12 Servos ×4

> **Categoria:** Controle / Voo · **Apenas 4 atuadores em toda a aeronave**

O Shahed-136 tem **apenas 4 superfícies de controle móveis e 4 servos**:

| Superfície | Movimento conjunto | Movimento diferencial |
|------------|--------------------|-----------------------|
| 2 **elevons** (asas) | Arfagem (pitch) | Rolagem (roll) |
| 2 **lemes de ponta de asa** | — | Guinada (yaw) |

Cada servo é uma unidade inteligente com:

| Componente | Papel |
|------------|-------|
| STM32F030 | Microcontrolador do servo |
| AS5600 | Encoder magnético de posição (feedback de ângulo) |
| EG2134 | Driver de potência do motor do servo |

Pouquíssimos atuadores = custo mínimo, mecânica mínima e calibração simples. Para uma munição que só precisa voar reto, curvar e mergulhar, quatro superfícies bastam.

---

### 4.13 Detonação da Ogiva

> **Categoria:** Controle / Voo · **Cadeia de segurança de três estágios**

Feito **deliberadamente simples** — exatamente como uma cadeia de detonação segura deve ser.

| Estágio | Função |
|---------|--------|
| Armar | Habilita o sistema somente após condições de voo |
| Pronto | Confirma aproximação do alvo |
| Detonar | Dispara o detonador no impacto / voo terminal |

| Componente | Papel |
|------------|-------|
| ATtiny13A | Microcontrolador de 8 pinos — intertravamento lógico |
| Relé HFD4/5-S | Chave de potência do detonador |

O princípio: **simples = confiável = sem detonações acidentais**. Um chip de centavos e um relé fazem a segurança, sem sofisticação que possa falhar.

---

### 4.14 Ignição do Motor (G108)

> **Categoria:** Controle / Voo · **Chave de potência isolada**

A placa G108 do FCU aciona a **bobina de ignição** do motor MD-550. Como fica fisicamente perto do motor — ambiente de altíssimo ruído elétrico — o isolamento é crítico:

| Componente | Papel |
|------------|-------|
| MOSFET AUIRF4905S | Chave de potência da bobina |
| Optoacoplador H11G1 | **Isolamento elétrico galvânico** entre lógica e potência |
| LM234 / LM258 | Referência de corrente e amplificação |

Os optoacopladores impedem que picos de alta tensão da ignição retornem pelo barramento e corrompam o sensível circuito de controle de voo.

---

## 5. Enlaces e fluxos de sinal

Relação completa das conexões representadas no diagrama:

| # | Origem | Destino | Tipo | Observação |
|---|--------|---------|------|------------|
| 1 | Constelação GNSS | CRPA | 🔵 GNSS | Recepção via arranjo antijamming |
| 2 | Constelação GNSS | INS | 🔵 GNSS | Recepção via u-blox do INS — perde travamento sob jamming |
| 3 | Constelação GNSS | FCU | 🔵 GNSS | Recepção direta via u-blox do G110 — perde travamento sob jamming |
| 4 | Interferência Eletrônica | Constelação | 🔴 Jamming | Ameaça ativa (aparece só no cenário de guerra eletrônica) |
| 5 | CRPA | CAN BUS | 🟢 Dados | Coordenadas corrigidas |
| 6 | INS | CAN BUS | 🟢 Dados | Estado inercial (atitude + trajetória estimada) |
| 7 | ADC | CAN BUS | 🟢 Dados | Velocidade + altitude |
| 8 | CAN BUS | FCU | 🟢 Dados | Fusão de navegação no controle de voo |
| 9 | FCU | Servos | 🟣 Comandos | Pitch / roll / yaw |
| 10 | FCU | Ogiva | 🟣 Comandos | Cadeia armar → pronto → detonar |
| 11 | FCU | Ignição (G108) | 🟣 Comandos | Partida do motor |
| 12 | PDU | CAN BUS | 🟠 Energia | Alimentação da espinha |
| 13 | Baterias | PDU | 🟠 Energia | Partida e reserva |
| 14 | Magneto | PDU | 🟠 Energia | Geração em voo |
| 15 | Motor | Magneto | 🟠 Energia | Fonte mecânica da geração |
| 16 | Ignição | Motor | 🟣 Comando | Aciona a bobina |

**Leitura funcional do fluxo principal (cruzeiro nominal):**

```text
Satélites → CRPA/INS/ADC → CAN BUS → FCU (fusão + waypoints)
         → comandos → Servos (correção de trajetória)
```

**Fluxo de energia (do combustível ao chip):**

```text
Gasolina → Motor MD-550 → Magneto → Retificador → PDU → módulos
                                Baterias 18650 ↗ (partida/reserva)
```

---

## 6. Cenário de missão — as 5 fases

O diagrama representa um voo completo como uma sequência de cinco fases, cada uma com um **subconjunto de módulos ativos** (os demais ficam em repouso/esmaecidos):

### Fase ① — Lançamento com booster de foguete
**Módulos ativos:** PDU · Baterias · FCU · CAN BUS

Posicionado em um **trilho inclinado**, o booster de foguete sob a fuselagem acende por 1–2 segundos, acelerando o drone até **~50 m/s**. Neste momento: alimentação vinda da **bateria** (o motor ainda não gira), o controle de voo desperta, e a navegação ainda não é necessária.

### Fase ② — Descarte do booster · partida do motor
**Módulos ativos:** FCU · CAN · Ignição · Motor · Magneto · Baterias · PDU

Ao atingir a velocidade necessária, o foguete é **descartado**. O G108 aciona a ignição, o **MD-550 dá a partida**, o **magneto** começa a gerar energia e assume a alimentação (bateria passa a reserva). A aeronave entra em voo autônomo.

### Fase ③ — Cruzeiro com navegação autônoma
**Módulos ativos:** GNSS · CRPA · INS · ADC · CAN · FCU · Servos · Motor · Magneto

O coração da missão. **Triplo posicionamento** (CRPA / INS / u-blox do FCU) + **fusão com dados aerodinâmicos**: o FCU compara a posição estimada com os **waypoints pré-programados**, comanda os **4 servos** para corrigir rumo e altitude, e parte num voo de longa distância (até **2.500 km**).

### Fase ④ — Sob interferência eletrônica ⚡
**Módulos ativos:** Jammer · GNSS · CRPA · INS · CAN · FCU · Servos

Ao entrar na área do alvo, o drone sofre **jamming de GPS**. O GPS civil (recepções diretas) **perde o travamento**. Mas:

- a **formação de feixe do CRPA** aponta o nulo para o jammer e **mantém o travamento** nos satélites;
- o **INS** assume a navegação inercial (dead-reckoning) enquanto necessário.

A aeronave mantém a trajetória. **Este trecho decide entre o sucesso e o fracasso da missão** — e é o motivo de toda a redundância de navegação do projeto.

### Fase ⑤ — Mergulho terminal · detonação no impacto
**Módulos ativos:** FCU · INS · CRPA · Servos · Ogiva

Após o último waypoint, a aeronave **mergulha** sobre o alvo. Os servos fazem as correções finais de trajetória; a cadeia de detonação (ATtiny13A) remove o intertravamento de segurança e **detona a ogiva de 50–90 kg no impacto**. Missão de uso único concluída.

---

## 7. Guerra eletrônica: por que o sistema resiste

O Shahed-136 foi projetado assumindo que **sempre será jamado** na aproximação final. A defesa em profundidade tem três anéis:

| Anel | Mecanismo | Princípio |
|------|-----------|-----------|
| **1** | CRPA (Kometa-M) | Filtra a interferência **na antena** — beamforming aponta o nulo para o jammer e o ganho para os satélites |
| **2** | INS (ADIS16488) | Navega **sem qualquer sinal externo** — dead-reckoning por giroscópios e acelerômetros |
| **3** | Redundância de receptores | Três vias GNSS independentes (CRPA ≠ u-blox do INS ≠ u-blox do FCU) |

**Efeitos observados quando o jamming é ativado:**

| Enlace | Estado | Motivo |
|--------|--------|--------|
| Constelação → INS (direto) | ❌ Perdido | Receptor civil sem proteção |
| Constelação → FCU (direto) | ❌ Perdido | Receptor civil sem proteção |
| Constelação → CRPA → CAN | ✅ Mantido | Beamforming anula o jammer na direção de chegada |
| INS → CAN (inercial) | ✅ Mantido | Não depende de sinal externo |
| Jammer → Constelação | 🔴 Ativo | Ameaça visível no cenário |

**Conclusão operacional:** para derrubar a navegação de um Shahed-136 modernizado não basta jamar o GPS — é preciso vencer o CRPA (potência direcional muito maior) **e** ainda assim o INS continuará conduzindo a aeronave aos waypoints com precisão decrescente lenta. É por isso que a munição é tão difícil de neutralizar com guerra eletrônica, e por isso que a interceptação cinética (antiaérea, caças, drones interceptores) virou a resposta predominante.

---

## 8. Especificações da plataforma

| Característica | Valor |
|----------------|-------|
| Envergadura | ≈ 2,5 m |
| Comprimento | ≈ 3,5 m |
| Massa de decolagem | ≈ 200 kg |
| Ogiva | 50–90 kg (conforme a variante) |
| Motor | MADO MD-550 · 550 cc · 2 tempos |
| Potência | ≈ 50 cv · hélice traseira direta |
| Velocidade de cruzeiro | ≈ 180–185 km/h |
| Alcance | até ≈ 2.500 km |
| Navegação | GNSS civil + INS + CRPA |
| Lançamento | Trilho inclinado + booster RATO |
| Controle | Autônomo — sem link de dados |
| DSPs F28335 a bordo | 9 (8 no FCU + 1 na PDU) |
| Atuadores | 4 servos (2 elevons + 2 lemes) |
| Barramento | CAN 2.0 · 1 Mbps |
| Custo unitário | US$ 20–50 mil (estimativa OSINT) |
| CRPA Kometa-M (retrofit) | US$ 50–60 mil (estimativa) |

---

## 9. Custo e produibilidade

| Decisão de projeto | Efeito no custo/produção |
|--------------------|--------------------------|
| Barramento CAN automotivo | Transceptores de centavos, produção bilionária, insancionável |
| 8 placas FCU idênticas (mesmo DSP, firmwares diferentes) | Uma única linha de montagem produz todas |
| Chips TI/ST/NXP genéricos | Estoque global, múltiplos fornecedores, sem controle de exportação eficaz |
| Apenas 4 servos com componentes de aeromodelismo | Mecânica mínima, calibração simples |
| Células 18650 comerciais | Compradas em qualquer mercado |
| Motor de motocicleta adaptado | Sem turbinas, sem metalurgia especial, manutenção trivial |
| Ogiva com ATtiny13A + relé | Chip de centavos fazendo a função crítica de segurança |
| **Única exceção:** CRPA Kometa-M | ~US$ 50–60 mil — e mesmo assim seu SDR (AD9361) é de fabricação chinesa |

O resultado é uma munição cujo **custo total é menor que o de muitos mísseis antiaéreos usados para interceptá-la** — uma inversão econômica que redefine o problema da defesa antiaérea.

---

## 10. Referências

1. **Stefan Nikolaj** — *"Geran-2 / Shahed-136 Hardware Analysis"* — análise detalhada das placas G-Boards, CRPA Kometa-M, INS, ADC, PDU, ignição e motor;
2. **War Sanctions (GUR da Ucrânia)** — portal de rastreamento de componentes com fotos de desmontagem dos módulos;
3. **ISIS (Middlebury Institute)** — *"Electronics in the Shahed-136"* — identificação dos componentes eletrônicos.

> Nota final: a configuração real varia entre lotes de fabricação e versões (iraniana original vs. russa Geran-2 vs. retrofit Kometa-M). Os valores aqui apresentados são aproximações de fontes abertas e servem ao propósito educacional de entender a arquitetura — não a um inventário exato de qualquer espécime específico.