int direcaoDesvio = 0; // 0 = não definido, 1 = direita, 2 = esquerda


int modoAutomatico = 0;  
// 0 = Modo manual
// 1 = Modo patrulha (detecta duas paredes e se posiciona no meio delas)
// 2 = Modo desvio simples


int eixoAtual = 0;   // 0 = primeiro eixo, 1 = segundo eixo


enum EstadoAuto {
  NORMAL,
  GIRANDO_1,
  MEDINDO_TEMPO,
  GIRANDO_2,
  INDO_PARA_MEIO,
  GIRANDO_90,
  PARADO
};


enum EstadoDesvio {
  DESVIO_FRENTE,
  DESVIO_VIRANDO,
};

EstadoDesvio estadoDesvio = DESVIO_FRENTE;
EstadoAuto estado = NORMAL;

unsigned long tempoInicio = 0;
unsigned long tempoTotal = 0;
unsigned long tempoMeio = 0;
unsigned long tempoGiro = 0;






#define TEMPO_180 756   // milisegundos de motor ligado pra dar uma volta de 180 graus
#define TEMPO_90 378   // milisegundos de motor ligado pra dar uma volta de 90 graus

#define TRIG_PIN 5
#define ECHO_PIN 32

#define DISTANCIA_MINIMA 10   // distância em cm (aproximadamente)
#define DISTANCIA_MINIMA2 25   // distância2 em cm (aproximadamente)






#include <Arduino.h>
#ifdef ESP32
#include <WiFi.h>
#include <AsyncTCP.h>
#elif defined(ESP8266)
#include <ESP8266WiFi.h>
#include <ESPAsyncTCP.h>
#endif
#include <ESPAsyncWebServer.h>


#define BUZZER_PIN 4


#define UP 1
#define DOWN 2
#define LEFT 3
#define RIGHT 4
#define UP_LEFT 5
#define UP_RIGHT 6
#define DOWN_LEFT 7
#define DOWN_RIGHT 8
#define TURN_LEFT 9
#define TURN_RIGHT 10
#define STOP 0

#define CURVA_RIGHT 12
#define CURVA_LEFT 11

#define CURVA2_RIGHT 14 // não utilizada por falta de consistência
#define CURVA2_LEFT 13 // não utilizada por falta de consistência

#define MODE 15

#define BUZINA 16


#define FRONT_RIGHT_MOTOR 0
#define BACK_RIGHT_MOTOR 1
#define FRONT_LEFT_MOTOR 2
#define BACK_LEFT_MOTOR 3

#define FORWARD 1
#define BACKWARD -1






struct MOTOR_PINS
{
  int pinIN1;
  int pinIN2;    
};

std::vector<MOTOR_PINS> motorPins = 
{
  {16, 17},  //FRONT_RIGHT_MOTOR
  {18, 19},  //BACK_RIGHT_MOTOR
  {27, 26},  //FRONT_LEFT_MOTOR
  {25, 33},  //BACK_LEFT_MOTOR   
};

const char* ssid     = "Carrin";
const char* password = "87654321";

AsyncWebServer server(80);
AsyncWebSocket ws("/ws");






// Código html pra controla o carrinho pelo celular
const char* htmlHomePage PROGMEM = R"HTMLHOMEPAGE(

