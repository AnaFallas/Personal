# Laboratorio 3 / Parte 3 – Aplicación: Juego de Adivinanza UART

## 1. Abreviaturas y Definiciones

- **UART**: Universal Asynchronous Receiver Transmitter  
- **FIFO**: First In, First Out  
- **LFSR**: Linear Feedback Shift Register  
- **FSM**: Finite State Machine  
- **ROM**: Read-Only Memory  

---

## 2. Contexto General

Este proyecto forma parte del **Laboratorio 3 – Interfaces con periféricos** del curso *EL3313 Taller de Diseño Digital*.  
El laboratorio se divide en tres partes principales:

1. **Parte 1 – FIFO**: Implementación de una memoria FIFO de 8 bits y 512 palabras mediante el IP *FIFO Generator*.  
2. **Parte 2 – Periférico UART**: Desarrollo de un periférico UART completo con registro de control, FIFOs de transmisión y recepción, y comunicación serial bidireccional.  
3. **Parte 3 – Aplicación**: Integración de los módulos anteriores en una aplicación funcional, un **juego de adivinanza** que interactúa con el usuario a través del puerto serie.

---

## 3. Desarrollo

### 3.1 Planteamiento del Diseño

El sistema implementa un **juego de adivinanza numérica** entre una **FPGA (anfitrión)** y una **computadora externa (cliente UART)**.

![diagrama_general](Imagenes/DiagramaGeneral.png)_Diagrama general de interconexión del sistema_

**Flujo general:**  
1. La FPGA genera un número aleatorio entre 1 y 255 usando un LFSR de 8 bits.  
2. El usuario, mediante un programa Python en la PC, envía números por UART para adivinar.  
3. El sistema compara el valor recibido con el número secreto y responde con mensajes almacenados en ROM.  
4. El usuario tiene 5 intentos. Si los agota o acierta, el sistema muestra el mensaje final y espera un nuevo juego.  

---

### 3.2 Componentes Principales del Sistema

#### 3.2.1 Módulo `lfsr8_seeded`
Genera números pseudoaleatorios de 8 bits con una semilla configurable.

```systemverilog
module lfsr8_seeded (
  input  logic       clk,
  input  logic       rst,
  input  logic       load_seed,
  input  logic [7:0] seed,
  input  logic       enable,
  output logic [7:0] rnd
);
```
**Descripción:**
- Implementa la ecuación de retroalimentación `x^8 + x^6 + x^5 + x^4 + 1`.  
- La salida se actualiza en cada ciclo cuando `enable=1`.  
- El estado 0 se evita para garantizar secuencias válidas.

---

#### 3.2.2 Módulo `tries_counter`
Controla el número de intentos disponibles (máximo 5).

```systemverilog
module tries_counter (
  input  logic clk,
  input  logic rst,
  input  logic new_game,
  input  logic consume_try,
  output logic [2:0] tries,
  output logic zero
);
```
**Funcionamiento:**
- Se inicializa con 5 intentos al inicio o con `new_game=1`.  
- Decrementa en cada intento fallido.  
- `zero` indica que no quedan intentos.  

---

#### 3.2.3 Módulo `timer_30s_16mhz`
Temporizador que mide 30 segundos a 16 MHz.

```systemverilog
module timer_30s_16mhz (
  input  logic clk,
  input  logic rst,
  input  logic start,
  input  logic kick,
  output logic timeout
);
```
- Permite reiniciar el juego si no hay interacción durante 30 s.  
- `timeout` se activa al finalizar el conteo.

---

#### 3.2.4 Módulo `rom_msgs_if` y `rom_msgs_pkg`
Interfaz y paquete de direcciones base para los mensajes almacenados en ROM.

| Mensaje | Dirección Base | Contenido |
|----------|----------------|-----------|
| BASE_WELCOME | 0 | “Adivina el número entre 1 y 255!” |
| BASE_HIGH | 35 | “El número es menor. Intenta de nuevo.” |
| BASE_LOW | 75 | “El número es mayor. Intenta de nuevo.” |
| BASE_CORRECT | 115 | “Felicidades! Adivinaste el número en X intentos.” |
| BASE_NOTRIES | 166 | “Lo siento! Se han acabado los intentos.” |
| BASE_NEWGAME | 208 | “Envia cualquier caracter para iniciar un nuevo juego.” |

---

#### 3.2.5 Módulo `uart_msg_streamer`
Envía secuencialmente los mensajes almacenados en la ROM a través del periférico UART.

```systemverilog
module uart_msg_streamer #(
  parameter int MAX_LEN = 512
)( ... );
```
**Características:**
- Recorre los bytes del mensaje desde la dirección base.  
- Reemplaza el carácter `'X'` por el número real de intentos usados.  
- Controla las señales de escritura (`wr_o`, `reg_sel_o`) hacia el periférico UART.  
- Señales `busy` y `done` indican estado de transmisión.

