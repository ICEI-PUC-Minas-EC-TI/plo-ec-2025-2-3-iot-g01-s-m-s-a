# Código do ESP-32

#include <Arduino.h>
#include <WiFi.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>
#include <WebSocketsServer.h>
#include <Wire.h>
#include <MPU6500_WE.h>
#include <Adafruit_BMP280.h>
#include "LittleFS.h"
#include <PubSubClient.h>
#include <ESP32Servo.h>

// CONFIGURAÇÕES DE HARDWARE E REDE
// Endereços I2C
#define MPU6500_ADDR 0x68
#define BMP280_ADDR  0x76

// Credenciais WiFi e MQTT
const char* ssid = "S23 de Flávio";       // SUBSTITUA SE NECESSÁRIO
const char* password = "senha1313";       // SUBSTITUA SE NECESSÁRIO

const char* BROKER_MQTT = "test.mosquitto.org";
const int BROKER_PORT = 1883;
const char* ID_MQTT = "esp32_aviao_seguranca_final";

// Definição dos Pinos
const int PIN_LED_LUZES = 2;      // LED de Alerta Visual
const int PIN_BUZZER = 4;         // Sirene 
const int PIN_PORTA_SERVO = 18;   // Servo 1 (Portas)
const int PIN_MASCARA_SERVO = 19; // Servo 2 (Máscaras)
const int CANAL_PWM_BUZZER = 12;  // Canal para controlar PWM buzzer

// Limiares de Sensores (Ajuste Fino)
const float LIMITE_PRESSAO_MINIMA = 900.0;  // hPa
const float LIMITE_TEMPERATURA_FOGO = 40.0; // °C
const float LIMITE_VIBRACAO = 0.5;          // G

// TÓPICOS MQTT
// Publicação
#define TOPICO_STATUS "aviao/sensor/status" 
#define TOPICO_PRESSAO "aviao/sensor/pressao"
#define TOPICO_TEMPERATURA_BMP "aviao/sensor/temperaturabmp"
#define TOPICO_TEMPERATURA_MPU "aviao/sensor/temperaturampu"
#define TOPICO_TURBULENCIA "aviao/sensor/turbulencia"
#define TOPICO_INCENDIO "aviao/sensor/incendio"

#define TOPICO_ACC_X "aviao/sensor/accx"
#define TOPICO_ACC_Y "aviao/sensor/accy"
#define TOPICO_ACC_Z "aviao/sensor/accz"
#define TOPICO_GYR_X "aviao/sensor/gyrx"
#define TOPICO_GYR_Y "aviao/sensor/gyry"
#define TOPICO_GYR_Z "aviao/sensor/gyrz"

#define TOPICO_ALERTA_INCENDIO "aviao/alerta/incendio"
#define TOPICO_ALERTA_DESPRESSURIZACAO "aviao/alerta/despressurizacao"
#define TOPICO_ALERTA_TURBULENCIA "aviao/alerta/turbulencia"

// Inscrição (Comandos)
#define TOPICO_SUBSCRIBE_LUZES  "aviao/atuador/luzes"
#define TOPICO_SUBSCRIBE_SIRENE "aviao/atuador/sirene"
#define TOPICO_SUBSCRIBE_PORTAS "aviao/atuador/portas"
#define TOPICO_SUBSCRIBE_MASCARAS "aviao/atuador/mascaras"
#define TOPICO_SUBSCRIBE_SIMULACAO_INCENDIO "aviao/simulacao/incendio"
#define TOPICO_SUBSCRIBE_SIMULACAO_DESPRESSURIZACAO "aviao/simulacao/despressurizacao"
#define TOPICO_SUBSCRIBE_SIMULACAO_TURBULENCIA "aviao/simulacao/turbulencia"

#define TOPICO_INFO_IP "aviao/info/ip"

// VARIÁVEIS GLOBAIS
Servo cleber;   // Servo Portas
Servo pryscila; // Servo Máscaras

MPU6500_WE mpu = MPU6500_WE(MPU6500_ADDR);
Adafruit_BMP280 bmp;

AsyncWebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);

WiFiClient espClient;
PubSubClient MQTT(espClient);

