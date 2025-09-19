# Laboratorio 2 / Ejercicio 3

## 1. Abreviaturas y definiciones
- **LFSR**: Linear Feedback Shift Register  

## 2. Desarrollo

### 2.1 Planteo del diseño


1. **Generador de Datos**: El LFSR genera un nuevo valor de 16 bits cada 2 segundos y lo envía al registro.  
2. **Escritura en Registro**: Cuando `WE` está habilitado, el valor generado se escribe en el registro de 16 bits.  
3. **Decodificador**: Los 4 dígitos hexadecimales del registro se envían al decodificador `hex_to_7segment`, que convierte cada dígito en una señal de control para el display.  
4. **Visualización en Displays**: Finalmente, los valores decodificados se muestran en los cuatro displays de 7 segmentos.  

**Tabla de Verdad: Decodificador Hex-to-7-Segmento**

#### 3. Tabla de Codificación
| Hex  | seg_out (gfedcba) | Display de 7 segmentos |
|------|------------------|----------------------|
| 0    | 1000000         | 0                    |
| 1    | 1111001         | 1                    |
| 2    | 0100100         | 2                    |
| 3    | 0110000         | 3                    |
| 4    | 0011001         | 4                    |
| 5    | 0010010         | 5                    |
| 6    | 0000010         | 6                    |
| 7    | 0111000         | 7                    |
| 8    | 0000000         | 8                    |
| 9    | 0010000         | 9                    |
| A    | 0001000         | A                    |
| B    | 0000011         | b (minúscula)        |
| C    | 0100111         | C                    |
| D    | 0100001         | d (minúscula)        |
| E    | 0000110         | E                    |
| F    | 0001110         | F                    |
| Default | 1111111      | Apagado              |

---

### 2.2 Módulos del diseño

#### 2.2.1 Módulo `top_with_LFSR`

```systemverilog
module top_with_LFSR(
    input  logic clk_i,
    input  logic reset_i,
    input  logic we_i,
    
    output logic [3:0]  an_o,
    output logic [6:0]  seg_o
);
```
Entradas
- clk_i: reloj principal de la FPGA.
- reset_i: señal de reset global.
- we_i: habilitación de escritura para el registro.

Salidas
- an_o: control de los anodos de los displays.
- seg_o: control de los segmentos del display.

Criterios de diseño
Este módulo integra todos los bloques principales:

- Instancia el LFSR, que genera valores pseudoaleatorios de 16 bits.
- La salida del LFSR (lfsr_data) se conecta al módulo top, que maneja registro, decodificación y multiplexado de displays.
- Se usa un clock wizard (clk_wiz_0) para generar un reloj interno de 10 MHz.

#### 2.2.2 Módulo Módulo top
```systemverilog
Copy code
module top #(
    parameter inicioTIMER = 16'd39999
)(
    input  logic        clk_i,
    input  logic        reset_i,
    input  logic        we_i,
    input  logic [15:0] data_i, 
    
    output logic [3:0]  an_o,
    output logic [6:0]  seg_o
);
```
Parámetros
- inicioTIMER: valor inicial del temporizador para multiplexar displays.

Entradas
- clk_i: reloj de sistema.
- reset_i: reset síncrono/asincrónico del sistema.
- we_i: habilitación de escritura para el registro.
- data_i: dato de 16 bits proveniente del LFSR.

Salidas
- an_o: habilitación de cada display de 7 segmentos (multiplexado).
- seg_o: salidas de segmentos activos en bajo.

Criterios de diseño
- Instancia un registro de 16 bits (register_16bits) que almacena los datos recibidos.
- Instancia el decodificador (deco7seg) que convierte cada nibble en su representación visual.
- Implementa lógica de multiplexado de displays con rotación cíclica de an_o y temporizador ajustable.

#### 2.2.3 Módulo Módulo register_16bits
```systemverilog
Copy code
module register_16bits #(
    parameter inicioCount = 32'd19999999
)(
    input  logic        clk_i,
    input  logic        reset_i,
    input  logic        we_i,
    input  logic [15:0] data_int,
    output logic [15:0] data_out
);
```
Parámetros
- inicioCount: número de ciclos de reloj para que el registro acepte una nueva escritura.

Entradas
- clk_i: reloj del sistema.
- reset_i: señal de reinicio del registro y temporizador.
- we_i: habilitación de escritura.
- data_int: dato de entrada a guardar (16 bits).

Salidas
- data_out: dato almacenado en el registro.

Criterios de diseño
- Implementa un temporizador que genera un flag cuando llega a cero.
- Solo se escribe un nuevo valor en el registro cuando:
flag = 1 (tiempo cumplido).
we_i = 1 (habilitación activa).

Esto permite controlar la frecuencia de actualización de los datos.

#### 2.2.4 Módulo Módulo deco7seg
```systemverilog
Copy code
module deco7seg (
    input        [3:0]  ent,
    output logic [6:0] segmento_out
);
```
Entradas
- ent: valor hexadecimal de 4 bits a decodificar.

Salidas
- segmento_out: control de segmentos activos en bajo para un display de 7 segmentos.

Criterios de diseño
- Implementa una tabla de codificación que convierte cada valor hex (0–F) en su respectiva representación de display.
- Los segmentos están activos en bajo, por lo que el bit en 0 significa segmento encendido.
- Incluye un caso default que apaga todos los segmentos (1111111).

#### 2.2.5 Módulo Interconexión General
top_with_LFSR es el nivel superior que integra todo.
El LFSR genera datos de prueba (lfsr_data).
El top recibe esos datos, los almacena en el register_16bits, los decodifica con deco7seg y los multiplexa en los 4 displays de 7 segmentos.
La frecuencia de refresco es controlada por inicioTIMER.

## 3 Testbench – `tb_top_with_LFSR`

Este testbench tiene como finalidad validar la integración del sistema completo a través del módulo **`top_with_LFSR`**, el cual conecta el generador LFSR, el registro de 16 bits y el decodificador de 7 segmentos.  

### Objetivos 
- Aplicar un **reset inicial** para asegurar el arranque correcto del sistema.  
- Habilitar periódicamente la escritura en el registro de 16 bits mediante la señal `we_i`, sincronizada con la señal `flag` interna.  
- Monitorear los cambios en el registro (`data_out`), los ánodos (`an_o`) y la salida de segmentos (`seg_o`).  
- Mostrar en consola los valores capturados en cada transición relevante.  

### Observaciones
La simulación fue ejecutada y el testbench cumplió con la generación de estímulos. Sin embargo, **el comportamiento observado en los displays y en la escritura de datos no fue el esperado**, mostrando inconsistencias.






