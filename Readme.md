# Guía de Instalación — ESP32 Marauder (Placa OG / MARAUDER_V4)

## 1. Objetivo

Documentar el proceso completo de compilación e instalación del firmware ESP32 Marauder sobre una placa ESP32 genérica (WROOM, sin PSRAM) con una pantalla táctil SPI no estándar, incluyendo todos los parches de compatibilidad necesarios para el core de Arduino 2.0.17.

Esta guía es de uso educativo, orientada a una capacitación sobre auditoría de redes WiFi en entornos propios o autorizados.

---

## 2. Requisitos previos

### Software
- Editor de código: **Arduino IDE 1.8.19** (legacy). No se recomienda Arduino IDE 2.x para este proceso, ya que el plugin de subida de SPIFFS que se usa más adelante no es compatible con esa versión.
- **Python 3** instalado:
  ```bash
  sudo apt install python3
  ```
- **pip3** y el módulo **pyserial** (requerido por `esptool.py` para la comunicación serial durante la subida del firmware):
  ```bash
  pip3 install pyserial --break-system-packages
  ```
  > El flag `--break-system-packages` es necesario en distribuciones Debian/Ubuntu/Kali recientes que gestionan Python como "entorno externamente administrado".

### Hardware

> ⚠️ Sección pendiente de completar — modelo definitivo de placa ESP32, GPS y componentes adicionales aún por confirmar.

| Componente | Modelo / Detalle |
|---|---|
| Placa ESP32 | ESP32-D0WDQ6 (sin PSRAM, variante WROOM), 4MB flash, dual core |
| Pantalla | ILI9486, 320×480, SKU MPI3501, touch resistivo XPT2046, diseño tipo "RPi" |
| Tarjeta microSD | *(pendiente — módulo lector SPI, formateada en FAT32)* |
| Módulo GPS | *(pendiente — modelo y velocidad de actualización)* |
| LED direccionable (NeoPixel) | *(pendiente — pin a confirmar en `config.h`)* |
| Botones físicos | *(pendiente — la pantalla incluye 3 botones sin usar aún)* |
| Batería / gestión de energía | *(pendiente — IP5306 vía I2C, SDA=GPIO33, SCL=GPIO22 según documentación oficial; a confirmar si se implementa)* |

---

## 3. Cableado (pantalla ILI9486 320×480)

| Pantalla (pin físico original) | Señal | Pin ESP32 |
|---|---|---|
| Pines 1, 17 | 3.3V | 3V3 |
| Pines 2, 4 | 5V (backlight) | 5V / VIN |
| Pines 6, 9, 14, 20, 25 | GND | GND |
| Pin 24 | LCD_CS | GPIO15 |
| Pin 18 | LCD_RS (D/C) | GPIO2 |
| Pin 22 | RST | **GPIO26** *(ver nota de conflicto con GPS)* |
| Pin 19 | LCD_SI / TP_SI (MOSI) | GPIO23 |
| Pin 23 | LCD_SCK / TP_SCK (SCK) | GPIO18 |
| Pin 21 | TP_SO (MISO) | GPIO19 |
| Pin 26 | TP_CS | GPIO21 |
| Pin 11 | TP_IRQ | No conectado |
| — | Backlight (TFT_BL) | GPIO33 *(pin libre, sin control real de brillo; requerido solo para que compile)* |

### Módulo GPS

| GPS | Pin ESP32 |
|---|---|
| TX del GPS | GPIO4 |
| RX del GPS | GPIO13 |
| VCC | Según datasheet del módulo |
| GND | GND común |

> **⚠️ Conflicto de pines resuelto:** el firmware de Marauder (bloque `MARAUDER_V4`) tiene **GPIO4 reservado por hardware para el GPS** (`GPS_TX`), en un UART distinto (`GPS_SERIAL_INDEX 2`) que no conviene reasignar por código. Como GPIO4 originalmente estaba ocupado por `TFT_RST`, se movió el reset de la pantalla a **GPIO26** (pin libre, sin lógica especial asociada) para liberar GPIO4 exclusivamente para el GPS.

