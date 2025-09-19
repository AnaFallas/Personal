# Ejercicio 5 Sistema de comunicación Morse

Este ejercicio tiene como objetivo implementar una comunicación unidireccional entre dos FPGA utilizando código Morse. Para lograrlo, se requiere el diseño de dos etapas: una transmisora, encargada de generar y enviar la señal en código Morse, y una receptora, responsable de interpretar dicha señal.

## 1. Abreviaturas y definiciones
- **FPGA**: Field Programmable Gate Arrays
- **Código Morse**: "El código Morse es un sistema de codificación de caracteres alfabéticos y numéricos que utiliza secuencias de señales, generalmente en forma de pulsos eléctricos, sonidos o luces, para transmitir mensajes a largas distancias. Fue desarrollado por Samuel Morse y Alfred Vail en la década de 1830, y desempeñó un papel crucial en las comunicaciones durante más de un siglo."[0]
- **ascii**: American Standard Code for Information Interchange

## 2. Referencias

[0] S. Parra. "Código Morse: ¿qué es, cómo funciona y qué tiene que ver con el Titanic?". NationalGeographic.com . Accesado: Setiembre 17, 2025.[En línea]. Disponible en: "https://www.nationalgeographic.com.es/ciencia/codigo-morse-que-es-como-funciona-que-tiene-que-ver-titanic_19830"

## 3. Desarrollo

El código Morse consiste en una representación de símbolos como letras y números en una combinación de pulsos cortos y pulsos largos. Donde un pulso largo equivale a tres pulsos cortos. Entre cada pulso, ya sea largo o corto, se debe dejar un tiempo de espera equivalente a un pulso corto, a esto se le llamará unidad de tiempo, y entre cada letra de una palabra se debe dejar tres unidades de tiempo. Entre palabras se deja un espacio vacío que tiene una duración de siete unidades de tiempo.
A continuación se resumen los códigos Morse para los caracteres soportados en este ejercicio.

| Caracter | Codigo  | Caracter | Codigo  | Caracter | Codigo  | Caracter | Codigo  |
|------|-------|------|-------|------|-------|------|-------|
| A    | .-    | B    | -...  | C    | -.-.  | D    | -..   |
| E    | .     | F    | ..-.  | G    | --.   | H    | ....  |
| I    | ..    | J    | .---  | K    | -.-   | L    | .-..  |
| M    | --    | N    | -.    | O    | ---   | P    | .--.  |
| Q    | --.-  | R    | .-.   | S    | ...   | T    | -     |
| U    | ..-   | V    | ...-  | W    | .--   | X    | -..-  |
| Y    | -.--  | Z    | --..  | 0    | ----- | 1    | .---- |
| 2    | ..--- | 3    | ...-- | 4    | ....- | 5    | ..... |
| 6    | -.... | 7    | --... | 8    | ---.. | 9    | ----. |


### 3 Transmisor

Mediante ocho de los switches presentes en la FPGA basys3 se toma como entrada un valor ascii que se debe transformar a su equivalente en el código Morse una vez se presione un botón de transmisión. La transmisión de datos se realiza únicamente hasta que se reciba por parte del receptor una señal *ready* que indica que está disponible para recibir datos.
El transmisor también se encarga de activar un *buzzer* que reproduce el código morse que se va a transmitir.

#### 3.1 Consideraciones generales

El diseño para el transmisor se modularizó de la siguiente manera:

 - **debouncer**: este módulo se encarga de eliminar el efecto de rebotes y ruido en pulsadores e interruptores.

- **ascii_latch**: Se encarga de codificar las señales de los switches a un valor ascii

- **ascii_to_morse**: Toma una valor ascii y lo convierte a un equivalente en código Morse.
- **morse_transmisor**: Contiene la lógica de las máquinas de estados que controlan el proceso de la preparación de datos para ser transmitidos.
- **top_transmisor**: Es el archivo que contiene las instancias de los módulos anteriores y se encarga de permitir la comunicación entre cada una de estas etapas.

