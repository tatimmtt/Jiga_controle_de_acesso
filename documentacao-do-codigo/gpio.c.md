# gpio.c

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file gpio.c
 * @brief Implementação das funções de controle GPIO
 */

#define _XOPEN_SOURCE 500

#include "gpio.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

/* ---------- Função auxiliar de escrita robusta ---------- */

ssize_t write_all(int fd, const void *buf, size_t count) {
    size_t written = 0;
    const char *p = buf;

    while (written < count) {
        ssize_t n = write(fd, p + written, count - written);
        if (n < 0) {
            if (errno == EINTR) continue;  // Interrompido por sinal, tenta novamente
            return -1;
        }
        written += n;
    }
    return (ssize_t)written;
}

/* ---------- Funções de GPIO ---------- */

int exportar_gpio(const char *gpio_num) {
    int f = open("/sys/class/gpio/export", O_WRONLY);

    if (f < 0) {
        // Em sistemas sem suporte a sysfs GPIO, opera em modo simulado
        // (útil para desenvolvimento e testes)
        return 0;
    }

    if (write_all(f, gpio_num, strlen(gpio_num)) < 0) {
        // EBUSY significa que o GPIO já foi exportado (ok)
        if (errno != EBUSY) {
            perror("Erro ao escrever em /sys/class/gpio/export");
            close(f);
            return -1;
        }
    }

    close(f);
    usleep(200000);  // Aguarda 200ms para o kernel processar
    return 0;
}

int configurar_gpio_saida(const char *gpio_num) {
    char path[80];
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/direction", gpio_num);

    int f = open(path, O_WRONLY);
    if (f < 0) {
        // Modo simulado se o arquivo não existir
        return 0;
    }

    if (write_all(f, "out", 3) < 0) {
        perror("Erro ao configurar direction");
        close(f);
        return -1;
    }

    close(f);
    return 0;
}

int abrir_gpio_value(const char *gpio_num) {
    char path[80];
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/value", gpio_num);

    int f = open(path, O_WRONLY);
    if (f < 0) {
        // Retorna -1 para indicar modo simulado
        return -1;
    }

    return f;
}

int inicializar_gpios(int *fd_step, int *fd_dir) {
    // Inicializa GPIO 60 (STEP)
    exportar_gpio(GPIO_60);
    configurar_gpio_saida(GPIO_60);
    *fd_step = abrir_gpio_value(GPIO_60);

    // Inicializa GPIO 112 (DIR)
    exportar_gpio(GPIO_112);
    configurar_gpio_saida(GPIO_112);
    *fd_dir = abrir_gpio_value(GPIO_112);

    printf("✓ GPIOs inicializados (STEP=%s, DIR=%s)\n", GPIO_60, GPIO_112);
    if (*fd_step < 0 || *fd_dir < 0) {
        printf("  [MODO SIMULADO - hardware GPIO não disponível]\n");
    }

    return 0;
}

int gpio_write(int fd, const char *val) {
    if (fd < 0) {
        // Modo simulado: apenas retorna sucesso
        return 0;
    }

    // Reposiciona para o início do arquivo
    if (lseek(fd, 0, SEEK_SET) < 0) {
        perror("lseek gpio");
        return -1;
    }

    // Escreve o valor (0 ou 1)
    if (write_all(fd, val, 1) < 0) {
        perror("write gpio");
        return -1;
    }

    fsync(fd);  // Garante que a escrita chegue ao hardware
    return 0;
}

```
{% endtab %}
{% endtabs %}
