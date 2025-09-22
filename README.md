---
icon: terminal
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: false
  metadata:
    visible: true
---

# JIGA DE CONTROLE DE ACESSO

## Resumo

Este projeto tem como objetivo otimizar os processos de teste e desenvolvimento do software _Defense IA_, por da implementação de uma solução na área da eletrônica, utilizando equipamentos e integração com motor de passo, driver de motor e sistemas embarcados.



## Introdução

### Definição do Problema

O módulo de controle de acesso do _Defense IA_ integra-se a dispositivos físicos, como controladoras de acesso. Um dos principais desafios enfrentados pela equipe de desenvolvimento é a limitação de testes físicos, especialmente para colaboradores em regime remoto, que não conseguem realizar a leitura de cartões nos dispositivos para gerar eventos no sistema. Além disso, há a necessidade de realizar testes de carga, simulando acessos em massa, com volumes superiores a 1 milhão de registros, para avaliar a performance e a capacidade do banco de dados em armazenar e processar eventos de acesso. Este projeto concentra-se na resolução de dois problemas principais:

* Simulação de eventos de acesso por meio da ativação remota da passagem de cartões nos dispositivos,  através de uma interface web, permitindo a execução de testes mesmo em ambientes fora do laboratório, como no trabalho remoto.
* Testes de performance voltados para a validação da escalabilidade e da robustez do banco de dados, por meio da simulação de grandes volumes de eventos, superiores a 1 milhão de registros, com o objetivo de avaliar a capacidade de armazenamento, processamento e resposta do sistema sob carga intensa.

### Motivação do Projeto

A motivação central é o desenvolvimento de uma JIGA de testes que possa ser integrada à esteira de desenvolvimento do _Defense IA_. Essa JIGA utilizará dispositivos eletrônicos motor de passo, driver de motor e sistemas embarcados para simular fisicamente a passagem de cartões nos leitores, gerando eventos reais no sistema. O projeto representa uma aplicação prática de conhecimentos acadêmicos e técnicos, com potencial de impacto direto na eficiência dos testes e na qualidade do produto final. A integração entre hardware e software permitirá a automação de testes e a ampliação da cobertura de cenários, contribuindo para a evolução contínua da solução.

## Objetivos

### Objetivo Geral

Desenvolver uma JIGA de testes para o módulo de controle de acesso do software Defense IA, integrando conhecimentos de eletrônica e desenvolvimento de software, com o propósito de solucionar limitações nos testes físicos e de performance enfrentadas pela equipe.

### Objetivos específicos

* Montar fisicamente a JIGA de testes.
* Integrar sistema embarcado com motor de passo e cartão de acesso.
* Desenvolver interface web para controle remoto.
* Implementar controle de temperatura e acionamento de ventoinha.

## Metodologia

Este trabalho utiliza a metodologia de Pesquisa-Ação, por ser adequada à investigação de problemas reais com aplicação prática de soluções. A Pesquisa-Ação é caracterizada pela integração entre a produção de conhecimento e a intervenção no contexto estudado, permitindo que o pesquisador atue diretamente na realidade observada. O processo metodológico foi dividido em quatro etapas principais:&#x20;

(1) diagnóstico do problema, por meio de levantamento bibliográfico e observação prática;&#x20;

(2) planejamento da ação, com definição das estratégias e recursos necessários para o desenvolvimento do projeto físico;&#x20;

(3) implementação, onde o conhecimento sintetizado foi aplicado na construção e execução do projeto; e&#x20;

(4) avaliação e reflexão, com análise dos resultados obtidos e apresentação das conclusões aos professores.

## Arquitetura do projeto

{% tabs %}
{% tab title="BeagleBone" %}
\[Cliente Web (PC/Celular)]\
⇅ Ethernet/Wi-Fi\
\[BeagleBone]\
├── Servidor Web (Apache/Nginx + Backend Python/C++)\
├── Interface com GPIOs (via PRU ou libgpiod)\
├── Ventoinha\
└── Driver de Motor (ex: DRV8824)\
↓\
\[Motor de Passo]



**Esquema de ligação entre BeagleBone e driver de motor**

| BeagleBone | DRV8824 |
| ---------- | ------- |
| GPIO1\_6   | STEP    |
| GPIO1\_7   | DIR     |
| GPIO1\_2   | ENABLE  |

**Esquema de ligação entre BeagleBone e ventoinha**

+12V (Fonte externa)\
│\
├────────────┐\
│                                   │\
\[Ventoinha] -  \[Diodo 1N4007]\
│                                  │\
│                               ┌┴┐\
└────┬──────┘  └──┐\
&#x20;              │                              │\
\[Dreno do MOSFET]          │\
&#x20;              │                              │\
\[Fonte do MOSFET] ───┘\
&#x20;              │\
&#x20;            GND (comum com BeagleBone)



BeagleBone GPIO\_Z ──┬─\[Resistor 1kΩ]─┬─> \[Gate do MOSFET]\
&#x20;                                            │                               │\
&#x20;                                         GND                         GND
{% endtab %}

{% tab title="ESP32" %}
\[Cliente Web (PC/Celular)]\
⇅ Wi-Fi\
\[ESP32]\
├── Servidor Web (HTTP ou WebSocket)\
├── Controle do Motor de Passo (via GPIO)\
└── Driver de Motor (ex: A4988, DRV8825)\
↓\
\[Motor de Passo]
{% endtab %}
{% endtabs %}

## Materiais e equipamentos utilizados

| Equipamento                          | Quantidade |
| ------------------------------------ | ---------- |
| BeagleBoard                          | 1          |
| Resistor (1k ohms)                   | 1          |
| Resistor (10k ohms)                  | 1          |
| Transistor (IRLZ44N)                 | 1          |
| Diodo (1N4007)                       | 1          |
| Driver de motor (DR8825)             | 1          |
| Motor de passo (NEMA 17)             | 1          |
| Madeira MDF                          | 7          |
| Controlador de acesso (SS 3532 MF W) | 2          |
| Fonte (5V)                           | 1          |

## Desenvolvimento e implementação

### Código para controle do motor de passo

{% tabs %}
{% tab title="C" %}
```javascript
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// Define os números dos GPIOs usados para controlar o motor
#define STEP1 60  // GPIO 60
#define STEP2 61  // GPIO 61
#define STEP3 62  // GPIO 62
#define STEP4 63  // GPIO 63

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

    return 0;
}

```
{% endtab %}
{% endtabs %}

## Resultados e avaliação