<!DOCTYPE html>
<html>
  <head>
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <style>
        .modeButton{
    font-size: 32px;
    font-weight: bold;
    font-family: monospace;
}

    /* setas */
    .arrows {
      /* padding-left: 20px; */
      text-align:center;  
      font-size:100px;
      color:black;
    }
    /* setas circulares */
    .circularArrows {
      font-size:80px;
      color:black;  
    }
    /* caixas de botão */
    td {
      text-align: center; /* Centraliza horizontalmente */
      vertical-align: middle; /* Centraliza verticalmente */
      /* width: 120px;  Define uma largura fixa para melhor visualização */
      /* height: 120px;  Define uma altura fixa para melhor visualização */
      background-color:white;
      border-radius:20%;
      box-shadow: 5px 5px #888888;
    }


    td:active {
      transform: translate(5px,5px);
      box-shadow: none; 
    }

    .noselect {
      -webkit-touch-callout: none; /* iOS Safari */
        -webkit-user-select: none; /* Safari */
         -khtml-user-select: none; /* Konqueror HTML */
           -moz-user-select: none; /* Firefox */
            -ms-user-select: none; /* Internet Explorer/Edge */
                user-select: none; /* Non-prefixed version, currently
                                      supported by Chrome and Opera */
    }
    </style>
  </head>
  <body class="noselect" align="center" style="background-color:black">
     
    <h1 style="color: white;text-align:center;font-family: 'monospace';">Carro Omnidirecional</h1>
    <h2 style="color: white;text-align:center;font-family: 'monospace';">Wi-Fi</h2>
    
    <table id="mainTable" style="width:400px;margin:auto;table-layout:fixed" CELLSPACING=10>
<tr>
    <td ontouchstart='onTouchStartAndEnd("5")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(225deg);">&#10145;</span>
    </td>
    <td ontouchstart='onTouchStartAndEnd("1")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(270deg);">&#10145;</span>
    </td>
    <td ontouchstart='onTouchStartAndEnd("6")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(315deg);">&#10145;</span>
    </td>
</tr>

<tr>
    <td ontouchstart='onTouchStartAndEnd("3")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(180deg);">&#10145;</span>
    </td>
    <td ontouchstart='onTouchStartAndEnd("16")' ontouchend='onTouchStartAndEnd("0")'>
      <span class="arrows" style="display: inline-block;">&#9675;</span>
  </td>
    <td ontouchstart='onTouchStartAndEnd("4")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(0deg);">&#10145;</span>
    </td>
</tr>

<tr>
    <td ontouchstart='onTouchStartAndEnd("7")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(135deg);">&#10145;</span>
    </td>
    <td ontouchstart='onTouchStartAndEnd("2")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(90deg);">&#10145;</span>
    </td>
    <td ontouchstart='onTouchStartAndEnd("8")' ontouchend='onTouchStartAndEnd("0")'>
        <span class="arrows" style="display: inline-block; transform: rotate(45deg);">&#10145;</span>
    </td>
</tr>

<tr>
        <td ontouchstart='onTouchStartAndEnd("9")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8634;</span></td>
        <td></td>
        <td ontouchstart='onTouchStartAndEnd("10")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8635;</span></td>
</tr>

<tr>
        <td ontouchstart='onTouchStartAndEnd("11")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8624;</span></td>
        <td ontouchstart='onTouchStartAndEnd("15")' ontouchend='onTouchStartAndEnd("0")'>
            <span class="modeButton">MODE</span>
        </td>
                <td ontouchstart='onTouchStartAndEnd("12")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8625;</span></td>
</tr>
<!--
<tr>
  <td ontouchstart='onTouchStartAndEnd("13")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8662;</span></td>
  <td></td>
  <td ontouchstart='onTouchStartAndEnd("14")' ontouchend='onTouchStartAndEnd("0")'><span class="circularArrows" >&#8663;</span></td>
</tr>
-->

    <!--<tr>
  <td>↖</td> 5 
  <td>↑</td> 1 
  <td>↗</td> 6 
</tr>
<tr>
  <td>←</td> 3 
  <td></td>  Espaço vazio 
  <td>→</td> 4 
</tr>
<tr>
  <td>↙</td> 7 
  <td>↓</td> 2 
  <td>↘</td> 8 
</tr>
<tr>
  <td>↶</td> 9 
  <td></td>  Espaço vazio 
  <td>↷</td> 10 
</tr>-->
    </table>

    <script>
      var webSocketUrl = "ws:\/\/" + window.location.hostname + "/ws";
      var websocket;
      
      function initWebSocket() 
      {
        websocket = new WebSocket(webSocketUrl);
        websocket.onopen    = function(event){};
        websocket.onclose   = function(event){setTimeout(initWebSocket, 2000);};
        websocket.onmessage = function(event){};
      }

      function onTouchStartAndEnd(value) 
      {
        websocket.send(value);
      }
          
      window.onload = initWebSocket;
      document.getElementById("mainTable").addEventListener("touchend", function(event){
        event.preventDefault()
      });      
    </script>
    
  </body>
