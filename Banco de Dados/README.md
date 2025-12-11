# Apresentação da Solução
## Como rodar o projeto:

### 1. Configuração do Banco
O sistema utiliza o Google Cloud SQL. É necessário criar um banco chamado `IoT` e liberar o IP nas configurações de rede.

### 2. Configuração do Python
Instale as dependências:
pip install paho-mqtt mysql-connector-python

Edite o arquivo `main.py` com suas credenciais do banco e execute:
python main.py


## Código ultilizado:
import paho.mqtt.client as mqtt
import mysql.connector
import sys

# CONFIGURAÇÕES DO GOOGLE CLOUD SQL 
CLOUD_SQL_IP = '34.41.42.27'       
CLOUD_SQL_SENHA = 'SUA_SENHA_AQUI' 

# Configurações do Avião
TOPICO_ALVO = "aviao/sensor/temperaturabmp"
BROKER = "test.mosquitto.org"
PORTA = 1883

print(f"Tentando conectar ao Google Cloud SQL no IP: {CLOUD_SQL_IP}...")

try:
    con = mysql.connector.connect(
        host=CLOUD_SQL_IP,
        user='root',
        password=CLOUD_SQL_SENHA,
        database='IoT' 
    )
except mysql.connector.Error as err:
    print(f"ERRO DE CONEXÃO: {err}")
    print("Dica: Verifique se o IP está correto e se liberou '0.0.0.0/0' nas Redes.")
    sys.exit()

if con.is_connected():
    print(">>> SUCESSO! Conectado ao banco de dados do Google. <<<")
    cursor = con.cursor()
    
    
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS dadosIoT (
        id INT AUTO_INCREMENT, 
        mensagem TEXT(512), 
        data_hora TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id)
    );
    """)

def on_connect(client, userdata, flags, rc):
    print(f"Conectado ao Broker MQTT! Assinando: {TOPICO_ALVO}")
    client.subscribe(TOPICO_ALVO)

def on_message(client, userdata, msg):
    mensagem = str(msg.payload.decode())
    print(f"Recebido do Avião: {mensagem}°C")

    try:
        # Reconecta o cursor para garantir
        cursor = con.cursor()
        sql = "INSERT INTO dadosIoT (mensagem) VALUES (%s)"
        cursor.execute(sql, (mensagem,))
        con.commit()
        print(">> Enviado para a nuvem Google Cloud!")
    except mysql.connector.Error as err:
        print(f"Erro ao salvar na nuvem: {err}")

if __name__ == '__main__':
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message
    client.connect(BROKER, PORTA, 60)
    client.loop_forever()

## Imagens do funcionamento do banco de dados:
<img width="597" height="267" alt="funcionamento 4" src="https://github.com/user-attachments/assets/ae56f65e-6220-42e1-a32b-c93bcc47c051" />
<img width="508" height="756" alt="funcionamento 3" src="https://github.com/user-attachments/assets/1d3910b9-e2ec-40d2-bc66-9fd215f8062d" />
<img width="1916" height="1018" alt="funcionamento 2" src="https://github.com/user-attachments/assets/3c858a0d-9c6c-4e22-9c65-65812cd19fb8" />
<img width="1919" height="801" alt="funcionamento 1" src="https://github.com/user-attachments/assets/fb936580-271c-492c-9ebf-a270da08d0b7" />
