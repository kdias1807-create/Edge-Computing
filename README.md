# Edge-Computing

# 🍷 Checkpoint 05 – Vinheria Agnello (IoT)

**Disciplina:** Edge Computing & Computer Systems – Engenharia de Software (FIAP)
**Professor:** Lucas Demetrius Augusto
**Turma:** 1ESPQ

**Integrantes do grupo:**

| Nome | RM |
|Antonio do Nascimento Ferreira de Sousa| |573706|
|Kaio Nincao Maia Dias| |568834|
|Matheus Martins Santos|570884|
|Leonardo Gonçalves Cardoso da Fonseca | 569475 |
| Juliano Galhardo de Oliveira | 572493 |
|kaue Fernando jaques lopes |571353 |

🔗 **Projeto no Wokwi:** https://wokwi.com/projects/477901588478288897

---

## 1. Contexto

A Vinheria Agnello deseja evoluir o monitoramento da sua adega e se integrar ao ecossistema de negócios da União Europeia. Na fase 1 foi monitorada apenas a luminosidade. Na **fase 2** o proprietário pediu também:

- monitoramento de **temperatura** e **umidade** do depósito;
- **dados históricos** para gráficos, análises e rastreabilidade;
- uso de **Smart Data Models** e do padrão **NGSIv2 / FIWARE**, com comunicação via **MQTT**.

Condições ideais de armazenamento de vinho:

| Variável | Valor ideal |
|----------|-------------|
| Temperatura | ~13 °C (variação máxima de 3 °C) |
| Umidade | ~70 % (faixa de 60 % a 80 %) |
| Luminosidade | Penumbra (iluminação muito suave) |

## 2. Objetivo desta entrega

Implementar no **ESP32** a leitura dos sensores **DHT (temperatura e umidade)** e **LDR (luminosidade)**, enviar os dados via **Wi-Fi + MQTT** e permitir a **leitura e escrita** dos dados pelo aplicativo **MyMQTT**.

## 3. Arquitetura

```
 ┌───────────────┐   Wi-Fi / MQTT    ┌──────────────────────┐   MQTT    ┌─────────────────┐
 │ ESP32         │ ────────────────► │ Broker MQTT          │ ◄──────── │ App MyMQTT      │
 │ DHT + LDR     │ ◄──────────────── │ test.mosquitto.org   │ ────────► │ (leitura/escrita)│
 │ LED (cmd)     │    :1883          └──────────┬───────────┘           └─────────────────┘
 └───────────────┘                              │ (evolução)
                                                ▼
                                   IoT Agent MQTT → Orion (NGSIv2)
                                   → STH-Comet / MongoDB (histórico)
```

O firmware usa o formato de tópicos e payloads **Ultralight 2.0**, o mesmo esperado pelo **IoT Agent MQTT do FIWARE**. Assim, basta apontar o broker para a stack FIWARE (Docker Compose com Mosquitto, IoT Agent, Orion, STH-Comet e MongoDB) para que os dados sejam convertidos em entidades NGSIv2 e armazenados com histórico.

## 4. Hardware

| Componente | Função | Pino do ESP32 |
|-----------|--------|----------------|
| ESP32 DevKit | Microcontrolador com Wi-Fi | – |
| Sensor DHT | Temperatura e umidade | GPIO 4 |
| LDR | Luminosidade (entrada analógica ADC1) | GPIO 34 |
| LED onboard | Atuador controlado por comando MQTT | GPIO 2 |

> ⚠️ **Observação sobre o sensor:** o enunciado cita o **DHT11**, mas o código deste projeto (e a simulação no Wokwi) usa o **DHT22**, declarado como `DHT dht(PIN_DHT, DHT22);`. Os dois usam a mesma biblioteca e os mesmos pinos. Para usar um DHT11 físico, troque `DHT22` por `DHT11` no código.

## 5. Bibliotecas

| Biblioteca | Uso |
|-----------|-----|
| `WiFi.h` (nativa do ESP32) | Conexão Wi-Fi |
| `PubSubClient` | Cliente MQTT |
| `DHT sensor library` (Adafruit) | Leitura do DHT |

No Wokwi, o arquivo `libraries.txt` deve conter:

```
PubSubClient
DHT sensor library
```

## 6. Configurações MQTT

| Parâmetro | Valor |
|-----------|-------|
| Wi-Fi (SSID) | `Wokwi-GUEST` (sem senha) |
| Broker | `test.mosquitto.org` |
| Porta | `1883` |
| ID do cliente | `fiware_021_makale` |
| Tópico de publicação (dados) | `/TEF/device021/attrs` |
| Tópico de assinatura (comandos) | `/TEF/device021/cmd` |