#### 3.2 Señales

A continuación se compilan las señales de entrada y salida que tiene esta sección del proyecto.

**Señales de Entrada:**
- `sw [7:0]`: Corresponde al arreglo de switches en la FPGA. 8 bits en total que representan un valor ascii.
- `btn`: Esta señal proviene de un botón integrado sobre la FPGA. Permite el inicio de la interpretación del valor ascii. Debe pasar por una etapa que elimina el efecto de rebotes y ruido.
- `ready`: Esta señal proviene de la otra FPGA que actúa como receptor. Se utiliza uno de los pines GPIO. Es necesaria para que el transmisor envíe el mensaje y tenga seguridad de que será recibido.


**Señales de Salida:**
- `morse_out`: Esta señal corresponde a una salida de los pines GPIO y es la que se interconecta con la otra placa. Esta permite la comunicación serial del mensaje en código Morse.
- `buzzer`: Igualmente corresponde al mensaje en código Morse. Esta señal se utiliza exclusivamente para recrear audiblemente el mensaje.


#### 3.2 Diseño

Se diseñó la siguiente máquina de estados finitos encargada de efectuar la transmisión efectiva del mensaje. A continuación se brinda una rápida descripción de cada uno de los estados.

![metastable](.\fsmtransm.svg) _Gráfico de elaboración propia_

- `IDLE`: En este estado la máquina se encuentra en un estado de espera hasta que reciba la señal del botón que confirma la selección de los switches.
- `LATCH_CHAR`: Cuando se reciba la señal ready se puede avanzar al siguiente estado.
- `LOAD_SYMBOL`: Este estado se encarga de comprobar si un dato fue enviado correctamente y también asigna la duración del pulso. En caso que la letra haya sido enviada completamente (symbol_idx >= lenght) se avanza al estado `DONE`. En caso contrario se avanza al estado `TRANSMIT_SYMBOL`.
- `TRANSMIT_SYMBOL`: Este estado es el que se encarga de hacer que los pulsos tengan la duración correcta. Una vez que el símbolo fue enviado se avanza al estado `SYMBOL_SPACE`.
- `SYMBOL_SPACE`: Luego de que se envía un símbolo debe ir seguido por un espacio de una unidad de tiempo, esto sucede durante este estado. Cuando este proceso finaliza se retorna al estado `LOAD_SYMBOL` quien revisa si ya fue enviada toda la letra.
- `DONE`: Este estado se encarga de reiniciar las variables necesarias y luego se devuelve a un estado `IDLE`.


#### 3.3 Testbench

Para probar el funcionamiento del transmisor se realizó un testbench en el que se utilizó como entrada el número hexadecimal 41 que representa el valor ascii para "A". En código morse se representa con ".-". En la siguiente imágen se puede apreciar la respuesta obtenida.

![metastable](./testbench%2041.png) _Captura de la simulación post-implementación_

### 4 Receptor

El receptor se encarga de interpretar la señal Morse recibida desde el transmisor, decodificarla a su equivalente ASCII y mostrarla en un display. El sistema debe ser capaz de identificar pulsos cortos (puntos), pulsos largos (rayas) y los espacios entre ellos, considerando un margen de error para robustecer la detección.

#### 4.1 Consideraciones generales

El diseño para el receptor se modularizó de la siguiente manera:

- **morse_rx_symbol_timer**: Este módulo se encarga de medir la duración de los pulsos HIGH y LOW de la señal Morse, detectar flancos de subida y bajada, y proporcionar conteos temporales precisos para su clasificación.

- **morse_receptor**: Contiene la máquina de estados principal que clasifica los pulsos en puntos (.), rayas (-) y espacios, construye progresivamente el código Morse recibido y controla el proceso de decodificación cuando se completa un carácter.

- **morse_to_ascii**: Decodificador que convierte las secuencias Morse completas a su equivalente en caracteres ASCII según la tabla estándar.
  