### Diagrama esquemático

> *(pendiente de adjuntar imagen/diagrama)*

---

## 4. Configuración del entorno Arduino IDE

### 4.1 Instalar el core de ESP32

1. `Archivo > Preferencias` → agregar en "URLs adicionales de gestor de placas":
   ```
   https://espressif.github.io/arduino-esp32/package_esp32_index.json
   ```
2. `Herramientas > Placa > Gestor de Placas` → buscar **"esp32"** → instalar la versión **2.0.17** específicamente (no la 3.x).

   > La versión 2.0.17 es obligatoria: la 3.x usa un toolchain (GCC 14) que genera errores de `multiple definition` irresolubles con el código actual de Marauder, y además presenta incompatibilidades con el plugin de subida de SPIFFS.

### 4.2 Parchear `platform.txt`

Editar:
```bash
sudo nano ~/.arduino15/packages/esp32/hardware/esp32/2.0.17/platform.txt
```

Agregar `-w` al final de la línea `build.extra_flags.esp32`:
```
build.extra_flags.esp32=-DARDUINO_USB_CDC_ON_BOOT=0 -w
```

Agregar `-zmuldefs` al final de la línea `compiler.c.elf.libs.esp32`:
```
compiler.c.elf.libs.esp32=-zmuldefs -lesp_ringbuf -lefuse -lesp_ipc -ldriver -lesp_pm ...
```

> Este parche es necesario porque el código de Marauder define ciertos símbolos (`index_html`, `operationInProgress`, `ieee80211_raw_frame_sanity_check`, entre otros) en más de un archivo `.h` incluido en múltiples `.cpp` — comportamiento tolerado en toolchains antiguos pero rechazado por defecto en versiones modernas de GCC. `-zmuldefs` le indica al enlazador que lo permita.

### 4.3 Instalar el plugin de subida de SPIFFS

1. Crear la carpeta `tools` dentro del sketchbook:
   ```bash
   mkdir -p ~/Arduino/tools
   ```
