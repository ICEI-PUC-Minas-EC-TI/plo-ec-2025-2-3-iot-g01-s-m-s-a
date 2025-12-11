# Interface de Monitoramento Móvel (Dashboard)

Este diretório contém os arquivos de configuração para a interface móvel do projeto. Utilizamos um aplicativo MQTT para criar um **Dashboard** que permite visualizar a telemetria do avião em tempo real e enviar comandos de simulação.

## Aplicativo Utilizado

Para utilizar a interface, é necessário instalar o seguinte aplicativo para smartphone (Android):

* **Nome:** MQTT Dashboard Client
* **Desenvolvedor:** Doikov Evgenii
* **Link:** [Disponível na Google Play Store](https://play.google.com/store/apps/details?id=com.doikov.mqttclient)

## Arquivos Neste Diretório

* **`MqttDashboardClient_Backup`** : Este arquivo contém o layout completo e pré-configurado de todos os painéis, botões e gráficos mostrados nas demonstrações do projeto.

## Funcionalidades do Painel

Ao importar o arquivo de configuração, você terá acesso imediato às seguintes abas e controles:

### 1. Cabine e Ambiente
* **Telemetria:** Temperatura Interna (MPU), Temperatura Externa (BMP) e Pressão Atmosférica.
* **Controle Manual de Atuadores:** Botões interativos para acionamento direto:
    * **Portas:** Switch para Abrir/Fechar a porta manualmente.
    * **Máscaras:** Switch para Liberar/Recolher as máscaras de oxigênio.

### 2. Alertas de Segurança
* **Indicadores de Emergência:** Luzes de alerta que se ativam automaticamente em caso de:
    * 🔥 Incêndio
    * 💨 Despressurização
    * ✈️ Turbulência

### 3. Sensores Inerciais (IMU)
* **Acelerômetro e Giroscópio:** Leitura em tempo real dos eixos X, Y e Z para monitoramento da estabilidade da aeronave.

### 4. Controle de Simulação
* **Botões de Teste:** Switches para ativar/desativar manualmente os modos de emergência (Incêndio, Despressurização, Turbulência) para fins de apresentação e testes de bancada.

### 5. Conectividade
* **WiFi IP:** Exibe o endereço IP atual do ESP32 para facilitar o acesso à página Web do Horizonte Artificial.

---

## Como Importar o Painel

1.  Baixe o arquivo de configuração disponível nesta pasta para o seu celular.
2.  Extraia o arquivo compactado, para obter o arquivo original.
3.  Abra o aplicativo **MQTT Dashboard**.
4.  Vá no menu de configurações e selecione **"Restore / Import"** (ou similar).
5.  Selecione o arquivo baixado.
6.  O painel carregará automaticamente com todos os tópicos MQTT já configurados.

> **Nota:** Certifique-se de configurar o *Broker* (endereço do servidor MQTT) no aplicativo caso ele não seja importado automaticamente com o arquivo.
