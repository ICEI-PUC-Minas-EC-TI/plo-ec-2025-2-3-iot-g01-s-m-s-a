# Código-Fonte do Firmware (ESP32)

Este diretório contém o código-fonte C++ desenvolvido para o microcontrolador **ESP32**, responsável por gerenciar todo o sistema de segurança da aeronave. O projeto foi estruturado utilizando o **Visual Studio Code** com a extensão **PlatformIO**.

## Estrutura dos Arquivos

* **`Código Fonte.cpp`**: Arquivo principal contendo a lógica do sistema. Inclui:
    * `setup()`: Inicialização de sensores (I2C), conexão WiFi, configuração de pinos PWM/Servos e WebServer.
    * `loop()`: Gerenciamento de multitarefas (Timers não-bloqueantes), leitura de sensores, filtro complementar (IMU) e comunicação MQTT/WebSocket.
* **`index.html`**: Arquivo de interface (Frontend) hospedado na memória do ESP32 via LittleFS. Contém o código HTML/JavaScript responsável por renderizar o **Horizonte Artificial 3D** e conectar-se ao WebSocket.
* **`platformio.ini`**: Arquivo de configuração do ambiente de desenvolvimento. Define a placa (board), velocidade do monitor serial e as bibliotecas externas necessárias para compilação.

## Dependências e Bibliotecas

Para compilar este projeto, as seguintes bibliotecas são utilizadas (gerenciadas automaticamente pelo `platformio.ini`):

1.  **Sensores:**
    * `MPU6500_WE`: Leitura e calibração do Acelerômetro e Giroscópio.
    * `Adafruit_BMP280`: Leitura de Pressão Atmosférica e Temperatura.
2.  **Conectividade:**
    * `PubSubClient`: Protocolo MQTT para envio de telemetria e recebimento de comandos.
    * `ESPAsyncWebServer` & `AsyncTCP`: Servidor Web assíncrono para hospedar a interface local.
    * `WebSocketsServer`: Comunicação em tempo real para o Horizonte Artificial.
3.  **Atuadores:**
    * `ESP32Servo`: Controle preciso dos servomotores (Portas e Máscaras).

## Principais Funcionalidades Implementadas

### 1. Multitarefa
O código não utiliza `delay()` no loop principal. Foram implementados timers baseados em `millis()` para separar tarefas críticas:
* **100ms (Alta Prioridade):** Leitura do IMU, cálculo de vibração e atualização do WebSocket.
* **1000ms (Baixa Prioridade):** Leitura de Pressão/Temperatura (BMP280) e envio de telemetria pesada via MQTT.

### 2. Filtro Complementar (IMU)
Implementação matemática para fusão de sensores (Acelerômetro + Giroscópio) com fator `ALPHA = 0.96`, eliminando o ruído do acelerômetro e o *drift* do giroscópio para obter ângulos de *Pitch*, *Roll* e *Yaw* estáveis.

### 3. Interface Web (Horizonte Artificial 3D)
O sistema hospeda uma página HTML na memória flash do ESP32 (via sistema de arquivos **LittleFS**).
* **Representação 3D:** A página renderiza um modelo 3D da aeronave que se move em tempo real.
* **Comunicação:** Utiliza **WebSockets** (porta 81) para enviar pacotes JSON contendo os ângulos de Euler calculados pelo filtro. Isso permite que o avião na tela imite instantaneamente os movimentos físicos do protótipo (Digital Twin).

### 4. Modos de Simulação
Sistema de flags que permite simular emergências sem a necessidade de condições físicas reais, ativadas via MQTT ou Serial:
* 🔥 Incêndio (Luzes estroboscópicas + Sirene bitonal).
* 💨 Despressurização (Máscaras caem + Sirene contínua).
* ✈️ Turbulência (Alertas de vibração).