// VARIÁVEIS DO FILTRO
float anguloPitch = 0;
float anguloRoll = 0;
float anguloYaw = 0;
unsigned long lastTimeFilter = 0;
const float ALPHA = 0.96; 

int ledFase = 0; 
int sireneFase = 0;
int modoLedGlobal = 0;
int modoSireneGlobal = 0;
unsigned long ledTimer = 0;
unsigned long sireneTimer = 0;
bool estadoSirene = false;

bool lastStateIncendio = false;
bool lastStateDespressurizacao = false;
bool lastStateTurbulencia = false;

unsigned long timerRUIDO = 0;      
const unsigned long INTERVALO_RUIDO = 100; // 10Hz (Mantém animação fluida)

// Timer 2: Lento (Clima/Pressão/MQTT pesado)
unsigned long timerCLIMA = 0;      
const unsigned long INTERVALO_CLIMA = 1000; // 1Hz (Lê BMP a cada 1 segundo)

// Variáveis Globais para armazenar o último valor lido
float globalPressao = 1013.25;
float globalTempBMP = 25.0;
float globalVibracao = 0.0;

const unsigned long INTERVALO_LEITURA = 150; //(ajuste se precisar)

unsigned long ultimoTempoTurbulencia = 0; 
const unsigned long TEMPO_ESPERA_TURBULENCIA = 3000; // 5000ms = 5 segundos
unsigned long ultimoTempoDespressurizacao = 0;
const unsigned long TEMPO_ESPERA_DESPRESSURIZACAO = 3000; // 5000ms = 5 segundos
unsigned long ultimoTempoIcendio = 0;
const unsigned long TEMPO_ESPERA_INCENDIO = 3000; // 5000ms = 5 segundos

// Flags de Simulação (Serial)
bool simDespressurizacao = false;
bool simIncendio = false;
bool simTurbulencia = false;
bool manualPortaAberta = false;    // Guarda se a porta foi aberta via App
bool manualMascarasSoltas = false; // Guarda se as máscaras foram soltas via App

char msgBuffer[50];  
char jsonBuffer[256];

// Protótipo das Funções
void initMQTT();
void reconnectMQTT();
void mqtt_callback(char* topic, byte* payload, unsigned int length);
void VerificaConexoesWiFIEMQTT();
void monitoraSensores();
void gerenciarLeds(int modo);
void gerenciarSirene(int modo);
void verificarComandosSerial();
void atualizarFiltroIMU();
void conectarWiFiComEfeitos();

// SETUP
void setup() {
  Serial.begin(115200);
  Wire.begin();
  Wire.setClock(400000);
  
  // Configuração dos Servos
  cleber.setPeriodHertz(50);
  cleber.attach(PIN_PORTA_SERVO, 500, 2400);
  pryscila.setPeriodHertz(50);
  pryscila.attach(PIN_MASCARA_SERVO, 500, 2400);

  // Posição Inicial
  cleber.write(0);
  pryscila.write(50);

  // Configuração dos Pinos (LEDs e Buzzer)
  pinMode(PIN_LED_LUZES, OUTPUT);
  digitalWrite(PIN_LED_LUZES, LOW);
  // digitalWrite(PIN_BUZZER, LOW); // Comentado pois vamos usar PWM

  ledcSetup(CANAL_PWM_BUZZER, 2000, 8); // Configura canal de som
  ledcAttachPin(PIN_BUZZER, CANAL_PWM_BUZZER); // Liga o pino ao canal

  // Inicialização de Rede e Arquivos
  initMQTT();
  if (!LittleFS.begin()) {
    Serial.println("Aviso: Falha ao montar LittleFS");
  }

  conectarWiFiComEfeitos();

  // Inicialização de Sensores
  if (!bmp.begin(BMP280_ADDR)) {
    Serial.println("ERRO: BMP280 não encontrado!");
  } else {
    Serial.println("BMP280 OK");
  }

  if (!mpu.init()) {
    Serial.println("ERRO: MPU6500 não encontrado!");
  } else {
    Serial.println("MPU6500 OK");
    Serial.println("Calibrando... não mexa!");
    delay(1000);
    mpu.autoOffsets(); 
    Serial.println("Calibrado.");
  }
  
  // Servidor Web e WebSocket
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send(LittleFS, "/index.html", "text/html");
  });
  server.serveStatic("/", LittleFS, "/");
  server.begin();
  webSocket.begin();
  
  // Instruções no Serial
  Serial.println("\n--- SISTEMA PRONTO ---");
  Serial.println("Use o Monitor Serial para testes manuais:");
  Serial.println("- Digite '1' para simular despressurização");
  Serial.println("- Digite '2' para simular incêndio");
  Serial.println("- Digite '3' para simular turbulência");
  Serial.println("- Digite '4' para voltar ao normal");
  Serial.println("------------------------");
  
  lastTimeFilter = millis(); 
}

