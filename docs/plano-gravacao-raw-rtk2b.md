# Gravação RAW local na base RTK (ArduSimple simpleRTK2B + ESP32-S3) e pós-processamento PPP/PPK

**Cliente:** LuftPlan Soluções Integradas a Projetos (Curitiba/PR)
**Data:** 2026-10-04 · **Versão:** 1.0 (análise e plano; nada foi executado no equipamento)
**Formato:** handoff para outro agente/engenheiro
**Documento complementar:** [`automacoes-esp32s3.md`](automacoes-esp32s3.md) traz automações de campo e escritório sobre a arquitetura A1.

**Convenções:**
- **[HIPÓTESE]**: inferência minha, ainda não comprovada por fonte. Cada uma tem um teste na seção 7.
- **[não verificado]**: não encontrei fonte primária nesta sessão.
- **[F#]**: referência à lista de fontes (seção 10).
- **[trecho de busca]**: o fato veio do resumo de um mecanismo de busca, não da leitura do documento original. Confiança menor; confirmar no original.

**Limitação desta pesquisa:** o proxy desta sessão bloqueou os sites de fabricantes e de órgãos públicos (u-blox, ArduSimple, Espressif, IBGE, NRCan, MDPI, NCBI). Por isso:
- Os documentos u-blox foram lidos em cópias oficiais hospedadas no GitHub da SparkFun: Integration Manual R13 e Datasheet ZED-F9P-02B R03.
- ESP-IDF, RTKLIB, RTKBase, pyubx2 e os firmwares ESP32 foram lidos diretamente no código-fonte, com commit indicado.
- IBGE-PPP, CSRS-PPP, RBMC, SW Maps e os artigos científicos vieram de trechos de busca e estão marcados como tal.

---

## 1. Resumo executivo

1. **Recomendação principal (Arquitetura A1):** usar o **ESP32-S3 + SD como gravador dedicado na UART1 do ZED-F9P**. A **ponte Bluetooth atual fica intocada na UART2** (soquete XBee). O S3 grava o stream UBX bruto (`UBX-RXM-RAWX` + `UBX-RXM-SFRBX` + `UBX-NAV-PVT`) a 1 Hz, com UART1 a 230400 bps. São cerca de **10 MB/h** (~240 MB/dia).
2. **Por quê:**
   - **SW Maps e GNSS Master continuam funcionando sem nenhuma mudança.** O ESP32-S3 **não tem Bluetooth Classic** [F7]. Se ele substituísse a ponte, o SPP no Android quebraria.
   - **Não precisa de RTC:** o tempo GNSS já vem dentro de cada mensagem.
   - **A vazão é trivial para o cartão SD.**
   - **Tudo usa padrões abertos:** UBX → RINEX com RTKLIB (licença BSD-2).
3. **Pós-processamento da base:**
   - **Principal: IBGE-PPP.** Já entrega SIRGAS2000 época 2000,4 de forma automática [F15]. Ocupação de **≥ 4 h** recomendada pelo IBGE [F15, trecho de busca]. Com ZED-F9P + antena patch, o PPP só atingiu "poucos cm" com **≥ 2,5 h** [F20].
   - **Verificação independente: CSRS-PPP.**
   - **Sessões curtas perto de estação RBMC** (ex.: UFPR em Curitiba): **PPK estático com RTKLIB-EX**. Uma sessão de 1 h chegou a "poucos cm" na horizontal [F20].
4. **Ganho rápido sem hardware novo:**
   - Habilitar RAWX/SFRBX na porta da ponte, **só na camada RAM** (reverte ao desligar), e usar o "Log to file" do SW Maps [F18].
   - Serve para validar a cadeia de pós-processamento antes do gravador existir.
   - Exige firmware HPG ≥ 1.30, que é o mínimo para a UART2 emitir UBX [F2].
5. **Alternativa mais robusta, para quando houver caster local e base fixa recorrente:** RTKBase (AGPL-3.0) em Raspberry Pi ou mini-PC [F10].
6. **Principal risco técnico:** a antena ANN-MB-00 (se for ela) **não tem calibração ANTEX pública verificada**. Isso pode introduzir viés vertical em PPP [HIPÓTESE; teste T15]. Trocar a antena está fora do escopo (ver seção 11).

---

## 2. Perguntas em aberto (com a suposição adotada)

| # | Pergunta ao cliente | Por que importa | Suposição adotada neste plano |
|---|---|---|---|
| Q1 | Qual a variante exata do RTK2B: Budget, Pro, V3 ou Lite? Foto da placa ajuda. | O roteamento das UARTs muda. No Budget o soquete XBee é **fixo na UART2**. No Pro/V3 o soquete é **selecionável UART1/UART2** [F5]. | **simpleRTK2B Budget**: XBee = UART2; UART1 nos pinos TX1/RX1 e no conector Pixhawk JST-GH [F5]. |
| Q2 | Qual a versão de firmware do ZED-F9P? Ver `UBX-MON-VER` no u-center ou PyGPSClient. | Antes do HPG 1.30 a UART2 **não emite UBX** [F2]. Isso só afeta o ganho rápido via SW Maps. | ZED-F9P-04B com **HPG 1.32** [F2]. |
| Q3 | Qual é a ponte ESP32 atual: ArduSimple "BT+BLE Bridge" ou montagem própria? Onde está ligada, com qual baud e qual firmware? | Define o que não pode ser tocado e qual protocolo (SPP/BLE) os celulares usam. | ArduSimple BT+BLE Bridge (ESP32 clássico, SPP+BLE) no soquete XBee/UART2 a 115200 [F6, trecho de busca]. |
| Q4 | Qual placa ESP32-S3: devkit, flash e PSRAM (ex.: DevKitC-1 N8R2 ou N16R8)? | Pinagem livre e tamanho do buffer. Nos módulos com PSRAM octal os GPIO35–37 ficam ocupados [HIPÓTESE]. | DevKitC-1 com PSRAM. Pinos SD fora de GPIO35–37. |
| Q5 | Qual módulo de SD (modelo ou foto) e qual cartão (capacidade, classe, marca)? | Módulos comuns são **somente SPI** e às vezes exigem 5 V no regulador. O FAT do ESP-IDF vem **sem exFAT** [F8]. | Módulo SPI com regulador e level shifter; microSD SDHC 8–32 GB "high endurance", em FAT32. |
| Q6 | Qual antena (ANN-MB-00, AS-ANT2B-CAL ou outra)? Há calibração? | A correção de centro de fase (PCO/PCV) afeta PPP e PPK [F16]. | ANN-MB-00, sem calibração ANTEX [não verificado]. |
| Q7 | Quantos receptores existem: só um (base = rover em momentos diferentes) ou base + rover? | Define se é preciso RTCM local agora. | **Um receptor hoje.** O plano grava na base; o RTCM local fica para a Fase 7, opcional. |
| Q8 | Como o sistema é alimentado em campo (power bank, LiPo, bateria 12 V)? | Quedas de energia são o principal risco de corrupção do SD. | Power bank USB 5 V. |
| Q9 | Qual a tolerância de entrega (ex.: GCP 3 cm H / 5 cm V)? Qual o referencial e qual a altitude (geométrica ou normal)? | Define a duração das sessões e os critérios de aceite. | SIRGAS2000 (2000,4), UTM fuso 22S, altitude normal via hgeoHNOR2020 [F17]. |
| Q10 | É necessário corrigir rovers em tempo real **sem internet** já na primeira fase? | Exige enlace de RTCM (Wi-Fi local ou rádio). | Desejável, não obrigatório (Fase 7). |
| Q11 | Qual o sistema operacional do PC de processamento? | Binários do RTKLIB (Windows/Linux) e scripts Python. | Windows ou Linux com Python 3. |
| Q12 | Qual a duração típica de permanência da base numa obra? | Viabilidade de PPP (≥ 4 h) versus PPK. | 4–8 h por dia de campo. |

---

## 3. Respostas técnicas por tema (escopo 4.1–4.5)

### 3.1 Hardware e receptor (4.1)

**Chip GNSS.** Todas as variantes simpleRTK2B (Budget, Pro, Lite) usam o **u-blox ZED-F9P** [F5, trecho de busca]. A hipótese do cliente está confirmada; resta confirmar a variante (Q1).

| Item | Valor | Fonte |
|---|---|---|
| Sinais (ZED-F9P-01B/02B/04B) | GPS L1C/A+L2C · GLONASS L1OF+L2OF · Galileo E1B/C+E5b · BeiDou B1I+B2I · QZSS L1C/A+L2C · SBAS L1C/A | Integration Manual R13 §3.1.2 [F2] |
| Variante L1/L5 | ZED-F9P-15B (HPG L1L5 1.40): GPS L1C/A+L5, Galileo E1+E5a, BeiDou B1I+B2a | [F2] |
| **BeiDou no Brasil** | Os satélites BDS-3 **não transmitem B2I** (transmitem B1I, B3I, B1C, B2a, B2b). Com o F9P-04B, o BeiDou aqui fica quase só com **B1I (monofrequência)**. | [F21, trecho de busca] + **[HIPÓTESE]** sobre a constelação visível |
| Taxa máxima (datasheet 02B, HPG 1.13) | RAW: **20 Hz** com GPS+GLO+GAL+BDS e **25 Hz** com menos constelações. RTK: 8 Hz (4 GNSS) a 20 Hz (só GPS). | Datasheet R03 Tabela 2 [F3] |
| Observação de firmware | A ArduSimple indica limite prático de ~7 Hz de navegação com HPG 1.32 (e 10 Hz com 1.13). Irrelevante para base a 1 Hz. | [F5b, trecho de busca] |
| Mensagens RAW | `UBX-RXM-RAWX` (0x02 0x15): payload **16 + 32·numMeas**. `UBX-RXM-SFRBX` (0x02 0x13): payload **8 + 4·numWords**. | Estrutura em pyubx2 1.3.8 [F11]; manual §3.15 [F2] |
| Firmware atual | F9P-04B: **HPG 1.32** · F9P-05B: **HPG 1.51** (R01, 01-nov-2024) · F9P-15B: HPG L1L5 1.40 | [F2], [F4, trecho de busca] |
| Precisão RTK (spec) | 0,01 m + 1 ppm CEP (H); 0,01 m + 1 ppm R50 (V) | Datasheet [F3] |

**Portas do ZED-F9P e roteamento no simpleRTK2B Budget:**

| Porta ZED | Onde aparece na placa | Uso hoje (suposto) | Uso proposto |
|---|---|---|---|
| UART1 | Pinos Arduino **TX1/RX1** e conector **Pixhawk JST-GH**. Exige ligar **IOREF** à tensão do host (3,3 V) [F5]. | Livre | **Gravador ESP32-S3** (UBX out; UBX in opcional para configuração) |
| UART2 | **Soquete XBee** e pinos TX2/RX2 [F5] | Ponte Bluetooth (NMEA out / RTCM in) | **Não mexer** |
| USB | Conector "POWER+GPS" [F5] | Configuração no PC | Configuração, backup e firmware (com aprovação) |
| SPI/I2C | Disponíveis no módulo. SPI exige D_SEL=0, o que desliga a UART1 [F3]. No Pro há I2C/Qwiic [F5]. | — | Não usar |

**Padrões de fábrica das portas** [F2 §3.1.3]:
- UART1: 38400 bps, NMEA out; UBX e RTCM habilitados sem mensagens.
- UART2: 38400 bps, RTCM in/out. **UBX out só a partir do HPG 1.30.** UBX in padrão a partir do 1.32. NMEA desligado.
- O manual também avisa: "Não use a UART2 como única interface com o host" (sem firmware update/safeboot por ela).

**Log interno.** A hipótese "não tem log" está **parcialmente errada**. O ZED-F9P tem **log em flash interna**, mas **apenas de posições** (máx. 1 Hz, comprimidas, ~0,1 m) e de strings do host, via `UBX-LOG-*` e `CFG-LOGFILTER-*` [F2 §3.6]. **Não grava RAWX/SFRBX.** Para PPP/PPK **é obrigatório um gravador externo**.

### 3.2 Gravação local (4.2)

#### Throughput (cálculo com fórmula verificável)

Premissas [HIPÓTESE, medir em T04]:
- Céu aberto em Curitiba.
- GPS 10 satélites (8 com L2C), GLONASS 7×2, Galileo 8×2, BeiDou 8 (B1I).
- Total N ≈ 60 sinais por época; faixa de 40 a 80.
- Moldura UBX de 8 bytes.

Fórmulas:
- RAWX por época = 8 + 16 + 32·N bytes.
- SFRBX ≈ **0,8 kB/s**, independente da taxa de medição. Cálculo por constelação: GPS L1 10 palavras/6 s · L2C 10 palavras/12 s · GLONASS 4 palavras/2 s por sinal · Galileo 8 palavras/2 s por sinal · BDS 10 palavras/6 s. Palavras por constelação conforme [F2 §3.15].

| N sinais | Taxa | Total (kB/s) | MB/h | Carga UART @115200 | @230400 | @460800 |
|---|---|---|---|---|---|---|
| 60 | **1 Hz** | **2,8** | **9,9** | 24 % | **12 %** | 6 % |
| 60 | 2 Hz | 4,7 | 16,9 | 41 % | 20 % | 10 % |
| 60 | 5 Hz | 10,5 | 37,9 | **91 % (inviável)** | 46 % | 23 % |
| 60 | 10 Hz | 20,3 | 72,9 | 176 % | 88 % | 44 % |
| 80 | 1 Hz | 3,4 | 12,2 | 29 % | 15 % | 7 % |
| 80 | 5 Hz | 13,7 | 49,4 | 119 % | 60 % | 30 % |

Carga = bytes·10 bits ÷ baud (8N1).

**Conclusões:**
- **Base a 1 Hz com UART1 a 230400 bps** deixa folga para NAV-PVT/MON-COMMS e, no futuro, RTCM.
- 5 Hz exige pelo menos 460800 bps.
- Se a banda for insuficiente, o próprio receptor descarta mensagens: "Once the buffer space is exceeded, new messages to be sent will be dropped" [F2 §3.8]. O monitoramento usa `UBX-MON-COMMS` [F2].
- **Armazenamento:** ~240 MB/dia a 1 Hz. Um cartão de 32 GB comporta mais de 100 dias.
- **Escrita no SD:** SDSPI tem teto de 20 MHz (`SDMMC_FREQ_DEFAULT`) [F8]. SDMMC no S3 chega a 40 MHz, 1/4 linhas [F8]. As duas ficam ordens de grandeza acima de 3–20 kB/s.
- **O gargalo real é a latência**: pausas de coleta de lixo interna do cartão [HIPÓTESE; medir em T06]. Isso se resolve com buffer.

#### Buffer, perda de bytes, flush e rotação (projeto do firmware gravador)

- **RX da UART:** driver ESP-IDF `uart_driver_install` com ring buffer de RX ≥ 32 KB e fila de eventos.
  - Monitorar `UART_FIFO_OVF` e `UART_BUFFER_FULL` [F8].
  - A FIFO de hardware tem só 128 bytes (`SOC_UART_FIFO_LEN`) [F8], por isso a tarefa de RX precisa de prioridade alta.
  - O S3 também suporta UART via DMA (UHCI) [F8]. Opcional; usar se T07 mostrar perdas.
- **Buffer de aplicação:** ring buffer de 64–256 KB, em PSRAM se houver.
  - 64 KB a 2,8 kB/s cobrem cerca de 23 s de travamento do SD.
  - A ideia de tarefas separadas (RX → ring buffer → consumidores SD/BT/TCP) é a mesma do firmware SparkFun RTK Everywhere [F12]. Ele serve de referência de projeto (licença MIT).
- **Tarefa de escrita:** grava blocos de 4–16 KB, múltiplos de 512 B.
  - `fsync`/`f_sync` a cada **10 s** e na rotação. Perda máxima numa queda de energia ≈ intervalo de sync + buffer.
  - Não habilitar `CONFIG_FATFS_IMMEDIATE_FSYNC` (sync a cada write); o Kconfig avisa sobre a queda de desempenho [F8].
- **Rotação:** um arquivo por hora (~10 MB). Limita o estrago de uma corrupção e facilita a cópia.
  - Nome em UTC derivado de `UBX-NAV-PVT` quando as flags de data/hora válidas estiverem ativas. Exemplo: `LPB1_20261004_1400.ubx`.
  - Antes disso: `LPB1_boot0042_s001.ubx`, renomeado depois.
- **Tempo:** **não precisa de RTC.** Cada RAWX traz `rcvTow`, `week` e `leapS` [F11]. O relógio do ESP32 só nomeia arquivos.
- **Integridade:** um parser UBX leve conta mensagens válidas e erros de checksum, sem filtrar nada: grava **todos os bytes**. Um CSV de saúde por minuto registra bytes recebidos, mensagens OK, erros de checksum, eventos de overflow, uso máximo do ring, latência máxima de escrita, espaço livre, `carrSoln`/numSV e tensão.
- **Queda de energia:**
  - UBX é auto-sincronizável. Um arquivo truncado continua convertível (o RTKLIB ressincroniza no `B5 62`) [HIPÓTESE; T09].
  - A estrutura FAT pode corromper se o corte acontecer durante a atualização de diretório. Mitigações: rotação horária, cartão de qualidade e supervisor de tensão (ADC) que fecha o arquivo abaixo de um limiar. Capacitor de reserva é opcional [HIPÓTESE].
- **Sistema de arquivos:** FAT32. O FatFs do ESP-IDF vem com `FF_FS_EXFAT 0` [F8]. Limite de 4 GiB por arquivo, irrelevante com rotação horária. Formatar cartões > 32 GB em FAT32 é **destrutivo** e exige aprovação.
- **Framework:** ESP-IDF v5.x em C, para controle fino de UART, FATFS, `esp_http_server` e softAP. Arduino-ESP32 core 3.x é aceitável. MicroPython não é recomendado para esta função [HIPÓTESE: pausas de GC].

#### Bluetooth: o que o ESP32-S3 suporta e o impacto

- **O ESP32-S3 suporta só Bluetooth LE**, sem BR/EDR Classic. Tabela oficial do ESP-IDF: "ESP32-S3 | Bluetooth Classic: N | Bluetooth LE: Y" [F7]. Em `soc_caps.h` do S3 não existe `SOC_BT_CLASSIC_SUPPORTED`, presente no ESP32 [F8]. **Hipótese confirmada.**
- **iPhone:** a Apple não suporta SPP (fora do MFi), então iOS só funciona por BLE [F12 docs]. O SW Maps no iOS conecta por BLE [F12 docs, F19].
- **Android:** SPP é o caminho mais comum. Há relatos de **desconexões do SW Maps via BLE no Android** com a ponte BLE da ArduSimple [F19, trecho de busca].
- **GNSS Master** é da ArduSimple, **somente Android**, e suporta BT, BLE, USB e TCP [F18b, trecho de busca].
- **Impacto:**
  - Se o S3 **substituir** a ponte (Arquitetura A2), usuários Android perdem o SPP e passam a depender de BLE. Risco de regressão no fluxo principal, que precisa ser testado antes (T18).
  - Se o S3 for **só gravador** (A1), não há impacto nenhum.
  - Wi-Fi e BLE no S3 dividem o mesmo rádio por time-slicing [F8 `coexist.rst`]. Mais um motivo para não acumular BLE + Wi-Fi + SD no mesmo chip.

#### Wi-Fi AP local (configuração e download sem internet)

- Rodar softAP + `esp_http_server` no S3 com quatro rotas: `/status` (JSON de saúde), `/files` (lista), `/download?f=` e `/cfg` (só leitura na Fase 1). WPA2 com senha.
- **Arquivos grandes:** retirar o cartão e ler no PC é mais simples e mais robusto. O download por Wi-Fi compete pelo SD com a gravação, por isso precisa de mutex e prioridade menor que a gravação [HIPÓTESE; T13].

#### Configuração reproduzível do receptor

- **Interface:** `CFG-VALSET` com camadas **RAM / BBR / Flash** [F2 §3.1; campo `layers` em F11].
- **Procedimento sem risco:**
  1. Aplicar **só em RAM**, que reverte ao desligar.
  2. Validar.
  3. Só então gravar em Flash, **com aprovação**.
- **Versionamento:**
  - Arquivo texto `KEY,valor` no mesmo formato usado pelo RTKBase (`receiver_cfg/U-Blox_ZED-F9P_rtkbase.cfg`) [F10], guardado em git.
  - Backup binário completo com **`ubxsave`**, que lê todo o banco via `CFG-VALGET` e grava como `CFG-VALSET`.
  - Restauração com **`ubxload`** (carrega na camada RAM).
  - Diferenças com **`ubxcompare`**.
  - Os três vêm do pacote `pyubxutils` 1.0.6, BSD-3 [F13].
  - O PyGPSClient (BSD-3) também carrega arquivos de configuração no formato da ArduSimple [F13].

### 3.3 Operação da base (4.3)

**1. Survey-in ou coordenada fixa.** O RAW gravado é o mesmo nos dois casos. O modo `CFG-TMODE-*` afeta a solução e o RTCM, não as observações [HIPÓTESE; T16].

| Modo | Coordenada da base | Efeito no RTK do rover | Efeito no PPP/PPK da base |
|---|---|---|---|
| Survey-in (`CFG-TMODE-MODE=SURVEY_IN`, `SVIN_MIN_DUR`, `SVIN_ACC_LIMIT`) [F2] | Média de soluções autônomas (erro de metro) | Todo o levantamento sai **deslocado** pelo erro da base; a geometria relativa fica correta | Nenhum. O RAW gera a coordenada correta depois. |
| Fixo (`CFG-TMODE-MODE=FIXED`, `POS_TYPE`, `ECEF_*` ou `LAT/LON/HEIGHT` + `_HP`) [F2] | Conhecida (PPP anterior ou vértice) | O rover sai no referencial da coordenada informada. O receptor emite `INF-WARNING` se a coordenada estiver errada em mais de ~50 m [F2]. | Nenhum |

- **Fluxo recomendado em obra nova:** survey-in curto + gravação RAW → PPP da base → transladar os pontos RTK por Δ = (PPP − survey-in). É prática comum [HIPÓTESE; T17].
- **Em revisitas:** modo fixo com a coordenada PPP já guardada num arquivo por obra.
- **Aviso do manual:** a base em survey-in **não deve receber RTCM** [F2].

**2. Gravar RAW e transmitir RTCM ao mesmo tempo.** **Sim, é possível.**
- O ZED emite UBX e RTCM na mesma porta ou em portas diferentes.
- **RTCM recomendado** para base em configuração padrão [F2 §3.1.5.5.3]:
  - `1005` (ARP);
  - MSM4 `1074` (GPS), `1084` (GLONASS), `1094` (Galileo), `1124` (BeiDou);
  - `1230` (vieses de código GLONASS).
- **Regras do manual** [F2]:
  - Todas as observações na mesma taxa.
  - Não misturar MSM4 com MSM7.
  - Se o RTCM sair em várias portas, a configuração deve ser idêntica em todas.
- **Opções de enlace sem internet** (detalhar só na Fase 7):

| Opção | Viabilidade | Observação |
|---|---|---|
| **S3: softAP + caster NTRIP/TCP** com o RTCM vindo da UART1 | **[HIPÓTESE] viável** para poucos clientes. O `esp32-xbee` já implementa caster NTRIP em ESP32 clássico [F9]. | O celular do rover entra no Wi-Fi da base e o SW Maps usa `192.168.4.1:2101`. Alcance limitado (dezenas a ~100 m) [HIPÓTESE]. O código do esp32-xbee é GPL-3.0. |
| **RTKBase** (Raspberry Pi / Orange Pi) | **Comprovado.** Caster local, servidor RTCM, gravação RAW, conversão RINEX e interface web [F10]. | Gera RTCM **a partir do UBX** com `str2str` (MSM7 por padrão em `settings.conf`) [F10], então um único stream serve aos dois usos. |
| Rádio no soquete XBee | Ocupa o soquete da ponte BT | Hardware novo, fora de escopo |
| `pygnssutils gnssserver` em notebook | Socket server/caster NTRIP simples em Python, BSD-3 [F13] | Bom para teste de bancada |

**3. Metadados obrigatórios** (ficha de campo + cabeçalho RINEX):

| Item | Por quê | Onde vai |
|---|---|---|
| Nome e código do ponto/marco; foto do marco e da montagem | Rastreabilidade | `-hm`/`-hn` do convbin |
| **Altura da antena**: vertical ou inclinada, **até qual ponto físico** (ARP) e medida 2× (início e fim) | Erro de altura vai direto para a cota | `-hd h/e/n` (h = marco→ARP) |
| **Modelo da antena + radome no padrão IGS** (20 caracteres) e se há calibração ANTEX | O serviço aplica PCO/PCV pelo nome. Nome diferente do IGS = sem correção [F15, F16]. | `-ha num/tipo` |
| Receptor, número de série e firmware (`UBX-MON-VER`) | Rastreabilidade | `-hr num/tipo/versão` |
| Início e fim (UTC), duração, intervalo, interrupções | Duração mínima e auditoria | ficha + CSV de saúde |
| Obstruções e multicaminho (croqui, máscara) | Qualidade | ficha |
| Operador e empresa | RINEX/auditoria | `-ho` |
| Hash/versão do arquivo de configuração | Reprodutibilidade | ficha |

- **Duração mínima por método:** ver 3.4.3.
- **Ocupação estática:** tripé ou pino fixo; antena nivelada e **orientada ao norte**, por convenção das calibrações [HIPÓTESE].

### 3.4 Pós-processamento (4.4)

#### 3.4.1 Fluxo resumido (detalhe na seção 8)

1. Arquivos `.ubx` horários.
2. Concatenar.
3. `convbin` (RTKLIB-EX 2.5.1, BSD-2) gera RINEX:
   - **2.11** GPS+GLONASS para o IBGE-PPP;
   - **3.04** multi-GNSS para CSRS-PPP e PPK.
4. Compactação Hatanaka + zip/gz.
5. Envio ao serviço ou processamento no RTKLIB.
6. Transformação e altitude: SIRGAS2000 2000,4 + hgeoHNOR2020.

#### 3.4.2 Comparação de serviços e softwares

| Critério | **IBGE-PPP** | **CSRS-PPP (NRCan)** | **PPK RTKLIB-EX + RBMC** |
|---|---|---|---|
| Entrada | RINEX ou Hatanaka, compactado. **Só RINEX ≤ 2.11** [F15b, trecho de busca da Emlid]. **Máx. 20 MB** por arquivo [F15, trecho de busca]. | RINEX 2/3. Arquivo < 100 MB descompactado e < 6 dias [F16, trecho de busca]. Exige conta. | RINEX do rover + RINEX da RBMC + efemérides |
| Constelações usadas | GPS + GLONASS [F15] | GPS + GLONASS. **Galileo PPP-AR só E1/E5a desde a v5 (14-mai-2025)** [F16b]. O F9P-04B rastreia E5b, então o Galileo provavelmente **não é usado** [HIPÓTESE; T15]. | Todas que base e rover tenham em comum (o BDS do F9P aqui é quase só B1I) |
| Duração | Sem mínimo obrigatório; **≥ 4 h recomendadas** [F15, trecho de busca] | Sem mínimo. PPP-AR da v3: ~45 min contra ~3 h na v2 [F16c, trecho de busca]. Emlid recomenda ≥ 1 h, de preferência 24 h a 30 s. | Depende da linha de base: Tabela 3.2 de IBGE (2008) [F22] (valores **[não verificado]** nesta sessão). Literatura com F9P: 1 h → poucos cm H [F20]. |
| Precisão esperada (F9P + patch) | Sem número específico para F9P [não verificado]. O manual 2023 traz gráficos de precisão por tempo [F15]. | **PPP ≥ 2,5 h → "poucos cm"** com ZED-F9P + ANN-MB [F20] | **1 h estático → "poucos cm" H**, com taxa de fix de 80 % [F20] |
| Saída | **SIRGAS2000 época 2000,4 (automático)** + ITRF/IGS20 na época do levantamento [F15] | ITRF2020/IGS20 na época do levantamento, mais referenciais canadenses [F16] | Mesmo referencial da coordenada da base RBMC. Os relatórios descritivos dão SIRGAS2000 [F23]. |
| Compatível com entregas da LuftPlan | **Direto**, sem transformação | Exige transformação ITRF2020(t) → SIRGAS2000(2000,4) com modelo de velocidades. Melhor usar como **controle**: comparar ITRF com ITRF em relação à saída ITRF do IBGE-PPP. | Direto, se a base RBMC estiver em SIRGAS2000 2000,4 |
| Lock-in | Serviço público gratuito com API documentada [F15c] | Serviço gratuito com conta | **Zero:** software aberto e dados abertos |

#### 3.4.3 Melhor abordagem por objetivo (com evidência)

**(a) Georreferenciar a base e depois corrigir levantamentos com RTK local**
- **Sessão:** **≥ 4 h** de RAW a 1 Hz, decimado para 15 s ou 30 s na entrega ao IBGE-PPP.
  - O IBGE recomenda ≥ 4 h [F15].
  - A literatura mostra que o F9P + patch precisa de ≥ 2,5 h para poucos cm em PPP [F20].
  - Para controle: CSRS-PPP sobre o mesmo RINEX.
  - Se houver RBMC a ≤ 20–30 km, processar também PPK estático [HIPÓTESE sobre o limite]. Três soluções independentes.
- **Precisão absoluta da base:** "poucos cm" em H, conforme [F20]. Vertical tipicamente pior. Sem calibração da antena, pode haver viés [HIPÓTESE; T15].
- **Precisão final dos pontos RTK:** erro absoluto da base + erro relativo RTK (1 cm + 1 ppm CEP, especificação u-blox em condições ideais) [F3].

**(b) Georreferenciar voos de drone e SLAM**
- A base grava RINEX a **1 Hz durante todo o voo/levantamento**, mais uma margem de ~10 min antes e depois [HIPÓTESE]. A coordenada da base vem de (a).
- O processamento PPK do drone ou SLAM fica fora do escopo. A contribuição da base é o seu erro absoluto, igual a (a).
- **Alternativa sem base própria:** RINEX 3 de 1 s da RBMC, publicado em arquivos de **15 minutos, a cada 15 minutos** [F14, trecho de busca]. A linha de base costuma ficar longa fora de Curitiba e da região de Guarapuava.

#### 3.4.4 Como obter os dados RBMC

- **15 s, RINEX 2 e 3, arquivos diários:** `https://geoftp.ibge.gov.br/informacoes_sobre_posicionamento_geodesico/rbmc/` (subpasta `dados/ANO/DIA_DO_ANO/`) [F14].
- **1 s, RINEX 3, arquivos de 15 min:**
  - Pasta: `.../rbmc/dados_RINEX3_1s/ANO/DIA/HORA/`.
  - Nome: `XXXX00BRA_R_AAAADDDHHMM_15M_01S_MO.crx.gz`.
  - Desde 01-jan-2020 [F14, trecho de busca].
  - Juntar arquivos: guia oficial IBGE "Passo a passo RINEX 1s" [F14].
- **Coordenadas e antena da estação:** relatórios descritivos `.../rbmc/relatorio/Descritivo_XXXX.pdf` [F23].
- **Estações perto de Curitiba:** **UFPR** (Curitiba) e **PRGU** (Guarapuava). A PRCV (Cascavel) está fora de operação desde 2020 [trecho de busca; confirmar o status atual no portal].
- **Latência dos arquivos diários de 15 s:** [não verificado].
- **Automação:** um script Python (urllib) baixa por data e estação. O IBGE-PPP tem API [F15c].

#### 3.4.5 Como provar que o resultado está correto (validação)

1. **Vértice conhecido:** ocupar um marco SAT/BDG do IBGE com coordenada SIRGAS2000 (ou um ponto próximo à UFPR com coordenada conhecida). Comparar a solução IBGE-PPP com o valor oficial.
2. **Três soluções independentes da mesma sessão:** IBGE-PPP, CSRS-PPP (comparado em ITRF, na mesma época, com a saída ITRF do IBGE-PPP) e PPK com RBMC.
3. **Comparar com o RTK em tempo real** via RBMC-IP, no mesmo ponto e no mesmo dia.
4. **Repetibilidade:** duas sessões em dias diferentes no mesmo marco.
5. **Critérios propostos**, a ajustar com Q9: |ΔH| ≤ 3 cm e |ΔV| ≤ 6 cm entre soluções e contra o vértice; repetibilidade ≤ 2 cm H [HIPÓTESE de tolerância; depende da entrega].

### 3.5 NTRIP e otimizações do uso atual (4.5)

**1. Boas práticas com a RBMC-IP**
- **Caster:** `170.84.40.52:2101` ou `gps-ntrip.ibge.gov.br:2101`, com cadastro gratuito [F24, trecho de busca de documento da Esteio, data não verificada].
- **Mountpoints:** `XXXX0` = RTCM 3.2 e `XXXX1` = RTCM 3.0 [F24]. Conferir a sourcetable atual. O ZED-F9P aceita MSM4/5/7 e também 1004/1012 legadas [F2 Tabela 6]. Preferir o fluxo com MSM, se a sourcetable confirmar [HIPÓTESE].
- **Estação mais próxima:** o erro RTK cresce ~1 ppm (1 cm a cada 10 km) [F3], e a fixação degrada em linhas longas. Manter abaixo de ~20–30 km [HIPÓTESE].
- **GGA:** mountpoints de base única normalmente não exigem GGA (só redes VRS). O SW Maps envia de qualquer forma, sem prejuízo [HIPÓTESE].
- **Perda de conexão:** o F9P mantém o fix até a idade da correção passar de `CFG-NAVSPG-CONSTR_DGNSSTO` (chave existe [F11]; valor padrão [não verificado]). Depois cai para float/3D.
- **Redundância:**
  - Segundo mountpoint cadastrado.
  - Base local (Fase 7).
  - **Sempre gravar RAW na base, e no rover quando possível**, para PPK de contingência.

**2. Melhorias na ponte, apenas as ligadas a RAW e NTRIP**
- **Sem tocar na ponte:** o gravador S3 pode **escutar em paralelo, de forma passiva e em alta impedância, a linha RX da UART2** (RTCM que a ponte entrega ao ZED) numa segunda UART do S3 (o S3 tem 3 [F8]).
  - Grava o RTCM recebido da RBMC-IP.
  - Isso serve para diagnóstico (idade e falhas da correção).
  - Também permite **PPK de contingência**: `convbin -r rtcm3 -tr ...` converte o RTCM da RBMC em RINEX [F1] [HIPÓTESE elétrica; T19].
- **Saúde:** registrar `carrSoln` e numSV do `NAV-PVT` mais o contador de bytes RTCM por minuto no CSV.

**3. Limitações do SW Maps e do GNSS Master**
- **SW Maps "Log to file":** grava **tudo** que chega pela conexão GNSS [F18, trecho de busca].
  - Se RAWX/SFRBX não estiverem habilitados, o arquivo `.ubx` sai **só com NMEA**, e o RTKLIB falha sem SFRBX [F18, fórum SparkFun].
  - RAWX pesa no Bluetooth. A recomendação é usar USB OTG quando possível [F18].
  - Contorno: habilitar RAWX+SFRBX a 1 Hz na UART2 (HPG ≥ 1.30), só na RAM, ou usar o gravador dedicado.
  - O celular precisa ficar conectado durante toda a sessão.
- **iOS:** só BLE [F12 docs]. O SW Maps é a única opção gratuita citada para GNSS externo via BT no iOS [F19, trecho de busca].
- **GNSS Master:** só Android. Tem NTRIP client v1/v2, saída para NTRIP server/TCP e configuração de u-blox [F18b]. Gravação RAW [não verificado].

---

## 4. Matriz comparativa

### 4.1 Arquiteturas de gravação

| Critério | **A1. S3 como gravador dedicado na UART1, ponte intocada (RECOMENDADA)** | A2. S3 substitui a ponte (BLE + SD em paralelo) | B. Firmware aberto existente | C. Dispositivo auxiliar (RTKBase/str2str) | D. Sem gravador dedicado (RBMC como base + SW Maps "Log to file") |
|---|---|---|---|---|---|
| Custo de hardware | ~0 (S3 + SD já comprados; cabos) | ~0 | ~0 | Pi/mini-PC + caixa + energia | 0 |
| Esforço de implementação | Médio: firmware pequeno, só RX + SD + HTTP | Alto: BLE NUS + SD + Wi-Fi + compatibilidade com apps | Alto: nenhum projeto pronto atende S3 + SD + ponte (ver abaixo) | Baixo: imagem pronta + `install.sh` [F10] | Mínimo: só configuração |
| Robustez em campo | Boa com buffer, rotação e sync. SD sensível a queda de energia. | Média: um único chip faz tudo | Varia | **Muito boa** (Linux, `str2str` maduro), mas consome mais energia e o SD/boot do Pi também corrompe | Fraca para base: depende do celular conectado por horas |
| Risco de corrupção do SD | Médio (mitigável) | Médio | Varia | Médio (SD do Pi) | Baixo (memória do celular) |
| RTC ou carimbo de tempo | Não precisa (tempo GNSS) | Não precisa | — | Precisa de hora correta; o RTKBase espera sincronizar antes de gravar (`check_timesync.sh`) [F10] | Não precisa |
| Impacto no SW Maps/GNSS Master | **Nenhum** | Android passa para BLE (relatos de instabilidade [F19]) | Depende | Nenhum (RTKBase fica na base) | Nenhum, mas o celular fica preso à base |
| Dependência de terceiros | ESP-IDF (Apache-2.0) | ESP-IDF + apps | Mantenedor do projeto | RTKBase (AGPL-3.0) + RTKLIB | App fechado (SW Maps) |
| Lock-in | Nenhum | Baixo | Baixo | Nenhum | Médio |
| Melhor uso | **Base em obra, rotina diária** | — | Referência de código | Base fixa e permanente, caster local | Teste rápido e rover |

**B. Firmwares abertos levantados** (estado em 2026-10-04):

| Projeto | Licença | Último commit | Alvo | SD | BT | Observação |
|---|---|---|---|---|---|---|
| `nebkat/esp32-xbee` v0.5.4 [F9] | GPL-3.0 | 2023-01-16 | ESP32 clássico (`CONFIG_IDF_TARGET="esp32"`) | Não | Chaves de configuração existem, mas `CONFIG_BT_ENABLED` está desligado | Firmware oficial do "WiFi NTRIP Master" da ArduSimple. **Caster/cliente/servidor NTRIP** e web. Parado desde 2023. |
| `sparkfun/SparkFun_RTK_Everywhere_Firmware` v3.4 [F12] | MIT (código) | 2026-09-27 | ESP32 clássico (`--fqbn esp32:esp32:esp32`) | Sim (SdFat) | SPP + BLE (NUS) | Muito ativo e completo (log, NTRIP, BLE), mas **amarrado ao hardware SparkFun**. Portar para S3 + RTK2B dá muito trabalho [HIPÓTESE]. Ótima referência de arquitetura. |
| `jancelin/RoverGNSS_RTK_BT_esp32_f9p` [F25] | sem arquivo de licença | 2024-09-18 | ESP32 | Não | SPP | Marcado "experimental code!!!". Ponte BT para SW Maps. |
| `HamishB/uBlox_PPP_logger` [F25] | GPL-3.0 | 2024-07-29 | Arduino Feather M0 | Sim | Não | Gravador de UBX para PPP (CSRS). Não é ESP32. |
| `PaulZC/F9P_RAWX_Logger` / `Eric-FR/F9x_RAWX_Logger` [F25] | sem arquivo na raiz | 2020 / 2024 | Feather M0 | Sim | Não | Referências históricas |
| `hvandermarel/ubxlogger` [F25] | Apache-2.0 | 2025-12-04 | OpenWrt/SBC | Sim | — | Alternativa à C (scripts de gravação + RINEX) |

**Conclusão sobre B:** **nenhum** projeto encontrado atende simultaneamente ESP32-S3, SD e ponte Bluetooth para o RTK2B. O caminho é firmware próprio pequeno (A1), reaproveitando ideias do SparkFun (MIT) e, se necessário, o caster do esp32-xbee (GPL-3.0).

### 4.2 Métodos de pós-processamento

| Critério | IBGE-PPP | CSRS-PPP | PPK RTKLIB-EX + RBMC | RTK NTRIP (referência) |
|---|---|---|---|---|
| Custo | 0 | 0 (conta) | 0 | 0 (dados móveis) |
| Esforço | Baixo | Baixo (+ transformação) | Médio (ajuste de parâmetros) | Baixo |
| Robustez | Alta | Alta | Média (sensível à linha de base e às configurações) | Depende do celular e da internet |
| Precisão esperada (F9P + patch) | Poucos cm H com ≥ 2,5–4 h [F15, F20] | Idem [F20] | Poucos cm H com 1 h em linha curta [F20] | 1 cm + 1 ppm (spec) [F3] |
| Saída em SIRGAS2000 2000,4 | **Direta** | Indireta | Direta (via coordenada da RBMC) | Direta (RBMC-IP) |
| Lock-in | Baixo | Baixo | **Nenhum** | Baixo |
| Risco | RINEX 2.11 + limite de 20 MB | Galileo E5b não usado | GLONASS entre receptores de marcas diferentes; linha longa | Perda de sinal celular |

---

## 5. Arquitetura recomendada (A1)

### 5.1 Diagrama

```
                         ┌──────────── simpleRTK2B (ZED-F9P) ────────────┐
 Antena ── SMA ────────► │                                               │
                         │ UART2 (XBee) ◄──RTCM──┐  ──NMEA──►            │
                         │                       │                       │──► Ponte BT atual (inalterada)
                         │                       └───────────────────────│◄── SW Maps / GNSS Master (NTRIP RBMC-IP)
                         │ UART1 (TX1/RX1, IOREF=3V3)                    │
                         │   TX1 ──UBX: RAWX+SFRBX+NAV-PVT(+MON-COMMS)──┐│
                         │   RX1 ◄──(opcional: CFG-VALSET do S3)───────┐││
                         │ USB "POWER+GPS" ◄──► PC (config/backup)     │││
                         └─────────────────────────────────────────────┼┼┘
                                                                       ││
          ┌──────────────── ESP32-S3 (gravador) ───────────────────────┼┼──┐
          │ UART1 RX ◄──────────────────────────────────────────────────┘│  │
          │ UART1 TX ───────────────────────────────────────────────────┘   │
          │ [opc.] UART2 RX ◄── tap passivo da linha RTCM → ZED RX2 (Fase 7)│
          │  Task RX (alta prioridade) → Ring buffer 64–256 KB (PSRAM)      │
          │  Task SD: blocos 4–16 KB, f_sync 10 s, rotação 1 h, FAT32       │
          │  Task saúde: CSV/min (bytes, ovf, ckerr, latência SD, Vin)      │
          │  softAP + HTTP: /status /files /download                        │
          │ SPI (SDSPI ≤20 MHz) ou SDMMC 1/4 bits ──► módulo microSD        │
          └─────────────────────────────────────────────────────────────────┘
 Alimentação: 5 V comum, GND comum; supervisor de tensão no ADC → fecha arquivo em queda [HIPÓTESE]
```

### 5.2 Configuração do receptor (arquivo versionado `config/zed-f9p/base_logger_v1.cfg`)

As chaves conferidas no **Integration Manual R13** [F2] estão marcadas com ✔. As demais existem no banco de chaves do pyubx2 1.3.8 [F11]. Confirmar no próprio receptor com `CFG-VALGET` (NAK = não suportada).

| Chave | Valor | Motivo | Conf. |
|---|---|---|---|
| `CFG-UART1-BAUDRATE` | 230400 | 12 % de carga a 1 Hz (seção 3.2) | ✔ (UART1 suporta até 921600 [F2]) |
| `CFG-UART1OUTPROT-UBX` | 1 | — | ✔ |
| `CFG-UART1OUTPROT-NMEA` | 0 | Libera banda (o RTKBase faz igual [F10]) | ✔ |
| `CFG-UART1OUTPROT-RTCM3X` | 0 (Fase 7: 1) | — | ✔ |
| `CFG-UART1INPROT-UBX` | 1 | O S3 pode enviar configuração (opcional) | ✔ |
| `CFG-RATE-MEAS` | 1000 (ms) | 1 Hz na base | [F11] |
| `CFG-RATE-NAV` | 1 | — | [F11] |
| `CFG-MSGOUT-UBX_RXM_RAWX_UART1` | 1 | Observações | [F11] (o RTKBase usa [F10]) |
| `CFG-MSGOUT-UBX_RXM_SFRBX_UART1` | 1 | Efemérides; **o RTKLIB falha sem SFRBX** [F18] | [F11] [F10] |
| `CFG-MSGOUT-UBX_NAV_PVT_UART1` | 1 | Tempo para nome de arquivo, estado e diagnóstico | [F11] [F10] |
| `CFG-MSGOUT-UBX_MON_COMMS_UART1` | 10 | Uso do buffer de TX (detecta descarte no receptor) | [F11]; mensagem citada em [F2] |
| `CFG-MSGOUT-UBX_NAV_SAT_UART1` | 10 (opcional) | Diagnóstico de céu e obstrução | [F11] |
| `CFG-NAVSPG-DYNMODEL` | 2 (stationary) na base | O RTKBase usa 2 [F10]. Afeta só a solução, não o RAW [HIPÓTESE]. | [F11] |
| `CFG-SIGNAL-SBAS_ENA` | 0 (opcional) | O RTKBase desliga [F10]. Sem uso em PPP/PPK. | [F11] |
| UART2 / USB | **sem alteração** | Preserva a ponte | — |

**Fase 7 (RTCM local), a acrescentar:**
- `CFG-TMODE-MODE`, `CFG-TMODE-SVIN_MIN_DUR`, `CFG-TMODE-SVIN_ACC_LIMIT` (ou modo FIXED com `CFG-TMODE-POS_TYPE`, `ECEF_*`/`LAT/LON/HEIGHT` e `_HP`) ✔ [F2].
- `CFG-MSGOUT-RTCM_3X_TYPE1005_UART1`, `…1074…`, `…1084…`, `…1094…`, `…1124…`, `…1230…` ✔ [F2].
- ID de estação: `CFG-RTCM-DF003_OUT` ✔ (o manual R13 grafa "DF003_OUT8"; confirmar no ID).

**Camadas:** aplicar primeiro em RAM. Gravar em BBR+Flash **só após o teste T03 e com aprovação** [F2 §3.1].

### 5.3 Fluxo de dados e armazenamento

| Item | Valor (N≈60, 1 Hz) |
|---|---|
| Entrada na UART1 | ~2,8 kB/s RAWX+SFRBX + ~0,1 kB/s NAV-PVT [HIPÓTESE: NAV-PVT tem payload de 92 B] |
| Arquivo | ~10 MB/h → ~240 MB/dia → 32 GB ≈ 130 dias |
| RINEX 3.04 de 1 s | Maior que o UBX [HIPÓTESE]. Decimar para 15–30 s para PPP (IBGE: limite de 20 MB [F15]). |
| RTCM MSM4 (Fase 7) | ~0,7 kB/s para 4 constelações [estimativa minha pelas definições de campo do MSM4; medir] |

---

## 6. Plano de execução (fases curtas)

> **Regra geral:** nada destrutivo sem aprovação explícita do cliente. Isso inclui reflash, formatação de cartão e gravação em Flash/BBR do receptor. Antes de qualquer passo assim: **backup + rollback descritos**.

### Fase 0: Inventário e backup (sem alterar nada)

| Passo | Objetivo | Comandos / ações | Resultado esperado | Validação |
|---|---|---|---|---|
| 0.1 | Responder Q1–Q12 | Fotos da placa, da ponte, do S3 e do módulo SD; `UBX-MON-VER` | Planilha de inventário | Revisão |
| 0.2 | Ferramentas no PC | `pip install pyubx2 pygnssutils pyubxutils pygpsclient`; RTKLIB-EX v2.5.1 [F1] | `ubxsave -h` e `convbin` respondem | — |
| 0.3 | **Backup completo da configuração do ZED** pelo USB | `ubxsave --port COMx --baudrate 38400 --outfile zed_backup_AAAAMMDD.ubx` (sintaxe exata: ver `ubxsave -h`) [F13] | Arquivo de backup | **T02** |
| 0.4 | Backup do firmware/configuração da ponte BT | Anotar nome BT, PIN, baud e app. **Não regravar.** | Ficha | — |
| 0.5 | Repositório git `luftplan-gnss` | `config/`, `firmware/`, `scripts/`, `fichas/` | Versionamento ativo | — |

**Rollback geral do receptor:**
- `ubxload` com o arquivo de backup (carrega em RAM [F13]), conferir com `ubxcompare`.
- Se a configuração foi gravada em Flash: regravar o backup em Flash, **com aprovação**.

### Fase 1 (opcional, ganho rápido): validar a cadeia com SW Maps "Log to file"

| Passo | Objetivo | Comandos / ações | Resultado | Validação |
|---|---|---|---|---|
| 1.1 | Habilitar RAWX e SFRBX na UART2 **só na RAM** | `CFG-VALSET` layer=RAM: `CFG-UART2OUTPROT-UBX=1`, `CFG-MSGOUT-UBX_RXM_RAWX_UART2=1`, `CFG-MSGOUT-UBX_RXM_SFRBX_UART2=1` (exige HPG ≥ 1.30 [F2]) | A ponte passa a carregar UBX junto com NMEA | O SW Maps mantém fix/RTK |
| 1.2 | Gravar 1 h no SW Maps | Bluetooth GNSS → marcar "Log to file" [F18] | `.ubx` no celular | **T09** (convbin gera obs e nav) |
| 1.3 | Reverter | Desligar e religar o receptor (a RAM é apagada) | Configuração original | `ubxcompare` contra o backup = sem diferenças |

### Fase 2: Gravador em bancada (sem o receptor)

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 2.1 | Benchmark do SD | Exemplo `examples/storage/perf_benchmark` do ESP-IDF [F8]. **Atenção: formata o cartão.** Usar cartão dedicado, com aprovação. | Taxa e latência por modo (SDSPI/SDMMC) | **T05** |
| 2.2 | Firmware mínimo: UART RX → ring → SD | ESP-IDF 5.x; tarefas conforme 3.2; pinos SD fora de GPIO35–37 [HIPÓTESE PSRAM octal]; UART1 do S3 em GPIOs livres (evitar 0, 3, 45, 46 [HIPÓTESE strapping] e 43/44 do console) | Arquivos `.ubx` horários | **T07** (bit-exato) |
| 2.3 | Saúde e status | CSV/min + softAP `/status` | JSON com contadores | **T13** |
| 2.4 | Queda de energia | Cortes aleatórios | FS montável, perda ≤ 10 s + buffer | **T08** |

### Fase 3: Configuração do receptor para o gravador

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 3.1 | Aplicar `base_logger_v1.cfg` **em RAM** pelo USB | PyGPSClient ou script pyubx2 (`UBXMessage.config_set(layers=RAM, ...)`) [F11] | UART1 a 230400 emitindo UBX | **T03**, **T04** |
| 3.2 | Validar a UART1 num conversor USB-serial 3,3 V | `gnssstreamer --port COMy --baudrate 230400` (contagem por mensagem) [F13] | RAWX/SFRBX/NAV-PVT a 1 Hz | **T04** |
| 3.3 | Persistir (**com aprovação**) | `CFG-VALSET` layers=BBR+Flash com o mesmo arquivo | Sobrevive a desligar e religar | **T03b** |

### Fase 4: Integração receptor + gravador

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 4.1 | Fiação | ZED **TX1 → S3 RX**; (opcional) S3 TX → ZED RX1; **IOREF = 3,3 V** [F5]; GND comum | — | Multímetro: IOREF = 3,3 V |
| 4.2 | Gravação de 24 h em céu aberto | Ponte BT ligada e em uso normal | 24 arquivos de ~10 MB | **T10**, **T11** |
| 4.3 | Convivência com o fluxo atual | SW Maps + NTRIP RBMC-IP durante a gravação | Fix RTK igual ao de antes | **T12** |

### Fase 5: Campo e validação geodésica

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 5.1 | Sessão de 6 h num vértice SIRGAS2000 conhecido | Ficha de metadados completa | `.ubx` + ficha | — |
| 5.2 | Processar IBGE-PPP, CSRS-PPP e PPK (UFPR) | Seção 8 | Três soluções | **T14**, **T15** |
| 5.3 | Comparar com o RTK NTRIP no mesmo ponto | SW Maps, média de 3 min, 3 vezes | Coordenadas RTK | **T14** |
| 5.4 | Repetir em outro dia | — | — | **T17** |

### Fase 6: Pipeline de pós-processamento automatizado (Python)

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 6.1 | Script `ubx2rinex.py`: concatena, roda `convbin` com metadados da ficha (YAML), decima e compacta | Seção 8 | RINEX 2.11 (IBGE) e 3.04 | **T09** |
| 6.2 | Download RBMC por data e estação | URL da seção 3.4.4 | RINEX da base | Hash e tamanho |
| 6.3 | Relatório comparativo (ΔE, ΔN, ΔU) | pandas/pyproj | CSV/PDF | Critérios de 3.4.5 |

### Fase 7 (opcional): RTCM local sem internet

| Passo | Objetivo | Ação | Resultado | Validação |
|---|---|---|---|---|
| 7.1 | RTCM na UART1 junto com UBX | `CFG-UART1OUTPROT-RTCM3X=1` + MSM4/1005/1230 [F2] | Stream misto | **T20** (o convbin ignora o RTCM no `-r ubx`) |
| 7.2 | Caster ou TCP no softAP do S3 | Demultiplexar frames RTCM (0xD3) para os clientes | O rover recebe correções | **T21** |
| 7.3 | Comparar com o RTKBase num Pi | — | Escolha da solução definitiva | Matriz 4.1 |

---

## 7. Plano de testes individuais (bancada → campo; uma variável por teste)

| ID | Objetivo | Variável única | Procedimento | Critério de aceite |
|---|---|---|---|---|
| T01 | Inventário de firmware | — | Ler `UBX-MON-VER` | Versão registrada. Se < HPG 1.30, pular a Fase 1. |
| T02 | Backup e restauração idênticos | Configuração | `ubxsave` → `ubxload` → `ubxcompare` [F13] | 0 diferenças |
| T03 | Configuração em RAM aplicada | Arquivo `.cfg` | `CFG-VALGET` de cada chave | 100 % das chaves com o valor esperado; nenhum NAK |
| T03b | Persistência (após aprovação) | Camada Flash | Desligar e religar; repetir T03 | Idem |
| T04 | Taxa e banda na UART1 | Baud = 230400 | 1 h com `gnssstreamer` no PC; `MON-COMMS` | RAWX = SFRBX-ok = NAV-PVT = 3600 ± 1; 0 erros de checksum; pico do buffer TX < 50 % [HIPÓTESE de limiar] |
| T05 | Desempenho do SD (cartão dedicado) | Modo SDSPI × SDMMC (um por vez) | `perf_benchmark` [F8] | Escrita sustentada ≥ 100 kB/s (> 30× a demanda) [HIPÓTESE de limiar] |
| T06 | Pior latência de escrita | Tempo (24 h) | Registrar a latência máxima de `write` no CSV | Latência máx. < tempo coberto pelo buffer (64 KB ÷ 2,8 kB/s ≈ 23 s) com margem 5× |
| T07 | Gravação bit-exata | Fonte simulada | O PC reproduz um `.ubx` real a 230400 (pyserial) durante 6 h | SHA-256 do enviado = SHA-256 dos arquivos concatenados; 0 eventos `UART_FIFO_OVF`/`BUFFER_FULL` |
| T08 | Queda de energia | Cortes | 20 cortes aleatórios durante a gravação | Cartão monta 20/20; perda ≤ 10 s + buffer por corte; arquivos anteriores íntegros |
| T09 | Conversão para RINEX | Arquivo | `convbin` (seção 8) | `.obs` e `.nav` gerados; épocas = duração × taxa (± 0,1 %); sem saltos |
| T10 | Gravação de 24 h com o receptor | Tempo | Fase 4.2 | 24 arquivos; CSV sem overflow; T09 aprovado para todos |
| T11 | Nome por tempo GNSS | Partida a frio | Ligar sem céu e depois liberar | Arquivo pré-fix renomeado; horário UTC ± 1 s do NAV-PVT |
| T12 | Convivência com a ponte BT | Gravador ligado × desligado | Mesmo ponto: tempo até fix RTK e estabilidade no SW Maps (Android e iOS) | Sem diferença perceptível: tempo até fix ± 20 %; 0 desconexões BT em 1 h |
| T13 | Status por Wi-Fi sem afetar a gravação | softAP ativo | Baixar arquivos pelo HTTP durante a gravação | T07 continua bit-exato; download funcional |
| T14 | Exatidão em vértice | Método (um por vez) | IBGE-PPP, CSRS-PPP, PPK UFPR e RTK NTRIP no vértice | Contra o vértice: \|ΔH\| ≤ 3 cm, \|ΔV\| ≤ 6 cm (ajustar com Q9) |
| T15 | Efeito da antena e do Galileo | Nome da antena no RINEX / Galileo on-off | Reprocessar no CSRS com "NONE" × outro nome; conferir no relatório quais sinais foram usados | Diferença registrada; decide se a calibração (fora do escopo) é necessária |
| T16 | RAW independe do TMODE | Modo da base | 1 h survey-in × 1 h fixo; conferir RAWX | Mensagens RAWX presentes e completas nos dois modos |
| T17 | Repetibilidade e translação Δ | Dia | Duas sessões; aplicar Δ = PPP − survey-in em pontos RTK de teste | Repetibilidade ≤ 2 cm H; pontos transladados batem com o vértice dentro de T14 |
| T18 | (Só se A2) SW Maps Android por BLE com o S3 | Protocolo BLE | 2 h conectado com NTRIP | 0 desconexões; RTCM chega ao receptor |
| T19 | Tap passivo do RTCM | Ligação do tap | Comparar o fix RTK com e sem tap; converter o RTCM gravado | RTK sem degradação; `convbin -r rtcm3` gera RINEX da RBMC |
| T20 | Stream misto UBX+RTCM | Protocolos na UART1 | `convbin -r ubx` no arquivo misto | RINEX idêntico ao do arquivo só-UBX |
| T21 | Caster local no S3 | Clientes (1 → 3) | Rover via SW Maps no softAP | Idade da correção < 2 s; RTK fixo em ≥ 95 % do tempo a 30 m |

---

## 8. Fluxo de pós-processamento completo

### 8.1 Preparação

```bash
# 1) Concatenar os arquivos horários da sessão (UBX é stream; concatenar é seguro [HIPÓTESE, validado em T09])
cat LPB1_20261004_1[0-5]00.ubx > LPB1_20261004_sessao.ubx      # Windows: copy /b a.ubx+b.ubx saida.ubx
```

### 8.2 `convbin` (RTKLIB-EX 2.5.1 [F1])

As opções foram conferidas no help do código-fonte. Pontos de atenção:
- `-hd` é **h/e/n**.
- `-ha` e `-hr` usam "/" como separador e o parser **ignora campos vazios** (`strtok`). **Sempre preencher o número** antes do tipo, senão o tipo cai no campo errado.
- `-y` pode ser repetido.

**RINEX 3.04 multi-GNSS** (CSRS-PPP, PPK, arquivo):
```bash
convbin -r ubx -v 3.04 -od -os -oi -ot -ol \
  -hm LPB1 -hn 0001 -ho "Operador/LuftPlan" \
  -hr "SN123/U-BLOX ZED-F9P/HPG 1.32" \
  -ha "0001/ANTTYPE_IGS_20CHARS" \
  -hd "1.5230/0/0" \
  -d rinex3 LPB1_20261004_sessao.ubx
```

**RINEX 2.11 GPS+GLONASS, 30 s** (IBGE-PPP: só RINEX ≤ 2.11 e ≤ 20 MB [F15]):
```bash
convbin -r ubx -v 2.11 -ti 30 -y E -y C -y J -y S -y I -od -os -oi -ot -ol \
  -hm LPB1 -hn 0001 -ho "Operador/LuftPlan" \
  -hr "SN123/U-BLOX ZED-F9P/HPG 1.32" -ha "0001/ANTTYPE_IGS_20CHARS" -hd "1.5230/0/0" \
  -d rinex2 LPB1_20261004_sessao.ubx
```

Notas:
- Opções de receptor u-blox existem (ex.: `-ro "-TADJ=…"`, `-STD_SLIP=`, `-MAX_STD_CP=`) [F1 `ublox.c`]. **Não usar** sem motivo testado.
- A antena sem nome IGS fica sem PCO/PCV no IBGE-PPP [F15]. Registrar o nome real ou "NONE" e testar o efeito (T15).
- Compactação Hatanaka (`rnx2crx`) + zip/gz é recomendada pelos dois serviços [F15, F16] (licença do RNXCMP: [não verificado]).

### 8.3 IBGE-PPP (principal)

1. Enviar o `.zip` com o RINEX 2.11 pela web ou pela API [F15c].
2. Informar o modelo e a altura da antena, ou usar "Não alterar RINEX".
3. Ler o relatório:
   - SIRGAS2000 (2000,4): φ, λ, h e desvios-padrão;
   - ITRF/IGS20 na época do levantamento.
4. **Altitude normal:** h → H com **hgeoHNOR2020** (substitui o MAPGEO2015) [F17]. O RTKLIB não tem esse modelo nas opções `out-geoid` [F1], então manter a saída elipsoidal e converter com a ferramenta do IBGE.
5. UTM fuso 22S: pyproj (EPSG:31982 = SIRGAS 2000 / UTM 22S).

### 8.4 CSRS-PPP (controle)

- Enviar o RINEX 3.04 ou o 2.11, modo estático, saída ITRF.
- Comparar **ITRF × ITRF** (mesma época) com a saída ITRF do IBGE-PPP. Isso evita depender da transformação para SIRGAS2000.
- Se for preciso SIRGAS2000 a partir do CSRS: aplicar os parâmetros IGS20 → SIRGAS2000 publicados pelo IBGE + modelo de velocidades. A busca citou Tx = −0,48 cm, Ty = −0,19 cm, Tz = −0,69 cm e escala de 0,69 ppb, com fonte não identificada [trecho de busca; **não usar sem conferir no documento oficial do IBGE**].

### 8.5 PPK estático com RTKLIB-EX (`rnx2rtkp` ou RTKPOST)

Ponto de partida; os nomes das opções foram conferidos em `src/options.c` [F1]. Validar em T14.

```ini
pos1-posmode       =static        # 3
pos1-frequency     =l1+l2         # 2
pos1-elmask        =15
pos1-navsys        =13            # 1 gps + 4 glo + 8 gal (BDS do F9P aqui é quase só B1I)
pos1-ionoopt       =brdc          # linha curta (<~20 km); linha longa: testar est-stec [HIPÓTESE]
pos1-tropopt       =saas          # linha longa: est-ztd [HIPÓTESE]
pos1-sateph        =brdc
pos2-armode        =continuous    # estático
pos2-gloarmode     =off           # receptores de marcas diferentes (RBMC × u-blox); testar autocal [HIPÓTESE]
ant2-postype       =xyz           # coordenada SIRGAS2000 do relatório descritivo da estação [F23]
ant2-anttype       =<nome IGS da antena da RBMC, conforme relatório>
ant1-anttype       =<nome da antena da LuftPlan, ou vazio se não calibrada>
file-rcvantfile    =igs20.atx     # ANTEX IGS20 (o IBGE usa IGS20.ATX após a semana GPS 2238 [F15])
out-solformat      =llh
out-height         =ellipsoidal
```

- Comando: `rnx2rtkp -k ppk_static.conf -o saida.pos rover.obs base_rbmc.obs rover.nav base.nav`.
- A coordenada resultante fica no referencial da coordenada informada para a base: SIRGAS2000 2000,4 se vier do relatório da RBMC.

---

## 9. Riscos e pontos de falha

| # | Risco | Prob. | Impacto | Mitigação | Teste |
|---|---|---|---|---|---|
| R1 | Corrupção do SD por queda de energia | Média | Perda de sessão | Rotação horária, `f_sync` a cada 10 s, cartão "high endurance", supervisor de tensão, fonte estável | T08 |
| R2 | Perda de bytes na UART (overflow) | Baixa a 1 Hz | Buracos no RINEX | Ring de 64–256 KB, tarefa de RX com prioridade alta, contadores de overflow, 230400 bps | T04, T07 |
| R3 | O receptor descarta mensagens (banda) | Baixa | Épocas faltando | Dimensionamento da seção 3.2; `MON-COMMS` | T04 |
| R4 | Nível lógico errado (IOREF desligado) | Média | Comunicação falha ou dano | IOREF = 3,3 V [F5]; medir antes de ligar | 4.1 |
| R5 | Pinos do S3 conflitantes (PSRAM octal, strapping) | Média | Boot falha ou SD instável | Pinos fora de 35–37, 0, 3, 45, 46 [HIPÓTESE]; conferir o datasheet do módulo | 2.2 |
| R6 | Antena sem calibração ANTEX | Alta (se ANN-MB) | Viés vertical de cm [HIPÓTESE] | Registrar no RINEX; avaliar em T15; calibração ou troca fora do escopo | T15 |
| R7 | IBGE-PPP rejeita o arquivo (versão, tamanho) | Média | Atraso | RINEX 2.11, decimar para 30 s, só GPS+GLONASS, compactar | T09 |
| R8 | Galileo e BeiDou do F9P pouco aproveitados em PPP | Alta | Menos satélites → convergência mais lenta | Sessões ≥ 4 h; PPK com GAL quando a base RBMC tiver E5b [HIPÓTESE] | T15 |
| R9 | Coordenada da base em survey-in deslocando todo o RTK | Alta se esquecido | Erro sistemático de metro | Procedimento Δ (3.3) ou modo fixo com coordenada PPP | T17 |
| R10 | Gravar em Flash uma configuração errada | Baixa | Ponte ou rover deixam de funcionar | Só RAM até validar; backup com `ubxsave`; rollback com `ubxload` | T02, T03 |
| R11 | GLONASS AR entre marcas diferentes no PPK | Média | Fix errado | `gloarmode=off` ou `autocal`; comparar com PPP | T14 |
| R12 | Altura ou ponto de medição da antena errado | Média | Erro vertical direto | Ficha com medida dupla e foto; ARP definido | T14 |
| R13 | Dependência de serviços online (IBGE, NRCan) | Baixa | Atraso | O PPK com RTKLIB é 100 % local; manter os dois caminhos | — |
| R14 | Licença GPL ao reaproveitar código do esp32-xbee | Baixa | Obrigação de publicar o código ao distribuir o firmware | Uso interno ou reescrita; preferir referências MIT/Apache | — |

---

## 10. Fontes (acesso em 2026-10-04)

**Documentos u-blox e ArduSimple**
- **F2:** u-blox, *ZED-F9P Integration Manual*, UBX-18010802 **R13 (30-ago-2023)**. Cobre HPG 1.12, 1.13, 1.32 e L1L5 1.40. Lido na cópia do repositório `sparkfun/SparkFun_RTK_EVK` (`docs/assets/component_documentation/`). Original: https://content.u-blox.com/sites/default/files/ZED-F9P_IntegrationManual_UBX-18010802.pdf
- **F3:** u-blox, *ZED-F9P-02B Data Sheet*, UBX-21023276 **R03 (24-mar-2023)**, HPG 1.13. Mesmo repositório. A versão 04B (UBX-21044850) não foi lida: https://cdn.sparkfun.com/assets/a/8/5/f/5/ZED-F9P-04B_DataSheet_UBX-21044850.pdf
- **F4:** u-blox, release notes HPG 1.51 (UBXDOC-963802114-13110, R01 01-nov-2024) e HPG 1.50 (R01 05-jul-2024) [trecho de busca]: https://content.u-blox.com/sites/default/files/documents/ZED-F9P-FW100HPG151_RN_UBXDOC-963802114-13110.pdf
- u-blox, *F9 HPG 1.32 Interface Description* UBX-22008968, protocolo 27.31. **Não foi possível ler (bloqueado).** As chaves foram conferidas via F2 e F11: https://cdn.sparkfun.com/assets/learn_tutorials/2/7/8/1/u-blox-F9-HPG-1.32_InterfaceDescription_UBX-22008968.pdf
- **F5:** ArduSimple, *User Guide simpleRTK2B Budget*, *simpleRTK2B V3 hookup guide* e *User Guide simpleRTK2B Pro* [trechos de busca]: https://ardusimple.com/?p=2074 · https://ardusimple.com/simplertk2b-v3-hookup-guide · https://www.ardusimple.com/?p=36066
- **F5b:** ArduSimple, *ZED-F9P firmware update with simpleRTK2B* [trecho de busca]: https://www.ardusimple.com/zed-f9p-firmware-update-with-simplertk2b/
- **F6:** ArduSimple, *Bluetooth / BT+BLE Bridge* [trecho de busca]: https://www.unmannedsystemstechnology.com/company/ardusimple/bluetooth-ble-bridge/ · https://ardusimple.com/?p=41835

**Espressif / ESP32-S3**
- **F7:** Espressif, ESP-IDF `docs/en/api-guides/bt-architecture/overview.rst` (tabela "Chip Bluetooth Capability"), repositório `espressif/esp-idf`, commit **4d59230 (2026-09-25)**.
- **F8:** Mesmo commit do ESP-IDF:
  - `components/soc/esp32s3/include/soc/soc_caps.h`: BLE 5.0, SDMMC 2 slots, UART×3, FIFO 128 B, UHCI;
  - `components/fatfs/src/ffconf.h` (`FF_FS_EXFAT 0`) e `components/fatfs/Kconfig` (`FATFS_IMMEDIATE_FSYNC`);
  - `docs/en/api-reference/peripherals/sdmmc_host.rst` (20/40 MHz), `sdspi_host.rst` (SDSPI ≤ `SDMMC_FREQ_DEFAULT`), `uart.rst` (eventos `UART_FIFO_OVF`/`UART_BUFFER_FULL`);
  - `docs/en/api-guides/coexist.rst`;
  - `examples/storage/perf_benchmark/README.md`.
  - https://github.com/espressif/esp-idf

**Firmwares e software abertos**
- **F1:** rtklibexplorer, **RTKLIB-EX** (ex-demo5) **v2.5.1**, commit 81943c1 (2026-09-28), licença BSD-2. Lidos `app/consapp/convbin/convbin.c` (help/opções), `src/options.c` e `src/rcv/ublox.c`: https://github.com/rtklibexplorer/RTKLIB
- **F9:** nebkat, **esp32-xbee** v0.5.4, commit 8fc008a (2023-01-16), GPL-3.0: https://github.com/nebkat/esp32-xbee
- **F10:** Stefal, **RTKBase** v2.7.0, commit 2ce7ce0 (2025-12-20), AGPL-3.0. `README.md`, `settings.conf.default`, `run_cast.sh`, `receiver_cfg/U-Blox_ZED-F9P_rtkbase.cfg`: https://github.com/Stefal/rtkbase
- **F11:** semuconsulting, **pyubx2 1.3.8** (2026-09-28), BSD-3. `ubxtypes_configdb.py` (IDs de chave), `ubxtypes_get.py` (estrutura RAWX/SFRBX) e `ubxtypes_set.py` (camadas do CFG-VALSET): https://github.com/semuconsulting/pyubx2
- **F12:** SparkFun, **RTK Everywhere Firmware v3.4**, commit 4d84581 (2026-09-27), código MIT. `Firmware/Dockerfile` (FQBN esp32:esp32:esp32), `Tasks.ino` (ring buffer) e `Bluetooth.ino` (UUIDs NUS). "F12 docs" = `docs/connecting_bluetooth.md`, `docs/quickstart-*.md` e `docs/gis_software_ios.md` (iOS sem SPP): https://github.com/sparkfun/SparkFun_RTK_Everywhere_Firmware
- **F13:** semuconsulting, **pygnssutils 1.2.8** (`gnssstreamer`, `gnssserver`, `gnssntripclient`), **pyubxutils 1.0.6** (`ubxsave`, `ubxload`, `ubxcompare`) e **PyGPSClient 1.7.7**, todos BSD-3 (PyPI e GitHub): https://github.com/semuconsulting/pygnssutils · https://github.com/semuconsulting/PyGPSClient
- **F25:** Outros repositórios, com data do último commit:
  - HamishB/uBlox_PPP_logger (2024-07-29, GPL-3.0);
  - hvandermarel/ubxlogger (2025-12-04, Apache-2.0);
  - jancelin/RoverGNSS_RTK_BT_esp32_f9p (2024-09-18);
  - PaulZC/F9P_RAWX_Logger (2020-01-21);
  - Eric-FR/F9x_RAWX_Logger (2024-05-03);
  - sparkfun/OpenLog_Artemis_GNSS_Logger (2024-10-15).
  - Referência adicional: rtklibexplorer, "Building a simple u-blox F9P data logger with a SparkFun OpenLog board" (2019): https://rtklibexplorer.wordpress.com/2019/10/25/building-a-simple-u-blox-f9p-data-logger-with-a-sparkfun-openlog-board/

**IBGE, NRCan e RBMC**
- **F14:** IBGE, RBMC: dados de 1 s em RINEX 3 (15 min) e guia "Passo a passo RINEX 1s" [trecho de busca]: https://www.ibge.gov.br/geociencias/informacoes-sobre-posicionamento-geodesico/rede-geodesica/16258-rede-brasileira-de-monitoramento-continuo-dos-sistemas-gnss-rbmc.html · https://geoftp.ibge.gov.br/informacoes_sobre_posicionamento_geodesico/rbmc/ · https://www.ibge.gov.br/np_download/novoportal/geociencias/rbmc/Passo_a_passo_RINEX_1s.pdf
- **F15:** IBGE, *Manual do Usuário IBGE-PPP*, versão **junho/2023** (IGS20; limite de 20 MB; ANTEX IGS20 após a semana GPS 2238; recomendação de tempo) [trechos de busca]: https://biblioteca.ibge.gov.br/visualizacao/livros/liv101677.pdf · https://www.ibge.gov.br/geociencias/informacoes-sobre-posicionamento-geodesico/servicos-para-posicionamento-geodesico/16334-servico-online-para-pos-processamento-de-dados-gnss-ibge-ppp.html
- **F15b:** Emlid, *IBGE-PPP workflow* ("RINEX only of version 2.11 or older"; GPS+GLONASS) [trecho de busca]: https://docs.emlid.com/reachrs2/tutorials/post-processing-workflow/ibge-workflow
- **F15c:** IBGE, *API do Serviço IBGE-PPP*: https://servicodados.ibge.gov.br/api/docs/ppp?versao=1
- **F16:** NRCan, *CSRS-PPP* (limites de arquivo; antenas IGS/NGS; doc. "CSRS-PPP v5 PCO") [trechos de busca]: https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/sample_doc_files/CSRS-PPP_v5_PCO_EN.pdf
- **F16b:** NRCan, *CSRS-PPP Updates*: v5 em 14-mai-2025 com Galileo PPP-AR E1/E5a; Repro3 em 29-jan-2026 [trecho de busca]: https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/ppp-update.php
- **F16c:** S. Banville, *CSRS-PPP Version 3: Tutorial* (PPP-AR ~45 min contra ~3 h) [trecho de busca]: https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/sample_doc_files/NRCan_CSRS-PPP-v3_Tutorial_EN.pdf
- **F17:** IBGE, *hgeoHNOR2020* (substitui o MAPGEO2015) [trecho de busca]: https://www.ibge.gov.br/en/geosciences/digital-surface-models/digital-surface-models/31312-model-to-convert-altitudes-hgeohnor2020modeloconversaoaltitudesgeometricasgnss-datumverticalsgb.html
- **F22:** IBGE, *Recomendações para Levantamentos Relativos Estáticos – GPS* (abr/2008), Tabela 3.2: https://www.ibge.gov.br/geociencias/metodos-e-outros-documentos-de-referencia/outros-documentos-tecnicos-geo/16376-recomendacoes-para-levantamentos-relativos-estaticos-gps.html
- **F23:** IBGE, relatórios descritivos das estações RBMC (ex.: `Descritivo_PECA.pdf`): https://geoftp.ibge.gov.br/informacoes_sobre_posicionamento_geodesico/rbmc/relatorio/
- **F24:** Esteio, *RBMC-IP em tempo real* (caster, porta e mountpoints 0/1; data não verificada) [trecho de busca]: https://www.esteio.com.br/downloads/pdf/RBMC%20IP.pdf · IBGE RBMC-IP: https://www.ibge.gov.br/geociencias/informacoes-sobre-posicionamento-geodesico/servicos-para-posicionamento-geodesico/16332-rbmc-ip-rede-brasileira-de-monitoramento-continuo-dos-sistemas-gnss-em-tempo-real.html

**Aplicativos de campo**
- **F18:** Fórum SparkFun, "SW Maps, SparkFun Surveyor, Invalid *.UBX log files" (Log to file; RAWX/SFRBX; RTKLIB exige SFRBX) [trecho de busca]: https://community.sparkfun.com/t/sw-maps-sparkfun-surveyor-invalid-ubx-log-files/45251 · Mettatec, documentação do SW Maps: https://docs.mettatec.com/swmaps/
- **F18b:** ArduSimple, *Introducing GNSS Master app* e rtkdata, *GNSS Master* [trecho de busca]: https://www.ardusimple.com/tutorials/introducing-gnss-master-app/ · https://docs.rtkdata.com/integration-hub/ntrip-clients-and-field-software/gnss-master
- **F19:** rtkdata, *SW Maps*, e fórum Quectel/SparkFun sobre BLE (relato de desconexões no Android com a ponte BLE da ArduSimple) [trecho de busca]: https://docs.rtkdata.com/integration-hub/ntrip-clients-and-field-software/sw-maps

**Artigos científicos**
- **F20:** Wielgocka, N.; Hadas, T.; Kaczmarek, A.; Marut, G. *Feasibility of Using Low-Cost Dual-Frequency GNSS Receivers for Land Surveying*. **Sensors 2021, 21(6), 1956**, doi:10.3390/s21061956. ZED-F9P + ANN-MB-00; estático 1 h → poucos cm H, fix 80 %; PPP ≥ 2,5 h [trecho de busca/resumo]: https://elib.uni-stuttgart.de/bitstream/11682/12776/1/sensors-21-01956.pdf
- **F21:** Sobre os sinais do BDS-3 (B1I, B3I, B1C, B2a, B2b; sem B2I) [trecho de busca]: https://www.spirent.com/blogs/beidou-phase-3-signals
- Complementares (antena patch × geodésica; não usados para números de PPP):
  - *Evaluation of Low-Cost GNSS Receiver under Demanding Conditions in RTK Network Mode*, Sensors 2021 (PMC8401267);
  - *Low-Cost GNSS and Real-Time PPP… ZED-F9P*, Remote Sensing 2022, 14(20), 5100.

---

## 11. Observações fora de escopo (só listadas)

1. Calibração ou troca da antena por um modelo com calibração IGS/NGS (impacto direto em PPP e altitude).
2. Rádio dedicado (LoRa/XBee 900 MHz) para RTCM base → rover com alcance de quilômetros.
3. Usar o USB OTG do ESP32-S3 como **host USB** do ZED-F9P (CDC-ACM) em vez da UART1.
4. Um segundo receptor como rover dedicado, com gravação RAW própria, para PPK de drone e SLAM.
5. RTKBase permanente no escritório (base fixa da LuftPlan) publicando no RTK2go ou num caster próprio.
6. Atualizar o firmware do ZED-F9P (HPG 1.32 → versão mais recente aplicável ao módulo). Destrutivo; exige aprovação e plano de rollback próprio.
7. Integrar o pipeline Python ao fluxo de entregas BIM/GIS (metadados ISO 19115 dos levantamentos).
8. Avaliar a conformidade com a Norma Técnica do INCRA para georreferenciamento de imóveis rurais, se houver esse tipo de serviço.