---

#### 3.2.6 Módulo `sevenseg_one_active_low`
Muestra el número de intentos restantes en el display de 7 segmentos.

#### 3.2.7 Módulo Principal `top_game`

Integra todos los bloques anteriores y maneja la FSM principal del juego.

```systemverilog
module top_game (
  input  logic clk_in,
  input  logic reset_button,
  input  logic newgame_button,
  input  logic [15:0] seed_switches,
  output logic [15:0] led,
  output logic [6:0] seg,
  output logic dp,
  output logic [3:0] an,
  input  logic rx,
  output logic tx
);
```
**Funciones del módulo:**
- Inicializa el LFSR con la semilla seleccionada.  
- Coordina los mensajes de la ROM con el streamer UART.  
- Controla intentos, comparaciones y reinicio del juego.  
- Muestra en LEDs y display el estado actual del sistema.

---

### 3.3 Máquina de Estados Finita (FSM)

La FSM controla las fases principales del juego:

![fsm](Imagenes/FSM.png)_Diagrama de la máquina de estados del juego_

---

| **Estado / Grupo** | **En el diagrama** | **Contexto funcional** |
|---------------------|--------------------------------|-------------------------------|
| **G_IDLE** | En el bloque **Inicialización** → estado inicial (punto negro) → `G_IDLE`| FSM en reposo esperando `start_new_game_pulse` (botón o reinicio). |
| **G_LATCH_TARGET** | En bloque **Inicialización**| Carga el número aleatorio desde el LFSR (semilla). |
| **G_SEND_WELCOME / G_WAIT_WELCOME** | En bloque **Inicialización** | Envía el mensaje de bienvenida por UART y espera que termine la transmisión. |
| **G_WAIT_INPUT** | En bloque **Bucle de juego** | Espera la entrada del usuario desde la UART (el número a adivinar). |
| **G_PROCESS** | En bloque **Procesamiento**| Compara el número ingresado (`guess`) con el número objetivo (`target`). |
| **G_SEND_FEEDBACK / G_WAIT_FEEDBACK** | En bloque **Mensajes** | Envía y espera mensaje de “mayor” o “menor” según el resultado de la comparación. |
| **G_SEND_CORRECT / G_WAIT_CORRECT** | En bloque **Mensajes**| Envía mensaje de acierto y luego prepara el mensaje de “nuevo juego”. |
| **G_SEND_NOTRIES / G_WAIT_NOTRIES** | En bloque **Mensajes**| Se activa cuando se acaban los intentos; envía mensaje de “sin intentos”. |
| **G_SEND_NEWGAME / G_WAIT_NEWGAME** | En bloque **Mensajes** (final del flujo)| Muestra el prompt “envía cualquier carácter para iniciar un nuevo juego”. |
| *(G_NG_POLL_RX / G_NG_ISSUE_LEER / G_NG_WAIT_LEER)* | En bloque **Nuevo juego** (fase final del diagrama)| Implementan el **reinicio real** cuando llega un carácter desde UART. |

## 4. Programa en Python (Cliente UART)

El siguiente programa permite interactuar con la FPGA mediante comunicación serie, actuando como **cliente UART**.  
Fue diseñado para ejecutarse en PC con Windows, Linux o macOS y requiere la librería `pyserial`.

### 4.1 Descripción General

- Se conecta al puerto serie especificado por el usuario.  
- Crea un hilo (`thread`) que **lee continuamente** los mensajes enviados por la FPGA.  
- Permite al usuario escribir números (1–255) para enviarlos al juego.  
- Valida el rango localmente y muestra errores de entrada.  
- Finaliza limpiamente al presionar `Ctrl + C`.

### 4.2 Código

```python
import serial
import threading
import time

def game_message(ser):
    """Lee y muestra todo lo recibido desde la FPGA"""
    while True:
        msg = ser.readline().decode(errors="ignore").strip()
        if msg:
            print(f"{msg}")
        time.sleep(0.05)

def val_input(usr_input):
    """Valida el rango permitido (1–255)."""
    if not 1 <= usr_input <= 255:
        print(f"Error local: {usr_input} no es válido (rango 1–255)." )
        return False
    return True

def main():
    puerto = input("Ingrese el puerto serie: ").strip()
    ser = serial.Serial(puerto, baudrate=115200, timeout=0.5)
    time.sleep(2)
    print(f"Conexión establecida en {ser.port}\n")

    reader_thread = threading.Thread(target=game_message, args=(ser,), daemon=True)
    reader_thread.start()

    while True:
        try:
            usr_input = input().strip()
            if not usr_input.isdigit():
                print("Error local: Debe ingresar un número entero.")
                continue
            usr_input = int(usr_input)
            if val_input(usr_input):
                ser.write(f"{usr_input}\r\n".encode())
                print(f" Enviando: {usr_input}")
        except KeyboardInterrupt:
            break

    ser.close()
    print("Conexión cerrada.")

if __name__ == "__main__":
    main()
```

