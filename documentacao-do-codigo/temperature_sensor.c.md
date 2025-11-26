# temperature\_sensor.c

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file temperature_sensor.c
 * @brief Implementação do driver do sensor GY-906 (MLX90614)
 * VERSÃO CORRIGIDA - Fix para leitura incorreta
 */

#include "temperature_sensor.h"
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <string.h>
#include <errno.h>
#include <time.h>

/* ========== FUNÇÕES PRIVADAS ========== */

/**
 * @brief Lê palavra (16 bits) do sensor via SMBus
 * CORRIGIDO: Usa protocolo SMBus correto
 */
static int i2c_read_word(int fd, unsigned char reg) {
    // Método 1: Usar função SMBus nativa do Linux
    #ifdef I2C_SMBUS_WORD_DATA
    union i2c_smbus_data data;
    struct i2c_smbus_ioctl_data args;

    args.read_write = I2C_SMBUS_READ;
    args.command = reg;
    args.size = I2C_SMBUS_WORD_DATA;
    args.data = &data;

    if (ioctl(fd, I2C_SMBUS, &args) < 0) {
        return -1;
    }

    return data.word & 0xFFFF;
    #else
    // Método 2: Leitura manual (fallback)
    unsigned char buf[3];

    // Escreve registrador
    if (write(fd, &reg, 1) != 1) {
        return -1;
    }

    // Aguarda sensor processar
    usleep(50000);  // 50ms (aumentado de 10ms)

    // Lê 3 bytes: LSB, MSB, PEC
    if (read(fd, buf, 3) != 3) {
        return -1;
    }

    // Combina LSB e MSB (little-endian)
    int data = (buf[1] << 8) | buf[0];

    return data;
    #endif
}

/**
 * @brief Converte dado bruto para temperatura em Celsius
 * CORRIGIDO: Adiciona validação de dados
 */
static float raw_to_celsius(int raw) {
    // VALIDAÇÃO: Verificar se dado é válido
    if (raw < 0 || raw == 0xFFFF) {
        // Valor inválido
        return -999.0;
    }

    // Verificar range válido do MLX90614
    // Range: -70°C a +382°C em Kelvin = 203.15K a 655.15K
    // Em raw: 10157 a 32757
    if (raw < 10000 || raw > 33000) {
        // Valor fora do range esperado
        fprintf(stderr, "⚠️  Valor raw fora do range: %d (0x%04X)\n", raw, raw);
        return -999.0;
    }

    // Fórmula do MLX90614: Temp(K) = raw * 0.02
    float temp_kelvin = (float)raw * 0.02;
    float temp_celsius = temp_kelvin - 273.15;

    // Validação final da temperatura
    if (temp_celsius < -70 || temp_celsius > 382) {
        fprintf(stderr, "⚠️  Temperatura calculada fora do range: %.2f°C\n", temp_celsius);
        return -999.0;
    }

    return temp_celsius;
}

/**
 * @brief Thread que lê temperatura periodicamente
 */
