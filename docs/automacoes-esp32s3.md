# Automações com o ESP32-S3: campo e escritório

**Cliente:** LuftPlan Soluções Integradas a Projetos
**Data:** 2026-10-04 · **Versão:** 1.0
**Complementa:** [`plano-gravacao-raw-rtk2b.md`](plano-gravacao-raw-rtk2b.md) (doravante "Plano v1")

**Convenções:** as mesmas do Plano v1.
- **[HIPÓTESE]**: inferência ainda não comprovada.
- **[não verificado]**: sem fonte primária nesta sessão.
- **[trecho de busca]**: fato retirado de resumo de buscador.
- **[F#]**: fontes do Plano v1.
- **[G#]**: fontes novas, na seção 11.

**Premissa que não muda:** a Arquitetura **A1** do Plano v1. O S3 é ligado à **UART1** do ZED-F9P e a ponte Bluetooth atual continua na **UART2**, intocada. Toda automação abaixo é **opcional, incremental e testável isoladamente**. Nenhuma delas exige reflash do receptor ou gravação em Flash/BBR.

---

## 1. Resumo executivo

O S3 deixa de ser só gravador e passa a ser o **"assistente de campo" da base**. As automações de maior retorno, na ordem de implementação:

1. **Sessão de ocupação pelo celular (F2).** Uma página web (portal cativo no Wi-Fi do S3) recebe ponto, altura medida duas vezes, antena e operador, e grava `session.json` ao lado do UBX. Isso **elimina a ficha em papel** e alimenta automaticamente os metadados do `convbin` no escritório.
2. **Painel de saúde com alarmes (F5).** Mostra tempo ocupado contra as metas de PPP (2,5 h / 4 h), satélites, lacunas, antena e jamming (`UBX-MON-RF`), espaço no SD e bateria.
3. **Descarga automática no escritório (O1/O2).**
   - Por cabo USB, o S3 aparece como **pendrive** (TinyUSB MSC) [G1].
   - Pelo Wi-Fi do escritório, um script baixa os arquivos novos com verificação **SHA-256** [G2, G3].
4. **Pipeline sem intervenção (O3/O4).** UBX → RINEX → **API do IBGE-PPP** [G5] → resultado em SIRGAS2000 → banco de sítios. Em paralelo, PPK com download automático da RBMC pela **API RBMC do IBGE** [G6].
5. **Ciclo fechado (F3).** As coordenadas PPP voltam ao S3 (`sites.csv`). Na próxima visita, a base **reconhece o marco** e se configura em modo fixo, **sempre com confirmação humana**.
6. **Conexões alternativas ao Bluetooth (F7/F11).**
   - O S3 pode ser **cliente NTRIP** via hotspot do celular.
   - Pode servir **NMEA por TCP** ao SW Maps, que aceita "NMEA over IP" no Android e no iOS [G7, trecho de busca].
   - Isso dá redundância ao fluxo BT atual, mas **exige a regra de uma única fonte de RTCM** (seção 4.2).

Quase tudo tem **exemplo oficial do ESP-IDF** como ponto de partida: file server com SD, portal cativo, MSC, ESP-NOW, OTA com rollback [G1–G4]. A biblioteca SparkFun u-blox GNSS v3 (MIT) tem exemplos de gravação de RAWX em SD no ESP32, cliente/servidor NTRIP e base em modo fixo [G8].

---

## 2. Perguntas adicionais (com suposição adotada)

| # | Pergunta | Por que importa | Suposição |
|---|---|---|---|
| Q13 | Modelos dos celulares (Android/iPhone) e se fazem hotspot de 2,4 GHz | O ESP32-S3 só opera em 2,4 GHz. O hotspot do iOS vem em 5 GHz por padrão e precisa ser trocado [G9]. | Ambos fazem hotspot de 2,4 GHz |
| Q14 | Rede do escritório: Wi-Fi WPA2-PSK ou corporativa (802.1X)? Existe NAS ou servidor? | Descarga e OTA por Wi-Fi | WPA2-PSK; PC Windows/Linux sempre ligado ou NAS |
| Q15 | O pino **EXTINT** do ZED está acessível na sua placa? | Marcação de eventos com tempo GNSS (`UBX-TIM-TM2`). No **Pro**, EXTINT e Timepulse estão no conector JST [G10, trecho de busca]. No Budget: [não verificado]. | Não acessível. Marcações feitas pelo S3 (precisão de ms). |
| Q16 | Quantas bases/dispositivos haverá? | Nomes únicos, mDNS, frota | 1 hoje, até 3 no futuro |
| Q17 | Há internet (hotspot) na maioria das obras? | Viabiliza F7/F8 | Na maioria sim, mas não em todas |
| Q18 | A LuftPlan aceita o S3 **injetar RTCM** no receptor (UART1) como alternativa ao SW Maps? | Risco de duas fontes de RTCM simultâneas | Sim, desde que exista uma chave de modo exclusivo |

---

## 3. Catálogo de automações

Legenda:
- **Esforço:** P (≤ 2 dias), M (≤ 1 semana) ou G (> 1 semana), para quem já domina ESP-IDF [HIPÓTESE de estimativa].
- **Risco:** chance de afetar a medição.

| ID | Automação | Onde | Base técnica / evidência | Esforço | Risco | Teste |
|---|---|---|---|---|---|---|
| **F1** | Gravação automática ao ligar, nome por tempo GNSS e rotação | Campo | Plano v1 §3.2 | (já no v1) | Baixo | T07–T11 do v1 |
| **F2** | **Sessão de ocupação pelo celular** (portal cativo + formulário → `session.json`) | Campo | Exemplos `captive_portal` e `file_serving` (SD) do ESP-IDF [G2] | M | Baixo | A01, A02 |
| **F3** | **Reconhecimento de marco + modo base automático** (`sites.csv` → TMODE FIXED em RAM, com confirmação) | Campo | `CFG-TMODE-*` [F2]; exemplo `Example12_setStaticPosition` [G8] | M | **Médio** (coordenada errada contamina o RTK) | A03, A04 |
| **F4** | **Perfis de configuração** (`/profiles/*.cfg` → `CFG-VALSET` RAM + verificação por `CFG-VALGET`) | Campo | Interface de configuração [F2]; exemplos VALSET/VALGET [G8] | P | Médio | A05 |
| **F5** | **Painel de saúde e alarmes** (LED/buzzer/web) | Campo | `UBX-MON-RF` (estado da antena, jamming) [F2 §3.10, §3.14.2]; `UBX-NAV-PVT`; contadores do v1 | P–M | Baixo | A06 |
| **F6** | **Marcadores de evento** (botão → CSV; ou EXTINT → `UBX-TIM-TM2` → evento no RINEX) | Campo | Time mark [F2 §3.13.2]; o RTKLIB-EX decodifica TIM-TM2 e grava evento (flag 5) no RINEX [F1 `ublox.c`, `rinex.c`] | P | Baixo | A07 |
| **F7** | **Cliente NTRIP no S3** via hotspot (RBMC-IP), com GGA, failover de mountpoint e estação mais próxima | Campo | Exemplos `Example15/16_NTRIPClient(_WithGGA)` [G8]; `ntrip_client.c` do esp32-xbee [F9] | M | **Médio** (regra de fonte única de RTCM) | A08, A09 |
| **F8** | **Servidor NTRIP no S3** (base → caster na internet) | Campo | `Example14_NTRIPServer` [G8]; `ntrip_server.c` [F9]; `str2str` aceita `ntrips://` e `ntripc://` [F1] | M | Baixo | A10 |
| **F9** | **Caster local no softAP** (rovers sem internet) | Campo | Plano v1 Fase 7; `ntrip_caster.c` [F9] | M | Baixo | T21 do v1 |
| **F10** | **RTCM base→rover por ESP-NOW** (exige um 2º ESP32 no rover) | Campo | ESP-NOW: pacotes de 250 B (v1) / 1470 B (v2), criptografia CCMP [G4]; ~250 m em visada direta (SparkFun) [G11] | M | Baixo | A11 |
| **F11** | **NMEA por TCP para o SW Maps** (sem Bluetooth) | Campo | SW Maps aceita NMEA over IP (Android/iOS) [G7, trecho de busca]; SparkFun usa servidor TCP na porta 2948 para apps iOS [G9] | P | Baixo | A12 |
| **F12** | **Hora GNSS para outros equipamentos** (NTP local no softAP; PPS opcional) | Campo | `settimeofday()` [G12]; o lwIP do ESP-IDF traz só **cliente** SNTP, então o servidor NTP é código próprio [HIPÓTESE] | M | Baixo | A13 |
| **F13** | **Proteção de energia** (Vin baixo → fecha arquivo e avisa) | Campo | ADC + supervisor (Plano v1 R1) | P | Baixo | T08 do v1 |
| **O1** | **S3 como pendrive USB** (SD exposto por MSC) | Escritório | Exemplo `tusb_msc`: o PC e o firmware **não acessam ao mesmo tempo** [G1]; S3 tem USB OTG [F8] | P–M | Baixo | A14 |
| **O2** | **Descarga automática por Wi-Fi do escritório** (mDNS + manifest SHA-256) | Escritório | `espressif/mdns` [G3]; SHA em hardware (`SOC_SHA_SUPPORTED`) [F8]; modo APSTA [G12] | M | Baixo | A15 |
| **O3** | **Pipeline UBX → RINEX → IBGE-PPP (API) → banco** | Escritório | API IBGE-PPP: `POST https://servicodados.ibge.gov.br/api/v1/ppp` [G5]; `convbin` [F1] | M | Baixo | A16 |
| **O4** | **PPK automático com RBMC** (estação mais próxima + download + `rnx2rtkp`) | Escritório | API RBMC: `relatorio`, `rinex2` (15 s), `rinex3/1s` [G6]; RTKLIB-EX [F1] | M | Baixo | A17 |
| **O5** | **Gestão de configuração** (git + `ubxsave`/`ubxcompare` + hash no `session.json`) | Escritório | pyubxutils [F13] | P | Baixo | A05 |
| **O6** | **Atualização do firmware do S3 por Wi-Fi com rollback** | Escritório | OTA + *App Rollback* (habilitado por padrão) [G4b] | P | Baixo (com rollback) | A18 |
| **O7** | **Relatório por sessão e camada GIS** (GeoPackage para QGIS) | Escritório | Python (pyproj, sqlite/GeoPackage) | P–M | Baixo | A16 |
| **O8** | **Retorno de coordenadas ao S3** (`sites.csv` por upload HTTP) | Escritório | Rota de upload do `file_serving` [G2] | P | Médio (ver F3) | A04 |

---

## 4. Detalhamento

### 4.1 Campo: sessão, gravação e base

**F2. Sessão de ocupação pelo celular**
- **Conexão:** o técnico entra no Wi-Fi `LPB1-xxxx` (WPA2). O portal cativo abre a página sozinho (DNS e DHCP opção 114) no Android, iOS e Windows. Não redireciona HTTPS [G2].
- **Campos do formulário:** código do ponto; tipo de marco; **altura 1 e altura 2** (com alerta se diferirem > 3 mm [HIPÓTESE de limiar]); tipo de medida (vertical/inclinada) e ponto de referência (ARP); antena (lista com nomes IGS); operador; observações. Botões "Iniciar ocupação" e "Encerrar".
- **Na tela de encerramento:** nova medição de altura e resumo (duração, % de épocas, satélites médios, alarmes).
- **Saída:**
  - `session.json` com os arquivos UBX da sessão;
  - hash do perfil de configuração;
  - `MON-VER` lido do receptor por UBX poll na UART1;
  - horários GNSS de início e fim.
- **Ganho:** os metadados do RINEX (`-hm`, `-hd`, `-ha`, `-hr`) saem do `session.json` automaticamente (O3). Sem retrabalho e sem erro de digitação no escritório.

**F3. Reconhecimento de marco e modo base automático, com travas**
1. O S3 lê `sites.csv` do SD: id, descrição, φ, λ, h (SIRGAS2000), σ, origem (IBGE-PPP/PPK), data.
2. Após um fix 3D, compara a posição autônoma com os sítios. Se houver candidato a menos de **R** metros (R = 5 m [HIPÓTESE; o erro autônomo é ~1,5 m CEP pela spec [F3]]), **pergunta no painel**: "Base sobre o marco X? Confirme o código".
3. Com confirmação: `CFG-VALSET` **em RAM** com `CFG-TMODE-MODE=FIXED`, `POS_TYPE=LLH`, `LAT/LON/HEIGHT(+_HP)`. A altura usada é a do ARP: h do marco + altura da antena.
4. Liga as mensagens RTCM, se o perfil pedir.
5. **Travas:**
   - Sem confirmação, cai em survey-in (padrão seguro).
   - Monitorar o aviso `UBX-INF-WARNING` "Base station position seems incorrect" (> ~50 m) [F2]. Se aparecer, voltar a survey-in e alarmar.
   - **Marcos a menos de R entre si** exigem digitar o código.
   - **O RAW sempre é gravado**, então qualquer erro é recuperável em PPK/PPP.
   - Registrar no `session.json` o modo e a coordenada usados. Assim o escritório sabe se precisa aplicar a translação Δ (Plano v1 §3.3).

**F4. Perfis de configuração**
- Arquivos `KEY,valor` no formato RTKBase [F10]: `base_logger.cfg`, `base_rtcm.cfg`, `rover_ntrip_s3.cfg`, `diagnostico.cfg`.
- **Antes de aplicar:** `CFG-VALGET` das mesmas chaves, para guardar os valores anteriores (rollback de sessão).
- **Depois de aplicar:** `CFG-VALGET` de verificação. Se divergir ou vier NAK, alarmar e reverter.
- **Somente camada RAM.** Desligar o receptor volta ao estado de fábrica ou ao último salvo pelo PC.
- O S3 **nunca grava em Flash/BBR** (regra de firmware).

**F5. Painel de saúde e alarmes**

| Indicador | Origem | Alarme proposto [HIPÓTESE de limiares] |
|---|---|---|
| Tempo de ocupação × metas | relógio da sessão | verde ≥ 4 h (meta IBGE [F15]); amarelo ≥ 2,5 h (literatura F9P [F20]) |
| Épocas RAWX recebidas/esperadas | contador | < 99 % na última hora |
| Satélites / sinais por época | `numMeas` da RAWX | < 20 sinais por > 5 min |
| Antena (aberta/curto) | `UBX-MON-RF` `antStatus` [F2 §3.10] | qualquer estado ≠ OK |
| Interferência | `UBX-MON-RF` (indicador de jamming) [F2 §3.14.2] | acima do limiar por > 1 min |
| UART / ring / SD | contadores do v1 | qualquer overflow; latência do SD > 2 s |
| Espaço no SD / bateria | FS / ADC | < 1 GB / Vin < limiar → fechar arquivo |
| Correções (se F7/F9) | idade do RTCM e bytes por min | > 10 s sem RTCM |

Sinalização: LED RGB (padrões), buzzer opcional e página `/status` atualizada por WebSocket (exemplo `ws_echo_server` [G2]).

**F6. Marcadores de evento**
- **Sem EXTINT:** o botão no S3 grava no CSV de eventos o tempo GNSS do último `NAV-PVT` mais o atraso local (precisão de ms). Serve para "altura medida", "antena mexida" e "obstrução temporária".
- **Com EXTINT** (Pro, ou se Q15 confirmar): o GPIO do S3 aciona o EXTINT. O ZED emite `UBX-TIM-TM2` com o tempo da borda [F2 §3.13.2] (só a última borda subida/descida por época). O `convbin` converte em **evento no RINEX** (flag 5) [F1].
- **Nível lógico:** conferir IOREF/VCC do pino [F2 §3.9.6].

### 4.2 Campo: conexões

**Regra de ouro: uma única fonte de RTCM por vez no receptor** [HIPÓTESE de precaução; não achei documentação u-blox sobre o comportamento com duas entradas RTCM simultâneas]. O firmware do S3 tem uma **chave de modo exclusiva**: `RTCM_SRC = app_bt | s3_ntrip | s3_espnow | none`. Quando o S3 for a fonte, orientar o técnico a **não** ligar o NTRIP no SW Maps; a tela do S3 avisa.

| Modo | Caminho dos dados | Internet | Android | iOS | Observações |
|---|---|---|---|---|---|
| **M1. Atual** | ZED UART2 ↔ ponte BT ↔ SW Maps / GNSS Master (NTRIP) | celular | ✔ SPP/BLE | ✔ BLE | Inalterado (A1) |
| **M2. S3 NTRIP + NMEA TCP** | Hotspot do celular ← S3 (STA): NTRIP RBMC-IP → UART1 RX do ZED; NMEA (UART1 TX) → servidor TCP do S3 → SW Maps "NMEA over IP" | celular (hotspot) | ✔ [G7] | ✔ [G7, G9] | Elimina o Bluetooth. Exige NMEA na UART1 junto com UBX (+~0,5 kB/s; cabe a 230400). Hotspot em 2,4 GHz. |
| **M3. softAP do S3** | Celular/notebook → `192.168.4.1` (status, sessão, download, caster local) | não | ✔ | ✔ | Celular sem dados móveis enquanto conectado |
| **M4. ESP-NOW** | S3 base → ESP32 rover → UART do rover | não | — | — | ~250 m em visada direta [G11]; mesmo canal Wi-Fi nas duas pontas [G11] |
| **M5. Wi-Fi do escritório** | S3 (STA) → PC/NAS | LAN | — | — | Descarga, OTA, sincronização de `sites.csv` |
| **M6. USB MSC** | S3 → PC como pendrive | não | — | — | O firmware para de gravar enquanto o PC monta [G1] |

O S3 pode operar **AP + STA ao mesmo tempo** (`WIFI_MODE_APSTA`) [G12]: painel local e hotspot do celular juntos.

**F7. Cliente NTRIP no S3: automações específicas**
- **Estação mais próxima automática:** baixar a *sourcetable* do caster RBMC-IP. Os registros `STR` do protocolo NTRIP trazem latitude e longitude [HIPÓTESE: a RBMC-IP preenche esses campos; conferir]. Alternativa: coordenadas pela API RBMC `relatorio/{estacao}` [G6], com cache no SD. Escolher a mais próxima da posição autônoma.
- **Failover:** lista ordenada (mais próxima, segunda mais próxima…). Troca se ficar sem RTCM por mais de N s ou se a conexão cair, com reconexão exponencial.
- **GGA:** enviar a partir do `NAV-PVT` a cada 10 s, se o mountpoint exigir. A RBMC é de base única; provavelmente não exige [HIPÓTESE].
- **Gravar o RTCM recebido** no SD (`*.rtcm3`) para PPK de contingência com `convbin -r rtcm3 -tr` [F1].
- **Credenciais RBMC-IP** guardadas no NVS do S3, nunca em arquivo no SD.

**F11. NMEA por TCP**
- Servidor TCP (porta configurável; a SparkFun usa 2948 [G9]) envia NMEA GGA/RMC/GST.
- Mensagens: `CFG-MSGOUT-NMEA_ID_GGA_UART1` (e RMC/GST) = 1, mais `CFG-UART1OUTPROT-NMEA=1` [F11].
- O parser do S3 separa NMEA de UBX: grava tudo no SD e repassa ao TCP só as linhas NMEA.

**F12. Hora GNSS na rede local**
- O S3 ajusta o próprio relógio por `settimeofday()` a partir do `NAV-PVT` [G12] e responde a pedidos NTP no softAP. Serve para sincronizar notebook, controle do drone ou câmeras.
- **Precisão:** de ms, limitada ao Wi-Fi. Com o pino TIMEPULSE do ZED num GPIO, melhora o ajuste interno [HIPÓTESE].

### 4.3 Escritório

**O1. Pendrive USB (MSC)**
- Conectar o S3 ao PC pela porta **USB nativa** do devkit (não a porta "UART"/ponte). O cartão aparece como unidade removível.
- O exemplo oficial deixa claro que **firmware e PC não acessam a partição ao mesmo tempo** [G1]. Por isso:
  1. ao detectar o USB do host, o firmware fecha o arquivo;
  2. pausa a gravação;
  3. expõe o MSC.
- **Em campo, a gravação não pode depender disso:** USB de power bank não enumera como host [HIPÓTESE].

**O2. Descarga automática por Wi-Fi**
- O S3 entra no Wi-Fi do escritório e se anuncia por mDNS (ex.: `lpb1.local`, serviço `_http._tcp`) [G3].
- O script no PC/NAS:
  1. Consulta `GET /manifest.json` (lista de arquivos, tamanho e **SHA-256 calculado no S3** com SHA em hardware [F8]).
  2. Baixa os arquivos novos (rota do `file_serving` [G2]).
  3. Confere o hash.
  4. Marca como copiado: `POST /ack`.
- **Política de exclusão no S3:** apagar só arquivos **com ack e mais velhos que X dias**, e só quando o espaço livre ficar abaixo do limiar. Nunca automaticamente sem cópia verificada.

**O3. Pipeline com a API do IBGE-PPP**

```text
[novos .ubx + session.json] → concatena por sessão → convbin (metadados do session.json)
   → RINEX 2.11 GPS+GLO 30 s (IBGE) e RINEX 3.04 (arquivo/PPK) → compacta
   → POST api/v1/ppp → JSON com link do resultado → baixa e extrai SIRGAS2000 + ITRF
   → grava em sites.gpkg/sites.csv + relatório PDF/MD → (O8) envia sites.csv ao S3
```

- **Parâmetros documentados da API** [G5, trecho de busca; **conferir na documentação oficial antes de codificar**]:
  - `arquivo`
  - `tipo-levantamento` (estático/cinemático)
  - `modelo-antena`
  - `altura-antena` (metros, ponto decimal, do marco ao ARP)
  - `marco-ibge`
  - `email`
  - `autorizacao-uso`
  - A resposta é JSON e o resultado também pode ser baixado pela URL retornada.
- **Esqueleto Python (ilustrativo):**

```python
import json, requests
s = json.load(open("session.json"))
files = {"arquivo": open(s["rinex211_zip"], "rb")}
data = {"tipo-levantamento": "estatico",            # conferir grafia aceita na doc
        "modelo-antena": s["antena_igs"],
        "altura-antena": f'{s["altura_arp_m"]:.4f}',
        "email": "geodesia@luftplan.example"}
r = requests.post("https://servicodados.ibge.gov.br/api/v1/ppp", files=files, data=data, timeout=600)
r.raise_for_status(); resultado = r.json()          # guardar JSON bruto junto da sessão
```

- **CSRS-PPP (controle):** a NRCan oferece interface web, o aplicativo "PPP Direct" e um **script de submissão automática distribuído sob pedido** [G13, trecho de busca]. Pedir o script à NRCan, ou manter o CSRS manual como amostragem (ex.: 1 em cada 5 sessões).

**O4. PPK automático com a RBMC**
1. A partir da posição aproximada da sessão, escolher a estação mais próxima pela API `relatorio/{estacao}` [G6], com cache local da lista.
2. Baixar:
   - estático: `rinex2/{estacao}/{ano}/{dia}` (15 s) [G6];
   - cinemático ou drone: `rinex3/1s/{estacao}/{ano}/{dia}/{hora}/{minuto}/{tipo}` (blocos de 15 min) [G6].
3. Gerar o `.conf` a partir do modelo do Plano v1 §8.5. A coordenada e a antena da base vêm do relatório da estação.
4. Rodar `rnx2rtkp`, extrair a média e o σ e comparar com o IBGE-PPP. **Sinalizar** se |ΔH| > 3 cm ou |ΔV| > 6 cm (critérios do v1, ajustáveis).

**O5–O8**
- **O5 (configuração):** repositório git com os perfis. Toda sessão registra o hash do perfil e o `MON-VER`. Um `ubxsave` + `ubxcompare` periódico (no PC, pelo USB do ZED) detecta divergência entre o receptor e o repositório [F13].
- **O6 (OTA):** atualizar o S3 por Wi-Fi do escritório. O *App Rollback* (padrão) volta à versão anterior se a nova falhar [G4b]. Recomendado confirmar a versão nova **só depois** de um autoteste: UART recebendo UBX válido e SD montado. É o modo de confirmação manual descrito na documentação [G4b].
- **O7 (relatório e GIS):** por sessão, um MD/PDF com metadados, durações, alarmes, resultados IBGE/CSRS/PPK e diferenças. Mais uma camada GeoPackage (pontos e sítios) para o QGIS.
- **O8 (retorno ao S3):** enviar o `sites.csv` atualizado pela rota de upload [G2]. O S3 valida o esquema e a soma de verificação antes de substituir o arquivo, guardando a versão anterior.

---

## 5. Arquitetura de firmware modular (S3)

```
┌─────────────────────────── ESP32-S3 (ESP-IDF 5.x) ─────────────────────────────┐
│ uart_gnss (UART1)  ──► ring_raw ──► sd_logger (UBX/RTCM/NMEA brutos, rotação)    │
│        │                    └────► ubx_nmea_parser ──► estado (PVT, MON-RF, ...) │
│        ◄── cfg_manager (VALSET/VALGET só em RAM; perfis; rollback de sessão)     │
│ rtcm_router: fonte única {none|ntrip_client|espnow_rx} ──► UART1 TX → ZED RX1    │
│              saída {caster_local|ntrip_server|espnow_tx} ◄── RTCM vindo do ZED   │
│ net: softAP + captive portal │ STA (hotspot/escritório) │ mDNS │ NTP local (opc.)│
│ web: /status(ws) /session /files /manifest /ack /upload(sites.csv) /ota          │
│ usb_msc: expõe SD ao PC (exclusão mútua com sd_logger)                           │
│ health: LED/buzzer, CSV de saúde, Vin (ADC), alarmes                             │
│ storage: SD (FAT32) dados │ NVS credenciais/configuração do S3                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Regras invioláveis do firmware:**
1. O `sd_logger` grava **todos os bytes** recebidos. Parsing e filtros nunca descartam nada do arquivo bruto.
2. O `cfg_manager` só escreve na camada **RAM** e sempre verifica com VALGET.
3. Há uma única fonte de RTCM ativa (chave `RTCM_SRC`).
4. As tarefas de rede e web têm **prioridade menor** que `uart_gnss` e `sd_logger`.
5. Credenciais ficam no NVS, nunca no SD.
6. Um erro em qualquer módulo de rede **não pode** parar a gravação. O watchdog reinicia o módulo, não o chip.

**Bibliotecas e referências:**
- ESP-IDF (Apache-2.0) e seus exemplos [G1–G4].
- Componente `espressif/mdns` [G3].
- Para UBX: parser próprio mínimo, ou a SparkFun u-blox GNSS v3 (MIT) [G8] se o projeto for em Arduino-ESP32.
- O caster e o cliente NTRIP do esp32-xbee são GPL-3.0 [F9]. Reaproveitar só para uso interno ou reescrever.

---

## 6. Ciclo ponta a ponta (campo ↔ escritório)

```
CAMPO                                   ESCRITÓRIO
─────                                   ──────────
Liga base → gravação automática (F1)
Celular → portal → session.json (F2)
S3 reconhece marco? ──sim──► confirma → FIXED + RTCM (F3)
        └─não──► survey-in + RAW
Painel: tempo/qualidade/alarmes (F5)
Encerrar → altura final → resumo
                         ── USB MSC (O1) ou Wi-Fi + mDNS (O2) ──►  ingest + SHA-256
                                                                  convbin (metadados do session.json)
                                                                  IBGE-PPP API (O3) ∥ PPK RBMC API (O4)
                                                                  comparação + relatório + GPKG (O7)
                         ◄── sites.csv atualizado (O8) ──────────  banco de sítios
Próxima visita: F3 usa a coordenada PPP
```

---

## 7. Formatos de arquivo (contratos entre S3 e escritório)

**`session.json`** (um por ocupação):
```json
{
  "schema": "luftplan.session/1",
  "device": "LPB1", "fw_s3": "1.2.0", "zed_monver": "HPG 1.32",
  "profile": {"name": "base_logger", "sha256": "…"},
  "site": {"id": "OBRA123-M1", "marker_type": "pino", "fixed_mode_used": false},
  "antenna": {"igs_type": "XXXXXXXXXXXXXXX NONE", "serial": "0001",
              "height_measured_m": [1.5230, 1.5232], "height_type": "vertical",
              "height_ref": "ARP", "height_final_m": 1.5231},
  "operator": "Fulano",
  "gnss_time": {"start": "2026-10-04T13:02:11Z", "end": "2026-10-04T19:05:40Z"},
  "files": [{"name": "LPB1_20261004_1300.ubx", "bytes": 10123456, "sha256": "…"}],
  "quality": {"epochs_expected": 21809, "epochs_ok": 21790, "alarms": ["ant_short@14:22Z"]},
  "events": [{"t": "2026-10-04T13:05:00Z", "type": "altura_medida"}],
  "rtcm_src": "none"
}
```

**`sites.csv`:** `id,descricao,lat,lon,h_elip,sigma_h,sigma_v,ref,epoca,origem,data,sessao_ref`, com `ref=SIRGAS2000`, `epoca=2000.4` e `origem=IBGE-PPP|PPK-RBMC`.

**`manifest.json`:** lista de `{name, bytes, sha256, closed, acked}`. Arquivos abertos (sendo gravados) aparecem com `closed=false` e não são baixados.

**`health_AAAAMMDD.csv`:** colunas do Plano v1 §3.2, mais `antStatus`, `jam`, `rtcm_age`, `rtcm_src` e `vin`.

---

## 8. Roteiro de implementação (encaixe no Plano v1)

| Etapa | Depende de | Entrega | Testes |
|---|---|---|---|
| E1 | Fases 2–4 do v1 (gravador validado) | F5 (saúde) + F13 (energia) + `/status` | A06, T08 |
| E2 | E1 | F2 (portal + `session.json`) + O1 (MSC) | A01, A02, A14 |
| E3 | E2 | O3 (pipeline IBGE-PPP) + O7 (relatório) | A16 |
| E4 | E3 | O4 (PPK RBMC) + comparação | A17 |
| E5 | E3 | O2 (Wi-Fi do escritório + manifest) + O6 (OTA) | A15, A18 |
| E6 | E3, E5 | F4 (perfis) + O8 + **F3** (reconhecimento de marco) | A03–A05 |
| E7 | Q17/Q18 | F7 (cliente NTRIP) + F11 (NMEA TCP) | A08, A09, A12 |
| E8 | opcional | F8, F9, F10, F12, F6 | A07, A10, A11, A13 |

Começar por E1–E3: são as que mais tiram trabalho manual sem tocar na medição.

---

## 9. Testes individuais (uma variável por teste)

| ID | Objetivo | Variável | Procedimento | Critério de aceite |
|---|---|---|---|---|
| A01 | O portal cativo abre sozinho | Sistema do celular | Conectar Android e iPhone ao softAP | A página abre sem digitar URL em ≥ 2 de 3 tentativas por aparelho. Fallback: `192.168.4.1` funciona sempre. |
| A02 | `session.json` correto e gravação contínua | Uso do formulário | Preencher e salvar 10× durante a gravação | JSON válido pelo esquema; T07 do v1 bit-exato durante o teste |
| A03 | Reconhecimento de marco | Distância ao sítio | Base a 0, 3 e 20 m de um sítio cadastrado | Propõe o sítio a 0 e 3 m; não propõe a 20 m; **nunca** aplica sem confirmação |
| A04 | Trava de coordenada errada | Coordenada do sítio | Cadastrar o sítio com erro de 100 m e confirmar | Detecta `INF-WARNING`, volta a survey-in e alarma; o RAW segue gravando |
| A05 | Perfil aplicado e revertido | Perfil | Aplicar `base_rtcm`, verificar, reverter | VALGET 100 % igual ao arquivo; depois de reverter, `ubxcompare` contra o backup dá 0 diferenças |
| A06 | Alarmes disparam | Falha simulada (uma por vez) | Desconectar a antena; encher o SD; baixar Vin | Alarme correto em < 10 s; gravação mantida, exceto no caso de Vin, que fecha o arquivo |
| A07 | Evento chega ao RINEX | Fonte do evento | Botão (CSV) / EXTINT (TIM-TM2) | CSV com tempo ± 50 ms do PVT [HIPÓTESE de tolerância]; RINEX com evento flag 5 (se EXTINT) |
| A08 | Cliente NTRIP do S3 | Fonte de RTCM = S3 | Hotspot + RBMC-IP; SW Maps sem NTRIP | Fix RTK; idade do RTCM < 3 s em 95 % do tempo; `.rtcm3` gravado e convertível |
| A09 | Failover | Mountpoint indisponível | Primeiro mountpoint inválido | Troca para o segundo em < 30 s |
| A10 | Servidor NTRIP | Caster de teste (`str2str ntripc://` num notebook) | Base envia; rover consome | Stream contínuo por 1 h; reconexão automática |
| A11 | ESP-NOW | Distância | 50, 150 e 250 m em visada direta | Perda de pacotes < 1 % até a maior distância com RTK fixo no rover |
| A12 | NMEA TCP para o SW Maps | Plataforma | Android e iPhone no hotspot | Posição e qualidade do fix exibidas; 1 h sem desconexão |
| A13 | NTP local | Cliente | Notebook sincroniza com o S3 | Offset < 20 ms [HIPÓTESE] |
| A14 | Pendrive USB | Conexão USB | Plugar durante a gravação | Gravação fecha o arquivo antes de expor o MSC; arquivos íntegros (SHA); retoma ao desplugar |
| A15 | Descarga automática | Wi-Fi do escritório | 3 dias de dados | 100 % dos arquivos fechados copiados com SHA conferido; nenhuma exclusão sem ack |
| A16 | Pipeline IBGE-PPP | Sessão real | Rodar o pipeline sem intervenção | Resultado SIRGAS2000 armazenado; RINEX com metadados iguais ao `session.json` |
| A17 | PPK automático | Estação RBMC | Sessão perto da UFPR | Estação correta escolhida; diferença para o IBGE-PPP dentro dos critérios do v1 |
| A18 | OTA com rollback | Imagem defeituosa de propósito | Enviar firmware que falha no autoteste | Volta à versão anterior sozinho e a gravação continua |

---

## 10. Riscos específicos das automações

| Risco | Mitigação |
|---|---|
| Modo fixo automático com marco errado | Confirmação obrigatória por código, raio pequeno, monitor de `INF-WARNING`, RAW sempre gravado (A03, A04) |
| Duas fontes de RTCM simultâneas | Chave `RTCM_SRC` exclusiva e aviso no painel [HIPÓTESE de precaução] |
| A rede trava ou atrasa a gravação | Prioridades das tarefas, watchdog por módulo, testes A02/A14 com T07 bit-exato |
| Credenciais (RBMC-IP, Wi-Fi) vazarem pelo SD | NVS; SD sem segredos; softAP com WPA2 |
| Exclusão acidental de dados | Exclusão só com ack + SHA + idade + falta de espaço; nunca manual pela web na Fase 1 |
| Parâmetros da API do IBGE mudarem | Pipeline valida a resposta; guarda o JSON bruto; fallback de envio manual pela web |
| Hotspot do iPhone em 5 GHz | Instrução na ficha de campo (2,4 GHz) [G9] |
| Licença GPL (código do esp32-xbee) | Uso interno ou reescrita; preferir MIT/Apache |

---

## 11. Fontes novas (acesso em 2026-10-04)

**Exemplos e documentação do ESP-IDF** (`espressif/esp-idf`, commit **4d59230**, 2026-09-25)
- **G1:** `examples/peripherals/usb/device/tusb_msc/README.md`: MSC com SD; acesso exclusivo PC/firmware; alvo ESP32-S3. Também `tusb_ncm` (rede por USB).
- **G2:** `examples/protocols/http_server/file_serving/README.md` (download e upload em FAT no SD, SDSPI e SDMMC); `captive_portal/README.md` (DNS + DHCP opção 114; Android, iOS e Windows; não redireciona HTTPS); `ws_echo_server`; `restful_server`.
- **G3:** `examples/protocols/http_server/restful_server/main/idf_component.yml` (dependência `espressif/mdns: ^1.8.0`).
- **G4:** `docs/en/api-reference/network/esp_now.rst`: v1.0 com 250 B e v2.0 com 1470 B; criptografia CCMP/PMK/LMK; até 20 pares.
- **G4b:** `docs/en/api-reference/system/ota.rst` (*App Rollback*, `CONFIG_BOOTLOADER_APP_ROLLBACK`, modos de confirmação); `examples/system/ota/README.md`.
- **G12:** `docs/en/api-guides/wifi-driver/overview.rst` (`WIFI_MODE_APSTA`); `docs/en/api-reference/system/system_time.rst` (`settimeofday()`, cliente SNTP).
- https://github.com/espressif/esp-idf

**IBGE / NRCan**
- **G5:** IBGE, *API do Serviço IBGE-PPP*: `POST https://servicodados.ibge.gov.br/api/v1/ppp` e parâmetros [trecho de busca]: https://servicodados.ibge.gov.br/api/docs/ppp?versao=1
- **G6:** IBGE, *API da RBMC*: `.../api/v1/rbmc/relatorio/{estacao}`, `.../rinex2/{estacao}/{ano}/{dia}`, `.../rinex3/1s/{estacao}/{ano}/{dia}/{hora}/{minuto}/{tipo}` [trecho de busca]: https://servicodados.ibge.gov.br/api/docs/rbmc?versao=1
- **G13:** NRCan/IGS, *Updates to the CSRS-PPP online service* (IGS Workshop 2018) e artigos: web, PPP Direct e script de submissão automática sob pedido [trecho de busca]: https://files.igs.org/pub/resource/pubs/workshop/2018/IGSWS-2018-PY04-05.pdf

**Aplicativos e SparkFun**
- **G7:** rtkdata, *SW Maps*: NMEA over IP (cliente TCP) no Android e iOS [trecho de busca]: https://docs.rtkdata.com/integration-hub/ntrip-clients-and-field-software/sw-maps · https://docs.rtkdata.com/integration-hub/app-playbook/sw-maps
- **G8:** SparkFun, **u-blox GNSS v3** library **3.1.15**, último commit 2026-08-28, código MIT. Exemplos usados como referência:
  - `Data_Logging/DataLoggingExample7_OpenLogESP32_SPI_SDIO` (RAWX+SFRBX em SD no ESP32; buffer de 64 KB; escrita em blocos de 2048 B);
  - `ZED-F9P/Example12_setStaticPosition`;
  - `Example14_NTRIPServer`;
  - `Example15/16/17_NTRIPClient*`;
  - `Callbacks/CallbackExample3_TIM_TM2`;
  - `VALGET_and_VALSET`.
  - https://github.com/sparkfun/SparkFun_u-blox_GNSS_v3
- **G9:** SparkFun RTK Everywhere Firmware v3.4, `docs/gis_software_ios.md` (servidor TCP na porta 2948; hotspot do iOS precisa estar em 2,4 GHz).
- **G10:** ArduSimple, *simpleRTK2B Pro* (EXTINT e Timepulse no conector JST) [trecho de busca]: https://www.ardusimple.com/product/simplertk2b-pro/
- **G11:** SparkFun RTK Everywhere Firmware v3.4, `docs/menu_radios.md` e `docs/correction_transport.md` (ESP-NOW ~250 m em visada direta; mesmo canal Wi-Fi; ponto-multiponto).

**RTKLIB**
- **F1 (complemento):** RTKLIB-EX 2.5.1:
  - `src/rcv/ublox.c` (`decode_timtm2`);
  - `src/rinex.c` (flag de evento externo 5);
  - `app/consapp/str2str/str2str.c` (`ntrips://`, `ntripc://`, `file://…::T::S=`).

**ZED-F9P**
- **F2 (complemento):** Integration Manual R13: §3.9.6 EXTINT; §3.10 supervisor de antena (`UBX-MON-RF`, `ANTSTATUS`); §3.13.2 time mark (`UBX-TIM-TM2`); §3.14.2 detecção de jamming.

---

## 12. Fora de escopo (só listado)

1. Display e-ink/OLED no S3 para status sem celular.
2. Rádio LoRa para RTCM de longo alcance.
3. Painel web de frota (várias bases) num servidor do escritório.
4. Envio de alertas pela internet (Telegram/MQTT) quando houver hotspot.
5. Conversão RINEX no próprio S3 (desaconselhada: CPU/RAM; fazer no escritório).
6. Integração direta com o software de processamento de drone/SLAM.