### 4.3 Flujo del Programa

1. **Inicialización:** se abre el puerto UART a 115200 baudios.  
2. **Recepción continua:** el hilo `game_message` imprime todo lo que llega de la FPGA.  
3. **Entrada del usuario:** se valida el número ingresado (1–255).  
4. **Envío:** se transmite la cadena `"<numero>\r\n"` hacia la FPGA.  
5. **Terminación:** al presionar `Ctrl + C`, se cierra la conexión limpiamente.


## 5. Testbench – `tb_top_game`

El testbench verifica el funcionamiento del sistema completo **sin UART física**, validando la FSM, la ROM y las respuestas del sistema.

**Objetivos principales:**
- Validar FSM, ROM, contador de intentos y reinicio.  
- Comprobar sustitución de `'X'` en mensajes correctos.  
- Evaluar temporizador y manejo de timeout.  

### 5.1 Estructura General

El testbench instancia el módulo `top_game` y genera señales de reloj de 100 MHz.  
Además, simula la recepción y transmisión UART para realizar **autoverificación** completa.

```systemverilog
`timescale 1ns/1ps
module tb_top_game;
  logic clk_in = 0;
  logic reset_button, newgame_button;
  logic [15:0] seed_switches;
  logic rx, tx;
  logic [15:0] led;
  logic [6:0] seg;
  logic dp;
  logic [3:0] an;

  top_game dut (...);
  always #5 clk_in = ~clk_in;
```
---

### 5.2 Utilidades y Tareas de Apoyo

El testbench incluye una serie de **tareas automáticas** y **funciones auxiliares** que facilitan la validación:

- **`autobaud_estimate`**: estima automáticamente el tiempo de bit (bit time) de la transmisión UART.  
- **`uart_rx_capture_byte`**: captura un byte transmitido por el DUT, reconstruyendo su valor.  
- **`capture_tx_packet_until_crlf`**: recibe una cadena completa hasta CR+LF (`\r\n`).  
- **`expect_msg`**: compara el mensaje recibido con una referencia esperada y marca OK/FAIL.  
- **`guess_and_expect_low/high/correct`**: simula intentos del jugador y espera las respuestas apropiadas.  

```systemverilog
task automatic expect_msg(input string expected_ascii, input string tag);
  byte_t  got_bytes[];
  byte_t  ref_bytes[];
  capture_tx_packet_until_crlf(got_bytes, bit_ns_est, 2000, 256);
  mk_ref(expected_ascii, ref_bytes);
  if (!pkt_equals_ref(got_bytes, ref_bytes))
    $error("[%s] [EXPECT:%s] FAIL", t_us(), tag);
  else
    $display("[%s] [EXPECT:%s] OK", t_us(), tag);
endtask
```
---

### 5.3 Secuencia de Prueba Automática

La simulación principal ejecuta la tarea `rom_selftest_full`, que prueba los dos escenarios clave:

1. **Adivinanza correcta**:  
   - Se generan números bajos, altos y el valor exacto.  
   - Se espera el mensaje `"Felicidades! Adivinaste el numero en X intentos."`.

2. **Sin intentos disponibles**:  
   - Se envían 5 intentos incorrectos.  
   - Se espera `"Lo siento! Se han acabado los intentos."` seguido del mensaje de nuevo juego.

```systemverilog
initial begin
  reset_button   = 1'b1;
  newgame_button = 1'b0;
  seed_switches  = 16'h0042;
  rx             = 1'b1;

  pulse_reset_release();
  autobaud_estimate(ok_ab, bit_ns_est, 10000);
  rom_selftest_full();
  $finish;
end
```

**Resultados**  
El testbench se logró ejecutar en simulación Post-Shynthesis y Post-Implementation, en la cual muestran los mensajes UART capturados con marcas de tiempo, verificando automáticamente el comportamiento del juego.

![sim](Imagenes/simulacion.png)_Simulación del sistema completo en el testbench_

---

---

## 5. Observaciones

- El sistema logra una comunicación coherente con la PC a través del UART a 115200 baudios.  
- Los mensajes son transmitidos correctamente desde la ROM y procesados por el `uart_msg_streamer`.  
- El display muestra correctamente los intentos restantes.   

**Limitaciones encontradas:**
- Se observaron retrasos en la actualización del display durante la transmisión de mensajes extensos.  
- El testbench requiere ciclos adicionales de espera para sincronizar lecturas UART simuladas. 
---

