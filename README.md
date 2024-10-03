# Projeto IoT: Leitura de Dados do DHT11 e LDR com ESP32 e MyMQTT

## Descrição do Projeto

Este projeto demonstra a captura de dados de sensores ambientais, utilizando o sensor de temperatura e umidade **DHT11** e o sensor de luminosidade **LDR**, com um **ESP32**. Os dados capturados são enviados via protocolo **MQTT** e podem ser lidos e controlados por um aplicativo MQTT, como o **MyMQTT**. Além disso, o projeto inclui funções de controle remoto de um **LED** (ligar e desligar) via MQTT.

## Tabela de Conteúdo
1. [Pré-requisitos](#pré-requisitos)
2. [Diagrama de Circuito](#diagrama-de-circuito)
3. [Configuração do Ambiente](#configuração-do-ambiente)
4. [Configuração MQTT](#configuração-mqtt)
5. [Código Fonte](#código-fonte)
6. [Como ver o host para o MQTT](#como-ver-o-host-para-o-mqtt)
7. [Execução do Projeto](#execução-do-projeto)
8. [Resultados](#resultados)
9. [Leitura de Tópico MQTT](#leitura-de-tópico-mqtt)
10. [Controle do LED (Liga/Desliga) via MQTT](#controle-do-led-ligadesliga-via-mqtt)
11. [Código para Solicitação de Dados Temporais via Método GET HTTP](#código-para-solicitação-de-dados-temporais-via-método-get-http)
12. [Referências](#referências)
13. [Integrantes](#integrantes)

## Pré-requisitos

Antes de começar, você precisará dos seguintes componentes:

### Hardware:
- **ESP32**
- **Sensor DHT11** (Temperatura e Umidade)
- **Sensor LDR** (Resistor Dependente de Luz)
- **LED** com Resistor (220Ω)
- Resistores de **10KΩ** e **220Ω** (para o LDR)
- **Protoboard** e **Jumpers**

### Software:
- **Arduino IDE** (com suporte ao ESP32)
- Biblioteca **DHT sensor library** (Adafruit)
- Biblioteca **PubSubClient** (para MQTT)
- Aplicativo **MyMQTT** (disponível para Android)
- **Python** (com `paho-mqtt` e `requests`)

## Diagrama de Circuito

Monte o circuito conforme o diagrama abaixo:

### Conexões:
- **DHT11**:
  - **VCC** -> 3.3V
  - **GND** -> GND
  - **DATA** -> GPIO 4 (ESP32)

- **LDR**:
  - Um lado do **LDR** conectado ao 3.3V e o outro ao **GPIO 34**, com um resistor de **10KΩ** conectado entre o GPIO 34 e o GND.

- **LED**:
  - O ânodo do **LED** vai para o **GPIO 2** (ESP32).
  - O cátodo do LED vai para **GND** com um resistor de **220Ω** em série.

- **ESP32**:
  - Alimentado por **USB** ou fonte de **3.3V**.

## Configuração do Ambiente

### Passo 1: Configurando a IDE do Arduino para ESP32
1. Abra a **IDE Arduino**.
2. Vá em **Arquivo > Preferências**.
3. Adicione o seguinte URL no campo **"URLs adicionais para gerenciadores de placas"**:
   ```
   https://dl.espressif.com/dl/package_esp32_index.json
   ```
4. Vá em **Ferramentas > Placas > Gerenciador de Placas** e procure por **ESP32**.
5. Instale o pacote **ESP32 by Espressif Systems**.

### Passo 2: Instalação de Bibliotecas
Na IDE Arduino, vá em **Sketch > Incluir Biblioteca > Gerenciar Bibliotecas** e instale as seguintes:
- **DHT sensor library** (by Adafruit)
- **Adafruit Unified Sensor**
- **PubSubClient** (para MQTT)

### Passo 3: Configuração do Broker MQTT
Instale um broker MQTT em um servidor local ou utilize um serviço gratuito como o **[test.mosquitto.org](https://test.mosquitto.org/)**.

Configure o **MyMQTT** para se conectar ao broker, inserindo o endereço IP do servidor MQTT, porta (geralmente **1883**), e tópicos de publicação e assinatura.

## Configuração MQTT

Os dados serão publicados nos seguintes tópicos:
- `iot/esp32/dht11` – Publica a temperatura e a umidade lidas pelo DHT11.
- `iot/esp32/ldr` – Publica os dados de luminosidade lidos pelo LDR.
- `iot/esp32/led` – Recebe comandos para controlar o LED (ligar/desligar).

Você pode configurar outros tópicos conforme necessário.

## Código Fonte

O código a seguir captura os dados do sensor DHT11 e do LDR, publica os dados via MQTT e permite a leitura e controle pelo aplicativo MyMQTT.

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>

#define DHTPIN 4      // Pin where the DHT11 is connected
#define DHTTYPE DHT11 // Define the type of DHT sensor
#define LDRPIN 34     // Pin for LDR sensor
#define LEDPIN 2      // Pin for LED control

const char* ssid = "Seu_SSID";      // Substitua pelo nome da sua rede WiFi
const char* password = "Sua_Senha"; // Substitua pela senha do WiFi
const char* mqtt_server = "test.mosquitto.org";

WiFiClient espClient;
PubSubClient client(espClient);
DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  dht.begin();
  pinMode(LEDPIN, OUTPUT); // Define o LED como saída
}

void setup_wifi() {
  delay(10);
  Serial.println("Conectando ao WiFi...");
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("WiFi conectado.");
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Tentando conectar ao MQTT...");
    if (client.connect("ESP32Client")) {
      Serial.println("Conectado ao MQTT!");
    } else {
      Serial.print("Falha ao conectar. Estado: ");
      Serial.print(client.state());
      delay(5000);
    }
  }
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Captura e publica os dados do DHT11
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  if (isnan(h) || isnan(t)) {
    Serial.println("Falha ao ler o DHT11");
    return;
  }

  String payload = "Temp: " + String(t) + " C, Umidade: " + String(h) + "%";
  client.publish("iot/esp32/dht11", payload.c_str());
  Serial.println(payload);

  // Captura e publica os dados do LDR
  int ldrValue = analogRead(LDRPIN);
  String ldrPayload = "Luminosidade: " + String(ldrValue);
  client.publish("iot/esp32/ldr", ldrPayload.c_str());
  Serial.println(ldrPayload);

  delay(2000); // Atraso de 2 segundos entre leituras
}
```

## Como ver o host para o MQTT

Para visualizar e identificar o **host** (endereço IP ou nome de domínio) que você deve inserir no aplicativo **MyMQTT** para se conectar ao broker MQTT, siga estas orientações de acordo com o ambiente em que o broker está rodando:

### 1. **Broker MQTT rodando localmente (na sua rede local)**

Se o broker MQTT está sendo executado em uma máquina local (como seu próprio computador ou um Raspberry Pi) na mesma rede Wi-Fi que o dispositivo ESP32 e o aplicativo MyMQTT, você precisará encontrar o **endereço IP local** dessa máquina.

#### Como encontrar o IP local:
- **No Windows**:
  1. Abra o **Prompt de Comando** (digite `cmd` na barra de pesquisa).
  2. Digite o comando:
     ```bash
     ipconfig
     ```
  3. Procure pelo adaptador de rede que está conectado (Wi-Fi ou Ethernet) e encontre o **Endereço IPv4**. O endereço IP terá um formato como `192.168.0.x` ou `192.168.1.x`.

- **No Linux ou Mac**:
  1. Abra o terminal e digite:
     ```bash
     ifconfig
     ```
  2. Localize o adaptador de rede (geralmente `wlan0` ou `eth0`), e o **Endereço IP** será algo como `192.168.x.x`.

#### Exemplo:
Se o endereço IP local da máquina for `192.168.1.50`, esse será o **host** que você deve inserir no MyMQTT e no código do ESP32.

### 2. **Broker MQTT rodando em um servidor online (exemplo: test.mosquitto.org)**

Se você está utilizando um broker MQTT em um servidor remoto ou um broker público, como **Mosquitto**, você pode usar o **nome de domínio** ou o **endereço IP** do serviço.

#### Exemplo de broker público:
- **test.mosquitto.org**: Este é um broker público gratuito para testes. Nesse caso, o **host** que você deve utilizar no MyMQTT e no código do ESP32 será simplesmente `test.mosquitto.org`.

### 3. **Broker MQTT rodando em um Raspberry Pi ou outro dispositivo IoT na sua rede**

Se você estiver usando um dispositivo como um **Raspberry Pi** para rodar o broker MQTT, você pode encontrar o endereço IP do Raspberry Pi usando os mesmos métodos descritos acima (usando `ifconfig` no Raspberry Pi ou verificando o roteador da sua rede).

### Verificando o endereço IP no seu roteador
Outra forma de encontrar o IP de um dispositivo na sua rede local é acessando o painel de administração do roteador (geralmente acessível pelo navegador em `192.168.0.1` ou `192.168.1.1`), onde você poderá ver uma lista de dispositivos conectados com seus respectivos endereços IP.

### Conclusão

O **host** é o **endereço IP** ou o **nome de domínio** do servidor onde o broker MQTT está rodando. Você pode obtê-lo conforme o ambiente (local ou remoto) onde o broker está em execução. No **MyMQTT**, você deve inserir esse host junto com a **porta 1883** para se conectar corretamente ao broker.

## Execução do Projeto

1. **Monte o circuito** conforme o diagrama.
2. **Faça o upload do código** para o ESP32 via **Arduino IDE**.
3. Abra o **monitor serial** para verificar a conexão e o envio dos dados MQTT.
4. No aplicativo **MyMQTT**:
   - **Conecte-se** ao broker MQTT.
   - Insira o **host** (endereço IP ou nome de domínio do broker MQTT).
   - Especifique a **porta 1883** (padrão para comunicação MQTT).
   - Assine os tópicos `iot/esp32/dht11` e `iot/esp32/ldr` para visualizar os dados enviados pelo ESP32.
   - Para **controlar o LED**, envie comandos de ligar/desligar para o tópico `iot/esp32/led` utilizando os comandos MQTT.

## Resultados

Os dados de temperatura, umidade e luminosidade serão atualizados no aplicativo **MyMQTT** em tempo real. A partir disso, você poderá monitorar o ambiente de forma remota, visualizar e analisar as leituras dos sensores.

## Leitura de Tópico MQTT

Este código Python usa a biblioteca `paho-mqtt` para se conectar ao broker MQTT e ler dados de um tópico específico:

```python
import paho.mqtt.client as mqtt
import sys

# Definições:
Broker = "{{url}}"
PortaBroker = 1883
KeepAliveBroker = 60
TopicoSubscribe = "/TEF/device001/attrs/p"

# Callback - Conexão ao broker realizada
def on_connect(client, userdata, flags, rc):
    print("[STATUS] Conectado ao Broker. Resultado de conexão: "+str(rc))
    # Faz subscribe automático no tópico
    client.subscribe(TopicoSubscribe)

# Callback - Mensagem recebida do broker
def on_message(client, userdata, msg):
    MensagemRecebida = str(msg.payload)
    print("[MSG RECEBIDA] Tópico: "+msg.topic+" / Mensagem: "+MensagemRecebida)

# Programa principal:
print("[STATUS] Inicializando MQTT...")
# Inicializa MQTT:
client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

client.connect(Broker, PortaBroker, KeepAliveBroker)
client.loop_forever()
```

## Controle do LED (Liga/Desliga) via MQTT

### Código para **Ligar o LED** via MQTT:

```python
import paho.mqtt.client as mqtt
import sys

# Definições:
Broker = "{{url}}"
PortaBroker = 1883
KeepAliveBroker = 60
TopicoSubscribe = "/TEF/device001/cmd"

client = mqtt.Client()
client.connect(Broker, PortaBroker, KeepAliveBroker)
client.publish(TopicoSubscribe, "device001@on|")
```

### Código para **Desligar o LED** via MQTT:

```python
import paho.mqtt.client as mqtt
import sys

# Definições:
Broker = "{{url}}"
PortaBroker = 1883
KeepAliveBroker = 60
TopicoSubscribe = "/TEF/device001/cmd"

client = mqtt.Client()
client.connect(Broker, PortaBroker, KeepAliveBroker)
client.publish(TopicoSubscribe, "device001@off|")
```

## Código para Solicitação de Dados Temporais via Método GET HTTP

Aqui está o código Python para obter os dados temporais do potenciômetro via uma solicitação HTTP e exibir um gráfico dos dados ao longo do tempo.

```python
import requests
import matplotlib.pyplot as plt
import numpy as np

# Função para obter os dados de luminosidade a partir da API
def obter_dados_potentiometer(lastN):
    url = f"http://{{url}}:8666/STH/v1/contextEntities/type/device/id/urn:ngsi-ld:device:001/attributes/potentiometer?lastN={lastN}"

    headers = {
        'fiware-service': 'smart',
        'fiware-servicepath': '/'
    }

    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        data = response.json()
        luminosity_data = data['contextResponses'][0]['contextElement']['attributes'][0]['values']
        return luminosity_data
    else:
        print(f"Erro ao obter dados: {response.status_code}")
        return []

# Função para criar e exibir o gráfico
def plotar_grafico(potentiometer_data):
    if not potentiometer_data:
        print("Nenhum dado disponível para plotar.")
        return

    potentiometer = [entry['attrValue'] for entry in potentiometer_data]
    tempos = [entry['recvTime'] for entry in potentiometer_data]

    plt.figure(figsize=(12, 6))
    plt.plot(tempos, potentiometer, marker='o', linestyle='-', color='r')
    plt.title('Gráfico de medições do potenciômetro em função do Tempo')
    plt.xlabel('Tempo')
    plt.ylabel('Valores')
    plt.xticks(rotation=90)
    plt.grid(True)
    plt.tight_layout()
    plt.show()

# Solicitar ao usuário um valor "lastN" entre 1 e 100
```

## Referências

- [Documentação do MQTT](https://mqtt.org/)
- [Biblioteca paho-mqtt para Python](https://pypi.org/project/paho-mqtt/)
- [MyMQTT App para Android](https://play.google.com/store/apps/details?id=at.tripwire.mqtt.client)

## Integrantes

- **Caio Rossini** - RM: 555084
- **Lucas Serrano** - RM: 555170
- **Pedro Henrique Nobre** - RM: 557454
- **Thomaz Neves** - RM: 557789

