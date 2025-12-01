# Atendimento do Professor

## Terças e quintas. Professor de horário parcial. Nesses 2 dias, podem contar comigo!

# Semama 08 - Sistema de disparo de atuadores

## Impactos no seu Projeto

* Usar o botões minimalistas e multifunções;
* Usar botões touch ao invés de push buttons;
* Entender a importância do UML;
* Entendimento do sono profundo do ESP32.

# Problema:

#### Seu projeto pode ser conectado numa bateria ao invés da energia de uma tomada? Como fazer isso?

#### O cliente deseja conhecer o fluxo de comunicação durante o funcionamento do seu projeto. Como explicar de forma lúdica? UML!

---
# Usando Botões Touch

Você sabia que o ESP32 possui pinos GPIO (General Propouse Input Output) que atuam como botões **touch**?

E você pode assim substituir os seus botões da caixinha e colocar um papel alumínio de fora ou uma chapinha de refrigerante recortada. Passe Bombril para remover a tinta da lata;

Os 10 pinos capacitivos que detectam o toque no ESP32 são:

1) T0 (GPIO4)
2) T1 (GPIO0)
3) T2 (GPIO2)
4) T3 (GPIO15)
5) T4 (GPIO13)
6) T5 (GPIO12)
7) T6 (GPIO14)
8) T7 (GPIO27)
9) T8 (GPIO33)
10) T9 (GPIO32)

Você só precisa usar o comando **touchRead(TOUCH_PIN)** para ler o sensibilidade do toque. Valor de **touchRead** próximo de 10 estão detectando o toque. Próximos de 90, estão sem toque.

## Exemplo de Código:

```
#define TOUCH_PIN T8  // GPIO33

int threshold = 20;      // Limiar para detecção de toque
unsigned long previousMillis = 0; // Armazena o último tempo em que o touch foi lido
const long interval = 100; // Intervalo em milissegundos para leituras

void setup() {
  Serial.begin(115200);
  Serial.println("Inicializando Touch no ESP32...");
}

void loop() {
  // Verifica o tempo atual
  unsigned long currentMillis = millis();

  // Se o intervalo tiver passado, faz a leitura do sensor
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis; // Atualiza o último tempo de execução

    // Ler o valor do sensor capacitivo
    int touchValue = touchRead(TOUCH_PIN);

    // Imprimir o valor no monitor serial
    Serial.print("Touch Value: ");
    Serial.println(touchValue);

    // Verificar se o valor está abaixo do limiar
    if (touchValue < threshold) {
      Serial.println("Toque detectado!");
    }
  }
}

```

Dicas importante: caso o seu ESP32 esteja detectando valores malucos do pino no modo touch e você não está encostando no pino, faça uma leitura de 10 amostras no void setup(), tire a média e salve esse valor. Depois você pode que o usuário colocar o dedo no pino e tire +10 amostras e calcule novamente a média. Agora é só fazer o algorimo ignorar a média do valor quando inativo e considerar valores próximos da média quando ativo.

---
# Fazendo seu ESP32 Dormir

O ESP32 pode dormir dormir teoricamente por 136 anos, porque seu relógio interno pode contar até 2^64 microsegundos.

Claro que não poderemos testar isso, porque uma bateria não duraria todo esse tempo.

Esse código exemplo faz um pisca-pisca dormir por 10s.

Note que você nunca usará o **void loop()** novamente se usar o recurso do sono profundo, porque o ESP32 se reinicia todas as vezes que ele acorda. Portanto, seu código do projeto deverá estar todo dentro do **void setup()**.

No modo **sono profundo**, as varáveis da RAM são reiniciada a cada vez que o ESP32 acordar. Portanto, prepare seu código para "perder" dados no modo sono profundo.

No modo profundo, o ESP32 consome 10uA (microAmpères). **Isso é 2000x menor que a corrente de um LED vermelho**.


## Código 1: Faz um LED acender por 3s e hibernar por 7s

```
#define LED_PIN 2 // GPIO onde o LED está conectado

void setup() {
  Serial.begin(115200);

  // Configuração do LED como saída
  pinMode(LED_PIN, OUTPUT);

  // Acender o LED para indicar atividade
  digitalWrite(LED_PIN, HIGH);
  Serial.println("ESP32 despertou! LED ligado por 3 segundos.");

  // Variáveis para controlar o tempo usando millis()
  unsigned long startMillis = millis(); // Registrar o momento inicial
  unsigned long currentMillis;

  // Manter o LED aceso por 3 segundos
  while (true) {
    currentMillis = millis();
    if (currentMillis - startMillis >= 3000) { // 3 segundos (3000 ms)
      break; // Sai do loop após 3 segundos
    }
  }

  // Apagar o LED
  digitalWrite(LED_PIN, LOW);
  Serial.println("LED desligado. Indo para sono profundo...");

  // Configurar sono profundo por 7 segundos
  esp_sleep_enable_timer_wakeup(7 * 1000000); // 7 segundos em microsegundos
  esp_deep_sleep_start();
}

void loop() {
  // Nunca será utilizado no modo sono profundo.
}

```

