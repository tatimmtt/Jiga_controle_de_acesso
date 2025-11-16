# main.c

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file main.c
 * @brief Programa principal - Controle de Motor BeagleBone Black
 * 
 * Este programa integra todos os módulos para criar um sistema de controle
 * de motor de passo com interface web.
 * 
 * Funcionalidades:
 * - Controle de motor via GPIOs (STEP e DIR)
 * - Simulação de temperatura com proteção contra superaquecimento
 * - Interface web para monitoramento e controle
 * - API REST para integração com outros sistemas
 * 
 * Uso:
 *   ./motor_control [porta]
 * 
 * Exemplo:
 *   ./motor_control 8081
 * 
 * @author Seu Nome
 * @date 2025
 */

#define _XOPEN_SOURCE 500

#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

#include "gpio.h"
#include "motor.h"
#include "http_server.h"

/* ---------- Variáveis globais ---------- */

static motor_state_t motor_state;
static http_server_t server;
static int fd_gpio_step = -1;
static int fd_gpio_dir = -1;

/* ---------- Funções de limpeza e sinais ---------- */

/**
 * @brief Limpa recursos e encerra o programa
 * 
 * Fecha GPIOs, encerra servidor e libera recursos.
 * 
 * @param code Código de saída
 */
void cleanup_and_exit(int code) {
    printf("\n=== Encerrando programa ===\n");

    // Solicita parada do motor se estiver rodando
    if (motor_is_running(&motor_state)) {
        printf("→ Parando motor...\n");
        motor_stop(&motor_state);
        sleep(1);  // Aguarda motor parar
    }

    // Fecha servidor HTTP
    printf("→ Fechando servidor HTTP...\n");
    http_server_close(&server);

    // Fecha GPIOs
    if (fd_gpio_step >= 0) {
        printf("→ Fechando GPIO STEP...\n");
        close(fd_gpio_step);
    }
    if (fd_gpio_dir >= 0) {
        printf("→ Fechando GPIO DIR...\n");
        close(fd_gpio_dir);
    }

    // Limpa recursos do motor
    motor_cleanup(&motor_state);

    printf("✓ Limpeza concluída. Até logo!\n");
    exit(code);
}

/**
 * @brief Handler para SIGINT (Ctrl+C)
 * 
 * Captura Ctrl+C e encerra o programa de forma limpa.
 */
void sigint_handler(int signo) {
    (void)signo;
    printf("\n\n⚠ SIGINT recebido (Ctrl+C). Finalizando...\n");
    cleanup_and_exit(0);
}

/* ---------- Função principal ---------- */

/**
 * @brief Ponto de entrada do programa
 * 
 * Fluxo de execução:
 * 1. Parseia argumentos de linha de comando
 * 2. Configura handlers de sinais
 * 3. Inicializa GPIOs
 * 4. Inicializa sistema do motor
 * 5. Inicializa servidor HTTP
 * 6. Entra no loop de processamento de requisições
 */
int main(int argc, char *argv[]) {
    int port = DEFAULT_PORT;

    /* ----- Banner de inicialização ----- */
    printf("\n");
    printf("╔═══════════════════════════════════════════════════╗\n");
    printf("║   CONTROLE DE MOTOR - BEAGLEBONE BLACK            ║\n");
    printf("║   Sistema de Controle com Interface Web          ║\n");
    printf("╚═══════════════════════════════════════════════════╝\n");
    printf("\n");

    /* ----- Parse de argumentos ----- */
    if (argc >= 2) {
        char *end;
        long p = strtol(argv[1], &end, 10);

        if (end == argv[1] || *end != '\0' || p < 1 || p > 65535) {
            fprintf(stderr, "✗ Erro: Porta inválida '%s'\n", argv[1]);
            fprintf(stderr, "  Uso: %s [porta]\n", argv[0]);
            fprintf(stderr, "  Exemplo: %s 8081\n", argv[0]);
            return 1;
        }

        port = (int)p;
    }

    /* ----- Configuração de sinais ----- */
    printf("[1/4] Configurando handlers de sinais...\n");
    if (signal(SIGINT, sigint_handler) == SIG_ERR) {
        perror("signal");
        return 1;
    }
    printf("      ✓ Handler SIGINT configurado\n\n");

    /* ----- Inicialização dos GPIOs ----- */
    printf("[2/4] Inicializando GPIOs...\n");
    if (inicializar_gpios(&fd_gpio_step, &fd_gpio_dir) != 0) {
        fprintf(stderr, "⚠ Aviso: Falha ao inicializar GPIOs\n");
        fprintf(stderr, "  O programa continuará em MODO SIMULADO\n");
        fprintf(stderr, "  (útil para desenvolvimento sem hardware)\n\n");
    } else {
        printf("      ✓ GPIO STEP: %s\n", GPIO_60);
        printf("      ✓ GPIO DIR: %s\n\n", GPIO_112);
    }

    /* ----- Inicialização do sistema do motor ----- */
    printf("[3/4] Inicializando sistema do motor...\n");
    if (motor_init(&motor_state, fd_gpio_step, fd_gpio_dir) != 0) {
        fprintf(stderr, "✗ Erro ao inicializar motor\n");
        cleanup_and_exit(1);
    }
    printf("      ✓ Estado inicial: Temperatura 25°C, Motor parado\n\n");

    /* ----- Inicialização do servidor HTTP ----- */
    printf("[4/4] Inicializando servidor HTTP...\n");
    if (http_server_init(&server, port, &motor_state) != 0) {
        fprintf(stderr, "✗ Erro ao inicializar servidor HTTP\n");
        cleanup_and_exit(1);
    }
    printf("\n");

    /* ----- Instruções de uso ----- */
    printf("╔═══════════════════════════════════════════════════╗\n");
    printf("║  SERVIDOR PRONTO!                                 ║\n");
    printf("╚═══════════════════════════════════════════════════╝\n");
    printf("\n");
    printf("📡 Acesse a interface web em:\n");
    printf("   → http://localhost:%d\n", port);
    printf("   → http://IP_DA_BEAGLEBONE:%d (de outro dispositivo)\n", port);
    printf("\n");
    printf("🔌 Endpoints da API:\n");
    printf("   GET  /api/status       - Status do sistema (JSON)\n");
    printf("   POST /api/temperature  - Define temperatura\n");
    printf("   POST /api/rotate       - Inicia rotação 90°\n");
    printf("   POST /api/stop         - Para motor imediatamente\n");
    printf("\n");
    printf("💡 Dicas:\n");
    printf("   • Temperatura > 30°C ativa proteção de superaquecimento\n");
    printf("   • Motor executa: 90° horário → pausa 2s → 90° anti-horário\n");
    printf("   • Pressione Ctrl+C para encerrar\n");
    printf("\n");
    printf("════════════════════════════════════════════════════\n\n");

    /* ----- Loop principal do servidor ----- */
    http_server_run(&server);

    /* ----- Encerramento (nunca deve chegar aqui em uso normal) ----- */
    cleanup_and_exit(0);
    return 0;
}

```
{% endtab %}
{% endtabs %}
