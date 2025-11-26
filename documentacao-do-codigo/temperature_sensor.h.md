# temperature\_sensor.h

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file temperature_sensor.h
 * @brief Driver do sensor GY-906 (MLX90614) via I2C
 * @version 2.0 - Corrigido para I2C1 com validação de dados
 */

#ifndef TEMPERATURE_SENSOR_H
#define TEMPERATURE_SENSOR_H

#include <pthread.h>

/* ========== CONFIGURAÇÕES ========== */

// Dispositivo I2C1 (P9_17/P9_18)
#define I2C_DEVICE "/dev/i2c-1"

// Endereço I2C do MLX90614
#define MLX90614_ADDR 0x5A

// Registradores do MLX90614
#define MLX90614_TOBJ1 0x07  // Temperatura do objeto
#define MLX90614_TAMB  0x06  // Temperatura ambiente

// Intervalo de leitura (5 minutos = 300 segundos)
#define LEITURA_INTERVALO_SEC 10

// Range válido de temperatura do MLX90614
#define TEMP_MIN_CELSIUS -70.0   // Temperatura mínima suportada
#define TEMP_MAX_CELSIUS 382.0   // Temperatura máxima suportada
#define RAW_MIN 10000            // Valor raw mínimo esperado
#define RAW_MAX 33000            // Valor raw máximo esperado

/* ========== ESTRUTURAS ========== */

/**
 * @brief Estrutura do sensor de temperatura
 */
typedef struct {
    int fd_i2c;                    // File descriptor do I2C
    int temperatura_atual;         // Temperatura atual em °C
    int sensor_ativo;             // 1 = sensor funcionando, 0 = simulado
    pthread_t thread;             // Thread de leitura
    pthread_mutex_t lock;         // Mutex para acesso thread-safe
    int thread_rodando;           // Flag de controle da thread
    void (*callback)(int);        // Callback para novas leituras
} temp_sensor_t;

/* ========== FUNÇÕES PÚBLICAS ========== */

/**
 * @brief Inicializa o sensor de temperatura
 * @param sensor Ponteiro para estrutura do sensor
 * @param callback Função callback para notificar novas leituras (pode ser NULL)
 * @return 0 em sucesso, -1 em erro (modo simulado retorna 0)
 */
int temp_sensor_init(temp_sensor_t *sensor, void (*callback)(int));

/**
 * @brief Inicia thread de leitura periódica
 * @param sensor Ponteiro para estrutura do sensor
 * @return 0 em sucesso, -1 em erro
 */
int temp_sensor_start(temp_sensor_t *sensor);

/**
 * @brief Lê temperatura do sensor (bloqueante)
 * @param sensor Ponteiro para estrutura do sensor
 * @return Temperatura em °C, ou -999 em erro
 */
int temp_sensor_read(temp_sensor_t *sensor);

/**
 * @brief Obtém última temperatura lida (não bloqueante)
 * @param sensor Ponteiro para estrutura do sensor
 * @return Temperatura em °C
 */
int temp_sensor_get_temperature(temp_sensor_t *sensor);

/**
 * @brief Verifica se sensor está ativo (real) ou simulado
 * @param sensor Ponteiro para estrutura do sensor
 * @return 1 = sensor real ativo, 0 = modo simulado
 */
int temp_sensor_is_active(temp_sensor_t *sensor);

/**
 * @brief Finaliza sensor e libera recursos
 * @param sensor Ponteiro para estrutura do sensor
 */
void temp_sensor_cleanup(temp_sensor_t *sensor);

#endif // TEMPERATURE_SENSOR_H

```
{% endtab %}
{% endtabs %}