// LOOP PRINCIPAL
void loop() {
  VerificaConexoesWiFIEMQTT();
  MQTT.loop(); 
  webSocket.loop();
  verificarComandosSerial();
  
  // O filtro roda livre para garantir precisão nos ângulos
  atualizarFiltroIMU();

  unsigned long atual = millis();
  
  if (atual - timerRUIDO >= INTERVALO_RUIDO) {
    timerRUIDO = atual; 

    // LER DADOS BRUTOS (Faltava ler o Giroscópio aqui)
    xyzFloat acc = mpu.getGValues();
    xyzFloat gyr = mpu.getGyrValues(); // <--- ADICIONADO
    
    // CÁLCULO DE VIBRAÇÃO
    float gTotal = sqrt(acc.x * acc.x + acc.y * acc.y + acc.z * acc.z);
    globalVibracao = abs(gTotal - 1.0); 

    // MONITORAMENTO DE ALARMES
    monitoraSensores();

    // WEBSOCKET (Envia ÂNGULOS para o Horizonte Artificial)
    // Nota: O App espera "gyroX" mas enviamos o anguloRoll (radianos) para o desenho se mexer
    float toRad = 0.0174533;
    snprintf(jsonBuffer, sizeof(jsonBuffer), 
        "{\"gyroX\":%.4f,\"gyroY\":%.4f,\"gyroZ\":%.4f,\"tempBMP\":%.2f,\"pressao\":%.2f}",
        anguloRoll * toRad, 
        anguloPitch * toRad, 
        anguloYaw * toRad, 
        globalTempBMP, 
        globalPressao 
    );
    webSocket.broadcastTXT(jsonBuffer);
    
    //MQTT DADOS RÁPIDOS
    if(MQTT.connected()) {
         // Acelerômetro
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", acc.x); MQTT.publish(TOPICO_ACC_X, msgBuffer);
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", acc.y); MQTT.publish(TOPICO_ACC_Y, msgBuffer);
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", acc.z); MQTT.publish(TOPICO_ACC_Z, msgBuffer);
         
         // Giroscópio (Adicionado agora)
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", gyr.x); MQTT.publish(TOPICO_GYR_X, msgBuffer);
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", gyr.y); MQTT.publish(TOPICO_GYR_Y, msgBuffer);
         snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", gyr.z); MQTT.publish(TOPICO_GYR_Z, msgBuffer);
    }
  }

  if (atual - timerCLIMA >= INTERVALO_CLIMA) {
    timerCLIMA = atual;

    // Lê Sensores Lentos
    globalTempBMP = bmp.readTemperature();
    globalPressao = bmp.readPressure() / 100.0; 
    float tempMPU = mpu.getTemperature();

    // Envia MQTT Lento
    if(MQTT.connected()) {
        snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", globalTempBMP); MQTT.publish(TOPICO_TEMPERATURA_BMP, msgBuffer);
        snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", globalPressao); MQTT.publish(TOPICO_PRESSAO, msgBuffer);
        snprintf(msgBuffer, sizeof(msgBuffer), "%.2f", tempMPU); MQTT.publish(TOPICO_TEMPERATURA_MPU, msgBuffer);
    }
  }

  gerenciarLeds(modoLedGlobal);
  gerenciarSirene(modoSireneGlobal);
}

