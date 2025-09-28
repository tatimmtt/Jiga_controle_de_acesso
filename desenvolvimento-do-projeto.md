---
description: Descrição das etapas de desenvolvimento do projeto
---

# Desenvolvimento do projeto

## Controle do motor de passo

Para realizar o controle do motor de passo, será necessário utilizar os seguintes componentes:

* Um driver de motor compatível com o tipo de motor de passo escolhido;
* Um motor de passo, responsável pela movimentação precisa;
* A BeagleBone, que atuará como unidade de controle;
* Os pinos GPIO da BeagleBone, que serão utilizados para enviar os sinais de comando ao driver.

A BeagleBone será programada para gerar os pulsos e sinais de direção necessários ao funcionamento do motor, por meio dos pinos GPIO previamente definidos.

### Código para controle do motor de passo

Abaixo está o código utilizado para controle do driver do motor:

{% tabs %}
{% tab title="C" %}
```javascript
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// Define os números dos GPIOs usados para controlar o motor
#define STEP1 30  // GPIO 60
#define STEP2 31  // GPIO 61
#define STEP3 48  // GPIO 62
#define STEP4 51  // GPIO 63

// Array com os GPIOs
int gpios[] = {STEP1, STEP2, STEP3, STEP4};

// Sequência de ativação das bobinas do motor (motor unipolar de 4 fases)
int seq[4][4] = {
    {1, 0, 0, 0},
    {0, 1, 0, 0},
    {0, 0, 1, 0},
    {0, 0, 0, 1}
};

// Função para exportar e configurar um GPIO como saída
void export_gpio(int gpio) {
    char path[50];
    FILE *f = fopen("/sys/class/gpio/export", "w");
    if (f) {
        fprintf(f, "%d", gpio); // Exporta o GPIO
        fclose(f);
    }

    // Define o GPIO como saída
    sprintf(path, "/sys/class/gpio/gpio%d/direction", gpio);
    f = fopen(path, "w");
    if (f) {
        fprintf(f, "out");
        fclose(f);
    }
}

// Função para escrever valor (0 ou 1) em um GPIO
void write_gpio(int gpio, int value) {
    char path[50];
    sprintf(path, "/sys/class/gpio/gpio%d/value", gpio);
    FILE *f = fopen(path, "w");
    if (f) {
        fprintf(f, "%d", value);
        fclose(f);
    }
}

// Ativa os GPIOs de acordo com a sequência de passo
void set_step(int step[4]) {
    for (int i = 0; i < 4; i++) {
        write_gpio(gpios[i], step[i]);
    }
}

// Função para girar o motor
// steps: número de passos
// direction: 1 = horário, 0 = anti-horário
// delay_ms: tempo entre passos (em milissegundos)
void rotate(int steps, int direction, int delay_ms) {
    for (int i = 0; i < steps; i++) {
        int idx = direction ? i % 4 : (3 - (i % 4)); // Define a direção
        set_step(seq[idx]); // Aplica o passo
        usleep(delay_ms * 1000); // Espera entre passos
    }
}

// Função para desligar todas as bobinas (parar o motor)
void stop_motor() {
    int stop[4] = {0, 0, 0, 0};
    set_step(stop);
}

int main() {
    // Exporta e configura os GPIOs
    for (int i = 0; i < 4; i++) export_gpio(gpios[i]);

    // Interface simples via terminal
    char comando[20];
    printf("Digite o comando:\n");
    printf("1 - horario\n2 - anti-horario\n3 - girar\n4 - parar\n");
    scanf("%s", comando);

    // Executa o comando escolhido
    if (strcmp(comando, "1") == 0) {
        rotate(50, 1, 10); // Gira 90º no sentido horário
    } else if (strcmp(comando, "2") == 0) {
        rotate(50, 0, 10); // Gira 90º no sentido anti-horário
    } else if (strcmp(comando, "3") == 0) {
        while (1) rotate(1, 1, 10); // Gira continuamente
    } else if (strcmp(comando, "4") == 0) {
        stop_motor(); // Para o motor
    } else {
        printf("Comando inválido.\n");
    }
```
{% endtab %}
{% endtabs %}