> Para uso em placa física, altere `default_SSID` e `default_PASSWORD` para a sua rede Wi-Fi.

## 7. Funcionamento do firmware

1. Conecta ao Wi-Fi e ao broker MQTT (com reconexão automática, sem `delay()` bloqueante).
2. **LDR:** 1 leitura por segundo; a cada 5 leituras calcula a média e converte de 0–4095 (ADC de 12 bits) para **0–100 %**.
3. **DHT:** 1 leitura a cada 2 segundos (limite do sensor); as leituras válidas são promediadas no mesmo ciclo.
4. A cada 5 amostras, publica temperatura, umidade e luminosidade em **uma única mensagem**.
5. A cada 5 segundos publica o estado do LED.
6. Escuta comandos em `/TEF/device021/cmd` para ligar/desligar o LED onboard.

### Formato das mensagens (Ultralight 2.0)

**Leituras dos sensores** → `/TEF/device021/attrs`

```
t|23.5|h|60.0|l|45
```

| Chave | Significado | Unidade |
|-------|-------------|---------|
| `t` | Temperatura | °C |
| `h` | Umidade relativa | % |
| `l` | Luminosidade | % (0–100) |

**Estado do LED** → `/TEF/device021/attrs`

```
s|on
s|off
```

**Comandos** → `/TEF/device021/cmd`

| Mensagem | Efeito |
|----------|--------|
| `device021@on\|` | Liga o LED |
| `device021@off\|` | Desliga o LED |

## 8. Como replicar o projeto

### 8.1 Simulação no Wokwi

1. Acesse o link do projeto: https://wokwi.com/projects/477901588478288897
2. Clique em **▶ Start the simulation**.
3. Abra o **Serial Monitor** e confira os logs de conexão (`Conectado com sucesso na rede`, `Conectado com sucesso ao broker MQTT!`) e as leituras de temperatura, umidade e luminosidade.
4. Altere os valores do DHT e do LDR clicando nos componentes durante a simulação e observe a mudança nas leituras.

### 8.2 Execução em hardware real (opcional)

1. Instale a **Arduino IDE** e o suporte à placa **ESP32** (Gerenciador de Placas).
2. Instale as bibliotecas **PubSubClient** e **DHT sensor library** (Gerenciador de Bibliotecas).
3. Monte o circuito conforme a seção 4 (DHT no GPIO 4, LDR no GPIO 34, alimentação 3V3 e GND).
4. Abra `sketch.ino`, ajuste SSID e senha do Wi-Fi e, se necessário, o broker.
5. Selecione a placa **ESP32 Dev Module**, escolha a porta COM e faça o upload.

### 8.3 Leitura e escrita com o MyMQTT

1. Instale o app **MyMQTT** (Android) ou equivalente (ex.: MQTT Explorer / MQTTX).
2. Em **Settings**, configure:
   - **Broker:** `test.mosquitto.org`
   - **Port:** `1883`
3. **Leitura:** em *Subscribe*, assine o tópico `/TEF/device021/attrs`. Com a simulação rodando, as mensagens `t|..|h|..|l|..` e `s|on/off` aparecerão no app.
4. **Escrita:** em *Publish*, envie para o tópico `/TEF/device021/cmd` a mensagem `device021@on|` (liga o LED) ou `device021@off|` (desliga o LED).
5. Confirme no Serial Monitor a mensagem recebida e, no MyMQTT, a confirmação `s|on` / `s|off`.

> Como `test.mosquitto.org` é um broker **público**, outros usuários podem usar o mesmo tópico. Se notar mensagens estranhas, troque `device021` por um identificador exclusivo do grupo, tanto nos tópicos quanto no `topicPrefix` do código.

## 9. Arquivos do repositório

| Arquivo | Descrição |
|---------|-----------|
| `sketch.ino` | Código-fonte do ESP32 |
| `diagram.json` | Circuito do Wokwi |
| `libraries.txt` | Bibliotecas usadas |
| `README.md` | Este documento |


## 10. Próximos passos (evolução para FIWARE)

- Subir a stack FIWARE via Docker Compose (Mosquitto, IoT Agent MQTT, Orion, STH-Comet, MongoDB).
- Provisionar o dispositivo no IoT Agent e criar a subscrição do Orion para o STH-Comet (histórico).
- Modelar a entidade com o Smart Data Model **WeatherObserved** / **Device**, em NGSIv2.
- Incluir o **timestamp** (`TimeInstant`) de cada leitura, por exemplo via NTP no ESP32.
- Exportar o histórico para gráficos e relatórios de rastreabilidade.

---

© 2026 – Material da disciplina de Edge Computing & Computer Systems, FIAP.
