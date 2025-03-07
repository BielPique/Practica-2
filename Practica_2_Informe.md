# **Practica 2: Interrupciones**


En esta práctica hemos trabajado las interrupciones por GPIO (parte A) y las interrupciones por timer (parte B). Para el funcionamiento de esta práctica hemos utilizado un par de LEDS y un botón. En caso de no disponer de uno se puede simular su funcionamiento conectando un cable a la salida del PIN establecido en el código y otro en la salida GND (al juntar los dos cables se simula la actuación del botón).

## Parte A: Interrupciones por GPIO

En este apartado el código consiste en recrear la interrupción mediante GPIO. En nuestro caso hemos inicializado las interrupciones en el PIN 18, que el programa irá contando y sacando por pantalla durante 1 minuto.

```
#include <Arduino.h>

struct Button {
    const uint8_t PIN;
    volatile uint32_t numberKeyPresses;
    volatile bool pressed;
  };
  
  
  Button button1 = {18, 0, false};
  const int LED1 = 4;
  const int LED2 = 5;
  
  
  void IRAM_ATTR isr() {
    button1.numberKeyPresses += 1;
    button1.pressed = true;
  }
  
  
  void setup() {
    Serial.begin(115200);
    pinMode(button1.PIN, INPUT_PULLUP);
    attachInterrupt(button1.PIN, isr, FALLING);
  
  
    // Configurar LEDs
    pinMode(LED1, OUTPUT);
    pinMode(LED2, OUTPUT);
  }
  
  
  void loop() {
    if (button1.pressed) {
      Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
      button1.pressed = false;
     
      // Encender LEDs por 500ms
      digitalWrite(LED1, HIGH);
      digitalWrite(LED2, HIGH);
      delay(500);
      digitalWrite(LED1, LOW);
      digitalWrite(LED2, LOW);
    }
  
  
    // Detach Interrupt after 1 Minute
    static uint32_t lastMillis = 0;
    if (millis() - lastMillis > 60000) {
      lastMillis = millis();
      detachInterrupt(button1.PIN);
      Serial.println("Interrupt Detached!");
    }
  }
```


De esta manera, tenemos el botón en el puerto GPIO18 de la ESP32-S3 con salida en GND, y cada vez que le damos sale el mensaje por el serial monitor junto al parpadeo del LED.

## Parte B: Interrupciones por Timer

```
#include <Arduino.h>

volatile int interruptCounter = 0;
int totalInterruptCounter = 0;
bool ledState = false;  // Estado de los LEDs


const int LED1 = 4;
const int LED2 = 5;


hw_timer_t *timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;


void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}


void setup() {
  Serial.begin(115200);
  while (!Serial);  // Espera conexión serial


  pinMode(LED1, OUTPUT);
  pinMode(LED2, OUTPUT);


  // Configuración del temporizador
  timer = timerBegin(0, 80, true);       // Timer 0, divisor 80 -> 1 tick = 1us
  timerAttachInterrupt(timer, &onTimer, true);
  timerAlarmWrite(timer, 1000000, true); // Interrupción cada 1s (1,000,000 us)
  timerAlarmEnable(timer);


  Serial.println("Timer iniciado...");
}


void loop() {
  if (interruptCounter > 0) {
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);


    totalInterruptCounter++;


    Serial.print("An interrupt has occurred. Total number: ");
    Serial.println(totalInterruptCounter);


    // Alternar estado de los LEDs
    ledState = !ledState;
    digitalWrite(LED1, ledState);
    digitalWrite(LED2, ledState);
  }
}
```

En este segundo apartado hemos modificado el código de manera que las interrupciones se realicen por timer, activándose cada interrupción cada 1000000 microsegundos (1 segundo) con el respectivo mensaje por pantalla.


                                                 
## Parte Extra