// IMPLEMENTAÇÃO DO FILTRO 
void atualizarFiltroIMU() {
  // Controle de tempo delta (dt)
  unsigned long tAtual = millis();
  float dt = (tAtual - lastTimeFilter) / 1000.0;
  lastTimeFilter = tAtual;

  // Lê dados brutos
  xyzFloat acc = mpu.getGValues();     
  xyzFloat gyr = mpu.getGyrValues();   

  // --- CORREÇÃO DO GIRO "FANTASMA" (DEADZONE) ---
  // Se a rotação for menor que 0.2 graus/s, consideramos como 0 (ruído)
  // Aumente para 0.5 ou 1.0 se continuar girando sozinho.
  if (abs(gyr.x) < 0.2) gyr.x = 0;
  if (abs(gyr.y) < 0.2) gyr.y = 0;
  if (abs(gyr.z) < 0.2) gyr.z = 0;

  // Calcula inclinação baseada APENAS na gravidade (Acelerômetro)
  float accPitch = atan2(acc.y, sqrt(acc.x * acc.x + acc.z * acc.z)) * 57.296;
  float accRoll  = atan2(-acc.x, sqrt(acc.y * acc.y + acc.z * acc.z)) * 57.296;

  // Filtro Complementar
  anguloPitch = ALPHA * (anguloPitch + gyr.y * dt) + (1.0 - ALPHA) * accPitch;
  anguloRoll  = ALPHA * (anguloRoll  + gyr.x * dt) + (1.0 - ALPHA) * accRoll;
  
  // Yaw (Esse é o que mais gira sozinho, pois não tem acelerômetro para corrigir)
  anguloYaw   = anguloYaw + gyr.z * dt;
}

// Leitura do Serial para ativar flags de simulação
void verificarComandosSerial() {
    if (Serial.available() > 0) {
        String comando = Serial.readStringUntil('\n');
        comando.trim(); 
        comando.toUpperCase(); 

        if (comando == "1") {
            simDespressurizacao = !simDespressurizacao;
            Serial.print("SIMULAÇÃO PRESSÃO: ");
            Serial.println(simDespressurizacao ? "ATIVADA" : "DESATIVADA");
        }
        else if (comando == "2") {
            simIncendio = !simIncendio;
            Serial.print("SIMULAÇÃO FOGO: ");
            Serial.println(simIncendio ? "ATIVADA" : "DESATIVADA");
        }
        else if (comando == "3") {
            simTurbulencia = !simTurbulencia;
            Serial.print("SIMULAÇÃO TURBULÊNCIA: ");
            Serial.println(simTurbulencia ? "ATIVADA" : "DESATIVADA");
        }
        else if (comando == "4") {
            simDespressurizacao = false;
            simIncendio = false;
            simTurbulencia = false;
            Serial.println("TODAS AS SIMULAÇÕES PARADAS. Lendo sensores reais.");
        }
    }
}