2. Descargar la última release de [arduino-esp32fs-plugin](https://github.com/me-no-dev/arduino-esp32fs-plugin) y descomprimirla dentro de `~/Arduino/tools/`, de forma que quede:
   ```
   ~/Arduino/tools/ESP32FS/tool/esp32fs.jar
   ```
3. Reiniciar Arduino IDE. Debería aparecer la opción `Herramientas > ESP32 Sketch Data Upload`.

---

## 5. Instalación de librerías

Todas las librerías se descargan de sus repositorios originales (no existe un paquete oficial que las incluya todas juntas) y se colocan en `~/Arduino/libraries/`.

| Librería | Repositorio | Versión usada |
|---|---|---|
| LinkedList | https://github.com/ivanseidel/LinkedList | master |
| TFT_eSPI (fork Marauder) | https://github.com/justcallmekoko/TFT_eSPI | master |
| JPEGDecoder | https://github.com/Bodmer/JPEGDecoder | master |
| NimBLE-Arduino | https://github.com/h2zero/NimBLE-Arduino | master |
| Adafruit NeoPixel | https://github.com/adafruit/Adafruit_NeoPixel | master |
| ArduinoJson | https://github.com/bblanchon/ArduinoJson/releases/tag/v6.18.2 | **v6.18.2 (fija)** |
| SwitchLib | https://github.com/justcallmekoko/SwitchLib | master |
| ESPAsyncWebServer | https://github.com/ESP32Async/ESPAsyncWebServer | master |
| AsyncTCP | https://github.com/ESP32Async/AsyncTCP | master |
| ESP32Ping | https://github.com/marian-craciunescu/ESP32Ping | master |
| MicroNMEA | https://github.com/stevemarple/MicroNMEA | master |
| XPT2046_Touchscreen | https://github.com/PaulStoffregen/XPT2046_Touchscreen | master |
| EspSoftwareSerial | https://github.com/plerup/espsoftwareserial/releases/tag/6.17.1 | **6.17.1 (fija)** |
| Adafruit BusIO | https://github.com/adafruit/Adafruit_BusIO | master |
| Adafruit MAX1704X | https://github.com/adafruit/Adafruit_MAX1704X | master |

> **Nota sobre versiones:** `ArduinoJson` y `EspSoftwareSerial` **deben** instalarse en la versión exacta indicada (no "master"/última), ya que versiones más nuevas rompen la compatibilidad con el código de Marauder (ver sección de errores comunes). Para el resto de las librerías, "master" no presentó problemas al momento de esta guía, pero se recomienda anotar el commit/fecha de descarga para reproducibilidad futura, dado que "master" no es una versión fija en el tiempo.
>
> **Cuidado con duplicados:** verificar que no queden dos carpetas de la misma librería instaladas a la vez (por ejemplo, una instalada por ZIP y otra por el Gestor de Bibliotecas) — esto puede causar errores de compilación e incluso crashes del compilador (`arduino-builder`). Verificar con:
> ```bash
> ls ~/Arduino/libraries/
> ```

---

## 6. Descarga del firmware

Descargar o clonar el repositorio oficial:
```
https://github.com/justcallmekoko/ESP32Marauder
```

Abrir `esp32_marauder/esp32_marauder.ino` con Arduino IDE (los demás archivos del proyecto se cargan automáticamente como pestañas).

---

## 7. Configuración de `config.h`

### 7.1 Selección de placa

En la sección `//// BOARD TARGETS`, descomentar únicamente:
```cpp
#define MARAUDER_V4
```

### 7.2 Ajuste de resolución de pantalla

Dentro del bloque `#ifdef MARAUDER_V4`, reemplazar:
```cpp
#ifndef TFT_WIDTH
  #define TFT_WIDTH 240
#endif

#ifndef TFT_HEIGHT
  #define TFT_HEIGHT 320
#endif
```
por:
```cpp
#undef TFT_WIDTH
#define TFT_WIDTH 320
#undef TFT_HEIGHT
#define TFT_HEIGHT 480
```

> Esto evita depender del orden de inclusión de archivos, forzando la resolución correcta en todo el proyecto sin importar qué `.cpp` se compile primero.

### 7.3 Ajuste de `YMAX`

En el mismo bloque, cambiar:
```cpp
#define YMAX 320
```
por:
```cpp
#define YMAX TFT_HEIGHT
```

> `YMAX` estaba fijo en 320 y se usa para calcular las zonas táctiles de navegación (arriba/centro/abajo). Con una pantalla de 480px de alto, dejarlo fijo en 320 provocaba que el touch no respondiera correctamente en la parte inferior de la pantalla.

### 7.4 Mantener activo `HAS_IDF_3`

Dejar `#define HAS_IDF_3` **activo** (sin comentar) dentro del bloque `MARAUDER_V4`. No desactivar este flag — desactivarlo reactiva un bloque de código alternativo (`wifi_init_config_t` con funciones de ESP-IDF v3) que tampoco es compatible con el core 2.0.17, causando otro error de compilación.

---

## 8. Configuración de `User_Setup.h` (librería TFT_eSPI)

Editar:
```bash
nano ~/Arduino/libraries/TFT_eSPI/User_Setup.h
```

```cpp
#define RPI_ILI9486_DRIVER

#define TFT_WIDTH  320
#define TFT_HEIGHT 480

#define TFT_MISO 19
#define TFT_MOSI 23
#define TFT_SCLK 18
#define TFT_CS   15
#define TFT_DC    2
#define TFT_RST  26
#define TFT_BL   33

#define TOUCH_CS 21

#define LOAD_GLCD
#define LOAD_FONT2
#define LOAD_FONT4
#define LOAD_FONT6
#define LOAD_FONT7
#define LOAD_FONT8
#define LOAD_GFXFF
#define SMOOTH_FONT

#define SPI_FREQUENCY  20000000
#define SPI_READ_FREQUENCY  16000000
#define SPI_TOUCH_FREQUENCY  2500000
```

> **`RPI_ILI9486_DRIVER` vs `ILI9486_DRIVER`:** las pantallas ILI9486 vendidas para Raspberry Pi suelen incluir chips lógicos intermedios (74HC04, 74HC4094, etc.) que convierten SPI a paralelo de 16 bits. El driver genérico `ILI9486_DRIVER` no es compatible con ese esquema y produce una pantalla en blanco (backlight encendido, sin imagen). `RPI_ILI9486_DRIVER` es el modo específico de TFT_eSPI para este tipo de módulo.

### Resumen de configuración

| Parámetro | Valor |
|---|---:|
| Driver | `RPI_ILI9486_DRIVER` |
| Resolución | 320 × 480 |
| MISO | GPIO19 |
| MOSI | GPIO23 |
| SCLK | GPIO18 |
| CS | GPIO15 |
| DC | GPIO2 |
| RST | GPIO26 |
| Backlight (TFT_BL) | GPIO33 |
| Touch CS | GPIO21 |
| SPI | 20 MHz |
| SPI lectura | 16 MHz |
| SPI Touch | 2.5 MHz |

---

## 9. Parches de código fuente

### 9.1 `WiFiScan.cpp` — error `WIFI_AUTH_WPA3_ENTERPRISE`

El core 2.0.17 (ESP-IDF 4.4.7) no incluye esta constante (agregada recién en ESP-IDF 5.2+). Ubicar (~línea 3235):

```cpp
#ifdef HAS_IDF_3
    case WIFI_AUTH_WPA3_ENTERPRISE:
      authtype = "[WPA3]";
      break;
    #endif
```

Comentar únicamente las 3 líneas internas, dejando el `#ifdef`/`#endif` intactos:

```cpp
#ifdef HAS_IDF_3
    //case WIFI_AUTH_WPA3_ENTERPRISE:
    //  authtype = "[WPA3]";
    //  break;
    #endif
```

### 9.2 `Display.cpp` — calibración de touch

La función `setCalData()` no contempla un caso para `MARAUDER_V4`, por lo que el touch nunca recibe calibración real y usa valores crudos sin mapear. Agregar un caso nuevo en la rama `if (!landscape)`:

```cpp
if (!landscape) {
  #ifdef TFT_SHIELD
    uint16_t calData[5] = { 275, 3494, 361, 3528, 4 };
  #elif defined(MARAUDER_CYD_3_5_INCH)
    uint16_t calData[5] = { 239, 3560, 262, 3643, 4 };
  #elif defined(MARAUDER_V8)
    uint16_t calData[5] = { 312, 3431, 191, 3456, 2 };
  #elif defined(TFT_DIY)
    uint16_t calData[5] = { 339, 3470, 237, 3438, 2 };
  #elif defined(MARAUDER_V4)
    uint16_t calData[5] = { 230, 3654, 276, 3684, 6 };  // Calibración propia
  #endif
  #ifdef HAS_ILI9341
    tft.setTouch(calData);
  #endif
}
```

> Los valores `{ 230, 3654, 276, 3684, 6 }` se obtuvieron corriendo la rutina `tft.calibrateTouch()` en un sketch de prueba aislado, con la **misma rotación (`setRotation(0)`)** que usa Marauder internamente (`SCREEN_ORIENTATION`). Si se cambia de pantalla física, esta calibración debe rehacerse.

> **Nota sobre la interfaz táctil:** `MARAUDER_V4` usa un esquema de navegación por **zonas** (tercio superior = subir, tercio medio = seleccionar, tercio inferior = bajar), no botones dibujados con precisión de píxel. Esto es el diseño original de esa versión de hardware, no una limitación de nuestra configuración.

---

## 10. Compilación y subida

1. `Herramientas > Placa` → seleccionar **ESP32 Dev Module** (bajo el core 2.0.17).
2. `Herramientas > Puerto` → seleccionar el puerto correspondiente.
3. `Herramientas > Partition Scheme` → **"Minimal SPIFFS (1.9MB APP with OTA/190KB SPIFFS)"**.
   > Este ajuste debe reconfirmarse cada vez que se reinstala el core o se cambia de placa, ya que Arduino IDE 1.8.19 no siempre lo conserva entre sesiones.
4. `Herramientas > ESP32 Sketch Data Upload` (sube el sistema de archivos SPIFFS necesario para el firmware).
5. Compilar y subir `esp32_marauder.ino` (botón de subir, no solo verificar).

---

## 11. Errores comunes y soluciones

| Error | Causa | Solución |
|---|---|---|
| `SPIFFS Error: esptool not found!` | Incompatibilidad entre el plugin ESP32FS y el core instalado | Usar core **2.0.17** específicamente |
| `ModuleNotFoundError: No module named 'serial'` | Falta `pyserial` en el entorno Python del sistema | `pip3 install pyserial --break-system-packages` |
| `circular_queue.h: No such file or directory` | Versión nueva de `EspSoftwareSerial` reestructuró sus archivos internos | Instalar específicamente la versión **6.17.1** |
| `Adafruit_MAX1704X.h: No such file or directory` | Librería no instalada o instalada incompleta | Instalar desde el Gestor de Bibliotecas de Arduino IDE |
| `multiple definition of 'index_html'` / `ieee80211_raw_frame_sanity_check` | GCC moderno no tolera símbolos definidos en headers incluidos múltiples veces | Agregar `-zmuldefs` en `platform.txt` |
| `WIFI_AUTH_WPA3_ENTERPRISE' was not declared` | Constante no existe en ESP-IDF 4.4.7 (core 2.0.17) | Comentar el `case` específico en `WiFiScan.cpp` (ver sección 9.1) |
| `esp_event_send_internal' was not declared` | Se desactivó `HAS_IDF_3` sin querer, activando código para ESP-IDF v3 (aún más viejo) | Mantener `HAS_IDF_3` activo; parchear solo el `case` puntual |
| `TFT_BL' was not declared in this scope` | Falta definir el pin de backlight en `User_Setup.h` | Agregar `#define TFT_BL 33` (pin libre, sin conexión física real necesaria) |
| Pantalla en blanco (backlight encendido, sin imagen) | Uso de `ILI9486_DRIVER` en vez de `RPI_ILI9486_DRIVER` en pantallas ILI9486 tipo RPi con chips lógicos | Cambiar a `RPI_ILI9486_DRIVER` |
| Touch impreciso / activa zona incorrecta | Falta de calibración específica para `MARAUDER_V4`, y `YMAX` fijo en 320 en vez de `TFT_HEIGHT` | Ver sección 9.2 y 7.3 |
| `panic: runtime error: index out of range` (crash de `arduino-builder`) | Caché de includes corrupta, a veces agravada por librerías duplicadas | Limpiar `/tmp/arduino*` y verificar que no haya carpetas de librerías repetidas |
| `text section exceeds available space in board` | Esquema de partición reseteado al default tras reinstalar el core | Volver a seleccionar "Minimal SPIFFS" |
| `Could not find /index.html` (Evil Portal) | El Evil Portal requiere sí o sí una tarjeta microSD física; no existe carga por Serial en el firmware oficial | Instalar módulo lector de microSD |
| `GPS Not Found` / `Could not detect GPS baudrate` | TX/RX del GPS invertidos, o GND no compartido | Verificar cableado (sección 3) e invertir TX/RX si es necesario |

---

## 12. Función Evil Portal

Requiere obligatoriamente una tarjeta microSD (formateada en FAT32) con un archivo `index.html` en la raíz.

**Flujo de uso:**
1. Seleccionar/clonar un SSID (`WiFi > Scanners > Scan APs`, o `Add SSID`).
2. Colocar `index.html` en la raíz de la SD.
3. Iniciar `WiFi Attacks > Evil Portal`.
4. (Opcional) Activar `EPDeauth` para forzar a los clientes a desconectarse de la red real y conectarse al AP falso.
5. Los datos ingresados por la víctima se muestran en tiempo real por Serial y en pantalla.

---

## 13. Automatización del entorno con PlatformIO (opcional)

Para reducir el proceso manual de instalación de librerías y parches de `platform.txt`, se armó un `platformio.ini` equivalente:

```ini
[env:esp32dev]
platform = espressif32 @ 7.0.1
board = esp32dev
framework = arduino

board_build.partitions = min_spiffs.csv
upload_speed = 921600
monitor_speed = 115200

lib_deps =
    ivanseidel/LinkedList
    https://github.com/justcallmekoko/TFT_eSPI.git
    bodmer/JPEGDecoder
    h2zero/NimBLE-Arduino
    adafruit/Adafruit NeoPixel
    bblanchon/ArduinoJson @ 6.18.2
    https://github.com/justcallmekoko/SwitchLib.git
    ESP32Async/ESPAsyncWebServer
    ESP32Async/AsyncTCP
    marian-craciunescu/ESP32Ping
    stevemarple/MicroNMEA
    paulstoffregen/XPT2046_Touchscreen
    plerup/espsoftwareserial @ 6.17.1
    adafruit/Adafruit BusIO
    adafruit/Adafruit MAX1704X

build_flags =
    -w
    -Wl,-zmuldefs
    -D MARAUDER_V4
    -D HAS_IDF_3
    -D RPI_ILI9486_DRIVER
    -D TFT_WIDTH=320
    -D TFT_HEIGHT=480
    -D TFT_MISO=19
    -D TFT_MOSI=23
    -D TFT_SCLK=18
    -D TFT_CS=15
    -D TFT_DC=2
    -D TFT_RST=26
    -D TFT_BL=33
    -D TOUCH_CS=21
    -D LOAD_GLCD
    -D LOAD_FONT2
    -D LOAD_FONT4
    -D LOAD_FONT6
    -D LOAD_FONT7
    -D LOAD_FONT8
    -D LOAD_GFXFF
    -D SMOOTH_FONT
    -D SPI_FREQUENCY=20000000
    -D SPI_READ_FREQUENCY=16000000
    -D SPI_TOUCH_FREQUENCY=2500000
```

> `platform = espressif32 @ 7.0.1` corresponde exactamente al Arduino core 2.0.17 (ESP-IDF 4.4.7).
>
> Los parches de código fuente (sección 9) **no se pueden automatizar** vía configuración — siguen requiriendo edición manual de `WiFiScan.cpp` y `Display.cpp`.

---

## 14. Complementos de hardware y siguientes pasos

- [ ] Definir pin del LED NeoPixel (`HAS_NEOPIXEL_LED` ya está activo en `config.h`, falta confirmar GPIO).
- [ ] Habilitar y cablear los 3 botones físicos de la pantalla (`HAS_BUTTONS`, actualmente comentado).
- [ ] Confirmar implementación de gestión de batería (IP5306 vía I2C: SDA=GPIO33, SCL=GPIO22 según documentación oficial — **verificar que no choque con `TFT_BL` en GPIO33**).
- [ ] Adquirir e instalar módulo lector de microSD para habilitar Evil Portal, guardado de PCAP y wardriving con log.
- [ ] Diagrama esquemático de cableado completo.
- [ ] Case de impresión 3D: https://www.printables.com/model/651095-esp32-marauder-case

---

## 15. Marco de uso

Este material y el firmware demostrado deben usarse exclusivamente sobre redes propias o en entornos con autorización explícita, con fines educativos y de auditoría de seguridad. El uso de estas herramientas sobre redes o dispositivos de terceros sin autorización puede constituir un delito según la legislación local.