- **top_display**: Controlador de display que muestra el carácter ASCII recibido en formato hexadecimal utilizando dos displays de 7 segmentos con multiplexación.

- **top_morse**: Módulo superior que integra todos los componentes del receptor y gestiona la comunicación entre ellos.

#### 4.2 Señales

##### Señales de Entrada

- `clk_i`: Reloj del sistema de 10 MHz, proveniente del wizard de clocks de la FPGA.  
- `reset_i`: Señal de reset activa en alto, utilizada para reiniciar el estado del receptor.  
- `morse_in`: Entrada de la señal Morse serial (1 = pulso activo, 0 = silencio).  

##### Señales de Salida

- `an_o[1:0]`: Señal de selección de ánodos para los dos displays disponibles.  
- `seg_o[6:0]`: Señal de control para los segmentos de los displays de 7 segmentos.  
- `ascii_char[7:0]`: Carácter ASCII decodificado, disponible para monitoreo externo.
- `ready`: Fag para indicar cuándo el Rx está listo para recibir datos

#### 4.3 Diseño

##### 4.3.1 Máquina de Estados del Receptor

Se diseñó la siguiente máquina de estados finitos (FSM) encargada de la recepción y decodificación de la señal Morse:

![FSM Receptor](https://fsm_receptor.svg)  
*Gráfico de elaboración propia*

- **ST_IDLE**: Estado inicial de espera. El receptor está listo para detectar el inicio de un nuevo pulso Morse. Permanece aquí hasta detectar un flanco de subida.  
- **ST_BUSY**: Estado de medición activa. Evalúa la duración del pulso HIGH para clasificarlo como punto (.) o raya (-) según los umbrales temporales configurados.  
- **ST_GAP**: Estado de monitoreo de espacios. Analiza la duración del silencio (LOW) entre símbolos para determinar si corresponde a un espacio entre símbolos (1 unidad) o entre caracteres (3 unidades).  
- **ST_FINALIZE**: Estado de finalización. Cuando se detecta el término de un carácter, activa las señales de validación y prepara el sistema para recibir el siguiente carácter.  

##### 4.3.2 Parámetros Configurables

El receptor implementa parámetros que permiten ajustar su comportamiento según las condiciones de operación:

```SystemVerilog
parameter int unidad_clk = 1_000_000; // Ciclos de reloj por unidad de tiempo Morse
parameter int MARG_PCT   = 25;        // Margen de error del 25% para robustecer detección
```
Estos parámetros permiten que el receptor maneje variaciones temporales mediante:
- Punto (.): 1U ± 25% de margen
- Raya (-): 3U ± 25% de margen
- Espacio entre símbolos: ≥1U
- Espacio entre caracteres: ≥3U

##### 4.3.3 Detección y Clasificación de Símbolos

El módulo morse_rx_symbol_timer implementa la detección precisa de:
- rise_pulse: Detecta flancos ascendentes (transiciones 0→1)
- fall_pulse: Detecta flancos descendentes (transiciones 1→0)
- high_cnt: Conteo de ciclos de reloj durante pulsos HIGH
- low_cnt: Conteo de ciclos de reloj durante períodos LOW

##### 4.3.4 Implementación en Display

El sistema utiliza dos displays de 7 segmentos multiplexados para mostrar el resultado:
- Display 0: Muestra el nibble bajo del valor ASCII
- Display 1: Muestra el nibble alto del valor ASCII
- Frecuencia de multiplexación: 10 MHz para garantizar persistencia visual

4.4 Testbench y Verificación

Se desarrolló un testbench a nivel del modulo morse_receptor el cual demostró el funcionamiento de la máquina de estados y la decodificación, se baso en:
- Recepción y decodificación correcta de letras individuales (A, S, O)
- Detección precisa de puntos (.) y rayas (-)

IMAGEN 
Se desarrollo un teste bench a nivel TOP para la visualización correcta en displays de 7 segmentos pero este no genero los resultados esperados.