---
## Código 2: Faz publicação no Ubidots e Hiberna

```
#include "UbidotsEsp32Mqtt.h"

const char *WIFI_SSID = ""; // Put here your Wi-Fi SSID
const char *WIFI_PASS = ""; // Put here your Wi-Fi password

const char *UBIDOTS_TOKEN = "PEGAR O TOKEN INDIVIDUAL QUE CHEGOU NO SLACK DO SEU GRUPO"; // Token do Ubidots
const char *DEVICE_LABEL = "coloque_um_nome_sem_caracteres_especiais"; // Nome do dispositivo
const char *VARIABLE_LABEL1 = "sua variável"; // Variável 1
const char *VARIABLE_LABEL2 = "sua variável"; // Variável 2
const char *CLIENT_ID = ""; // Identificador único para o cliente

Ubidots ubidots(UBIDOTS_TOKEN, CLIENT_ID); // Instância do Ubidots

uint8_t pinPotenciometro = 34;
uint8_t pinBotao = 33;
uint8_t pinLED = 2;

/****************************************
 * Auxiliar Functions
 ****************************************/

void callback(char *topic, byte *payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
}

/****************************************
 * Main Functions
 ****************************************/

void setup() {
  // Configuração inicial
  Serial.begin(115200);
  ubidots.setDebug(true); // Habilita mensagens de debug
  ubidots.connectToWifi(WIFI_SSID, WIFI_PASS);
  ubidots.setCallback(callback);
  ubidots.setup();
  ubidots.reconnect();
  pinMode(pinBotao, INPUT_PULLUP);
  pinMode(pinPotenciometro, INPUT);
  pinMode(pinLED, OUTPUT);

  // Publica os dados antes de entrar no modo de sono profundo
  if (!ubidots.connected()) {
    ubidots.reconnect();
  }

  // Lê os valores dos sensores
  float value1 = analogRead(pinPotenciometro);
  bool value2 = digitalRead(pinBotao);

  // Exibe os valores no monitor serial
  Serial.println("Lendo dados:");
  Serial.println(value1);
  Serial.println(value2);

  // Publica os valores no Ubidots
  ubidots.add(VARIABLE_LABEL1, value1);
  ubidots.add(VARIABLE_LABEL2, value2);
  ubidots.publish(DEVICE_LABEL);
  
  Serial.println("Dados enviados para o Ubidots.");

  // Configura o sono profundo por 5 segundos
  Serial.println("Entrando em sono profundo por 5 segundos...");
  esp_sleep_enable_timer_wakeup(5 * 1000000); // Tempo em microsegundos
  esp_deep_sleep_start();
}

void loop() {
  // O ESP32 nunca chegará aqui no modo de sono profundo
}

```


---
# Entendendo o UML

A Unified Modeling Language (UML) é uma linguagem de modelagem visual padronizada que fornece uma maneira versátil, flexível e amigável de visualizar o design de um sistema. 

Com o UML, é possível especificar, visualizar, construir e documentar artefatos de sistemas de software. 

O UML oferece uma variedade de diagramas e vamos focar no Diagrama de Sequência: é um tipo de diagrama de interação que mostra como os objetos interagem em um determinado cenário de tempo. Ele ilustra a sequência de mensagens trocadas entre os objetos e são essenciais para modelar interações dinâmicas em sistemas de software. 

Vamos a um exemplo:

### Diagrama UML Completo

```
+-------------------+                +-------------------+                +-------------------+
|       ESP32       |                |      Ubidots      |                |       Sensor       |
|-------------------|                |-------------------|                |-------------------|
| - pinLED: int     |                | - token: String   |                | - value: float    |
| - pinButton: int  |                | - clientId: String|                |-------------------|
| - pinPot: int     |                |-------------------|                | + readAnalog()    |
|-------------------|                | + connectToWifi() |                | + readDigital()   |
| + setup()         |                | + setDebug()      |                |-------------------|
| + callback()      |                | + reconnect()     |                          |     
|-------------------|                | + add()           |                          |
        |                            | + publish()       |                          |
        |                                    |                                      |
        +------------------------------------+--------------------------------------+
                           usa                               lê

Diagrama de Sequência:
-----------------------------------------------------------------------------------------
ESP32 -> ESP32: setup()
ESP32 -> Serial: Serial.begin(115200)
ESP32 -> Ubidots: setDebug(true)
ESP32 -> Ubidots: connectToWifi(WIFI_SSID, WIFI_PASS)
ESP32 -> Ubidots: setCallback(callback)
ESP32 -> Ubidots: setup()
ESP32 -> Ubidots: reconnect()

loop {
    ESP32 -> Sensor: readAnalog(pinPotentiometro)
    ESP32 <- Sensor: Retorna valor analógico (float)

    ESP32 -> Sensor: readDigital(pinBotao)
    ESP32 <- Sensor: Retorna valor digital (bool)

    ESP32 -> Serial: Imprime "Lendo dados: " + valores
    ESP32 -> Ubidots: add(VARIABLE_LABEL1, value1)
    ESP32 -> Ubidots: add(VARIABLE_LABEL2, value2)
    ESP32 -> Ubidots: publish(DEVICE_LABEL)

    ESP32 -> Serial: Imprime "Dados enviados para o Ubidots"
}

ESP32 -> Serial: Imprime "Indo para sono profundo"
ESP32 -> ESP32: esp_sleep_enable_timer_wakeup(5 * 1000000)
ESP32 -> ESP32: esp_deep_sleep_start()

```

