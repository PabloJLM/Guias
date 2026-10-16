# Instalacion de  WSL + PlatformIO (o Arduino xd)

Guía para programar microcontroladores (ESP32, STM32, etc.) editando en Windows y **compilando en Linux** (WSL 2).

---

## 1. Instalar WSL y Ubuntu

Abre **PowerShell como administrador** y ejecuta:

```powershell
wsl --install -d Ubuntu-24.04
```

Si falla o dice que la instalación está dañada (por ejemplo `REGDB_E_CLASSNOTREG`), repara y reintenta sin pasar por Microsoft Store:

```powershell
wsl --update
wsl --install --web-download -d Ubuntu-24.04
```

1. Reinicia la PC si lo pide.
2. Se abre una terminal de Ubuntu y pide crear **usuario** y **contraseña**
3. Comprueba la versión:

```powershell
wsl --version
wsl -l -v
```

La distribución debe aparecer con `VERSION 2`.

> `wsl --shutdown`, `wsl --update` y los demás comandos que empiezan con `wsl --` se escriben en **PowerShell**, nunca dentro de Ubuntu.

---

## 2. Preparar Ubuntu

Abre Ubuntu (Windows Terminal → Ubuntu-24.04). Si arranca en `/mnt/c/...`, entra a tu carpeta con `cd ~`.

```bash
cd ~
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl python3-venv pipx rsync
sudo usermod -aG dialout $USER
```

`dialout` da permiso para usar puertos serie sin ser administrador.

Cierra Ubuntu y, en **PowerShell**, reinicia WSL para que el grupo se aplique:

```powershell
wsl --shutdown
```

Vuelve a abrir Ubuntu y comprueba:

```bash
groups
```

En la lista debe aparecer `dialout`.

---

## 3. Elegir herramienta de compilación

### Opción A: PlatformIO Core (recomendada)

PlatformIO es una herramienta de línea de comandos. No necesita la extensión de VS Code. Soporta Arduino, ESP-IDF, STM32 y otros frameworks desde un mismo archivo de configuración.

```bash
pipx ensurepath
pipx install platformio
exec $SHELL
pio --version
```

Debe imprimir la versión de PlatformIO Core.

### Opción B: Arduino

Hay tres formas:

**1. Arduino IDE en Windows** 
Se instala como cualquier programa de Windows. Compila con el toolchain de Windows, sin WSL, y el USB funciona directo. 

*NO RECOMENDABLE POR QUE WINDOWS RECOMPILA CADA LIBRERIA EN CADA ITERACION DE COMPILADO*

**2. `arduino-cli` dentro de WSL.** Compila en Linux por línea de comandos.

```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | BINDIR=$HOME/.local/bin sh
arduino-cli core update-index
arduino-cli core install esp32:esp32
arduino-cli compile --fqbn esp32:esp32:esp32s3 MiSketch
arduino-cli upload -p /dev/ttyACM0 --fqbn esp32:esp32:esp32s3 MiSketch
```

**3. Arduino IDE dentro de WSL (ventana gráfica con WSLg).** El IDE y el compilador corren en Linux y la ventana aparece en Windows.

```bash
sudo apt install -y libfuse2t64 libgtk-3-0t64 libnss3 libasound2t64 libgbm1 libxss1
chmod +x ~/arduino-ide_*.AppImage
~/arduino-ide_*.AppImage --no-sandbox
```

El AppImage ("Linux AppImage 64 bits") se descarga desde arduino.cc. Las placas y librerías se instalan aparte en Linux, y el USB también hay que pasarlo con `usbipd` 

---

## 4. Crear un proyecto y su `platformio.ini`

### 4.1 Dónde crear la carpeta

**Opción 1: el proyecto en Windows** (VS Code de Windows y cualquier programa lo ven). Es la que usa esta guía:

```bash
mkdir -p /mnt/c/Arduino/MiProyecto
cd /mnt/c/Arduino/MiProyecto
```

**Opción 2: el proyecto en Linux** (más rápido, pero los programas de Windows lo ven solo como ruta de red):

```bash
mkdir -p ~/proyectos/MiProyecto
cd ~/proyectos/MiProyecto
```

### 4.2 Generar la estructura