</html> 

)HTMLHOMEPAGE";
// Fim da parte de html






// Definição de quais pinos energiza pra roda ir pra frente ou pra trás
void rotateMotor(int motorNumber, int motorDirection)
{
  if (motorDirection == FORWARD)
  {
    digitalWrite(motorPins[motorNumber].pinIN1, HIGH);
    digitalWrite(motorPins[motorNumber].pinIN2, LOW);    
  }
  else if (motorDirection == BACKWARD)
  {
    digitalWrite(motorPins[motorNumber].pinIN1, LOW);
    digitalWrite(motorPins[motorNumber].pinIN2, HIGH);     
  }
  else
  {
    digitalWrite(motorPins[motorNumber].pinIN1, LOW);
    digitalWrite(motorPins[motorNumber].pinIN2, LOW);       
  }
}






// Função que lê a distância usando o sensor ultrassônico e retorna em centímetros (mas não de maneira precisa)
long lerDistanciaCM()
{
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duracao = pulseIn(ECHO_PIN, HIGH, 30000);

  // Se não recebeu eco, retorna valor alto
  if (duracao == 0) return 999;

  long distancia = duracao / 58;
  return distancia;
}






void processCarMovement(String inputValue)
{
  // Verifica se o comando recebido foi Mode
  if (inputValue.toInt() == MODE)
  {
  // Avança para o próximo modo se for
  modoAutomatico++;

  // Mantém valor entre 0 e 2, ao invés de ir pra 3 vai pra 0 assim ficando num ciclo
  if (modoAutomatico > 2) modoAutomatico = 0;

  // Modo Patrulha (procura duas paredes e se desloca pro ponto entre elas)
  if (modoAutomatico == 1) {
    Serial.println("MODO PATRULHA ATIVADO");
    eixoAtual = 0;
    estado = NORMAL;
  }

  // Modo Desvio de obstáculos, quando encontra um obstáculo desvia pra esquerda ou direita aleatóriamente
  else if (modoAutomatico == 2) {
    Serial.println("MODO DESVIO ATIVADO");
    estadoDesvio = DESVIO_FRENTE;
  }

  // Modo Manual, controle pelo celular
  else {
    Serial.println("MODO MANUAL ATIVADO");
    processCarMovement(String(STOP)); // Para o robô
  }

  return;
}






// Processa outros comandos
switch(inputValue.toInt())
{

    // ligar buzina
    case BUZINA:
      digitalWrite(BUZZER_PIN, HIGH);   
      break;

    // Curva pra direita
    case CURVA_RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, STOP);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);  
      break;

    // Curva pra esquerda
    case CURVA_LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, STOP);  
      break;

    case CURVA2_RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);  
      break;

    case CURVA2_LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);  
      break;


    // Movimentos básicos
    case UP:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);                  
      break;
  
    case DOWN:
      rotateMotor(FRONT_RIGHT_MOTOR, BACKWARD);
      rotateMotor(BACK_RIGHT_MOTOR, BACKWARD);
      rotateMotor(FRONT_LEFT_MOTOR, BACKWARD);
      rotateMotor(BACK_LEFT_MOTOR, BACKWARD);   
      break;
  
    case LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, BACKWARD);
      rotateMotor(FRONT_LEFT_MOTOR, BACKWARD);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);   
      break;
  
    case RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, BACKWARD);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, BACKWARD);  
      break;
  
    case UP_LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, STOP);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);  
      break;
  
    case UP_RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, STOP);  
      break;
  
    case DOWN_LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, BACKWARD);
      rotateMotor(FRONT_LEFT_MOTOR, BACKWARD);
      rotateMotor(BACK_LEFT_MOTOR, STOP);   
      break;
  
    case DOWN_RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, BACKWARD);
      rotateMotor(BACK_RIGHT_MOTOR, STOP);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, BACKWARD);   
      break;
  
    case TURN_LEFT:
      rotateMotor(FRONT_RIGHT_MOTOR, FORWARD);
      rotateMotor(BACK_RIGHT_MOTOR, FORWARD);
      rotateMotor(FRONT_LEFT_MOTOR, BACKWARD);
      rotateMotor(BACK_LEFT_MOTOR, BACKWARD);  
      break;
  
    case TURN_RIGHT:
      rotateMotor(FRONT_RIGHT_MOTOR, BACKWARD);
      rotateMotor(BACK_RIGHT_MOTOR, BACKWARD);
      rotateMotor(FRONT_LEFT_MOTOR, FORWARD);
      rotateMotor(BACK_LEFT_MOTOR, FORWARD);   
      break;
  
    case STOP:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, STOP);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, STOP);    
      digitalWrite(BUZZER_PIN, LOW);   // desliga buzina
      break;

    // Se receber comando inválido faz esse caso
    default:
      rotateMotor(FRONT_RIGHT_MOTOR, STOP);
      rotateMotor(BACK_RIGHT_MOTOR, STOP);
      rotateMotor(FRONT_LEFT_MOTOR, STOP);
      rotateMotor(BACK_LEFT_MOTOR, STOP);    
      digitalWrite(BUZZER_PIN, LOW);   // desliga buzina
      break;
  }
}


