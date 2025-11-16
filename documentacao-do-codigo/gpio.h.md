# gpio.h

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file gpio.h
 * @brief Interface para controle de GPIOs via sysfs no BeagleBone Black
 * 
 * Este módulo gerencia a configuração e controle dos pinos GPIO através
 * do sistema de arquivos /sys/class/gpio do Linux.
 */

#ifndef GPIO_H
#define GPIO_H

/* ===== INCLUDES NECESSÁRIOS ===== */
#include <sys/types.h>   // Define ssize_t e size_t
#include <stddef.h>      // Define size_t (alternativa/reforço)

/* Definição dos pinos GPIO utilizados */
#define GPIO_60 "60"    // Pino STEP - controla os pulsos de passo do motor
#define GPIO_112 "112"  // Pino DIR - controla a direção de rotação do motor

/**
 * @brief Exporta um GPIO para o userspace
 * 
 * Escreve o número do GPIO no arquivo /sys/class/gpio/export para
 * torná-lo acessível via sysfs.
 * 
 * @param gpio_num String com o número do GPIO (ex: "60")
 * @return 0 em sucesso, -1 em erro (exceto EBUSY que é ignorado)
 */
int exportar_gpio(const char *gpio_num);

/**
 * @brief Configura um GPIO como saída
 * 
 * Define a direção do GPIO como "out" através do arquivo direction.
 * 
 * @param gpio_num String com o número do GPIO
 * @return 0 em sucesso, -1 em erro
 */
int configurar_gpio_saida(const char *gpio_num);

/**
 * @brief Abre o arquivo value do GPIO para escrita
 * 
 * Retorna um descritor de arquivo que pode ser usado para escrever
 * valores (0 ou 1) no GPIO rapidamente.
 * 
 * @param gpio_num String com o número do GPIO
 * @return File descriptor (>=0) em sucesso, -1 em erro
 */
int abrir_gpio_value(const char *gpio_num);

/**
 * @brief Inicializa os GPIOs do motor (STEP e DIR)
 * 
 * Exporta, configura como saída e abre os file descriptors dos
 * pinos utilizados para controle do motor.
 * 
 * @param fd_step Ponteiro para armazenar o FD do pino STEP
 * @param fd_dir Ponteiro para armazenar o FD do pino DIR
 * @return 0 em sucesso, -1 em erro
 */
int inicializar_gpios(int *fd_step, int *fd_dir);

/**
 * @brief Escreve um valor no GPIO
 * 
 * Escreve "0" ou "1" no arquivo value do GPIO.
 * Caso o FD seja inválido (<0), opera em modo simulado.
 * 
 * @param fd File descriptor do GPIO (retornado por abrir_gpio_value)
 * @param val String com o valor: "0" ou "1"
 * @return 0 em sucesso, -1 em erro
 */
int gpio_write(int fd, const char *val);

/**
 * @brief Escreve dados de forma robusta, tratando interrupções
 * 
 * Garante que todos os bytes sejam escritos, mesmo que a chamada
 * write() seja interrompida (EINTR).
 * 
 * @param fd File descriptor
 * @param buf Buffer com os dados
 * @param count Quantidade de bytes a escrever
 * @return Número de bytes escritos, -1 em erro
 */
ssize_t write_all(int fd, const void *buf, size_t count);

#endif /* GPIO_H */

```
{% endtab %}
{% endtabs %}
