# motor.c

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file motor.c
 * @brief Implementação do controle do motor de passo
 */

#include "motor.h"
#include "gpio.h"
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

/* Protótipo da função executada na thread do motor */
static void *motor_worker(void *arg);

/* ---------- Funções públicas ---------- */

int motor_init(motor_state_t *state, int fd_step, int fd_dir) {
    pthread_mutex_init(&state->lock, NULL);
    state->current_temperature = 25;  // Temperatura inicial
    state->overheat = 0;
    state->stop_requested = 0;
    state->motor_running = 0;
    state->fd_step = fd_step;
    state->fd_dir = fd_dir;

    printf("✓ Sistema do motor inicializado\n");
    return 0;
}

void motor_set_temperature(motor_state_t *state, int temp) {
    pthread_mutex_lock(&state->lock);

    state->current_temperature = temp;

    // Verifica superaquecimento (>30°C)
    if (state->current_temperature > 30) {
        state->overheat = 1;
        state->stop_requested = 1;  // Solicita parada imediata
        printf("⚠ ALERTA: Temperatura %d°C > 30°C - SUPERAQUECIMENTO DETECTADO!\n", 
               state->current_temperature);
        printf("  → Motor será parado imediatamente\n");
    } else {
        state->overheat = 0;
    }

    pthread_mutex_unlock(&state->lock);
}

int motor_get_temperature(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);
    int temp = state->current_temperature;
    pthread_mutex_unlock(&state->lock);
    return temp;
}

int motor_is_overheated(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);
    int ov = state->overheat;
    pthread_mutex_unlock(&state->lock);
    return ov;
}

int motor_is_running(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);
    int running = state->motor_running;
    pthread_mutex_unlock(&state->lock);
    return running;
}

int motor_start_rotation(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);

    // Verifica se há superaquecimento
    if (state->overheat) {
        pthread_mutex_unlock(&state->lock);
        printf("✗ Movimento bloqueado: sistema superaquecido\n");
        return -1;
    }

    // Verifica se motor já está rodando
    if (state->motor_running) {
        pthread_mutex_unlock(&state->lock);
        printf("✗ Motor já está em movimento\n");
        return -1;
    }

    pthread_mutex_unlock(&state->lock);

    // Cria thread para executar o movimento
    if (pthread_create(&state->motor_thread, NULL, motor_worker, state) != 0) {
        perror("pthread_create");
        return -1;
    }

    // Detach para não precisar fazer join
    pthread_detach(state->motor_thread);

    printf("✓ Movimento iniciado em thread separada\n");
    return 0;
}

void motor_stop(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);

    if (state->motor_running) {
        state->stop_requested = 1;
        printf("✓ Parada do motor solicitada\n");
    } else {
        printf("ℹ Motor já está parado\n");
    }

    pthread_mutex_unlock(&state->lock);
}

void motor_cleanup(motor_state_t *state) {
    pthread_mutex_destroy(&state->lock);
}

/* ---------- Thread do motor ---------- */

/**
 * @brief Função executada pela thread do motor
 * 
 * Executa o movimento completo:
 * 1. Gira 90° no sentido horário (DIR=1)
 * 2. Aguarda 2 segundos
 * 3. Retorna 90° no sentido anti-horário (DIR=0)
 * 
 * A cada passo, verifica se há pedido de parada ou superaquecimento.
 */
static void *motor_worker(void *arg) {
    motor_state_t *state = (motor_state_t *)arg;

    // Marca motor como rodando
    pthread_mutex_lock(&state->lock);
    state->motor_running = 1;
    state->stop_requested = 0;
    pthread_mutex_unlock(&state->lock);

    printf("→ MOTOR: Iniciando movimento de 90° ida e volta\n");

    /* ----- FASE 1: Rotação horária (90°) ----- */
    printf("  Fase 1/3: Girando 90° (horário)...\n");
    gpio_write(state->fd_dir, "1");  // Define direção horária

    for (int i = 0; i < QUARTO_VOLTA; i++) {
        // Verifica condições de parada
        pthread_mutex_lock(&state->lock);
        int local_stop = state->stop_requested;
        int local_over = state->overheat;
        pthread_mutex_unlock(&state->lock);

        if (local_stop || local_over) {
            printf("✗ MOTOR: Interrompido na fase 1 (stop=%d, overheat=%d)\n", 
                   local_stop, local_over);
            goto cleanup;
        }

        // Gera pulso STEP
        gpio_write(state->fd_step, "1");
        usleep(1000);  // 1ms alto
        gpio_write(state->fd_step, "0");
        usleep(1000);  // 1ms baixo
    }

    /* ----- FASE 2: Pausa ----- */
    printf("  Fase 2/3: Aguardando 2 segundos...\n");
    for (int i = 0; i < 20; i++) {  // 20 * 100ms = 2s
        pthread_mutex_lock(&state->lock);
        int local_stop = state->stop_requested;
        int local_over = state->overheat;
        pthread_mutex_unlock(&state->lock);

        if (local_stop || local_over) {
            printf("✗ MOTOR: Interrompido na fase 2 (stop=%d, overheat=%d)\n", 
                   local_stop, local_over);
            goto cleanup;
        }

        usleep(100000);  // 100ms
    }

    /* ----- FASE 3: Retorno anti-horário (90°) ----- */
    printf("  Fase 3/3: Retornando 90° (anti-horário)...\n");
    gpio_write(state->fd_dir, "0");  // Define direção anti-horária

    for (int i = 0; i < QUARTO_VOLTA; i++) {
        pthread_mutex_lock(&state->lock);
        int local_stop = state->stop_requested;
        int local_over = state->overheat;
        pthread_mutex_unlock(&state->lock);

        if (local_stop || local_over) {
            printf("✗ MOTOR: Interrompido na fase 3 (stop=%d, overheat=%d)\n", 
                   local_stop, local_over);
            goto cleanup;
        }

        gpio_write(state->fd_step, "1");
        usleep(1000);
        gpio_write(state->fd_step, "0");
        usleep(1000);
    }

    printf("✓ MOTOR: Movimento completo executado com sucesso!\n");

cleanup:
    // Marca motor como parado
    pthread_mutex_lock(&state->lock);
    state->motor_running = 0;
    state->stop_requested = 0;
    pthread_mutex_unlock(&state->lock);

    return NULL;
}

```
{% endtab %}
{% endtabs %}