static void *temp_sensor_worker(void *arg) {
    temp_sensor_t *sensor = (temp_sensor_t *)arg;
    int leituras_validas = 0;
    int leituras_erro = 0;

    printf("🌡️  Thread de leitura de temperatura iniciada\n");
    printf("   → Intervalo: %d segundos (5 minutos)\n\n", LEITURA_INTERVALO_SEC);

    while (1) {
        // Verifica se deve parar
        pthread_mutex_lock(&sensor->lock);
        int continuar = sensor->thread_rodando;
        pthread_mutex_unlock(&sensor->lock);

        if (!continuar) {
            break;
        }

        // Lê temperatura
        int temp = temp_sensor_read(sensor);

        if (temp != -999) {
            leituras_validas++;

            pthread_mutex_lock(&sensor->lock);
            int temp_anterior = sensor->temperatura_atual;
            sensor->temperatura_atual = temp;
            sensor->sensor_ativo = 1;
            pthread_mutex_unlock(&sensor->lock);

            // Log com timestamp
            time_t now = time(NULL);
            struct tm *tm_info = localtime(&now);
            char time_str[20];
            strftime(time_str, sizeof(time_str), "%H:%M:%S", tm_info);

            printf("[%s] 🌡️  Leitura #%d: %d°C", 
                   time_str, leituras_validas, temp);

            // Alertas
            if (temp > 30) {
                printf(" ⚠️  SUPERAQUECIMENTO!\n");
            } else if (temp_anterior > 30 && temp <= 30) {
                printf(" ✓ Temperatura normalizada\n");
            } else {
                printf(" ✓ Normal\n");
            }

            // Chamar callback
            if (sensor->callback) {
                sensor->callback(temp);
            }

        } else {
            leituras_erro++;

            pthread_mutex_lock(&sensor->lock);
            sensor->sensor_ativo = 0;
            pthread_mutex_unlock(&sensor->lock);

            printf("⚠️  Erro ao ler sensor (tentativa %d/%d)\n", 
                   leituras_erro, leituras_validas + leituras_erro);

            // Se muitos erros consecutivos, avisar
            if (leituras_erro > 5 && leituras_validas == 0) {
                printf("⚠️  Muitos erros! Verifique conexão I2C\n");
            }
        }

        // Aguarda próxima leitura (verificando a cada 1s se deve parar)
        for (int i = 0; i < LEITURA_INTERVALO_SEC; i++) {
            pthread_mutex_lock(&sensor->lock);
            int deve_continuar = sensor->thread_rodando;
            pthread_mutex_unlock(&sensor->lock);

            if (!deve_continuar) {
                break;
            }

            sleep(1);
        }
    }

    printf("\n🌡️  Thread de temperatura finalizada\n");
    printf("   Estatísticas: %d leituras OK, %d erros\n\n", 
           leituras_validas, leituras_erro);

    return NULL;
}

/* ========== FUNÇÕES PÚBLICAS ========== */

int temp_sensor_init(temp_sensor_t *sensor, void (*callback)(int)) {
    if (!sensor) {
        fprintf(stderr, "Erro: ponteiro sensor é NULL\n");
        return -1;
    }

    printf("🌡️  Inicializando sensor de temperatura GY-906 (MLX90614)...\n");

    // Zerar estrutura
    memset(sensor, 0, sizeof(temp_sensor_t));
    sensor->callback = callback;
    sensor->temperatura_atual = 25;  // Temperatura inicial padrão
    sensor->fd_i2c = -1;
    sensor->sensor_ativo = 0;

    // Inicializar mutex
    if (pthread_mutex_init(&sensor->lock, NULL) != 0) {
        perror("Erro ao criar mutex do sensor");
        return -1;
    }

    // Abrir dispositivo I2C
    printf("   → Abrindo %s...\n", I2C_DEVICE);
    sensor->fd_i2c = open(I2C_DEVICE, O_RDWR);
    if (sensor->fd_i2c < 0) {
        printf("   ⚠️  Não foi possível abrir I2C: %s\n", strerror(errno));
        printf("   ⚠️  Dica: Execute 'sudo i2cdetect -y -r 1'\n");
        printf("   ⚠️  Continuando em MODO SIMULADO\n");
        return 0;  // Não é erro fatal
    }

    // Configurar endereço do slave
    printf("   → Configurando endereço 0x%02X...\n", MLX90614_ADDR);
    if (ioctl(sensor->fd_i2c, I2C_SLAVE, MLX90614_ADDR) < 0) {
        printf("   ⚠️  Erro ao configurar endereço I2C: %s\n", strerror(errno));
        close(sensor->fd_i2c);
        sensor->fd_i2c = -1;
        printf("   ⚠️  Continuando em MODO SIMULADO\n");
        return 0;
    }

    // Testar leitura COM DEBUG
    printf("   → Testando comunicação...\n");

    // Ler valor bruto primeiro
    int raw = i2c_read_word(sensor->fd_i2c, MLX90614_TOBJ1);

    if (raw < 0) {
        printf("   ⚠️  Erro ao ler sensor\n");
        close(sensor->fd_i2c);
        sensor->fd_i2c = -1;
        printf("   ⚠️  Continuando em MODO SIMULADO\n");
        return 0;
    }

    // DEBUG: Mostrar valor bruto
    printf("   → Valor bruto lido: %d (0x%04X)\n", raw, raw);

    // Converter para Celsius
    float temp = raw_to_celsius(raw);

    if (temp == -999.0) {
        printf("   ⚠️  Valor inválido do sensor (raw=%d)\n", raw);
        printf("   ⚠️  Verifique conexões:\n");
        printf("      • VCC → 3.3V (P9_3)\n");
        printf("      • GND → GND (P9_1)\n");
        printf("      • SDA → P9_18 (I2C1_SDA)\n");
        printf("      • SCL → P9_17 (I2C1_SCL)\n");
        close(sensor->fd_i2c);
        sensor->fd_i2c = -1;
        printf("   ⚠️  Continuando em MODO SIMULADO\n");
        return 0;
    }

    sensor->temperatura_atual = (int)(temp + 0.5);
    sensor->sensor_ativo = 1;

    printf("   ✓ Sensor respondendo!\n");
    printf("   ✓ Temperatura inicial: %.1f°C (raw=%d)\n", temp, raw);
    printf("✅ Sensor GY-906 inicializado com sucesso!\n\n");

    return 0;
}