```bash
pio boards esp32-s3          # buscar el identificador de tu placa
pio project init --board esp32-s3-devkitc-1
```

Crea esta estructura:

```
MiProyecto/
├── platformio.ini     <- configuración (placa, framework, librerías)
├── src/               <- tu código (main.cpp y demás .cpp/.h)
├── include/           <- cabeceras globales
├── lib/               <- librerías propias
├── test/
└── data/              <- (opcional) archivos para el sistema de archivos LittleFS
```

### 4.3 Primer programa de prueba

`src/main.cpp`:

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
}

void loop() {
  Serial.println("hola desde PlatformIO en WSL");
  delay(1000);
}
```

### 4.4 Ejemplos de `platformio.ini`

La sintaxis es: una sección por entorno (`[env:nombre]`), con `platform`, `board` y `framework`.

**ESP32-S3 con la plataforma oficial (Arduino-ESP32 2.x):**

```ini
[env:esp32s3]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200
; Si el puerto USB-C es el USB nativo del chip, para ver Serial por ahí:
build_flags = -DARDUINO_USB_CDC_ON_BOOT=1
```

**ESP32-S3 con Arduino-ESP32 3.x vía pioarduino, 16 MB de flash, PSRAM, LittleFS y librerías:**

```ini
[platformio]
; Solo si el proyecto está en Windows: manda compilación y librerías al disco de Linux.
workspace_dir = /home/TU_USUARIO/.pio_workspaces/mi_proyecto

[env:mi_placa]
; Verifica la URL vigente en el README de pioarduino.
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
framework = arduino
board = esp32-s3-devkitc-1

board_upload.flash_size = 16MB
board_upload.maximum_size = 16777216
board_build.arduino.memory_type = qio_qspi     ; flash QIO + PSRAM quad
board_build.partitions = default_16MB.csv      ; trae una partición con etiqueta "spiffs"
board_build.filesystem = littlefs

lib_deps = lovyan03/LovyanGFX

monitor_speed = 115200
```

Notas:
- `workspace_dir` sustituye a la carpeta `.pio/` del proyecto. Con ella, los fuentes pueden estar en Windows y la compilación ir sobre el disco de Linux, que es mucho más rápido. Cambia `TU_USUARIO` por tu usuario de Linux.
- LittleFS necesita una partición cuya etiqueta sea `spiffs`. Con `default_16MB.csv` la tiene.
- Lo que haya en la carpeta `data/` se sube a la placa con `pio run -t uploadfs`.

**Varias placas en el mismo proyecto:**

```ini
[platformio]
default_envs = esp32s3

[env]                       ; valores comunes a todos los entornos
framework = arduino
monitor_speed = 115200

[env:esp32s3]
platform = espressif32
board = esp32-s3-devkitc-1