// Função chamada quando acessam a página principal "/"
void handleRoot(AsyncWebServerRequest *request) 
{
  request->send_P(200, "text/html", htmlHomePage);
}

void handleNotFound(AsyncWebServerRequest *request) 
{
    request->send(404, "text/plain", "File Not Found");
}






// Função chamada sempre que ocorre um evento no WebSocket
void onWebSocketEvent(AsyncWebSocket *server, 
                      AsyncWebSocketClient *client, 
                      AwsEventType type,
                      void *arg, 
                      uint8_t *data, 
                      size_t len) 
{                      
  switch (type) 
  {
    case WS_EVT_CONNECT:
      Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
      //client->text(getRelayPinsStatusJson(ALL_RELAY_PINS_INDEX));
      break;
    case WS_EVT_DISCONNECT:
      Serial.printf("WebSocket client #%u disconnected\n", client->id());
      processCarMovement("0");
      break;
    case WS_EVT_DATA:
      AwsFrameInfo *info;
      info = (AwsFrameInfo*)arg;
      if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) 
      {
        std::string myData = "";
        myData.assign((char *)data, len);
        processCarMovement(myData.c_str());       
      }
      break;
    case WS_EVT_PONG:
    case WS_EVT_ERROR:
      break;
    default:
      break;  
  }
}






void setUpPinModes()
{
  // Sensor ultrassônico
  pinMode(TRIG_PIN, OUTPUT); //trig
  pinMode(ECHO_PIN, INPUT); //echo
  digitalWrite(TRIG_PIN, LOW); //trig começa desligado

  // Buzzer
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);


  // Configuração dos motores
  for (int i = 0; i < motorPins.size(); i++)
  {
    pinMode(motorPins[i].pinIN1, OUTPUT);
    pinMode(motorPins[i].pinIN2, OUTPUT);  
    rotateMotor(i, STOP);  
  }
}






// Função executada uma vez ao ligar o ESP
void setup(void) 
{
  // Configura sensores, motores e buzzer
  setUpPinModes();
  Serial.begin(115200);

  // Cria rede Wi-Fi própria
  WiFi.softAP(ssid, password);
  // Obtém endereço IP da rede criada
  IPAddress IP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(IP);

  server.on("/", HTTP_GET, handleRoot);
  server.onNotFound(handleNotFound);
  
  ws.onEvent(onWebSocketEvent);
  server.addHandler(&ws);
  
  // Inicia servidor HTTP
  server.begin();
  Serial.println("HTTP server started");
}






