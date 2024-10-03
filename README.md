# Projeto IoT: Leitura de Dados do DHT11 e LDR com ESP32 e MyMQTT

## Descrição do Projeto

Este projeto demonstra a captura de dados de sensores ambientais, utilizando o sensor de temperatura e umidade DHT11 e o sensor de luminosidade LDR, com um ESP32. Os dados capturados são enviados via protocolo MQTT, e podem ser lidos e controlados por um aplicativo MQTT, como o MyMQTT. 

O objetivo é proporcionar uma solução simples e funcional de monitoramento IoT, com interface amigável e dados em tempo real, acessíveis via dispositivos móveis.

## Tabela de Conteúdo
1. [Pré-requisitos](#pré-requisitos)
2. [Diagrama de Circuito](#diagrama-de-circuito)
3. [Configuração do Ambiente](#configuração-do-ambiente)
4. [Configuração MQTT](#configuração-mqtt)
5. [Código Fonte](#código-fonte)
6. [Execução do Projeto](#execução-do-projeto)
7. [Resultados](#resultados)
8. [Possíveis Melhorias](#possíveis-melhorias)
9. [Referências](#referências)

## Pré-requisitos

Antes de começar, você precisará dos seguintes componentes:

### Hardware:
- ESP32
- Sensor DHT11 (Temperatura e Umidade)
- Sensor LDR (Resistor Dependente de Luz)
- Resistores de 10KΩ e 220Ω (para o LDR)
- Protoboard e Jumpers

### Software:
- Arduino IDE (com suporte ao ESP32)
- Biblioteca "DHT sensor library" (Adafruit)
- Biblioteca "PubSubClient" (para MQTT)
- Aplicativo MyMQTT (disponível para Android)

## Diagrama de Circuito

Monte o circuito conforme o diagrama abaixo:

### Conexões:
- **DHT11**: 
  - VCC -> 3.3V
  - GND -> GND
  - DATA -> GPIO 4 (ESP32)
  
- **LDR**:
  - Um lado do LDR conectado ao 3.3V e o outro ao GPIO 34, com um resistor de 10KΩ conectado entre o GPIO 34 e o GND.

- **ESP32**: 
  - Alimentado por USB ou fonte de 3.3V.

### [Diagrama Esquemático] (adicione um link ou imagem aqui)

## Configuração do Ambiente

### Passo 1: Configurando a IDE do Arduino para ESP32
1. Abra a IDE Arduino.
2. Vá em **Arquivo > Preferências**.
3. Adicione o seguinte URL no campo "URLs adicionais para gerenciadores de placas":
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

Configure o MyMQTT para se conectar ao broker, inserindo o endereço IP do servidor MQTT, porta (geralmente 1883), e tópicos de publicação e assinatura.

## Configuração MQTT

Os dados serão publicados nos seguintes tópicos:
- `iot/esp32/dht11` – Publica a temperatura e a umidade lidas pelo DHT11.
- `iot/esp32/ldr` – Publica os dados de luminosidade lidos pelo LDR.

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

## Execução do Projeto

1. Monte o circuito conforme o diagrama.
2. Faça o upload do código para o ESP32 via Arduino IDE.
3. Abra o monitor serial para verificar a conexão e o envio dos dados MQTT.
4. No aplicativo MyMQTT:
   - Conecte-se ao broker MQTT.
   - Assine os tópicos `iot/esp32/dht11` e `iot/esp32/ldr` para visualizar os dados enviados pelo ESP32.
   
## Resultados

Os dados de temperatura, umidade e luminosidade serão atualizados no aplicativo MyMQTT em tempo real. A partir disso, você poderá monitorar o ambiente de forma remota, visualizar e analisar as leituras dos sensores.

## Possíveis Melhorias

- Implementação de sensores adicionais (por exemplo, sensores de gás ou movimento).
- Controle de dispositivos por comandos via MQTT (ex.: ligar/desligar luzes).
- Desenvolvimento de interface gráfica no MyMQTT ou aplicativo personalizado.

## Referências

- [ESP32 MQTT Tutorial](https://randomnerdtutorials.com/esp32-mqtt-publish-subscribe-arduino-ide/)
- [MyMQTT Android App](https://play.google.com/store/apps/details?id=at.tripwire.mqtt.client)

#Integrantes:
Caio Soares Rossini, Lucas Serrano, Thomaz Neves e Pedro Henrique Nobre 
RM: 555084, 555170,557789 e 557454