El objetivo de este apartado es generar un código que altere la frecuencia de parpadeo de un LED mediante dos pulsadores. De esta manera el LED empezará a parpadear a frecuencia inicial mediante interrupciones por timer y una vez se pulse uno de los dos botones, provocará que la frecuencia de parpadeo suba mientras que pulsando el otro la misma bajará. También hemos modificado el código para que los pulsadores que se leen en el timer se filtren y así evitamos rebotes. Para eso hemos usado la función “checkButtons()”.

```
#include <Arduino.h>
const int LED_PIN = 4;       // LED en GPIO4
const int BTN_UP = 18;       // Botón para aumentar la frecuencia
const int BTN_DOWN = 17;     // Botón para disminuir la frecuencia

volatile int interruptCounter = 0;
volatile int blinkDelay = 500; // Frecuencia inicial (500ms -> 1Hz)
volatile bool ledState = false;
volatile unsigned long lastPressUp = 0;
volatile unsigned long lastPressDown = 0;
const int debounceTime = 200; // Tiempo de debounce en ms

hw_timer_t *timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;

void IRAM_ATTR onTimer() {
    portENTER_CRITICAL_ISR(&timerMux);
    interruptCounter++;
    portEXIT_CRITICAL_ISR(&timerMux);
}

void IRAM_ATTR checkButtons() {
    unsigned long currentMillis = millis();

    // Comprobamos si el botón UP fue presionado (con debounce)
    if (digitalRead(BTN_UP) == LOW && (currentMillis - lastPressUp > debounceTime)) {
        lastPressUp = currentMillis;
        if (blinkDelay > 100) { // Evitar que sea demasiado rápido
            blinkDelay -= 50; // Aumenta la frecuencia
        }
    }

    // Comprobamos si el botón DOWN fue presionado (con debounce)
    if (digitalRead(BTN_DOWN) == LOW && (currentMillis - lastPressDown > debounceTime)) {
        lastPressDown = currentMillis;
        if (blinkDelay < 2000) { // Evitar que sea demasiado lento
            blinkDelay += 50; // Disminuye la frecuencia
        }
    }
}

void setup() {
    Serial.begin(115200);
    pinMode(LED_PIN, OUTPUT);
    pinMode(BTN_UP, INPUT_PULLUP);
    pinMode(BTN_DOWN, INPUT_PULLUP);

    // Configurar el temporizador
    timer = timerBegin(0, 80, true); // Timer 0, divisor 80 → 1 tick = 1µs
    timerAttachInterrupt(timer, &onTimer, true);
    timerAlarmWrite(timer, 500000, true); // 500ms inicial (1Hz)
    timerAlarmEnable(timer);

    Serial.println("Sistema iniciado...");
}

void loop() {
    if (interruptCounter > 0) {
        portENTER_CRITICAL(&timerMux);
        interruptCounter--;
        portEXIT_CRITICAL(&timerMux);

        // Alternar estado del LED
        ledState = !ledState;
        digitalWrite(LED_PIN, ledState);

        // Verificar pulsadores dentro del timer para evitar rebotes
        checkButtons();

        // Actualizar el tiempo del temporizador según la frecuencia modificada
        timerAlarmWrite(timer, blinkDelay * 1000, true);
    }
}
```

En cuanto a la práctica nosotros hemos conectado el botón de subida al GPIO18 y el de bajada al GPIO17, con el LED en el GPIO4. Así pues, alcanzada la inicialización por timer, el LED empieza a parpadear a 1Hz (500ms) de frecuencia inicial junto a un mensaje en el serial monitor indicando que se ha iniciado el sistema y cuando pulsamos el botón del GPIO18 disminuye de 50 en 50 el tiempo entre parpadeos causando que la frecuencia sea mayor con un mínimo de 100ms, para evitar que sea muy rápido. Lo mismo ocurre con el GPIO17, que en cambio de disminuir el tiempo entre parpadeos la aumenta de 50 en 50 para disminuir la frecuencia de parpadeo con un límite en 2000ms.   
