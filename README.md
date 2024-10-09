# Integração entre ESP32, PCF8574 e Joystick 5 Eixos

O objeto desse repositório é exemplificar o uso do 'PCF8574' (expansor de IO I²C), que é conectado eletricamente á um módulo de botões do tipo ' Joystick de 5 Eixos'. Ambos dispositivos são integrados ao microcontrolador ESP32, que realiza a leitura dos sinais de acionamento dos botões do joystick via comunicação I²C.

## Autor
- [Felipe Figueiredo Bezerra](https://github.com/FigFelipe)

## Ambiente de Desenvolvimento

 - **IDE**: Arduino IDE 2.3.3
 - **MCU:** Espressif ESP32-Wroom Dev Module

## Como funciona?

* No PCF8574, o terminal P0 é definido como Saída Digital, com objetivo de gerar um sinal lógico de valor UM. Esse valor lógico é conectado no terminal COM (comum) do módulo Joystick 5 eixos.
* Já os outros terminais do PCF8574, entre P1 e P7, são definidos como Entrada Digital, qual recebem um sinal de valor ZERO assim que qualquer botão do Joystick 5 eixos for pressionado.
* Ao pressionar qualquer botão do Joystick 5 eixos, o PCF8574 detecta mudança de status em suas entradas digitais, modificando (para o valor ZERO) o terminal de interrupção *INT.
* O sinal do terminal INT (PCF8574) é enviado para o GPIO do ESP32, o qual também detecta um sinal de interrupção externa.
* O ESP32 ao receber o sinal de interrupção externa realiza a comunicação via I²C com o PCF8574, realizando a leitura de todos os GPIOs de entrada do PCF8574, sendo assim determinando qual botão do Joystick de 5 eixos foi pressionado.

### 1. Conexão elétrica entre o módulo Joystick 5 Eixos e o módulo PCF8574

Os terminais do módulo joystick são conectados ao módulo PCF8574, designados conforme a tabela abaixo:

| Terminais Joystick    | Terminais PCF8574 |
|-----------------------|-------------------|
| COM                   | P0 como Saída Digital (é gerado no terminal o sinal lógico de valor UM) |
| UP                    | P1 como Entrada Digital |
| DWN                   | P2 como Entrada Digital |
| LFT                   | P3 como Entrada Digital |
| RHT                   | P4 como Entrada Digital |
| MID                   | P5 como Entrada Digital |
| SET                   | P6 como Entrada Digital |
| RST                   | P7 como Entrada Digital |

### 2. Conexão elétrica entre o módulo PCF8574 e ESP32 GPIO

A conexão do módulo PCF8574 com o ESP32 GPIO é dado somente pelo terminal de interrupção (*INT):

| Terminais PCF8574    | ESP32 GPIO |
|----------------------|------------|
| *INT                 | 23 |

## Dispositivos

### 1. Módulo Joystick 5 Eixos
O módulo Joystick de 5 eixos é um botão de multifunção, qual funciona como um push-button normal aberto, disponibilizando no total 5 funções de direção (Up, Down, Left, Right, Mid). Além disso, o módulo também possui 2 botões individuais do tipo push-button normal aberto (Set e Reset):

| Terminais             | Descrição |
|-----------------------|-----------|
| COM                   | Comum (para todos os botões do módulo) |
| UP                    | Joystick Push Button NO |
| DWN                   | Joystick Push Button NO |
| LFT                   | Joystick Push Button NO |
| RHT                   | Joystick Push Button NO |
| MID                   | Joystick Push Button NO |
| SET                   | Push Button NO |
| RST                   | Push Button NO |

### 2. Módulo PCF8574
O módulo PCF8574 é um expansor de I/O's multidirecional (funciona tanto como entrada ou saída), sendo controlado especificamente via comunicação I²C.

| Terminais             | Descrição |
|-----------------------|-----------|
| *INT                  | Notifica (enviando nível ZERO) quando os status dos pinos P0:P7 é modificado |
| P0                    | Configurar como Entrada Digital |
| P1                    | Configurar como Saída Digital |
| P2                    | Configurar como Saída Digital |
| P3                    | Configurar como Saída Digital |
| P4                    | Configurar como Saída Digital |
| P5                    | Configurar como Saída Digital |
| P6                    | Configurar como Saída Digital |
| P7                    | Configurar como Saída Digital |
| A0                    | Endereçamento I²C |
| A1                    | Endereçamento I²C |
| A2                    | Endereçamento I²C |

> **Observação:**
O terminal *INT é ativo em nível lógico ZERO, deve ser conectado á um resistor de pull-up (de preferência do valor de 10K Ohm).

O módulo PCF8574 possui o seguinte endereçamento para a comunicação I²C:

| Endereço (HEX)        | A0 | A1 | A2 | 
|-----------------------|----|----|----|
| 0x20                  | 0  | 0  | 0  |
| 0x21                  | 0  | 0  | 1  |
| 0x22                  | 0  | 1  | 0  |
| 0x23                  | 0  | 1  | 1  |
| 0x24                  | 1  | 0  | 0  |
| 0x25                  | 1  | 0  | 1  |
| 0x26                  | 1  | 1  | 0  |
| 0x27                  | 1  | 1  | 1  |

 
### 3. ESP32 Code
```C++
// Bibliotecas
#include <Wire.h>
#include <PCF8574.h>

#define PCF8574_LOW_LATENCY

// GPIO
#define pcf8574Int 23

// PCF8574 I2C Addressing Map (8 devices max)
// https://github.com/xreef/PCF8574_library
// ------------------------
// Address | A0 | A1 | A2 |
//  0x20   | 0  | 0  | 0  |
//  0x21   | 0  | 0  | 1  |
//  0x22   | 0  | 1  | 0  |
//  0x23   | 0  | 1  | 1  |
//  0x24   | 1  | 0  | 0  |
//  0x25   | 1  | 0  | 1  |
//  0x26   | 1  | 1  | 0  |
//  0x27   | 1  | 1  | 1  |
// ------------------------

// Objetos
PCF8574 ioExpander_1(0x20);
PCF8574::DigitalInput di;

// Variáveis
String joystickPressedButton;
volatile bool showPressedButton = false;

//variables to keep track of the timing of recent interrupts
volatile unsigned long buttonTime = 0;  
volatile unsigned long lastButtonTime = 0; 

// Interrupt Service Routine
void IRAM_ATTR isr() 
{
  // Software Debounce
  buttonTime = millis();
  
  if((buttonTime - lastButtonTime) > 900)
  {
    Serial.println("\nISR");
    showPressedButton = true;
    lastButtonTime = buttonTime;
  }

}

void ReadPcf8574Inputs()
{
  di = ioExpander_1.digitalReadAll();

  if(di.p1 == LOW)
  {
    joystickPressedButton = "Up";
  }
  else if(di.p2 == LOW)
  {
    joystickPressedButton = "Down";
  }
  else if(di.p3 == LOW)
  {
    joystickPressedButton = "Left";
  }
  else if(di.p4 == LOW)
  {
    joystickPressedButton = "Right";
  }
  else if(di.p5 == LOW)
  {
    joystickPressedButton = "Mid";
  }
  else if(di.p6 == LOW)
  {
    joystickPressedButton = "Set";
  }
  else if(di.p7 == LOW)
  {
    joystickPressedButton = "Reset";
  }

}

void setup() {
  // put your setup code here, to run once:
  Serial.begin(115200);
  Serial.println("Hello, ESP32!");

  // PinMode 'PCF8574'
  // Entradas
  ioExpander_1.pinMode(P1, INPUT_PULLUP); // UP
  ioExpander_1.pinMode(P2, INPUT_PULLUP); // DOWN
  ioExpander_1.pinMode(P3, INPUT_PULLUP); // LEFT
  ioExpander_1.pinMode(P4, INPUT_PULLUP); // RIGTH
  ioExpander_1.pinMode(P5, INPUT_PULLUP); // MID
  ioExpander_1.pinMode(P6, INPUT_PULLUP); // SET
  ioExpander_1.pinMode(P7, INPUT_PULLUP); // RESET

  // Saidas
  ioExpander_1.pinMode(P0, OUTPUT); // COM

  ioExpander_1.begin();

  // Realizar a varredura do 'joystick 5 eixos'
  // enviando o pino 'COM' para LOW
  ioExpander_1.digitalWrite(P0, LOW);

  // Interrupçoes
  // attachInterrupt(GPIOPin, ISR, Mode);

  // Onde:
  // - GPIO, define o pino da interrupção
  // - ISR, nome da função que executa qdo a interrupção ocorre
  // - Mode, define quando a interrupção deve ser ocorrida
  //    1. LOW (qdo o pino é nível 0)
  //    2. HIGH (qdo o pino é nível 1)
  //    3. CHANGE (qdo o nível do pino é alterado)
  //    4. FALLING (qdo o pino vai do estado 1 para o estado 0)
  //    5. RISING (qdo o pino vai do estado 0 para o estado 1)
  attachInterrupt(digitalPinToInterrupt(pcf8574Int), isr, FALLING);

  delay(1000);

}

void loop() {
  // put your main code here, to run repeatedly:
  delay(20);

  // Realiza a leitura das entradas do PCF8574
  ReadPcf8574Inputs();

  // Quando a interrupção ocorre, então exibir qual o botão foi pressionado
  if(showPressedButton)
  {
    Serial.printf("Joystick: %s", joystickPressedButton);
    showPressedButton = false;
  }

}

```