void monitoraSensores() {
    // 1. Atualiza Flags de Tempo baseadas nos sensores ou simulação
    if ((globalVibracao > LIMITE_VIBRACAO) || simTurbulencia) ultimoTempoTurbulencia = millis(); 
    if ((globalPressao < LIMITE_PRESSAO_MINIMA) || simDespressurizacao) ultimoTempoDespressurizacao = millis();
    if ((globalTempBMP > LIMITE_TEMPERATURA_FOGO) || simIncendio) ultimoTempoIcendio = millis();

    // 2. Determina o estado ATUAL (True/False)
    bool isTurbulencia = (millis() - ultimoTempoTurbulencia < TEMPO_ESPERA_TURBULENCIA);
    bool isDespressurizacao = (millis() - ultimoTempoDespressurizacao < TEMPO_ESPERA_DESPRESSURIZACAO);
    bool isIncendio = (millis() - ultimoTempoIcendio < TEMPO_ESPERA_INCENDIO);

    // 3. Controla Servos (Atuadores Físicos)
    if (manualPortaAberta) cleber.write(90); else cleber.write(0);                   
    if (isDespressurizacao || manualMascarasSoltas) pryscila.write(180); else pryscila.write(50);                      

    // 4. Lógica de Prioridade para LEDs e Sirene
    if (isIncendio) {
        modoLedGlobal = 3; modoSireneGlobal = 3; 
    }
    else if (isDespressurizacao) {
        modoLedGlobal = 1; modoSireneGlobal = 1; 
    }
    else if (isTurbulencia) {
        modoLedGlobal = 2; modoSireneGlobal = 2; 
    }
    else {
        modoLedGlobal = 0; modoSireneGlobal = 0;
    }
    
    // --- Check Incêndio ---
    if (isIncendio != lastStateIncendio) {
        lastStateIncendio = isIncendio; // Atualiza memória
        MQTT.publish(TOPICO_ALERTA_INCENDIO, isIncendio ? "true" : "false");
        Serial.print("Mudança Estado Incêndio: "); Serial.println(isIncendio);
    }

    // --- Check Despressurização ---
    if (isDespressurizacao != lastStateDespressurizacao) {
        lastStateDespressurizacao = isDespressurizacao;
        MQTT.publish(TOPICO_ALERTA_DESPRESSURIZACAO, isDespressurizacao ? "true" : "false");
         Serial.print("Mudança Estado Despress: "); Serial.println(isDespressurizacao);
    }

    // --- Check Turbulência ---
    if (isTurbulencia != lastStateTurbulencia) {
        lastStateTurbulencia = isTurbulencia;
        MQTT.publish(TOPICO_ALERTA_TURBULENCIA, isTurbulencia ? "true" : "false");
         Serial.print("Mudança Estado Turbulencia: "); Serial.println(isTurbulencia);
    }
}

// Gerenciador de Padrão de LED
void gerenciarLeds(int modo) {
    // MODO 0: Tudo Apagado
    if (modo == 0) {
        digitalWrite(PIN_LED_LUZES, LOW);
        ledFase = 0; 
        return;
    }

    unsigned long atual = millis();

    // MODO 1: EMERGÊNCIA CRÍTICA
    if (modo == 1) {
        switch (ledFase) {
            case 0: 
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 2000) { ledTimer = atual; ledFase = 1; }
                break;
            case 1: 
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 250) { ledTimer = atual; ledFase = 2; }
                break;
            case 2: 
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 500) { ledTimer = atual; ledFase = 3; }
                break;
            case 3: 
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 250) { ledTimer = atual; ledFase = 0; } 
                break;
            default: ledFase = 0; break;
        }
    }
    
    // MODO 2: TURBULÊNCIA
    else if (modo == 2) {
        switch (ledFase) {
            case 0: 
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 200) { ledTimer = atual; ledFase = 1; }
                break;
            case 1: 
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 200) { ledTimer = atual; ledFase = 2; }
                break;
            case 2: 
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 200) { ledTimer = atual; ledFase = 3; }
                break;
            case 3: 
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 1000) { ledTimer = atual; ledFase = 0; } 
                break;
            default: ledFase = 0; break; 
        }
    }
    // MODO 
    else if (modo == 3) {
        switch (ledFase) {
            case 0: // Flash 1
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 50) { ledTimer = atual; ledFase = 1; }
                break;
            case 1: // Intervalo
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 50) { ledTimer = atual; ledFase = 2; }
                break;
            case 2: // Flash 2
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 50) { ledTimer = atual; ledFase = 3; }
                break;
            case 3: // Intervalo
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 50) { ledTimer = atual; ledFase = 4; }
                break;
            case 4: // Flash 3
                digitalWrite(PIN_LED_LUZES, HIGH);
                if (atual - ledTimer >= 50) { ledTimer = atual; ledFase = 5; }
                break;
            case 5: // Pausa Longa
                digitalWrite(PIN_LED_LUZES, LOW);
                if (atual - ledTimer >= 800) { ledTimer = atual; ledFase = 0; } // Reinicia
                break;
            default: ledFase = 0; break; 
        }
    }
}

