# motor.h

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file motor.h
 * @brief Interface para controle do motor de passo
 * 
 * Este módulo gerencia o movimento do motor, temperatura e proteção
 * contra superaquecimento. O motor executa em thread separada.
 */

#ifndef MOTOR_H
#define MOTOR_H

#include <pthread.h>

/* Constantes de movimento */
#define VOLTA_COMPLETA 400   // Passos para uma volta completa (depende do driver)
#define QUARTO_VOLTA 100     // Passos para 90 graus (1/4 de volta)

/**
 * @brief Estado do sistema de controle do motor
 */
typedef struct {
    pthread_mutex_t lock;        // Mutex para acesso thread-safe
    int current_temperature;     // Temperatura atual em °C (simulada)
    int overheat;               // 1 = superaquecido (>30°C), 0 = normal
    int stop_requested;         // 1 = parada solicitada, 0 = normal
    int motor_running;          // 1 = motor em movimento, 0 = parado
    pthread_t motor_thread;     // Thread do motor
    int fd_step;               // File descriptor do pino STEP
    int fd_dir;                // File descriptor do pino DIR
} motor_state_t;

/**
 * @brief Inicializa o estado do motor
 * 
 * @param state Ponteiro para a estrutura de estado
 * @param fd_step File descriptor do GPIO STEP
 * @param fd_dir File descriptor do GPIO DIR
 * @return 0 em sucesso, -1 em erro
 */
int motor_init(motor_state_t *state, int fd_step, int fd_dir);

/**
 * @brief Define a temperatura do sistema
 * 
 * Atualiza a temperatura e verifica se há superaquecimento (>30°C).
 * Em caso de superaquecimento, solicita parada imediata do motor.
 * 
 * @param state Ponteiro para a estrutura de estado
 * @param temp Temperatura em °C
 */
void motor_set_temperature(motor_state_t *state, int temp);

/**
 * @brief Obtém a temperatura atual
 * 
 * @param state Ponteiro para a estrutura de estado
 * @return Temperatura em °C
 */
int motor_get_temperature(motor_state_t *state);

/**
 * @brief Verifica se há superaquecimento
 * 
 * @param state Ponteiro para a estrutura de estado
 * @return 1 se superaquecido, 0 caso contrário
 */
int motor_is_overheated(motor_state_t *state);

/**
 * @brief Verifica se o motor está em movimento
 * 
 * @param state Ponteiro para a estrutura de estado
 * @return 1 se rodando, 0 se parado
 */
int motor_is_running(motor_state_t *state);

/**
 * @brief Inicia movimento do motor (90 graus ida e volta)
 * 
 * Cria uma thread que executa:
 * 1. Rotação de 90° no sentido horário
 * 2. Pausa de 2 segundos
 * 3. Retorno de 90° no sentido anti-horário
 * 
 * O movimento pode ser interrompido por superaquecimento ou comando stop.
 * 
 * @param state Ponteiro para a estrutura de estado
 * @return 0 em sucesso, -1 se motor já estiver rodando ou superaquecido
 */
int motor_start_rotation(motor_state_t *state);

/**
 * @brief Solicita parada imediata do motor
 * 
 * Define a flag stop_requested que será verificada pela thread do motor.
 * 
 * @param state Ponteiro para a estrutura de estado
 */
void motor_stop(motor_state_t *state);

/**
 * @brief Limpa recursos do motor
 * 
 * @param state Ponteiro para a estrutura de estado
 */
void motor_cleanup(motor_state_t *state);

#endif /* MOTOR_H */

```
{% endtab %}
{% endtabs %}
