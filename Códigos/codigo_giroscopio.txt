#include <WiFi.h>
#include <ArduinoWebsockets.h>
#include <Wire.h>
#include <MPU6050.h>

using namespace websockets;

MPU6050 mpu;
WebsocketsClient client;

// Wi-Fi do robô
const char* ssid = "Carrin";
const char* password = "87654321";

// IP do robô
const char* websocket_server = "ws://192.168.4.1/ws";


int16_t last_ax = 0, last_ay = 0, last_gz = 0;
unsigned long lastSendTime = 0;
const int SEND_INTERVAL = 100; // ms

// Ajustáveis
const int ACCEL_THRESHOLD = 8000;    // Sensibilidade aceleração
const int GYRO_THRESHOLD = 5000;     // Sensibilidade giroscópio
const int DIAGONAL_THRESHOLD = 6000; // Sensibilidade diagonal

// Variáveis para conexão
unsigned long lastReconnectAttempt = 0;
const int RECONNECT_INTERVAL = 5000; // Tentar reconectar a cada 5 segundos
bool wasConnected = false;

void setup() 
{
  Serial.begin(115200);
  Wire.begin();

  // Inicializa MPU6050
  mpu.initialize();
  if(!mpu.testConnection())
  {
    Serial.println("MPU6050 falhou!");
    while(1);
  }
  
  // Configuração do MPU6050
  mpu.setFullScaleAccelRange(MPU6050_ACCEL_FS_2);  // ±2g
  mpu.setFullScaleGyroRange(MPU6050_GYRO_FS_250);  // ±250°/s
  mpu.setDLPFMode(MPU6050_DLPF_BW_5);

  // Conecta WiFi
  WiFi.begin(ssid, password);
  Serial.print("Conectando ao WiFi");
  while(WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConectado ao WiFi");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // Conecta WebSocket
  connectWebSocket();
}

void connectWebSocket() {
  Serial.print("Conectando ao WebSocket... ");
  bool connected = client.connect(websocket_server);
  if(connected) {
    Serial.println("Conectado!");
    wasConnected = true;
  } else {
    Serial.println("Falha!");
    wasConnected = false;
  }
}

String detectMovement(int16_t ax, int16_t ay, int16_t gz) {
  String comando = "0"; // Parado por padrão
  
  // Filtro simples para reduzir ruído
  ax = (ax + last_ax * 2) / 3;
  ay = (ay + last_ay * 2) / 3;
  gz = (gz + last_gz * 2) / 3;
  
  // Atualiza histórico
  last_ax = ax;
  last_ay = ay;
  last_gz = gz;
  
  // DETECÇÃO DE ROTAÇÃO (Giroscópio Z)
  if (abs(gz) > GYRO_THRESHOLD) {
    if (gz > GYRO_THRESHOLD) {
      return "9";  // Giro anti-horário (TURN_LEFT)
    } else if (gz < -GYRO_THRESHOLD) {
      return "10"; // Giro horário (TURN_RIGHT)
    }
  }
  
  // DETECÇÃO DE MOVIMENTOS LINEARES E DIAGONAIS
  bool moveX = abs(ax) > ACCEL_THRESHOLD;
  bool moveY = abs(ay) > ACCEL_THRESHOLD;
  
  if (moveX && moveY) {
    // MOVIMENTO DIAGONAL
    if (ax > DIAGONAL_THRESHOLD && ay > DIAGONAL_THRESHOLD) {
      comando = "6";  // Diagonal superior direita (UP_RIGHT)
    } 
    else if (ax > DIAGONAL_THRESHOLD && ay < -DIAGONAL_THRESHOLD) {
      comando = "8";  // Diagonal inferior direita (DOWN_RIGHT)
    } 
    else if (ax < -DIAGONAL_THRESHOLD && ay > DIAGONAL_THRESHOLD) {
      comando = "5";  // Diagonal superior esquerda (UP_LEFT)
    } 
    else if (ax < -DIAGONAL_THRESHOLD && ay < -DIAGONAL_THRESHOLD) {
      comando = "7";  // Diagonal inferior esquerda (DOWN_LEFT)
    }
  } 
  else if (moveY) {
    // MOVIMENTO VERTICAL
    if (ay > ACCEL_THRESHOLD) {
      comando = "1";  // Para frente (UP)
    } else if (ay < -ACCEL_THRESHOLD) {
      comando = "2";  // Para trás (DOWN)
    }
  } 
  else if (moveX) {
    // MOVIMENTO HORIZONTAL
    if (ax > ACCEL_THRESHOLD) {
      comando = "4";  // Para direita (RIGHT)
    } else if (ax < -ACCEL_THRESHOLD) {
      comando = "3";  // Para esquerda (LEFT)
    }
  }
  
  return comando;
}

void loop() 
{
  // Verifica e mantém conexão WebSocket
  unsigned long currentMillis = millis();
  
  if (!client.available()) {
    if (wasConnected) {
      Serial.println("WebSocket desconectado!");
      wasConnected = false;
    }
    
    // Tenta reconectar periodicamente
    if (currentMillis - lastReconnectAttempt >= RECONNECT_INTERVAL) {
      lastReconnectAttempt = currentMillis;
      connectWebSocket();
    }
    
    // Aguarda antes de tentar novamente
    delay(100);
    return;
  }
  
  // Processa mensagens WebSocket
  client.poll();
  
  // Lê dados do MPU6050
  int16_t ax, ay, az, gx, gy, gz;
  mpu.getMotion6(&ax, &ay, &az, &gx, &gy, &gz);
  
  // Detecta movimento
  String comando = detectMovement(ax, ay, gz);
  
  // Envia comando apenas se mudou ou após intervalo
  static String lastCommand = "0";
  
  if (comando != lastCommand || (currentMillis - lastSendTime) > SEND_INTERVAL) {
    if (client.available()) {
      client.send(comando);
    }
    
    // Debug no monitor serial
    if (comando != lastCommand) {
      Serial.print("Comando: ");
      Serial.print(comando);
      Serial.print(" | AX: ");
      Serial.print(ax);
      Serial.print(" | AY: ");
      Serial.print(ay);
      Serial.print(" | GZ: ");
      Serial.print(gz);
      
      // Traduz comando para texto
      String movimentos[] = {"STOP", "UP", "DOWN", "LEFT", "RIGHT", 
                            "UP_LEFT", "UP_RIGHT", "DOWN_LEFT", "DOWN_RIGHT",
                            "TURN_LEFT", "TURN_RIGHT"};
      int cmd = comando.toInt();
      if (cmd >= 0 && cmd <= 10) {
        Serial.print(" | Movimento: ");
        Serial.print(movimentos[cmd]);
      }
      Serial.println();
    }
    
    lastCommand = comando;
    lastSendTime = currentMillis;
  }
  
  delay(50); // Pequeno delay para estabilidade
}