// Gerenciador de Sirene (USANDO LEDC/PWM para não travar Servo)
void gerenciarSirene(int modo) {
    if (modo == 0) {
        ledcWrite(CANAL_PWM_BUZZER, 0); // Mudo
        estadoSirene = false;
        sireneFase = 0;
        return;
    }

    unsigned long atual = millis();
    unsigned long intervalo;
    int frequencia;

    //Despressurizacao
    if(modo == 1) {
      intervalo = 100;
      frequencia = 2000;
      if (atual - sireneTimer >= intervalo) {
            sireneTimer = atual;
            estadoSirene = !estadoSirene; 
            if (estadoSirene) {
                ledcSetup(CANAL_PWM_BUZZER, frequencia, 8);
                ledcWrite(CANAL_PWM_BUZZER, 128); // Volume médio
            } else {
                ledcWrite(CANAL_PWM_BUZZER, 0); // Silêncio
            }
        }
    }

    // Turbulencia
    else if(modo == 2) {
      intervalo = 500;
      frequencia = 1000;
      if (atual - sireneTimer >= intervalo) {
            sireneTimer = atual;
            estadoSirene = !estadoSirene; 
            if (estadoSirene) {
                ledcSetup(CANAL_PWM_BUZZER, frequencia, 8);
                ledcWrite(CANAL_PWM_BUZZER, 128); // Volume médio
            } else {
                ledcWrite(CANAL_PWM_BUZZER, 0); // Silêncio
            }
        }
    }

    // Incendio 
    else if (modo == 3) { 
        if (atual - sireneTimer >= 400) { 
            sireneTimer = atual; 
            estadoSirene = !estadoSirene; 
            ledcSetup(CANAL_PWM_BUZZER, estadoSirene ? 1200 : 800, 8); 
            ledcWrite(CANAL_PWM_BUZZER, 128);
        }
    }
}

// FUNÇÕES DE REDE (MQTT)
void initMQTT() {
    MQTT.setServer(BROKER_MQTT, BROKER_PORT);
    MQTT.setCallback(mqtt_callback);
}

void reconnectMQTT() {
    while (!MQTT.connected()) {
        // --- SEGURANÇA: ESCAPE SE O WIFI CAIR ---
        if(WiFi.status() != WL_CONNECTED) {
          Serial.println("WiFi caiu durante tentativa MQTT. Abortando...");
          return; 
        }

        Serial.print("* Tentando MQTT... ");
        
        // Tenta conectar
        if (MQTT.connect(ID_MQTT)) {
            Serial.println("Conectado!");
            
            // --- CORREÇÃO: O ENVIO DO IP TEM QUE SER AQUI (DEPOIS DE CONECTAR) ---
            MQTT.publish(TOPICO_INFO_IP, WiFi.localIP().toString().c_str());
            Serial.print("IP enviado para o topico: ");
            Serial.println(WiFi.localIP());
            // ---------------------------------------------------------------------

            MQTT.publish(TOPICO_STATUS, "ESP32 Online");
            
            // Reinscreve nos tópicos
            MQTT.subscribe(TOPICO_SUBSCRIBE_LUZES);
            MQTT.subscribe(TOPICO_SUBSCRIBE_SIRENE);
            MQTT.subscribe(TOPICO_SUBSCRIBE_PORTAS);
            MQTT.subscribe(TOPICO_SUBSCRIBE_MASCARAS);
            MQTT.subscribe(TOPICO_SUBSCRIBE_SIMULACAO_INCENDIO);
            MQTT.subscribe(TOPICO_SUBSCRIBE_SIMULACAO_DESPRESSURIZACAO);
            MQTT.subscribe(TOPICO_SUBSCRIBE_SIMULACAO_TURBULENCIA);
            
        } else {
            Serial.print("Falha rc=");
            Serial.print(MQTT.state());
            Serial.println(" tentando em 2s");
            delay(2000);
        }
    }
}

void VerificaConexoesWiFIEMQTT() {
    // PRIMEIRO: Verifica se o WiFi caiu
    if (WiFi.status() != WL_CONNECTED) {
        Serial.println("Detectado queda de WiFi!");
        conectarWiFiComEfeitos(); // Entra no loop de bips e luzes até voltar
        return; // Sai da função para não tentar MQTT sem internet
    }

    // DEPOIS: Se WiFi está OK, verifica MQTT
    if (!MQTT.connected()) {
        reconnectMQTT();
    }
}

