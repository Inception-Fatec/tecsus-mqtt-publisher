# DataPublisher

Firmware para a estação meteorológica baseada em **ESP32** desenvolvida pela Inception em parceria com a Tecsus. Responsável por coletar dados dos sensores e publicá-los via **MQTT** no broker **EMQX Cloud**.

---

## Visão geral

```
Placa ESP32 (DHT22)
      ↓ leitura a cada 5 min
  Payload JSON
      ↓ MQTT / TLS
  EMQX Cloud (Broker)
      ↓
  Subscriber (outro repositório)
```

O payload publicado segue o formato:

```json
{
  "uid": "AC67B23C2B78",
  "uxt": 1745763000,
  "tem": 28.4,
  "umi": 74.2,
  "plu": 3.1
}
```

| Campo | Descrição             | Tipo  |
|-------|-----------------------|-------|
| `uid` | MAC address da placa  | string |
| `uxt` | Unix timestamp (UTC)  | int   |
| `tem` | Temperatura (°C)      | float |
| `umi` | Umidade (%)           | float |
| `plu` | Pluviômetro (mm)      | float |

---

## Hardware necessário

- Placa **ESP32**
- Sensor **DHT22** (módulo V182) conectado ao conector **J5**
- Cabo **USB** para gravação do firmware

### Conexão do sensor DHT22 no conector J5

| Pino do módulo V182 | Pino do J5   | GPIO do ESP32 |
|---------------------|--------------|---------------|
| `+`                 | `3V3`        | —             |
| `out`               | `Modulo_2`   | GPIO 13       |
| `-`                 | `GND`        | —             |

> ⚠️ **Atenção:** nunca inverta a polaridade do sensor. Conectar `+` no `GND` danifica o componente permanentemente.

---

## Pré-requisitos

### Arduino IDE
Versão **2.x** ou superior — [download](https://www.arduino.cc/en/software)

### Suporte ao ESP32
1. Abra o Arduino IDE
2. Vá em **File → Preferences**
3. Em *Additional boards manager URLs*, adicione:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
4. Vá em **Tools → Board → Boards Manager**, busque `esp32` e instale **esp32 by Espressif Systems**

### Bibliotecas necessárias
Instale pelo **Sketch → Include Library → Manage Libraries**:

| Biblioteca                  | Autor     |
|-----------------------------|-----------|
| `PubSubClient`              | Nick O'Leary |
| `ArduinoJson`               | Benoit Blanchon |
| `DHT sensor library`        | Adafruit  |
| `Adafruit Unified Sensor`   | Adafruit  |

---

## Configuração

### 1. Copie o arquivo de configuração

```bash
cp config.example.h config.h
```

### 2. Preencha o `config.h` com suas credenciais

```cpp
#define WIFI_SSID     "nome_da_sua_rede"
#define WIFI_PASSWORD "senha_da_sua_rede"

#define MQTT_SERVER "m1396282.ala.us-east-1.emqxsl.com"
#define MQTT_PORT   8883
#define MQTT_USER   "seu_usuario_emqx"
#define MQTT_PASS   "sua_senha_emqx"
```

> As credenciais do broker EMQX estão no `.env` do repositório principal — use as mesmas.

---

## Gravando na placa

### 1. Conecte a placa via USB

### 2. Selecione a placa e porta no Arduino IDE
- **Tools → Board → esp32 → ESP32 Dev Module**
- **Tools → Port →** selecione a porta COM que apareceu (ex: `COM5` no Windows, `/dev/ttyUSB0` no Linux)

### 3. Compile e grave
Clique em **Upload** (ícone de seta →) ou use `Ctrl + U`

### 4. Acompanhe pelo Serial Monitor
Abra em **Tools → Serial Monitor** com baud rate **115200**

Saída esperada:
```
[Setup] UID da placa: AC67B23C2B78
[Setup] Tópico: estacoes/AC67B23C2B78/dados
[WiFi] Conectando à rede: MinhaRede...
[WiFi] Conectado. IP: 192.168.1.100
[NTP] Tempo sincronizado: 1745763000
[MQTT] Conectando ao broker EMQX...
[MQTT] Conectado
[MQTT] Publicado em estacoes/AC67B23C2B78/dados → OK | {"uid":"AC67B23C2B78","uxt":1745763000,"tem":28.4,"umi":74.2,"plu":3.1}
```

---

## Funcionamento

Após inicializar, a placa executa três processos paralelos via **FreeRTOS**:

| Processo          | Core | Intervalo | Responsabilidade                        |
|-------------------|------|-----------|-----------------------------------------|
| `taskLerSensores` | 0    | 5 min     | Lê temperatura e umidade do DHT22       |
| `taskMonitorWifi` | 1    | 30 seg    | Verifica e reconecta o WiFi se necessário |
| `loop()`          | 1    | 30 seg    | Monta e publica o payload no broker     |

---

## Solução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| `[WiFi] Conectando...` infinito | SSID ou senha errados | Verifique o `config.h` |
| `[MQTT] Conexão falhou. Código: 4` | Usuário/senha do broker incorretos | Verifique as credenciais EMQX no `config.h` |
| `[MQTT] Publicado → Falhou` | Placa desconectou do broker | A reconexão é automática, aguarde |
| Sensor retornando `NaN` | DHT22 mal encaixado ou danificado | Reencaixe o módulo V182 no J5 |
| Porta COM não aparece | Driver USB não instalado | Instale o driver CP210x ou CH340 conforme o chip USB da placa |

---

## Estrutura do repositório

```
DataPublisher/
  estacao_mqtt.ino    ← código principal do firmware
  config.h            ← credenciais locais (não versionado)
  config.example.h    ← modelo de configuração
  .gitignore
  README.md
```

---

## Observações

- O arquivo `config.h` está no `.gitignore` e **nunca deve ser commitado**
- O `uid` da placa é gerado automaticamente a partir do MAC address — cada placa terá um identificador único sem necessidade de configuração manual
- O tópico MQTT segue o padrão `estacoes/{uid}/dados`, compatível com o subscriber que escuta `estacoes/+/dados`