void loop()
{
  // Remove conexões WebSocket antigas
  ws.cleanupClients();


  // MODO 1 - PATRULHA
  if (modoAutomatico == 1)
  {
    long distancia = lerDistanciaCM();

    switch (estado)
    {
      // case NORMAL: anda pra frente
      case NORMAL:
        processCarMovement(String(UP));

        if (distancia <= DISTANCIA_MINIMA)
        {
          Serial.println("Obstáculo detectado! Girando 180°");
          processCarMovement(String(STOP));
          delay(200);

          tempoGiro = millis();
          processCarMovement(String(TURN_RIGHT));
          estado = GIRANDO_1;
        }
        break;


      // Primeiro giro de 180 graus
      case GIRANDO_1:
        if (millis() - tempoGiro >= TEMPO_180)
        {
          Serial.println("Giro 180° completo. Medindo tempo...");
          tempoInicio = millis();
          processCarMovement(String(UP));
          estado = MEDINDO_TEMPO;
        }
        break;


      // Mede comprimento do eixo pra poder andar até a metade depois, faz isso medindo quanto tempo fica andando
      case MEDINDO_TEMPO:
        if (distancia <= DISTANCIA_MINIMA)
        {
          processCarMovement(String(STOP));
          tempoTotal = millis() - tempoInicio;
          tempoMeio = tempoTotal / 2;
          Serial.printf("Tempo total: %lums, Meio: %lums\n", tempoTotal, tempoMeio);

          tempoGiro = millis();
          processCarMovement(String(TURN_RIGHT));
          estado = GIRANDO_2;
        }
        break;

      // Segundo giro de 180 graus
      case GIRANDO_2:
        if (millis() - tempoGiro >= TEMPO_180)
        {
          Serial.println("Segundo giro 180° completo. Indo para meio...");
          tempoInicio = millis();
          processCarMovement(String(UP));
          estado = INDO_PARA_MEIO;
        }
        break;

      // Anda até o meio do eixo
      case INDO_PARA_MEIO:
        if (millis() - tempoInicio >= tempoMeio)
        {
          Serial.println("Chegou no meio. Girando 90°...");
          processCarMovement(String(STOP));
          tempoGiro = millis();
          processCarMovement(String(TURN_RIGHT));
          estado = GIRANDO_90;
        }
        break;

      // Giro de 90 graus
      case GIRANDO_90:
        if (millis() - tempoGiro >= TEMPO_90)
        {
          processCarMovement(String(STOP));
          delay(100);
          
          eixoAtual++;
          Serial.printf("Eixo %d completado\n", eixoAtual);
          
          if (eixoAtual >= 1) // Se completou todos os eixos
          {
            Serial.println("PATRULHAMENTO COMPLETO!");
            estado = PARADO;
          }
          else
          {
            Serial.println("Iniciando próximo eixo...");
            processCarMovement(String(UP));
            estado = NORMAL;
          }
        }
        break;

      case PARADO:
        processCarMovement(String(STOP));
        break;
    }
  }






// MODO 2 - DESVIO SIMPLES

if (modoAutomatico == 2)
{
  // Lê distância
  long distancia = lerDistanciaCM();

  switch (estadoDesvio)
  {
    case DESVIO_FRENTE:
      processCarMovement(String(UP));

      if (distancia <= DISTANCIA_MINIMA2)
      {
        processCarMovement(String(STOP));
        delay(100);

        // começa a ir pra direita
        processCarMovement(String(RIGHT));
        estadoDesvio = DESVIO_VIRANDO;
      }
      break;

case DESVIO_VIRANDO:
  // Escolhe aleatoriamente uma direção (direita ou esquerda)
  if (direcaoDesvio == 0) {
    direcaoDesvio = random(1, 3); // 1 ou 2
  }
  
  // Enquanto houver obstáculo
  if (distancia <= DISTANCIA_MINIMA2) {
    // Vira na direção escolhida
    if (direcaoDesvio == 1) {
      processCarMovement(String(RIGHT));
    } else {
      processCarMovement(String(LEFT));
    }
    delay(1000);
  } else {
    // Caminho livre
    processCarMovement(String(STOP));
    delay(50);
    estadoDesvio = DESVIO_FRENTE; // Volta a andar
    direcaoDesvio = 0; // Reseta direção
  }
  break;
  }
}
}
