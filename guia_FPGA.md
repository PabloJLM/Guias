# Síntesis y Programación FPGA

Guía para sintetizar el diseño en la Tang Nano 9K y programarlo con Gowin EDA.

---

## Tang Nano 9K — Especificaciones relevantes

| Característica | Valor |
|---|---|
| FPGA | Gowin GW1NR-9C |
| LUTs | 8640 |
| Block RAM | 468 Kbits |
| Clock onboard | 27 MHz |
| LEDs | 6 (activos en bajo) |
| Botones | 2 (S1 = reset, S2 libre) |
| UART USB | Integrado (CH340 o similar) |

El MicroRV8-GT ocupa aproximadamente 800–1200 LUTs dependiendo del programa en ROM.

---

## Flujo en Gowin EDA

### 1. Crear proyecto

```
File → New Project
  Device:  GW1NR-9C
  Package: QFN88
  Speed:   C6/I5
```

### 2. Agregar archivos fuente

Agregar en este orden (importa para evitar errores de módulo no definido):

```
alu.v  regfile.v  cpu_core.v
instruction_memory.v  data_memory.v
gpio.v  uart.v  pwm.v  uart_loader.v
microrv8_system.v  tang_nano_top.v
tang_nano_9k.cst
```

Clic derecho en `tang_nano_top.v` → **Set as Top Module**.

### 3. Cargar programa en la ROM

**Opción A — hardcodeado en instruction_memory.v**

Editar el bloque `initial` con `$readmemh`:

```verilog
initial begin
    $readmemh("mi_programa.hex", rom);
end
```

El `.hex` debe estar en el directorio del proyecto Gowin.

**Opción B — cargar en caliente via UART** (sin resintetizar)

```bash
python3 tools/assembler.py mi_prog.asm --binary -o mi_prog.bin
python3 tools/uart_flash.py mi_prog.bin --port COM3
```

O usar directamente el `JoJoP_IDE.py`.

### 4. Sintetizar y programar

```
Synthesize → Place & Route → Generate Bitstream → Program Device
```

Verificar en el reporte de síntesis:
- Logic Elements < 8640
- BRAM: 1/26
- Timing Slack > 0 ns

### 5. Programar la FPGA

```
Tools → Programmer
  Cable: USB Cable → Query/Detect
  SRAM:  temporal, se borra al desconectar (usar para pruebas)
  Flash: permanente (usar para despliegue)
```

El archivo `.fs` generado está en: `[proyecto]/impl/pnr/[nombre].fs`

---

## Verificar que funciona

Con el programa de contador por defecto los 6 LEDs deben contar en binario (0 a 63) y reiniciar.

Si los LEDs no cambian:
- Verificar que S1 no esté presionado (reset activo bajo)
- Revisar el reporte de síntesis por advertencias en los módulos de memoria
- Confirmar que el `.cst` está incluido en el proyecto

---

## Cargar programas via UART

El `uart_loader` integrado permite reprogramar el CPU sin resintetizar.

```
programa.asm  →  assembler.py --binary  →  programa.bin  →  uart_flash.py  →  FPGA
```

Durante la carga el LED 5 se enciende (loader activo, CPU en reset). Al terminar, el CPU arranca con el nuevo programa.

### Protocolo de referencia

```
Byte 0:    0xAA  (sync 1)
Byte 1:    0x55  (sync 2)
Byte 2:    count[8]    (siempre 0 para programas < 256 instrucciones)
Byte 3:    count[7:0]  (cantidad de instrucciones)
Byte 4+5:  instrucción 0 (big-endian)
...
```

---

## Pinout Tang Nano 9K

```
Pin   Señal        Descripción
────  ───────────  ─────────────────────────────
52    sys_clk      27 MHz oscilador
4     sys_rst_n    Botón S1 (reset activo bajo)
10    led_n[0]     LED 0
11    led_n[1]     LED 1
13    led_n[2]     LED 2
14    led_n[3]     LED 3
15    led_n[4]     LED 4
16    led_n[5]     LED 5 (también indica loader activo)
17    uart_rx      RX del conversor USB-UART
18    uart_tx      TX del conversor USB-UART
25    pwm_out      Salida PWM
```

---

## Depuración en FPGA

**Timing BRAM:** La `instruction_memory` es síncrona (un ciclo de latencia). El estado `S_WAIT` del CPU absorbe este ciclo. Si hay problemas de timing, es normal a 27 MHz para este diseño.

**UART baud rate:** El divisor es `27_000_000 / 115200 = 234`. Si la comunicación es inestable, verificar que el terminal usa 115200 8N1.

**GPIO:** Los LEDs son activos en bajo. El top level ya invierte la señal (`led_n = ~gpio_out`). LEDs externos tienen corriente máxima de 8 mA por pin.

**Reset:** La placa tiene pull-up interno en S1. Se puede agregar un capacitor de 100 nF para estabilizar el power-on reset.

Para señales internas usar **Gowin Analyzer Oscilloscope** (`Tools → Gowin Analyzer Oscilloscope`) sin modificar el pinout.