void mqtt_callback(char* topic, byte* payload, unsigned int length) {
    String msg = "";
    for (int i = 0; i < length; i++) msg += (char)payload[i];
    
    if (String(topic) == TOPICO_SUBSCRIBE_PORTAS) {
        if (msg == "A" || msg == "abrir") {
            manualPortaAberta = true; 
            Serial.println("MQTT: Porta ABERTA");
        }
        if (msg == "F" || msg == "fechar") {
            manualPortaAberta = false; 
            Serial.println("MQTT: Porta FECHADA");
        }
    }

    if (String(topic) == TOPICO_SUBSCRIBE_MASCARAS) {
        if (msg == "L" || msg == "liberar") {
            manualMascarasSoltas = true;
            Serial.println("MQTT: Máscaras LIBERADAS");
        }
        if (msg == "R" || msg == "recolher") {
            manualMascarasSoltas = false;
            Serial.println("MQTT: Máscaras RECOLHIDAS");
        }
    } 
    
    if(String(topic) == TOPICO_SUBSCRIBE_SIMULACAO_DESPRESSURIZACAO) {
        if(msg == "S") {
            simDespressurizacao = true;
            Serial.println("SIMULAÇÃO PRESSÃO: ATIVADA ");
        }
        if(msg == "N") {
            simDespressurizacao = false;
            Serial.println("SIMULAÇÃO PRESSÃO: DESATIVADA ");
        }
    }
    
    if(String(topic) == TOPICO_SUBSCRIBE_SIMULACAO_INCENDIO) {
        if(msg == "S") {
            simIncendio = true;
            Serial.println("SIMULAÇÃO FOGO: ATIVADA ");
        }
        if(msg == "N") {
            simIncendio = false;
            Serial.println("SIMULAÇÃO FOGO: DESATIVADA ");
        }
    }
    if(String(topic) == TOPICO_SUBSCRIBE_SIMULACAO_TURBULENCIA) {
        if(msg == "S") {
            simTurbulencia = true;
            Serial.println("SIMULAÇÃO TURBULÊNCIA: ATIVADA ");
        }
        if(msg == "N") {
            simTurbulencia = false;
            Serial.println("SIMULAÇÃO TURBULÊNCIA: DESATIVADA ");
        }
        if(msg == "Desativar simulacao") {
            simDespressurizacao = false;
            simIncendio = false;
            simTurbulencia = false;
            Serial.println("TODAS AS SIMULAÇÕES PARADAS. Lendo sensores reais.");
        }
    }
}

void conectarWiFiComEfeitos() {
  // Se já estiver conectado, sai imediatamente
  if (WiFi.status() == WL_CONNECTED) return;

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.println("\nWiFi perdido! Iniciando reconexão...");

  // LOOP COM EFEITOS ENQUANTO TENTA CONECTAR
  while (WiFi.status() != WL_CONNECTED) {
    // Liga LED e Som (Bip Agudo Curto estilo Radar)
    digitalWrite(PIN_LED_LUZES, HIGH); 
    ledcSetup(CANAL_PWM_BUZZER, 2500, 8); 
    ledcWrite(CANAL_PWM_BUZZER, 100);     
    delay(50);                            
    
    // Desliga LED e Som
    digitalWrite(PIN_LED_LUZES, LOW);
    ledcWrite(CANAL_PWM_BUZZER, 0);       
    
    Serial.print(".");
    delay(450); // Completa 500ms
  }

  Serial.println("\nWiFi conectado!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // EFEITO DE SUCESSO (Drone)
  int melodia[] = {523, 659, 784, 1046}; 
  for (int i = 0; i < 4; i++) {
    ledcSetup(CANAL_PWM_BUZZER, melodia[i], 8); 
    ledcWrite(CANAL_PWM_BUZZER, 150);           
    digitalWrite(PIN_LED_LUZES, HIGH);          
    delay(100);                                 
    ledcWrite(CANAL_PWM_BUZZER, 0);             
    digitalWrite(PIN_LED_LUZES, LOW);           
    delay(30);                                  
  }
  
  // Bip final
  delay(50);
  ledcSetup(CANAL_PWM_BUZZER, 2000, 8);
  ledcWrite(CANAL_PWM_BUZZER, 150);
  delay(100);
  ledcWrite(CANAL_PWM_BUZZER, 0);
}