[env:nucleo_f303k8]
platform = ststm32
board = nucleo_f303k8
upload_protocol = stlink
```

Se elige con `pio run -e esp32s3` o `pio run -e nucleo_f303k8`.

**ESP-IDF en lugar de Arduino:**

```ini
[env:esp32s3_idf]
platform = espressif32
board = esp32-s3-devkitc-1
framework = espidf
```

Este framework exige estructura CMake (`src/CMakeLists.txt`) y `app_main()` en lugar de `setup()`/`loop()`

*Mucho mas control a bajo nivel, pero una mayor curva de aprendizaje*

*Recomendado solo si es un proyecto que requiera de interrupciones, control avanzado de perifericos o directamente el uso de RTOS*

---

## 5. Compilar y subir

Desde Ubuntu, dentro de la carpeta del proyecto:

```bash
pio run                     # compilar (la primera vez tarda)
pio run -t upload           # compilar y grabar el firmware
pio run -t uploadfs         # subir la carpeta data/ a LittleFS
pio device monitor -b 115200   # ver lo que imprime la placa (salir: Ctrl+C)
pio run -t clean            # borrar lo compilado
pio device list             # puertos que ve Linux
```

### Desde la terminal de VS Code en Windows

La terminal por defecto es PowerShell. Hay dos maneras de usar Ubuntu:

```powershell
wsl                                              # entra a Ubuntu en esa misma terminal
wsl -d Ubuntu-24.04 -- bash -lc "pio run"        # un comando suelto y vuelve a PowerShell
```

El `bash -lc` carga el entorno de Ubuntu y así encuentra `pio`. Con `wsl pio run` a secas puede decir que no encuentra el comando.

---

## 6. Pasar la placa a WSL con usbipd-win

Windows y Linux no pueden usar el mismo puerto USB a la vez. `usbipd-win` "presta" la placa a WSL. Mientras está prestada, Windows no la ve (el puerto COM desaparece).

### 6.1 Instalar (una sola vez)

Descarga el `.msi` de la página de releases de `usbipd-win` y ejecútalo. También sirve `winget install --interactive --exact dorssel.usbipd-win` si tienes `winget`. **Cierra y vuelve a abrir PowerShell** después de instalar.

### 6.2 Localizar la placa

Con la placa conectada, en PowerShell:

```powershell
usbipd list
```

Aparece una lista con una columna `BUSID` (por ejemplo `1-2`). Una ESP32 con USB nativo suele llamarse "USB JTAG/serial debug unit" (`303a:1001`). Con CP210x o CH340 aparece con ese nombre.

### 6.3 Compartir y adjuntar

El BUSID de abajo es un ejemplo: **reemplázalo por el tuyo** (sin los símbolos `<` `>`).

```powershell
usbipd bind --busid 1-2                          # una sola vez; PowerShell como administrador
usbipd attach --wsl --busid 1-2 --auto-attach    # no requiere administrador; déjalo abierto
```

- `bind` necesita administrador y se hace **una sola vez** por dispositivo.
- `attach` no necesita administrador. Necesita que haya una terminal de Ubuntu abierta para que WSL esté activo.
- `--auto-attach` vuelve a conectar la placa sola cada vez que se reinicia o cambia de modo (importante en las ESP32 con USB nativo, que cambian de identificador al entrar en modo de carga). Mientras la ventana esté abierta, la conexión se mantiene; si la cierras, la placa vuelve a Windows.

### 6.4 Comprobar en Ubuntu

```bash
ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

Debe aparecer `/dev/ttyACM0` (USB nativo) o `/dev/ttyUSB0` (CP210x/CH340). Ya puedes ejecutar `pio run -t upload`.

### 6.5 Devolver la placa a Windows

```powershell
usbipd detach --busid 1-2
```

O desconecta y vuelve a conectar el cable.

### 6.6 Atajo para no abrir otra ventana (opcional)

Desde Ubuntu se puede lanzar el `attach` de Windows:

```bash
echo "alias placa='powershell.exe -NoProfile -Command \"usbipd attach --wsl --busid 1-2 --auto-attach\"'" >> ~/.bashrc
source ~/.bashrc
```

Luego `placa` en una pestaña de Ubuntu la deja corriendo.

### 6.7 Si no hay permiso sobre el puerto

La documentación de Microsoft indica que, según la aplicación, puede hacer falta una regla `udev`. Con el usuario en el grupo `dialout` (sección 2) normalmente basta.

---

## 7. Editar con VS Code de Windows

1. Abre la carpeta del proyecto (`C:\Arduino\MiProyecto`) con **File → Open Folder**.
2. Edita con normalidad. No necesitas ninguna extensión para compilar.
3. En la terminal integrada ejecuta `wsl` para entrar a Ubuntu en esa carpeta, y compila con `pio run`.
4. Usa dos terminales: una para `usbipd attach` (PowerShell) y otra para compilar y subir (Ubuntu).

---

## 8. GUI propia (en progreso, quiza)

## Referencias

- [Conectar dispositivos USB a WSL (Microsoft Learn)](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)
- [usbipd-win en GitHub](https://github.com/dorssel/usbipd-win)
- [PlatformIO Core: instalación](https://docs.platformio.org/en/latest/core/installation/index.html)
- [Configuración de `platformio.ini`](https://docs.platformio.org/en/latest/projectconf/index.html)
- [pioarduino (Arduino-ESP32 3.x en PlatformIO)](https://github.com/pioarduino/platform-espressif32)
- [Arduino CLI](https://arduino.github.io/arduino-cli/)

- [De verdad desconfias de mi?](https://youtu.be/dQw4w9WgXcQ)
