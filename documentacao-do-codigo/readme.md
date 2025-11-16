# README

## 🔧 Sistema de Controle de Motor - BeagleBone Black

![BeagleBone Black](https://img.shields.io/badge/Platform-BeagleBone%20Black-green?style=for-the-badge) ![C](https://img.shields.io/badge/Language-C-blue?style=for-the-badge\&logo=c) ![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge) ![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)

**Sistema embarcado de controle de motor de passo com interface web, proteção térmica e API REST**

Características • Instalação • Como Usar • API • Hardware • Documentação

***

### 📖 Sobre o Projeto

Sistema completo desenvolvido em C para controle de motores de passo em BeagleBone Black. O projeto implementa um servidor HTTP embarcado (sem dependências externas), interface web responsiva e sistema de proteção contra superaquecimento, tudo executando diretamente no hardware.

#### 🎯 Objetivo

Criar uma solução de integração entre a BeagleBone driver de motor e motor de passo em sistemas embarcados, demonstrando conceitos de:

* Programação bare-metal em C
* Controle de GPIOs via sysfs
* Servidor HTTP implementado do zero
* Programação multi-thread (POSIX threads)
* Interface web

***

### ✨ Características

#### 🌟 Principais Funcionalidades

* ✅ **Interface Web Responsiva** - Controle visual via navegador.
* ✅ **API REST Completa** - Endpoints para integração com outros sistemas
* ✅ **Proteção Térmica Inteligente** - Bloqueio automático quando temperatura > 30°C
* ✅ **Execução Assíncrona** - Motor roda em thread separada sem bloquear interface
* ✅ **Zero Dependências** - Servidor HTTP implementado do zero em C puro
* ✅ **Thread-Safe** - Sincronização via mutex para acesso seguro ao estado
* ✅ **Parada de Emergência** - Comando stop interrompe motor imediatamente

#### 🎮 Controle do Motor

**Sequência de Movimento:**

{% stepper %}
{% step %}
### Rotação 90º sentido horário (100 passos)

Comando no motor para gerar um acesso no controlador de acesso.
{% endstep %}

{% step %}
### Pausa de 2 segundos


{% endstep %}

{% step %}
### Retorno 90º sentido anti-horário (100 passos)

Retorna no sentido anti-horário utilizando o pino DIR do driver do motor
{% endstep %}
{% endstepper %}

{% hint style="warning" %}
Proteções:

* Motor não inicia se já estiver rodando
* Motor não inicia se sistema estiver superaquecido
* Movimento pode ser interrompido via API ou interface
* Verificação contínua de condições de segurança.


{% endhint %}

### ✨ Hierarquia de responsabilidade

#### Nível 1: main.c

* Inicializa todos os outros módulos
* Cuida do ciclo de vida do programa

#### Nível 2: camada de negócio

* http\_server.c: recebe requisições e decide o que fazer
* motor.c: executa a lógica de controle do motor

#### Nível 3: camada de infraestrutura

* gpio.c: realiza os comandos de gpio
* web\_content.c: fornece os arquivo estáticos (HTMLL/CSS/JS)



### Descrição das responsabilidades por módulo

#### main.c

**Responsabilidades:**

* ✅ Inicializar tudo na ordem correta
* ✅ Capturar Ctrl+C e finalizar com segurança
* ✅ Liberar recursos ao sair
* ❌ NÃO faz lógica de negócio
* ❌ NÃO mexe diretamente em hardware

{% stepper %}
{% step %}
### Inicializações

* Define a porta http;
* Configura os handlers de sinais;
* Chama a função de inicialização dos GPIOS;
* Inicializa o sistema do motor;
* Inicializa o server HTTP;
{% endstep %}

{% step %}
### Loop principal



* Espera conexões de clientes(accept de conexões http)
* Quando o cliente conecta passa para o http\_server.c
{% endstep %}

{% step %}
### Encerramento

Quando acontece uma interrupção CTRL+C

* Para o motor, se estiver rodando
* Fecha os GPIOS
* Fecha as portas do servidor
* Encerra o programa
{% endstep %}
{% endstepper %}

#### gpio.c

**Responsabilidades:**

* ✅ Abstrair complexidade do sysfs
* ✅ Gerenciar file descriptors
* ✅ Escrever valores nos pinos
* ✅ Suportar modo simulado
* ❌ NÃO sabe nada sobre "motor" ou "aplicação"
* ❌ É só uma camada fina sobre o hardware

#### motor.c

**Responsabilidades:**

* ✅ Gerenciar estado completo do motor
* ✅ Executar movimento em thread separada
* ✅ Proteger contra superaquecimento
* ✅ Permitir parada de emergência
* ✅ Sincronizar acesso multi-thread
* ❌ NÃO sabe de HTTP ou interface web
* ❌ NÃO mexe diretamente em sysfs (usa gpio.c)

**Estrutura de dados:**

{% tabs %}
{% tab title="C" %}
```c
motor_state_t {
    mutex                 // Tranca a porta quando alguém está mexendo
    current_temperature   // Temperatura atual (25°C)
    overheat             // Alarme de superaquecimento (ligado/desligado)
    stop_requested       // Botão de emergência foi apertado?
    motor_running        // Motor está rodando agora?
    motor_thread         // ID da requisição que está movendo o motor
    fd_step              // Atalho para o pino STEP
    fd_dir               // Atalho para o pino DIR
}
```
{% endtab %}
{% endtabs %}

**Funcionamento:**

{% stepper %}
{% step %}
### Usuário clica "Girar motor"
{% endstep %}

{% step %}
### http\_server pede para motor\_start\_rotation()
{% endstep %}

{% step %}
### motor.c verifica

* Motor está rodando?  OK
* Temperatura está normal? OK
* Não há pedido de parada?  OK
{% endstep %}

{% step %}
### motor.c cria uma thread separada

thread executa o movimento enquanto main.c continua atendendo outros pedidos
{% endstep %}
{% endstepper %}

Dentro da thread existe outro fluxo:

{% stepper %}
{% step %}
### FASE 0 - Inicializa

* Marca "motor\_running = 1"
{% endstep %}

{% step %}
### FASE 1 - Girar 90º

* Liga DIR = 1 (sentido horário)
* Para cada passo (100 vezes):
  * Verifica se alguém pediu para parar
  * Verifica se a temperatura subiu muito
  * Se ok, faz um pulso STEP
    * STEP = 1 (high)
    * Espera 1ms
    * STEP = 0 (low)
    * Espera 1ms
{% endstep %}

{% step %}
### Fase 2 - pausa 2 segundos

* Fica parado
* Mas continua verificando se existe alguma interrupção no sistema (descrito acima)
* Verifica 20 vezes (a cada 100ms)
{% endstep %}

{% step %}
### Fase 3 - Volta 90º sentido anti-horário

* Liga DIR= 0 (sentido anti-horário)
* Faz 100 pulsos novamente


{% endstep %}

{% step %}
### Fase 4 - Encerra

* Marca "motor\_running = 0"
{% endstep %}
{% endstepper %}

**Sistema de proteção térmica:**

```
Temperatura ≤ 30°C:
    overheat = 0
    Motor pode rodar normalmente

Temperatura > 30°C:
    overheat = 1
    stop_requested = 1
    Motor para imediatamente
    Novos pedidos são BLOQUEADOS
```

Thread Safety:

```
Thread A quer ler temperatura:
    1. Bate na porta (pthread_mutex_lock)
    2. Entra na sala
    3. Lê temperatura = 25
    4. Sai da sala (pthread_mutex_unlock)

Thread B tenta entrar ao mesmo tempo:
    1. Bate na porta
    2. Porta trancada! Espera...
    3. Thread A sai
    4. Thread B entra
    5. Faz seu trabalho
    6. Sai
```



#### http\_server.c

**Responsabilidades:**

* ✅ Implementar servidor HTTP simples
* ✅ Parsear requisições
* ✅ Rotear para handlers corretos
* ✅ Servir arquivos estáticos
* ✅ Montar respostas HTTP válidas
* ❌ NÃO sabe de GPIO ou hardware
* ❌ NÃO executa lógica do motor (só pede para motor.c)

**Lógica implementada:**

{% stepper %}
{% step %}
### Aguarda requisições

* Aguarda uma requisição;
* Quando um cliente requisita, recebe o pedido e entende.
{% endstep %}

{% step %}
### Parse da requisição

* Cliente envia "GET api/status HTTP/1.1"
* Parse:
  * Método: GET
  * Caminho: /api/status
  * É uma requisição de status


{% endstep %}

{% step %}
### Roteia a requisição (routing)

```
  /api/status      → handle_api_status()
   /api/rotate      → handle_api_rotate()
   /api/stop        → handle_api_stop()
   /api/temperature → handle_api_temperature()
   /                → página HTML
   /style.css       → arquivo CSS
   /script.js       → arquivo JavaScript
```
{% endstep %}

{% step %}
### Preapara a resposta

* Pede informação para o motor.c
* Monta a resposta (JSON ou HTML)
* Envia a resposta para o cliente


{% endstep %}

{% step %}
### Encerra

* Depois de responder, fecha a conexão
* Volta a esperar a próxima requisição
{% endstep %}
{% endstepper %}

Exemplo de fluxo completo:

```
CLIENTE: GET /api/status
    ↓
SERVIDOR: "Ok, vou buscar o status"
    ↓
SERVIDOR chama: motor_get_temperature()
    ↓
MOTOR: "Temperatura é 25°C"
    ↓
SERVIDOR monta JSON: {"temperature": 25, ...}
    ↓
SERVIDOR responde: HTTP/1.1 200 OK
                    Content-Type: application/json

                    {"temperature": 25, "status": "Normal", ...}
```

**Handlers de API:**

**handle\_api\_status():**

```
1. Trava o motor (mutex)
2. Lê temperatura atual
3. Lê se está superaquecido
4. Lê se motor está rodando
5. Destrava motor (mutex)
6. Monta string JSON
7. Envia resposta
```

**handle\_api\_rotate():**

```
1. Pergunta ao motor: "Pode rodar?"
2. Motor verifica:
   - Já está rodando? NÃO pode
   - Superaquecido? NÃO pode
   - Tudo ok? PODE!
3. Se pode: cria thread e responde "200 OK"
4. Se não pode: responde "409 Conflict"
```

**handle\_api\_temperature():**

```
1. Recebe body: "temperature=35"
2. Faz parse: encontra "35"
3. Valida: está entre -50 e 200? Sim
4. Chama motor_set_temperature(35)
5. Motor verifica: 35 > 30? Sim! Superaqueceu!
6. Motor marca overheat = 1 e para motor
7. Servidor responde: "200 OK"
```

**handle\_api\_stop():**

```
1. Marca stop_requested = 1
2. Thread do motor vai ver isso e parar
3. Responde: "200 OK, solicitada parada"
```





#### web\_content.c

**Responsabilidades:**

* ✅ Guardar HTML/CSS/JS como strings
* ✅ Fornecer getters simples
* ❌ NÃO processa nada
* ❌ NÃO interage com outras partes

**O que ele tem:**

Três "pôsteres" grandes guardados na memória:

**1. HOME\_HTML:**

htmlCopiar![](data:image/svg+xml;utf8,%0A%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20width%3D%2216%22%20height%3D%2216%22%20viewBox%3D%220%200%2016%2016%22%20fill%3D%22none%22%3E%0A%20%20%3Cpath%20d%3D%22M10.8%208.63V11.57C10.8%2014.02%209.82%2015%207.37%2015H4.43C1.98%2015%201%2014.02%201%2011.57V8.63C1%206.18%201.98%205.2%204.43%205.2H7.37C9.82%205.2%2010.8%206.18%2010.8%208.63Z%22%20stroke%3D%22%23717C92%22%20stroke-width%3D%221.05%22%20stroke-linecap%3D%22round%22%20stroke-linejoin%3D%22round%22%2F%3E%0A%20%20%3Cpath%20d%3D%22M15%204.42999V7.36999C15%209.81999%2014.02%2010.8%2011.57%2010.8H10.8V8.62999C10.8%206.17999%209.81995%205.19999%207.36995%205.19999H5.19995V4.42999C5.19995%201.97999%206.17995%200.999992%208.62995%200.999992H11.57C14.02%200.999992%2015%201.97999%2015%204.42999Z%22%20stroke%3D%22%23717C92%22%20stroke-width%3D%221.05%22%20stroke-linecap%3D%22round%22%20stroke-linejoin%3D%22round%22%2F%3E%0A%3C%2Fsvg%3E%0A)

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Controle Motor</title>
    <link rel="stylesheet" href="/style.css">
  </head>
  <body>
    <h1>Controle Motor</h1>
    <div id="temperature">--°C</div>
    <button onclick="rotateMotor()">Girar Motor</button>
    <script src="/script.js"></script>
  </body>
</html>
```

**2. STYLE\_CSS:**

```css
body {
  background: #f6faf4;
  color: #1b3b2b;
}
button {
  background: #137a4f;
  color: white;
  padding: 10px;
}
```

**3. SCRIPT\_JS:**

```javascript
async function fetchStatus() {
  const response = await fetch('/api/status');
  const data = await response.json();
  document.getElementById('temperature').textContent = 
    data.temperature + '°C';
}

setInterval(fetchStatus, 2000); // A cada 2 segundos
```

**Como funciona:**

```
Cliente pede: GET /
    ↓
http_server chama: get_home_html()
    ↓
web_content devolve: HOME_HTML (string grande)
    ↓
http_server envia para cliente

Cliente pede: GET /style.css
    ↓
http_server chama: get_style_css()
    ↓
web_content devolve: STYLE_CSS (string grande)
    ↓
http_server envia para cliente
```

### 🔄 Fluxo de Dados Completo <a href="#fluxodedadoscompletoexemploreal" id="fluxodedadoscompletoexemploreal"></a>

Pedido do início ao fim:

#### Cenário: Usuário clica "Girar Motor 90°" <a href="#cenriousurioclicagirarmotor90" id="cenriousurioclicagirarmotor90"></a>

{% stepper %}
{% step %}
**No Navegador (Frontend - JavaScript)**

```javascript
// Botão foi clicado
async function rotateMotor() {
    // Desabilita botão
    button.disabled = true;

    // Faz requisição HTTP
    const response = await fetch('/api/rotate', {
        method: 'POST'
    });

    // Mostra resultado
    if (response.ok) {
        showToast("Motor iniciado!");
    }
}
```
{% endstep %}

{% step %}
**Requisição Viaja pela Rede**

```
POST /api/rotate HTTP/1.1
Host: 192.168.1.100:8081
Content-Length: 0

(corpo vazio)
```
{% endstep %}

{% step %}
**Chega no BeagleBone (http\_server.c)**

```c
// accept() detecta nova conexão
int client = accept(listen_fd, ...);

// Lê dados
recv(client, buffer, ...);
// buffer = "POST /api/rotate HTTP/1.1\r\n..."

// Parse
char method[16], path[512];
sscanf(buffer, "%s %s", method, path);
// method = "POST"
// path = "/api/rotate"

// Roteamento
if (strcmp(method, "POST") == 0 && 
    strcmp(path, "/api/rotate") == 0) {
    handle_api_rotate(client, motor_state);
}
```
{% endstep %}

{% step %}
**Handler Processa (http\_server.c)**

```c
void handle_api_rotate(int client_fd, motor_state_t *state) {
    // Tenta iniciar motor
    int result = motor_start_rotation(state);

    if (result == 0) {
        // Sucesso!
        respond_text(client_fd, 200, "OK", 
                    "Movimento iniciado");
    } else {
        // Falhou (já rodando ou superaquecido)
        respond_text(client_fd, 409, "Conflict",
                    "Motor não pode iniciar");
    }
}
```
{% endstep %}

{% step %}
**Motor Verifica Condições (motor.c)**

```c
int motor_start_rotation(motor_state_t *state) {
    pthread_mutex_lock(&state->lock);

    // Verifica condições
    if (state->overheat) {
        pthread_mutex_unlock(&state->lock);
        return -1; // Superaquecido!
    }

    if (state->motor_running) {
        pthread_mutex_unlock(&state->lock);
        return -1; // Já está rodando!
    }

    pthread_mutex_unlock(&state->lock);

    // Tudo ok! Cria thread
    pthread_create(&state->motor_thread, NULL, 
                   motor_worker, state);

    return 0; // Sucesso
}
```
{% endstep %}

{% step %}
**Thread do Motor Inicia (motor.c)**

```c
void *motor_worker(void *arg) {
    motor_state_t *state = arg;

    // Marca como rodando
    pthread_mutex_lock(&state->lock);
    state->motor_running = 1;
    pthread_mutex_unlock(&state->lock);

    // Fase 1: Horário
    gpio_write(state->fd_dir, "1");
    for (int i = 0; i < 100; i++) {
        // Verifica parada
        pthread_mutex_lock(&state->lock);
        if (state->stop_requested || state->overheat) {
            pthread_mutex_unlock(&state->lock);
            goto cleanup;
        }
        pthread_mutex_unlock(&state->lock);

        // Pulso
        gpio_write(state->fd_step, "1");
        usleep(1000);
        gpio_write(state->fd_step, "0");
        usleep(1000);
    }

    // ... pausa e retorno ...

cleanup:
    pthread_mutex_lock(&state->lock);
    state->motor_running = 0;
    pthread_mutex_unlock(&state->lock);
    return NULL;
}
```
{% endstep %}

{% step %}
**GPIO Mexe no Hardware (gpio.c)**

```c
int gpio_write(int fd, const char *val) {
    // Vai para início do arquivo
    lseek(fd, 0, SEEK_SET);

    // Escreve "1" ou "0"
    write(fd, val, 1);

    // Força gravação
    fsync(fd);

    return 0;
}
```
{% endstep %}

{% step %}
&#x20;**Kernel e Hardware**

```
/sys/class/gpio/gpio60/value recebe "1"
    ↓
Kernel detecta mudança
    ↓
Altera tensão no pino físico P9_12
    ↓
Driver DRV8825 detecta HIGH
    ↓
Bobina do motor é energizada
    ↓
Motor dá um passo!
```
{% endstep %}

{% step %}
**Resposta Volta ao Navegador**

```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 18

Movimento iniciado
```
{% endstep %}

{% step %}
**JavaScript Atualiza Interface**

```javascript
showToast("✓ Motor iniciado!");
button.disabled = false;

// A cada 2 segundos, busca status novo
setInterval(async () => {
    const status = await fetch('/api/status');
    const data = await status.json();
    updateDisplay(data); // Atualiza tela
}, 2000);
```
{% endstep %}
{% endstepper %}

### 🏗️Guia para buildar o projeto <a href="#fluxodedadoscompletoexemploreal" id="fluxodedadoscompletoexemploreal"></a>

Estrutura do projeto:

```bash
# Entrar no diretório do projeto
cd /var/lib/cloud9/Projeto_jiga
ls -la src/
# gpio.c, gpio.h
# motor.c, motor.h
# http_server.c, http_server.h
# web_content.c, web_content.h
# main.c
```

Flags de compilação:

```bash
-Wall                    # Ativa avisos importantes
-Wextra                  # Ativa avisos extras
-O2                      # Otimização nível 2 (velocidade)
-pthread                 # Habilita suporte a threads POSIX
-D_XOPEN_SOURCE=500     # Define padrão POSIX
-c                       # Compila mas NÃO linka (gera .o)
-o arquivo.o            # Nome do arquivo de saída
```

```bash
gcc -Wall -Wextra -O2 -pthread -D_XOPEN_SOURCE=500 -c src/gpio.c -o build/gpio.o
gcc -std=c99 -Wall -Wextra -O2 -pthread -D_XOPEN_SOURCE=500 -c src/motor.c -o build/motor.o
gcc -Wall -Wextra -O2 -pthread -D_XOPEN_SOURCE=500 -c src/http_server.c -o build/server.o
gcc -Wall -Wextra -O2 -pthread -D_XOPEN_SOURCE=500 -c src/web_content.c -o build/web_content.o
gcc -Wall -Wextra -O2 -pthread -D_XOPEN_SOURCE=500 -c src/main.c -o build/main.o

```

Processo de linkagem em um único binário:

```bash
gcc build/gpio.o build/motor.o build/http_server.o build/web_content.o build/main.o -o bin/motor_control -pthread
```