int temp_sensor_start(temp_sensor_t *sensor) {
    if (!sensor) {
        return -1;
    }

    pthread_mutex_lock(&sensor->lock);
    sensor->thread_rodando = 1;
    pthread_mutex_unlock(&sensor->lock);

    // Criar thread de leitura periódica
    if (pthread_create(&sensor->thread, NULL, temp_sensor_worker, sensor) != 0) {
        perror("Erro ao criar thread de temperatura");
        return -1;
    }

    // Detach (thread gerencia próprio ciclo de vida)
    pthread_detach(sensor->thread);

    return 0;
}

int temp_sensor_read(temp_sensor_t *sensor) {
    if (!sensor) {
        return -999;
    }

    pthread_mutex_lock(&sensor->lock);
    int fd = sensor->fd_i2c;
    pthread_mutex_unlock(&sensor->lock);

    // Modo simulado (sem hardware)
    if (fd < 0) {
        // Simula temperatura variando entre 22-28°C
        static int temp_sim = 25;
        static int contador = 0;

        contador++;

        // Varia suavemente
        if (contador % 3 == 0) {
            temp_sim += (rand() % 3) - 1;  // -1, 0, ou +1
            if (temp_sim < 22) temp_sim = 22;
            if (temp_sim > 28) temp_sim = 28;
        }

        return temp_sim;
    }

    // Leitura real do sensor
    int raw = i2c_read_word(fd, MLX90614_TOBJ1);

    if (raw < 0) {
        fprintf(stderr, "⚠️  Erro na leitura I2C\n");
        return -999;
    }

    // DEBUG em modo verbose
    #ifdef DEBUG_TEMP
    printf("DEBUG: raw=%d (0x%04X)\n", raw, raw);
    #endif

    float temp = raw_to_celsius(raw);

    if (temp == -999.0) {
        fprintf(stderr, "⚠️  Temperatura inválida (raw=%d/0x%04X)\n", raw, raw);
        return -999;
    }

    // Arredondar para inteiro
    return (int)(temp + 0.5);
}

int temp_sensor_get_temperature(temp_sensor_t *sensor) {
    if (!sensor) {
        return -999;
    }

    pthread_mutex_lock(&sensor->lock);
    int temp = sensor->temperatura_atual;
    pthread_mutex_unlock(&sensor->lock);

    return temp;
}

int temp_sensor_is_active(temp_sensor_t *sensor) {
    if (!sensor) {
        return 0;
    }

    pthread_mutex_lock(&sensor->lock);
    int ativo = (sensor->fd_i2c >= 0 && sensor->sensor_ativo);
    pthread_mutex_unlock(&sensor->lock);

    return ativo;
}

void temp_sensor_cleanup(temp_sensor_t *sensor) {
    if (!sensor) {
        return;
    }

    printf("🌡️  Finalizando sensor de temperatura...\n");

    // Parar thread
    pthread_mutex_lock(&sensor->lock);
    sensor->thread_rodando = 0;
    pthread_mutex_unlock(&sensor->lock);

    // Aguardar thread finalizar (timeout 3s)
    printf("   → Aguardando thread finalizar...\n");
    for (int i = 0; i < 30; i++) {
        usleep(100000);  // 100ms
    }

    // Fechar I2C
    if (sensor->fd_i2c >= 0) {
        close(sensor->fd_i2c);
        printf("   ✓ Dispositivo I2C fechado\n");
    }

    // Destruir mutex
    pthread_mutex_destroy(&sensor->lock);

    printf("   ✓ Sensor finalizado com sucesso\n");
}

```
{% endtab %}
{% endtabs %}

