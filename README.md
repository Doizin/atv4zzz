# atv4zzz
#include <WiFi.h>
#include <PubSubClient.h>

// Configurações Wi-Fi
const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configurações MQTT
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic_nivel = "wokwi/jkan/test-0/nivel";
const char* mqtt_topic_atuador = "wokwi/jkan/test-0/alerta";

// Pinos
const int ledPin = 2; // LED embutido no ESP32 (GPIO2)

// MQTT
WiFiClient espClient;
PubSubClient client(espClient);

// Variáveis de tempo
unsigned long tempoSensorAntes;
unsigned long tempoSensorDepois;
unsigned long tempoMsgRecebida;
unsigned long tempoAcionarLED;

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando no Wi-Fi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi conectado.");
}

void callback(char* topic, byte* payload, unsigned int length) {
  tempoMsgRecebida = millis();
  
  Serial.print("Mensagem recebida no topico: ");
  Serial.println(topic);

  String message;
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }

  Serial.print("Mensagem: ");
  Serial.println(message);

  if (String(topic) == mqtt_topic_atuador) {
    if (message == "on") {
      digitalWrite(ledPin, HIGH);
    } else {
      digitalWrite(ledPin, LOW);
    }

    tempoAcionarLED = millis();
    Serial.print("Tempo Broker → Atuador (LED): ");
    Serial.print(tempoAcionarLED - tempoMsgRecebida);
    Serial.println(" ms");
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Tentando conectar no MQTT...");
    if (client.connect("ESP32Client")) {
      Serial.println("Conectado");
      client.subscribe(mqtt_topic_atuador);
    } else {
      Serial.print("Falha, rc=");
      Serial.print(client.state());
      Serial.println(" Tentando novamente em 5 segundos");
      delay(5000);
    }
  }
}

void setup() {
  pinMode(ledPin, OUTPUT);
  Serial.begin(115200);

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Simular leitura do sensor
  tempoSensorAntes = millis();
  int nivel = random(10, 100); // Simula um valor de nível de água
  delay(100); // Simula tempo de leitura do sensor
  tempoSensorDepois = millis();

  Serial.print("Nivel de agua: ");
  Serial.println(nivel);

  Serial.print("Tempo Sensor → Leitura: ");
  Serial.print(tempoSensorDepois - tempoSensorAntes);
  Serial.println(" ms");

  // Publicar no MQTT
  client.publish(mqtt_topic_nivel, String(nivel).c_str());
  unsigned long tempoPublicacao = millis();
  Serial.print("Tempo Leitura → Publicacao MQTT: ");
  Serial.print(tempoPublicacao - tempoSensorDepois);
  Serial.println(" ms");

  delay(5000); // Espera antes da próxima leitura
}