## Diagrama UML Incompleto

```
+-------------------+                +-------------------+
|       ESP32       |                |      Ubidots      |
|-------------------|                |-------------------|
| - pinPot: int     |                | - token: String   |
| - pinButton: int  |                | - clientId: String|
|-------------------|                |-------------------|
| + setup()         |                | + connectToWifi() |
| + loop()          |                | + add()           |
|-------------------|                | + publish()       |
        |                                     |
        +-------------------------------------+
                           usa
```

Problemas no Diagrama:
-----------------------------------------------------------------------------------------
1. setup():
    - Conecta ao Wi-Fi diretamente sem verificar status.
    - Configura o Ubidots, mas não implementa reconexão.

2. loop():
    - Contém lógica redundante e misturada com leitura de sensores e envio.
    - Não verifica se o Ubidots está conectado antes de publicar.

3. Relação:
    - O ESP32 "usa" Ubidots de forma dependente, mas sem tratamento de erros.
    - Falta de separação entre as tarefas (leitura de sensores, envio de dados, etc.).

Exemplo de Fluxo de Execução:
-----------------------------------------------------------------------------------------
ESP32 -> Ubidots: connectToWifi(WIFI_SSID, WIFI_PASS)
ESP32 -> Sensor: analogRead(pinPot)
ESP32 -> Sensor: digitalRead(pinButton)
ESP32 -> Ubidots: add(variable, value)
ESP32 -> Ubidots: publish(device)

Problemas Identificados:
- O Wi-Fi pode desconectar sem tratamento no código.
- O código não considera reconexões nem modos de economia de energia.
- A leitura dos sensores e envio dos dados estão acoplados diretamente no loop(), tornando o código difícil de manter.
- 

### Problemas Comuns em Diagramas Incompletos

* Ausência de Modularidade: toda a lógica é colocada diretamente no setup() ou loop(), sem métodos auxiliares para leitura de sensores ou envio de dados.

* Falta de Tratamento de Erros: não verifica se a conexão Wi-Fi foi estabelecida antes de tentar enviar dados.

* Não implementa lógica para reconectar ao Ubidots ou ao Wi-Fi.

* Uso Ineficiente de Recursos: não utiliza modos de economia de energia, como o sono profundo.

* As leituras dos sensores são feitas continuamente, sem intervalos controlados por millis().

* Acoplamento Excessivo: a lógica de leitura de sensores e envio de dados está acoplada, dificultando testes e modificações futuras.

* Falta de Documentação e Comentários: o código não possui explicações suficientes, dificultando o entendimento.

## Livro: UML essencial : um breve guia para linguagem padrão, autor Martin Fowler
### Disponível na biblioteca virtual.

# Dinâmica em grupo

Turma, seu grupo foi encarregado de evoluir um projeto IoT. O ESP32 precisa ficar acordado por 10s e hibernar por +10s.


1) Tenha um display com o aviso "Botão X liga LED". Essa mensagem deve aparecer enquanto o ESP32 está acordado;
3) Tenha uma entrada touch para ligar o LED e uma msg dizendo que o LED ligou;
4) Mantenha o LED ligado até que o outra entrada touch seja tocada para desligar o LED (dentro do prazo de acordado);
5) Imprima uma msg dizendo que o LED foi desligado;
6) Coloque o ESP32 para dormir por 10s;
7) Ao retornar, o usuário tem 10s para ligar e desligar o LED usando botões touch;
8) Desenvolva um UML sequencial para explicar seu algoritmo.
9) Haverá uma votção para o melhor grupo que ganhará um prêmio: uma volta dentro do Inteli no Fusca como passageiro.

![Fusca](https://github.com/agodoi/m04-semana08/blob/main/imgs/fusca.jpg)
