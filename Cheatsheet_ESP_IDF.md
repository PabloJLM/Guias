# ESP32-IDF-Cheatsheet
Cheatsheet para programar microcontroladores ESP32 con el ESP32-IDF

**necesitas al menos ESP-IDF v5.2.X**

# INDICE

1. [GPIO – Entradas y Salidas](#gpio--entradas-y-salidas)
2. [FreeRTOS – Tareas y Delays](#freertos--tareas-y-delays)
3. [UART (Serial)](#uart-serial)
4. [I2C (Modo Maestro)](#i2c-modo-maestro)
5. [SPI (Modo Maestro)](#spi-modo-maestro)
6. [ADC](#adc-conversor-analógico-digital)
7. [DAC](#dac-conversor-digital-analógico)
8. [PWM (LEDC)](#pwm-ledc)
9. [Wi-Fi Modo Estación (Cliente)](#wi-fi-modo-estacion-cliente)
10. [ESP-NOW](#esp-now)
11. [NVS](#nvs-almacenamiento-no-volatil)
12. [Timers por Hardware](#timers-por-hardware)
13. [OTA / HTTPS](#ota--https)
14. [Logs y Debug](#logs-y-debug)
15. [Deep Sleep](#deep-sleep-modo-bajo-consumo)
16. [RTC](#rtc-real-time-clock-y-memoria-persistente)
17. [MQTT](#mqtt-cliente-conectado-a-broker)
18. [Wi-Fi Access Point](#wi-fi-access-point)
19. [Servo (usando PWM/LEDC)](#servo-usando-pwmledc)
20. [Botones](#botones-con-debounce-por-software)
21. [MPU 6050 Registros](#mpu-6050-registros)
22. [Pantalla I2C 16x2](#pantalla-i2c-16x2)
23. [FreeRTOS – Multitarea, Sensores y Deep Sleep](#parte-importante-freertos-multitarea-sensores-y-deep-sleep)


## GPIO – Entradas y Salidas

```c
#include "driver/gpio.h"

// Forma con numero
#define LED_GPIO 48

// Forma oficial con enum
// #define LED_GPIO GPIO_NUM_48

void app_main(void)
{
    gpio_reset_pin(LED_GPIO);                        // Limpia el pin
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);  // Define como salida
    gpio_set_level(LED_GPIO, 1);                     // Enciende LED
    vTaskDelay(pdMS_TO_TICKS(1000));
    gpio_set_level(LED_GPIO, 0);                     // Apaga LED
}
```
## FreeRTOS – Tareas y Delays
```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define LED_GPIO GPIO_NUM_2

void blink_task(void *pvParameter)//es un puntero de tipo generico que se usa para pasar datos a una tarea cuando se crea
{
    while (1) {
        gpio_set_level(LED_GPIO, 1);                 // LED ON
        vTaskDelay(pdMS_TO_TICKS(500));              // Espera 500 ms
        gpio_set_level(LED_GPIO, 0);                 // LED OFF
        vTaskDelay(pdMS_TO_TICKS(500));              // Espera 500 ms
    }
}

void app_main(void)
{
    gpio_reset_pin(LED_GPIO);                        // Limpia el pin
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);  // Configura pin como salida

    // Crea una tarea llamada “blink_task” con stack de 2048 bytes y prioridad 5
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
}

```

## UART (Serial)
```c
#include "driver/uart.h"

#define UART_TX_PIN 17
#define UART_RX_PIN 16

void app_main(void)
{
    // Configura UART: velocidad, bits, paridad, etc.
    uart_config_t uart_config = {
        .baud_rate = 115200,             // Velocidad en baudios
        .data_bits = UART_DATA_8_BITS,   // 8 bits de datos
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE
    };

    uart_param_config(UART_NUM_1, &uart_config);     // Aplica config
    uart_set_pin(UART_NUM_1, UART_TX_PIN, UART_RX_PIN,
                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
    uart_driver_install(UART_NUM_1, 1024, 0, 0, NULL, 0);

    const char *msg = "Hola ESP-IDF\r\n";
    uart_write_bytes(UART_NUM_1, msg, strlen(msg));  // Envia
}

```

## I2C (Modo Maestro)
```c
#include "driver/i2c.h"

#define I2C_SDA 21
#define I2C_SCL 22

void app_main(void)
{
    // Configuracion i2c
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_SDA,
        .scl_io_num = I2C_SCL,
        .master.clk_speed = 100000  // 100 kHz
    };

    i2c_param_config(I2C_NUM_0, &conf);              // Aplica config
    i2c_driver_install(I2C_NUM_0, conf.mode, 0, 0, 0); // Instala driver
}


```

## SPI (Modo Maestro)
```c
#include "driver/spi_master.h"

#define SPI_MISO 19
#define SPI_MOSI 23
#define SPI_CLK  18
#define SPI_CS   5

void app_main(void)
{
    // Configura las lineas spi
    spi_bus_config_t buscfg = {
        .miso_io_num = SPI_MISO,
        .mosi_io_num = SPI_MOSI,
        .sclk_io_num = SPI_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1
    };

    // Inicializa bus SPI2 con DMA automático
    spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO);
}


```

## ADC (Conversor Analógico-Digital)
```c
#include "driver/adc.h"

#define SENSOR_PIN ADC1_CHANNEL_6  // GPIO34

void app_main(void)
{
    adc1_config_width(ADC_WIDTH_BIT_12);             // Resolucion 12 bits (0–4095)
    adc1_config_channel_atten(SENSOR_PIN, ADC_ATTEN_DB_11); // Rango de voltaje 0–3.3 V
    int valor = adc1_get_raw(SENSOR_PIN);            // Lectura cruda del ADC
}


```

## DAC (Conversor Digital-Analógico)
```c
#include "driver/dac.h"

#define DAC_OUT DAC_CHANNEL_1  // GPIO25

void app_main(void)
{
    dac_output_enable(DAC_OUT);                      // Activa salida DAC
    dac_output_voltage(DAC_OUT, 180);                // Salida ≈ 180/255 * 3.3 V
}


```

## PWM (LEDC)
```c
#include "driver/ledc.h"

#define PWM_GPIO GPIO_NUM_4

void app_main(void)
{
    // Configura un temporizador para la señal PWM
    ledc_timer_config_t ledc_timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_10_BIT,   // 10 bits: 0–1023
        .freq_hz = 5000,                        // Frecuencia 5 kHz
        .clk_cfg = LEDC_AUTO_CLK
    };
    ledc_timer_config(&ledc_timer);

    // Configura el canal PWM asociado al GPIO
    ledc_channel_config_t ledc_channel = {
        .gpio_num = PWM_GPIO,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 512,                            // 50% de ciclo útil
        .hpoint = 0
    };
    ledc_channel_config(&ledc_channel);

    // Barrido de brillo de LED
    while (1) {
        for (int duty = 0; duty < 1023; duty += 10) {
            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
            vTaskDelay(pdMS_TO_TICKS(10));
        }
    }
}


```

## Wi-Fi Modo Estacion (Cliente)
```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"

void app_main(void)
{
    nvs_flash_init();                              // Inicializa NVS (requerido)
    esp_netif_init();                              // Inicializa capa de red
    esp_event_loop_create_default();               // Crea loop de eventos
    esp_netif_create_default_wifi_sta();           // Configura Wi-Fi STA

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);
    esp_wifi_set_mode(WIFI_MODE_STA);              // Modo cliente

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = "MiWiFi",
            .password = "12345678",
        },
    };
    esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
    esp_wifi_start();                              // Inicia conexión Wi-Fi
}


```

## ESP-NOW
```c
#include "esp_now.h"
#include "esp_wifi.h"

void onDataSent(const uint8_t *mac_addr, esp_now_send_status_t status)
{
    // Callback al enviar datos
}

void app_main(void)
{
    esp_now_init();                                // Inicializa protocolo ESP-NOW
    esp_now_register_send_cb(onDataSent);          // Registra callback
    esp_now_add_peer(&peerInfo);                   // Añade receptor
    esp_now_send(peer_addr, data, len);            // Envía mensaje
}


```

## NVS (Almacenamiento No Volatil)
```c
#include "nvs_flash.h"
#include "nvs.h"

void app_main(void)
{
    nvs_handle_t handle;
    nvs_flash_init();                              // Inicializa flash
    nvs_open("storage", NVS_READWRITE, &handle);   // Abre espacio "storage"
    nvs_set_i32(handle, "valor", 123);             // Guarda entero
    nvs_commit(handle);                            // Confirma cambios
    nvs_close(handle);
}


```

## Timers por Hardware
```c
#include "driver/timer.h"

#define TIMER_DIVIDER 80  // 1 µs por tick (80 MHz / 80)

void app_main(void)
{
    timer_config_t config = {
        .divider = TIMER_DIVIDER,
        .counter_dir = TIMER_COUNT_UP,
        .counter_en = TIMER_PAUSE,
        .alarm_en = TIMER_ALARM_EN,
    };
    timer_init(TIMER_GROUP_0, TIMER_0, &config);   // Inicializa timer 0 del grupo 0
    timer_start(TIMER_GROUP_0, TIMER_0);           // Comienza conteo
}


```

## OTA / HTTPS
```c
#include "esp_https_ota.h"

void app_main(void)
{
    esp_http_client_config_t ota_client = {
        .url = "https://servidor.com/firmware.bin",
    };
    esp_https_ota(&ota_client);                    // Descarga e instala nuevo firmware
}


```

## Logs y Debug
```c
#include "esp_log.h"

static const char *TAG = "APP";

void app_main(void)
{
    ESP_LOGI(TAG, "Iniciando sistema...");         // Log de información
    ESP_LOGW(TAG, "Advertencia detectada");        // Log de advertencia
    ESP_LOGE(TAG, "Error detectado");              // Log de error
}


```

## Tips

Usa ccache para acelerar compilaciones posteriores.

Ejecuta nvs_flash_init() antes de usar Wi-Fi o Bluetooth.

Prefiere ESP_LOGx() sobre printf() para depuración organizada.

Sustituye delay() por vTaskDelay(pdMS_TO_TICKS(ms)).

Tareas intensivas → prioridad baja; tareas rápidas → prioridad alta.

Usa colas (xQueueSend(), xQueueReceive()) para comunicar tareas.

Usa GPIO_NUM_X en lugar de números crudos para portabilidad entre chips.

Para ver pines disponibles: revisa el pinout del modelo exacto de ESP32.

## Deep Sleep (modo bajo consumo)
```c
#include "esp_sleep.h"

#define WAKE_BUTTON GPIO_NUM_0

void app_main(void)
{
    gpio_set_direction(WAKE_BUTTON, GPIO_MODE_INPUT);
    esp_sleep_enable_ext0_wakeup(WAKE_BUTTON, 0);  // Despierta con nivel bajo
    printf("Entrando en Deep Sleep...\n");
    esp_deep_sleep_start();                        // Entra en modo profundo
}

```

## RTC (Real-Time Clock y memoria persistente)
```c
#include "esp_sleep.h"
#include "esp_log.h"

RTC_DATA_ATTR int contador = 0;  // Se mantiene tras deep sleep

void app_main(void)
{
    contador++;
    ESP_LOGI("RTC", "Reinicios desde deep sleep: %d", contador);
    vTaskDelay(pdMS_TO_TICKS(2000));
    esp_deep_sleep(5 * 1000000); // Duerme 5 segundos
}

```

## MQTT (cliente conectado a broker)
```c
#include "esp_event.h"
#include "esp_log.h"
#include "mqtt_client.h"

static const char *TAG = "MQTT";

void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    switch (event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "Conectado al broker");
            esp_mqtt_client_publish(event->client, "/esp32/status", "online", 0, 1, 0);
            break;
        default:
            break;
    }
}

void app_main(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtt://test.mosquitto.org",  // Broker público
    };

    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}

```

## Wi-Fi Access Point
```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"

void app_main(void)
{
    nvs_flash_init();
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_ap();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);
    esp_wifi_set_mode(WIFI_MODE_AP);

    wifi_config_t ap_config = {
        .ap = {
            .ssid = "ESP32_AP",
            .password = "12345678",
            .ssid_len = strlen("ESP32_AP"),
            .max_connection = 4,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK
        },
    };
    esp_wifi_set_config(WIFI_IF_AP, &ap_config);
    esp_wifi_start();
}

```

## Servo (usando PWM/LEDC)
```c
#include "driver/ledc.h"

#define SERVO_GPIO GPIO_NUM_18

void app_main(void)
{
    // Servos funcionan con pulsos de 50 Hz (20 ms)
    ledc_timer_config_t timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_16_BIT,
        .freq_hz = 50,
        .clk_cfg = LEDC_AUTO_CLK
    };
    ledc_timer_config(&timer);

    ledc_channel_config_t channel = {
        .gpio_num = SERVO_GPIO,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 0,
        .hpoint = 0
    };
    ledc_channel_config(&channel);

    // Mueve el servo entre 0° y 180°
    while (1) {
        for (int angle = 0; angle <= 180; angle += 10) {
            uint32_t duty = (angle * (3276 - 1638) / 180) + 1638; // Mapea a 1–2 ms
            ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
            vTaskDelay(pdMS_TO_TICKS(500));
        }
    }
}

```
## Botones (con debounce por software)
```c
#include "driver/gpio.h"

#define BUTTON_GPIO GPIO_NUM_0
#define LED_GPIO    GPIO_NUM_2

void app_main(void)
{
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON_GPIO, GPIO_PULLUP_ONLY);  // Activa resistencia pull-up
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        int estado = gpio_get_level(BUTTON_GPIO);       // Lee nivel del pin
        if (estado == 0) {                              // Botón presionado (activo bajo)
            gpio_set_level(LED_GPIO, 1);
        } else {
            gpio_set_level(LED_GPIO, 0);
        }
        vTaskDelay(pdMS_TO_TICKS(50));                  // Pequeño delay anti-rebote
    }
}

```

# MPU 6050 REGISTROS

| Registro           | Dirección | Tamaño | Descripción                                |
|-------------------|-----------|--------|--------------------------------------------|
| **SMPLRT_DIV**     | 0x19      | 1 byte | Frecuencia de muestreo                     |
| **CONFIG**         | 0x1A      | 1 byte | Configuración del filtro digital           |
| **GYRO_CONFIG**    | 0x1B      | 1 byte | Configuración rango giroscopio             |
| **ACCEL_CONFIG**   | 0x1C      | 1 byte | Configuración rango acelerómetro           |
| **ACCEL_CONFIG2**  | 0x1D      | 1 byte | Configuración adicional acelerómetro       |
| **LP_ACCEL_ODR**   | 0x1E      | 1 byte | Low-power accel data rate                   |
| **WOM_THR**        | 0x1F      | 1 byte | Wake-on-motion threshold                   |
| **FIFO_EN**        | 0x23      | 1 byte | Habilitar FIFO                              |
| **I2C_MST_CTRL**   | 0x24      | 1 byte | Control I2C maestro                        |
| **I2C_SLV0_ADDR**  | 0x25      | 1 byte | Dirección esclavo I2C 0                     |
| **I2C_SLV0_REG**   | 0x26      | 1 byte | Registro del esclavo I2C 0                 |
| **I2C_SLV0_CTRL**  | 0x27      | 1 byte | Control de lectura/escritura I2C           |
| **PWR_MGMT_1**     | 0x6B      | 1 byte | Gestión de energía y modo sueño            |
| **PWR_MGMT_2**     | 0x6C      | 1 byte | Gestión de energía por eje y sensores      |
| **WHO_AM_I**       | 0x75      | 1 byte | Identificación del chip (valor 0x68)      |

## Acelerómetro – Datos de los 3 ejes (16 bits cada uno)

| Registro      | Dirección | Uso |
|---------------|-----------|-----|
| ACCEL_XOUT_H  | 0x3B      | MSB eje X |
| ACCEL_XOUT_L  | 0x3C      | LSB eje X |
| ACCEL_YOUT_H  | 0x3D      | MSB eje Y |
| ACCEL_YOUT_L  | 0x3E      | LSB eje Y |
| ACCEL_ZOUT_H  | 0x3F      | MSB eje Z |
| ACCEL_ZOUT_L  | 0x40      | LSB eje Z |

## Giroscopio – Datos de los 3 ejes (16 bits cada uno)

| Registro      | Dirección | Uso |
|---------------|-----------|-----|
| GYRO_XOUT_H   | 0x43      | MSB eje X |
| GYRO_XOUT_L   | 0x44      | LSB eje X |
| GYRO_YOUT_H   | 0x45      | MSB eje Y |
| GYRO_YOUT_L   | 0x46      | LSB eje Y |
| GYRO_ZOUT_H   | 0x47      | MSB eje Z |
| GYRO_ZOUT_L   | 0x48      | LSB eje Z |

## Temperatura

| Registro       | Dirección | Uso |
|----------------|-----------|-----|
| TEMP_OUT_H     | 0x41      | MSB temperatura |
| TEMP_OUT_L     | 0x42      | LSB temperatura |

---

### Tips

- Para leer acelerometro o giroscopio 
```c
int16_t val = (msb << 8) | lsb;
```
## EJEMPLO
```c
#include "driver/i2c.h"
#include "esp_log.h"

#define I2C_MASTER_SCL_IO           GPIO_NUM_22    // Pin SCL
#define I2C_MASTER_SDA_IO           GPIO_NUM_21    // Pin SDA
#define I2C_MASTER_NUM              I2C_NUM_0      // Canal I2C (0 o 1)
#define I2C_MASTER_FREQ_HZ          100000         // Frecuencia 100kHz
#define I2C_MASTER_TX_BUF_DISABLE   0
#define I2C_MASTER_RX_BUF_DISABLE   0
#define MPU6050_ADDR                0x68           // Dirección I2C del MPU6050
#define MPU6050_REG_WHO_AM_I        0x75           // Registro de identidad
#define MPU6050_PWR_MGMT_1          0x6B
#define MPU6050_ACCEL_XOUT_H        0x3B

static const char *TAG = "MPU6050";

/* Inicializa el bus I2C como master */
void i2c_master_init(void)
{
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_pullup_en = GPIO_PULLUP_ENABLE,
        .scl_pullup_en = GPIO_PULLUP_ENABLE,
        .master.clk_speed = I2C_MASTER_FREQ_HZ,
    };
    i2c_param_config(I2C_MASTER_NUM, &conf);
    i2c_driver_install(I2C_MASTER_NUM, conf.mode,
                       I2C_MASTER_RX_BUF_DISABLE,
                       I2C_MASTER_TX_BUF_DISABLE, 0);
}

/* Escribir un byte a un registro del MPU6050 */
esp_err_t mpu6050_write_byte(uint8_t reg, uint8_t data)
{
    return i2c_master_write_to_device(I2C_MASTER_NUM, MPU6050_ADDR,
                                      (uint8_t[]){reg, data}, 2,
                                      pdMS_TO_TICKS(100));
}

/* Leer bytes de un registro */
esp_err_t mpu6050_read_bytes(uint8_t reg, uint8_t *data, size_t len)
{
    return i2c_master_write_read_device(I2C_MASTER_NUM, MPU6050_ADDR,
                                        &reg, 1, data, len,
                                        pdMS_TO_TICKS(100));
}

/* Inicializa el sensor */
void mpu6050_init(void)
{
    uint8_t who_am_i = 0;
    mpu6050_write_byte(MPU6050_PWR_MGMT_1, 0x00); // Despierta el sensor
    mpu6050_read_bytes(MPU6050_REG_WHO_AM_I, &who_am_i, 1);
    ESP_LOGI(TAG, "MPU6050 WHO_AM_I = 0x%02X", who_am_i);
}

/* Leer datos de aceleración */
void mpu6050_read_accel(void)
{
    uint8_t data[6];
    mpu6050_read_bytes(MPU6050_ACCEL_XOUT_H, data, 6);

    int16_t ax = (data[0] << 8) | data[1];
    int16_t ay = (data[2] << 8) | data[3];
    int16_t az = (data[4] << 8) | data[5];

    ESP_LOGI(TAG, "Accel X:%d  Y:%d  Z:%d", ax, ay, az);
}

/* app_main: Inicializa I2C y lee sensor */
void app_main(void)
{
    i2c_master_init();
    mpu6050_init();

    while (1) {
        mpu6050_read_accel();
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

```
# PANTALLA I2C 16X2
```c
#include "esp_lcd_io_i2c.h"
#include "esp_lcd_panel_io.h"
#include "driver/i2c_master.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

// Pines y configuración
#define I2C_SDA_PIN 11
#define I2C_SCL_PIN 12
#define LCD_ADDR    0x27      // o 0x3F según el módulo
#define I2C_FREQ_HZ 50000     // Frecuencia I2C

static i2c_master_bus_handle_t i2c_bus = NULL;
static esp_lcd_panel_io_handle_t io_handle = NULL;

// Función base para enviar datos (modo 4 bits)
void lcd_write(uint8_t data, uint8_t rs)
{
    uint8_t high = data & 0xF0;
    uint8_t low  = (data << 4) & 0xF0;

    uint8_t seq[4] = {
        high | (rs ? 0x01 : 0x00) | 0x0C,
        high | (rs ? 0x01 : 0x00) | 0x08,
        low  | (rs ? 0x01 : 0x00) | 0x0C,
        low  | (rs ? 0x01 : 0x00) | 0x08
    };

    esp_lcd_panel_io_tx_param(io_handle, 0, seq, sizeof(seq));
    vTaskDelay(pdMS_TO_TICKS(2));
}

void lcd_cmd(uint8_t cmd)  { lcd_write(cmd, 0); }
void lcd_data(uint8_t data){ lcd_write(data, 1); }

// Inicialización estándar del LCD
void lcd_init()
{
    vTaskDelay(pdMS_TO_TICKS(100));
    lcd_write(0x30, 0); vTaskDelay(pdMS_TO_TICKS(10));
    lcd_write(0x30, 0); vTaskDelay(pdMS_TO_TICKS(10));
    lcd_write(0x20, 0); vTaskDelay(pdMS_TO_TICKS(10));

    lcd_cmd(0x28); // 4 bits, 2 líneas
    lcd_cmd(0x0C); // Display ON
    lcd_cmd(0x06); // Incrementar cursor
    lcd_cmd(0x01); // Limpiar
    vTaskDelay(pdMS_TO_TICKS(10));
}

// Posicionar cursor
void lcd_set_cursor(uint8_t col, uint8_t row)
{
    static const uint8_t offsets[] = {0x00, 0x40};
    lcd_cmd(0x80 | (col + offsets[row]));
}

// Imprimir texto
void lcd_print(const char *msg)
{
    while (*msg)
        lcd_data(*msg++);
}

// Configuración I2C y LCD
void lcd_begin(void)
{
    i2c_master_bus_config_t bus_cfg = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_NUM_0,
        .sda_io_num = I2C_SDA_PIN,
        .scl_io_num = I2C_SCL_PIN,
        .flags.enable_internal_pullup = true,
    };
    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_cfg, &i2c_bus));

    esp_lcd_panel_io_i2c_config_t io_cfg = {
        .dev_addr = LCD_ADDR,
        .scl_speed_hz = I2C_FREQ_HZ,
        .control_phase_bytes = 1,
        .lcd_cmd_bits = 8,
        .lcd_param_bits = 8,
    };
    ESP_ERROR_CHECK(esp_lcd_new_panel_io_i2c(i2c_bus, &io_cfg, &io_handle));

    lcd_init();
}

// Programa principal
void app_main(void)
{
    lcd_begin();

    lcd_set_cursor(0, 0);
    lcd_print("ESP-IDF LCD OK!");
    lcd_set_cursor(0, 1);
    lcd_print("Nueva API lista");

    while (1)
        vTaskDelay(pdMS_TO_TICKS(1000));
}

```
### Comandos de LCD
| Acción                | Código            | Descripción                       |
| --------------------- | ----------------- | --------------------------------- |
| `lcd_cmd(0x01)`       | Limpiar pantalla  | Borra todo el contenido del LCD   |
| `lcd_cmd(0x0C)`       | Display ON        | Muestra texto sin cursor          |
| `lcd_cmd(0x0E)`       | Cursor ON         | Muestra cursor parpadeante        |
| `lcd_set_cursor(x,y)` | Posicionar cursor | x = columna, y = fila (0 o 1)     |
| `lcd_print("texto")`  | Escribir texto    | Envía texto carácter por carácter |


# PARTE IMPORTANTE FreeRTOS: Multitarea, Sensores y Deep Sleep

## Conceptos clave de FreeRTOS
- Tarea (Task): Unidad de ejecución concurrente.

- Colas (Queue): Comunicación segura entre tareas.

- Delay (vTaskDelay): Pausa de tarea sin bloquear el sistema.

- Prioridades: Determinan qué tarea se ejecuta primero.

## Crear tareas para leer múltiples sensores
```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define SENSOR1_GPIO GPIO_NUM_34
#define SENSOR2_GPIO GPIO_NUM_35
#define LED_GPIO     GPIO_NUM_2

static const char *TAG = "MULTISENSOR";

/* tarea para leer sensor 1 */
void sensor1_task(void *pvParameter)
{
    while(1) {
        int val = gpio_get_level(SENSOR1_GPIO);  // Simulación
        ESP_LOGI(TAG, "Sensor1: %d", val);
        vTaskDelay(pdMS_TO_TICKS(500));          // Lee cada 0.5s
    }
}

/* Tarea para leer sensor 2 */
void sensor2_task(void *pvParameter)
{
    while(1) {
        int val = gpio_get_level(SENSOR2_GPIO);
        ESP_LOGI(TAG, "Sensor2: %d", val);
        vTaskDelay(pdMS_TO_TICKS(1000));         // Lee cada 1s
    }
}

/* Tarea para LED de estado */
void led_task(void *pvParameter)
{
    while(1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(200));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

/* app_main */
void app_main(void)
{
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    xTaskCreate(sensor1_task, "sensor1", 2048, NULL, 5, NULL);
    xTaskCreate(sensor2_task, "sensor2", 2048, NULL, 5, NULL);
    xTaskCreate(led_task,     "led",     1024, NULL, 3, NULL);//MAS PRIORIDAD, PRIORIDAD 3
}

```

### Explicacion de codigo

- Cada tarea corre de forma independiente.
- Puedes variar vTaskDelay() según la frecuencia de muestreo deseada.
- Las tareas no se bloquean entre sí; FreeRTOS gestiona la CPU automáticamente.

## Uso de colas para compartir datos entre tareas
```c
#include "freertos/queue.h"

QueueHandle_t sensor_queue;

void sensor_task(void *pvParam)
{
    int val;
    while(1) {
        val = gpio_get_level(SENSOR1_GPIO);
        xQueueSend(sensor_queue, &val, pdMS_TO_TICKS(10)); // Envia valor
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void process_task(void *pvParam)
{
    int sensor_val;
    while(1) {
        if(xQueueReceive(sensor_queue, &sensor_val, portMAX_DELAY)) {
            ESP_LOGI(TAG, "Procesando Sensor1: %d", sensor_val);
        }
    }
}

void app_main(void)
{
    sensor_queue = xQueueCreate(10, sizeof(int));
    xTaskCreate(sensor_task, "sensor", 2048, NULL, 5, NULL);
    xTaskCreate(process_task, "process", 2048, NULL, 5, NULL);
}

```


## Deep Sleep con “último mensaje”
```c
#include "esp_sleep.h"
#include "esp_log.h"
#include "driver/gpio.h"

#define WAKE_BUTTON GPIO_NUM_0
#define LED_GPIO    GPIO_NUM_2

void send_last_message(void)
{
    ESP_LOGI("DEEPSLEEP", "Enviando último mensaje antes de dormir...");
    gpio_set_level(LED_GPIO, 1);          // Indica visualmente
    vTaskDelay(pdMS_TO_TICKS(500));       // Pequeña espera para transmisión
    gpio_set_level(LED_GPIO, 0);
}

void app_main(void)
{
    gpio_set_direction(WAKE_BUTTON, GPIO_MODE_INPUT);
    gpio_set_pull_mode(WAKE_BUTTON, GPIO_PULLUP_ONLY);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    send_last_message();  // Ejecuta “mensaje de adiós”

    esp_sleep_enable_ext0_wakeup(WAKE_BUTTON, 0); // Despierta con botón
    ESP_LOGI("DEEPSLEEP", "Entrando en Deep Sleep...");
    esp_deep_sleep_start();
}

```
### Explicacion de codigo
- Puedes ejecutar funciones finales antes de dormir (send_last_message).

- Después llamas a esp_deep_sleep_start(), que detiene CPU y periféricos.

- Al despertar, el chip reinicia desde app_main.

