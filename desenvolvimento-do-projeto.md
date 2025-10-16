---
description: Descrição das etapas de desenvolvimento do projeto
---

# Desenvolvimento do projeto

## Controle do motor de passo

Para realizar o controle do motor de passo foi utilizado um driver de motor o DRV 8825, conforme documentado na seção [materiais-utilizados](materiais-utilizados/ "mention") o controle foi realizado utilizando o pino step.

Algumas considerações para o motor funcionar:

Alimentação para o motor é de 30volts

adicionado um capacitor de aclopamento para suprimir ruídos

utilizado os pinos, A1 A2 B1 B2 nas saídas das bobinas do motor, o motor é bipolar, portanto uma bobina foi conectada em A1 A2  e outra bobina foi conectada em B1 B2.

O driver foi configurado na função half step, ou seja ele tem o passo de 1,8º é necessário 200 passos em full step para 360º em half step é necessário 400 passos para a volta completa.

No projeto foi idealizado uma volta de 90º para realizar o acesso, e após 5 segundos foi utilizado o pino DIR para mudar a direção de horário para anti-horário e voltar o motor para posião inicial com os mesmos passos que foram realizados para girar 90º voltou -90º para posição inicial.

Os pinos M0, M1, M2 foram utilizados para configurar o driver em half step, para half step M0 foi configurado em HIGH M1 E M2 em LOW.

Além disso, também foram utilizados o pino STEP para mandar os pulsos e o pino DIR para setar a direção horário /anti-horario

### Código em C para passos do motor.

Para realizar o controle na beaglebone, foi criado um código em C para controle de 2 GPIOS uma para o pino STEP, outra para o pino DIR. Conforme código abaixo

{% tabs %}
{% tab title="C" %}
```javascript
#define _XOPEN_SOURCE 500   // Habilita extensões POSIX/XSI e define comportamento de headers da biblioteca C.
                            // O valor 500 refere-se ao X/Open 5 / POSIX.1c (aprox. POSIX.1-1995)
#include <stdio.h>          // I/O padrão(printf, sprintf, perror, snprintf)
#include <stdlib.h>         // utilitários padrão (malloc, free, atoi/strtol, exit, system)
#include <string.h>         // operações em string
#include <unistd.h>         // chamadas POSIX (read, write, close, usleep)
#include <fcntl.h>          // flags e funções de file control, (open, O_WRONLY)
#include <errno.h>          // errno e códigos de erro (EINTR,  etc)
#include <time.h>           //estruturas e funções de tempo (time_t, nanosleep)


#define GPIO_60 "60"    // STEP
#define GPIO_112 "112"  // DIR

#define VOLTA_COMPLETA 400  // define volta completa no modo Half step , M0 = HIGH; M1, M2 = LOW
#define QUARTO_VOLTA 100   // passos necessário para giro 90º

static int fd_gpio60 = -1;  //inicializa gpio60 com -1, indica que ainda não foi aberto
static int fd_gpio112 = -1; //inicializa gpio112 com -1, indica que ainda não foi aberto
/* ---------- utilitário de escrita robusta ----------
escreve todos os count no arquivo
retorna número de bytes lidos   
melhor que a função write (que pode escrever menos byte que solicitado)
*/
ssize_t write_all(int fd, const void *buf, size_t count) {
    size_t written = 0;     //variável que acumula quantos bytes já foram escritos
    const char *p = buf;    //ponteiro para bytes do buffer (gpio)
    while (written < count) { // loop que continua até que todos os bytes tenham sido escrito
        ssize_t n = write(fd, p + written, count - written);  //escreve até o arquivo acabar;
                                                             //p + written --> ajusta o ponteiro para a próxima posição. 
                                                             //count - writen --> é o tamanho restante
        if (n < 0) {                                        // verifica se n retorna valor negativo, se for negativo tem erro
            if (errno == EINTR) continue;                   // erro padrão errno = EINTR
            return -1;
        }
        written += n;                                       //incrementa o contador com a quantidade escrita na iteração
    }
    return (ssize_t)written;                                // após todos os bytes escritos, retorna o total escrito
}

/* ---------- GPIO export ---------- */
int exportar_gpio(const char *gpio_num) {
    int f = open("/sys/class/gpio/export", O_WRONLY);
    if (f < 0) {
        perror("Erro ao abrir /sys/class/gpio/export");     //trata erro se acontecer erro ao abrir o arquivo
        return -1;
    }
    if (write_all(f, gpio_num, strlen(gpio_num)) < 0) { //write_all garante que todos os bytes sejam escritos
        if (errno != EBUSY) {                           // EBUSY é um caso esperado, caso o GPIO já tenha sido exportado.
            perror("Erro ao escrever em /sys/class/gpio/export");
            close(f);
            return -1;
        }
    }
    close(f);
    usleep(200000);
    return 0;
}


/* ---------- Configura GPIO como saída ---------- 
        Escreve out no arquivo, para configurar o pino como saída
*/
int configurar_gpio_saida(const char *gpio_num) {
    char path[80];
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/direction", gpio_num);
    int f = open(path, O_WRONLY);
    if (f < 0) {
        perror("Erro ao abrir direction do GPIO");
        return -1;
    }
    if (write_all(f, "out", 3) < 0) {
        perror("Erro ao escrever direction");
        close(f);
        return -1;
    }
    close(f);
    return 0;
}


/* ---------- Monta o caminho do arquivo ---------- 
        Abre value para escrita
*/
int abrir_gpio_value(const char *gpio_num) {
    char path[80];
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/value", gpio_num);
    int f = open(path, O_WRONLY);
    if (f < 0) {
        perror("Erro ao abrir value do GPIO");
        return -1;
    }
    return f;
}



/* ---------- Inicializa os 2GPIOS usados pelo driver do motor ---------- */
int inicializar_gpios(int *fd_step, int *fd_dir) {
    if (exportar_gpio(GPIO_60) < 0) return -1;
    if (configurar_gpio_saida(GPIO_60) < 0) return -1;
    *fd_step = abrir_gpio_value(GPIO_60);
    if (*fd_step < 0) return -1;

    if (exportar_gpio(GPIO_112) < 0) return -1;
    if (configurar_gpio_saida(GPIO_112) < 0) return -1;
    *fd_dir = abrir_gpio_value(GPIO_112);
    if (*fd_dir < 0) return -1;

    printf("GPIOS inicializados com sucesso\n");
    return 0;
}

/* ---------- Escreve 1 ou 0 no descritor ---------- */
int gpio_write(int fd, const char *val) {
    if (fd < 0) return -1;              // verifica se fd é válido
    if (lseek(fd, 0, SEEK_SET) < 0) {   // reposiciona  o offset para o início do arquivo 
        perror("lseek gpio");
        return -1;
    }
    if (write_all(fd, val, 1) < 0) {
        perror("write gpio");
        return -1;
    }
    fsync(fd);      //forçar sincronização do descritor com o kernel 
    return 0;
}

/* ---------- Executa a rotina física para girar o motor 90° (sentido horário)
e depois retorna à posição inicial (retorno anti-horário). 
É uma função bloqueante que gera os pulsos STEP e seta o pino DIR adequadamente. ---------- */
void girar_motor_noventa_graus(int fd_step, int fd_dir) {
    if (fd_step < 0 || fd_dir < 0) {        // valida se fd_step e fd_dir são válidos
        fprintf(stderr, "Descriptors inválidos no girar_motor_noventa_graus\n");
        return;
    }

    // Direção horário
    gpio_write(fd_dir, "1");
    printf("[motor] Girando no sentido horário...\n");
    for (int i = 0; i < QUARTO_VOLTA; i++) {
        gpio_write(fd_step, "1");
        usleep(1000); // 1ms por pulso — ajuste conforme necessidade
        gpio_write(fd_step, "0");
        usleep(1000);
    }

    printf("[motor] Pausa 2s\n");
    sleep(2);

    // Retorno anti-horário
    gpio_write(fd_dir, "0");
    printf("[motor] Retornando no sentido anti-horário...\n");
    for (int i = 0; i < QUARTO_VOLTA; i++) {
        gpio_write(fd_step, "1");
        usleep(1000);
        gpio_write(fd_step, "0");
        usleep(1000);

```
{% endtab %}
{% endtabs %}

## Interface web para controle remoto do motor

Para atender o objetivo do projeto, que é realizar os comandos no motor para gerar um evento na controladora de acesso, foi desenvolvido um web server http e apis para chamar a função de girar motor, essa função irá girar o motor e gerar o evento do dispositivo de controle de acesso, atendendo o objetivo do projeto.

Segue abaixo o código que foi desenvolido para o webserver http.

{% tabs %}
{% tab title="C" %}
```javascript
#define _XOPEN_SOURCE 500   // Habilita extensões POSIX/XSI e define comportamento de headers da biblioteca C.
                            // O valor 500 refere-se ao X/Open 5 / POSIX.1c (aprox. POSIX.1-1995)
#include <stdio.h>          // I/O padrão(printf, sprintf, perror, snprintf)
#include <stdlib.h>         // utilitários padrão (malloc, free, atoi/strtol, exit, system)
#include <string.h>         // operações em string
#include <unistd.h>         // chamadas POSIX (read, write, close, usleep)
#include <fcntl.h>          // flags e funções de file control, (open, O_WRONLY)
#include <errno.h>          // errno e códigos de erro (EINTR,  etc)
#include <signal.h>         //manipulação de sinais (signal, signaction, SIGINT)
#include <time.h>           //estruturas e funções de tempo (time_t, nanosleep)

#include <sys/types.h>      //tipos básicos de sistema (pid_t, ssize_t, etc)
#include <sys/socket.h>     // api de sockets (socket, bind, listen, accept, etc)
#include <netinet/in.h>     // estruturas para endereço ipv4(strutc addr_in, etc)
#include <arpa/inet.h>      // conversões de endereço (inet,ntoa, inet_pton)

#define GPIO_60 "60"    // STEP
#define GPIO_112 "112"  // DIR

#define VOLTA_COMPLETA 400  // define volta completa no modo full step , M0 ,M1, M2 = LOW
#define QUARTO_VOLTA 100   // passos necessário para giro 90º
#define DEFAULT_PORT 8081 // porta default para o servidor HTTP
#define BACKLOG 8           // tamanho da fila de conexões pendentes para listen()
#define RECV_BUF 8192       // tamanho do buffer utilizado para requisições http com recv 8192 bytes (8kb) suficiente para requisições simples (header + body)

static int fd_gpio60 = -1;  //inicializa gpio60 com -1, indica que ainda não foi aberto
static int fd_gpio112 = -1; //inicializa gpio112 com -1, indica que ainda não foi aberto
static int listen_fd = -1; //descritor do socket de escuta, inicializar com -1 significa que ainda não foi aberto.

/* ---------- Executa a rotina física para girar o motor 90° (sentido horário)
e depois retorna à posição inicial (retorno anti-horário). 
É uma função bloqueante que gera os pulsos STEP e seta o pino DIR adequadamente. ---------- */
void girar_motor_noventa_graus(int fd_step, int fd_dir) {
    if (fd_step < 0 || fd_dir < 0) {        // valida se fd_step e fd_dir são válidos
        fprintf(stderr, "Descriptors inválidos no girar_motor_noventa_graus\n");
        return;
    }

    // Direção horário
    gpio_write(fd_dir, "1");
    printf("[motor] Girando no sentido horário...\n");
    for (int i = 0; i < QUARTO_VOLTA; i++) {
        gpio_write(fd_step, "1");
        usleep(1000); // 1ms por pulso — ajuste conforme necessidade
        gpio_write(fd_step, "0");
        usleep(1000);
    }

    printf("[motor] Pausa 2s\n");
    sleep(2);

    // Retorno anti-horário
    gpio_write(fd_dir, "0");
    printf("[motor] Retornando no sentido anti-horário...\n");
    for (int i = 0; i < QUARTO_VOLTA; i++) {
        gpio_write(fd_step, "1");
        usleep(1000);
        gpio_write(fd_step, "0");
        usleep(1000);
    }

    printf("[motor] Movimento completo.\n");
}




/* ---------- respostas HTTP ---------- */
void respond_raw(int client_fd, const char *status_line, const char *content_type, const char *body, size_t body_len) {
    char header[512];
    int n = snprintf(header, sizeof(header),
                     "%s\r\n"
                     "Content-Type: %s\r\n"
                     "Content-Length: %zu\r\n"
                     "Connection: close\r\n"
                     "\r\n",
                     status_line, content_type, body_len);
    if (n > 0) write_all(client_fd, header, (size_t)n);
    if (body_len > 0 && body != NULL) write_all(client_fd, body, body_len);
}
void respond_text(int client_fd, int status_code, const char *status_text, const char *body) {
    char status_line[64];
    snprintf(status_line, sizeof(status_line), "HTTP/1.1 %d %s", status_code, status_text);
    respond_raw(client_fd, status_line, "text/plain; charset=utf-8", body ? body : "", body ? strlen(body) : 0);
}
void respond_html(int client_fd, int status_code, const char *status_text, const char *html) {
    char status_line[64];
    snprintf(status_line, sizeof(status_line), "HTTP/1.1 %d %s", status_code, status_text);
    respond_raw(client_fd, status_line, "text/html; charset=utf-8", html ? html : "", html ? strlen(html) : 0);
}
void respond_css(int client_fd, const char *css) {
    respond_raw(client_fd, "HTTP/1.1 200 OK", "text/css; charset=utf-8", css, css ? strlen(css) : 0);
}
void respond_js(int client_fd, const char *js) {
    respond_raw(client_fd, "HTTP/1.1 200 OK", "application/javascript; charset=utf-8", js, js ? strlen(js) : 0);
}

/* ---------- extrai Host do header (fallback localhost:porta) ---------- */
void get_host_header(const char *req, char *out_host, size_t out_size, int port) {
    const char *p = strstr(req, "\r\nHost:");
    if (!p) p = strstr(req, "\nHost:");
    if (!p) {
        snprintf(out_host, out_size, "localhost:%d", port);
        return;
    }
    p = strchr(p, ':');
    if (!p) {
        snprintf(out_host, out_size, "localhost:%d", port);
        return;
    }
    p++;
    while (*p == ' ') p++;
    const char *end = strchr(p, '\r');
    if (!end) end = strchr(p, '\n');
    if (!end) end = p + strlen(p);
    size_t len = end - p;
    if (len >= out_size) len = out_size - 1;
    memcpy(out_host, p, len);
    out_host[len] = '\0';
}

/* ---------- conteúdo estático (seu HTML/CSS/JS) ---------- */
/* style.css (a partir do seu design / imagem) */
const char STYLE_CSS[] =
"/* Reset básico */\n"
"*{margin:0;padding:0;box-sizing:border-box}\n"
"body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,'Helvetica Neue',Arial,sans-serif;background:#fff;color:#1a5c2e;min-height:100vh}\n"
"header{background:linear-gradient(135deg,#22c55e,#16a34a);padding:2rem 1.5rem;box-shadow:0 4px 6px -1px rgba(34,197,94,0.3)}\n"
"header h1{color:#fff;text-align:center;font-size:2rem;font-weight:700}\n"
"main{max-width:900px;margin:0 auto;padding:4rem 1.5rem;text-align:center}\n"
"h2{font-size:1.75rem;color:#1a5c2e;margin-bottom:2rem;font-weight:600}\n"
"button{background:linear-gradient(135deg,#22c55e,#16a34a);color:#fff;font-size:1.125rem;font-weight:600;padding:1rem 2rem;border:none;border-radius:0.75rem;cursor:pointer;box-shadow:0 10px 30px -10px rgba(34,197,94,0.3);transition:all .3s cubic-bezier(.4,0,.2,1)}\n"
"button:hover:not(:disabled){transform:scale(1.05);box-shadow:0 20px 40px -15px rgba(34,197,94,0.4)}\n"
"button:active:not(:disabled){transform:scale(.98)}\n"
"button:disabled{opacity:.6;cursor:not-allowed}\n"
".toast{position:fixed;bottom:2rem;right:2rem;padding:1rem 1.5rem;border-radius:.5rem;color:#fff;font-weight:500;opacity:0;transform:translateY(100px);transition:all .3s ease;z-index:1000;box-shadow:0 10px 25px rgba(0,0,0,.2)}\n"
".toast.show{opacity:1;transform:translateY(0)}\n"
".toast.success{background:linear-gradient(135deg,#22c55e,#16a34a)}\n"
".toast.error{background:linear-gradient(135deg,#ef4444,#dc2626)}\n"
"@media(max-width:768px){header h1{font-size:1.5rem}h2{font-size:1.5rem}button{font-size:1rem;padding:.875rem 1.5rem}.toast{bottom:1rem;right:1rem;left:1rem}}\n";

/* script.js */
const char SCRIPT_JS[] =
"function mostrarToast(mensagem,tipo){const toast=document.getElementById('toast');toast.textContent=mensagem;toast.className=`toast ${tipo}`;setTimeout(()=>{toast.classList.add('show')},100);setTimeout(()=>{toast.classList.remove('show')},3000)}\n"
"async function gerarEvento(){const botao=document.getElementById('btnGerar');botao.disabled=true;botao.textContent='Gerando...';try{const response=await fetch('/api/gerar-evento',{method:'GET'});if(response.ok){mostrarToast('Evento gerado com sucesso!','success')}else{mostrarToast('Erro ao gerar evento','error')}}catch(e){console.error('Erro:',e);mostrarToast('Erro ao conectar com o servidor','error')}finally{botao.disabled=false;botao.textContent='Gerar evento'}}\n";

/* home HTML (usa link para /style.css e /script.js) */
const char HOME_HTML_TEMPLATE[] =
"<!DOCTYPE html>\n"
"<html lang=\"pt-BR\">\n"
"<head>\n"
"  <meta charset=\"UTF-8\">\n"
"  <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n"
"  <title>Sistema de Controle de Acesso</title>\n"
"  <link rel=\"stylesheet\" href=\"/style.css\">\n</head>\n"
"<body>\n"
"  <header>\n"
"    <h1>Sistema de Controle de Acesso</h1>\n"
"  </header>\n"
"  <main>\n"
"    <h2>Gerar eventos de acesso na controladora</h2>\n"
"    <button id=\"btnGerar\" onclick=\"gerarEvento()\">Gerar evento</button>\n"
"  </main>\n"
"  <div id=\"toast\" class=\"toast\"></div>\n"
"  <script src=\"/script.js\"></script>\n"
"</body>\n"
"</html>\n";

/* ---------- handler de requisições ----------
Responsabilidade:   ler a requisição http do cliente, identificar o endpoint solicitado e enviar a resposta apropriada

*/
void handle_client(int client_fd, int port) {   //recebe, client_fd, descritor do socket retornado por accept
                                                // recebe port, usada para montar a requisição
    char buf[RECV_BUF];                         // aloca um buffer na pilha com o tamanho definido por RECV_BUF 8kb
    ssize_t r = recv(client_fd, buf, sizeof(buf) - 1, 0);   // chama recv  para ler até sizeof(buf) -1 bytes dos socket client_fd
                                                            // r recebe o número de bytes lidos
    if (r <= 0) {                                           // tratamento de erro
        close(client_fd);
        return;
    }
    buf[r] = '\0';                                          // escreve \0 para encerrar o arquivo

    char method[16] = {0}, path[512] = {0};                 // declara buffers para extrair, method com limite de 15 caracteres
                                                            // path com limite de 511 caracteres, inicializados com 0 por segurança
    sscanf(buf, "%15s %511s", method, path);                //extrai os 2 primeiros tokens da requisição  (METHOD PATH HTTP/1.1)
    printf("Requisição: %s %s\n", method, path);            // log no stdout com método e caminho, útil para diagnóstico

    /* /api/gerar-evento -> aciona o motor */
    if (strcmp(method, "GET") == 0 && strcmp(path, "/api/gerar-evento") == 0) {     //verifica se a requisição é GET /api/gerar-evento
                                                                                    //strcmp compara as strings , condição é método e rota.
        respond_text(client_fd, 200, "OK", "Movimento iniciado. Aguarde retorno.\n");   // envia resposta 200 OK
        close(client_fd);                           // fecha conexão
        // Executa o motor (bloqueante)
        girar_motor_noventa_graus(fd_gpio60, fd_gpio112); //chama a rotina que gira o motor.
        return;
    }

    /* /style.css
    rotina para o css
    */
    if (strcmp(method, "GET") == 0 && strcmp(path, "/style.css") == 0) {            //verifica GET /style.css
        respond_css(client_fd, STYLE_CSS);                                          //respond_css envia Content-Type: text/css e o conteúdo STYLE_CSS.
        close(client_fd);                                                           //fecha conexão    
        return;
    }

    /* /script.js
    rotina para o js*/
    if (strcmp(method, "GET") == 0 && strcmp(path, "/script.js") == 0) {        // verifica GET e script.js
        respond_js(client_fd, SCRIPT_JS);                                       // respond_js envia content-typ: text/css e o conteúdo SCRIPT_JS
        close(client_fd);                                                       // fecha conexão
        return;
    }

    /* / -> home page */
    if (strcmp(method, "GET") == 0 && strcmp(path, "/") == 0) {             //Rota para a página principal
        respond_html(client_fd, 200, "OK", HOME_HTML_TEMPLATE);             // GET / retorna o HTML contido em HOME_HTML_TEMPLATE.
                                                                            //Resposta HTML enviada via respond_html.

        close(client_fd);                                                  //Fecha socket e retorna.
        return;
    }

    /* não encontrado */
    respond_text(client_fd, 404, "Not Found", "Endpoint não encontrado\n");
    close(client_fd);
}

/* ---------- cleanup / signals / servidor ---------- */
void cleanup_and_exit(int code) {
    if (fd_gpio60 >= 0) close(fd_gpio60);
    if (fd_gpio112 >= 0) close(fd_gpio112);
    if (listen_fd >= 0) close(listen_fd);
    printf("Cleanup feito. Saindo.\n");
    exit(code);
}

void sigint_handler(int signo) {
    (void)signo;
    printf("SIGINT recebido. Finalizando...\n");
    cleanup_and_exit(0);
}

int start_server(int port) {
    struct sockaddr_in addr;                    //cria struct sockaddr
    listen_fd = socket(AF_INET, SOCK_STREAM, 0); // socket cria uma conexão na rede, retorna um int, -1 em caso de erro, AF_NET --> Domínio ipv4, SOCK_STREAM -->tipo de socket TCP, 0 --> protocolo padrão para TCP
    if (listen_fd < 0) {                         //Verifica se aconteceu algum erro.
        perror("socket");
        return -1;
    }

    int opt = 1;
    if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {    //setsockpot é utilizado para configurar opções do socket
                                                                                    //listen_fd --> descritor do socket
                                                                                    //SOL_SOCKET --> level, camada onde a opção será aplicada (SOL_SOCKET é genérica, não é específica TCP ou UDP)
                                                                                    //SO_REUSEADOR --> option_name, permite reutilizar a mesma porta rapidamente, mesmo se o socket anterior ainda estiver no estado TIME_WAIT, Sem isso, reiniciar o servidor rapidamente pode gerar erro bind: Address already in use.
                                                                                    //&opt --> option_value, ponteiro do valor que será utilizado, 1 ativa, 0 desativa.
                                                                                    //sizeof(opt) --> tamanhho do valor apontado
                                                                                    // retorna -1 em caso de erro
                                                                                    // SO_REUSEADOR não permite conexões simultâneas, uma precisa ser fechada para outr abrir
        perror("setsockopt");       
        close(listen_fd);                                                           // libera o socket
        return -1;
    }

    memset(&addr, 0, sizeof(addr));                                                 // memset preenche um bloco de memória. A função está garantindo que addr inicialize zerada.
    addr.sin_family = AF_INET;      // define a família para ipv4
    addr.sin_addr.s_addr = INADDR_ANY;  //define o endereço ip onde o socket vai escutar INADDR_ANY, indica todas as interfaces, aceita qualquer conexão.
    addr.sin_port = htons(port);        // a porta escutada será port

    if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {    // bind associa um socket a um endereço de IP e uma porta de rede
                                                                        // listen_fd --> descritor do socket (criado previamente)
                                                                        // addr --> ponteiro para a estrutura do endereço
                                                                        // sizeof(addr)--> tamanho da estrutura do endereço 
                                                                        // verifica se aconteceu algum erro
        perror("bind");
        close(listen_fd);
        return -1;
    }

    if (listen(listen_fd, BACKLOG) < 0) {                           // listen é utilizada para colocar socket em modo passivo, prepara o socket para receber conexões de clients
                                            // listen_fd --> descritor do socket
                                            // BACKLOG --> Número máximo de conexões na fila
                                            
        perror("listen");                   // trata o erro
        close(listen_fd);
        return -1;
    }

    printf("HTTP server escutando na porta %d\n", port);     // após tudo inicializado, printa que o server está escutando na porta inicializada.
    return listen_fd;                                       // retorna listen_fdS
}

/* ---------- main ---------- */
int main(int argc, char *argv[]) {
    int port = DEFAULT_PORT;                                            //Porta padrão 8081
    if (argc >= 2) {
        char *end;
        long p = strtol(argv[1], &end, 10);
        if (end == argv[1] || *end != '\0' || p < 1 || p > 65535) {
            fprintf(stderr, "Porta inválida: %s\n", argv[1]);
            return 1;
        }
        port = (int)p;
    }

    if (signal(SIGINT, sigint_handler) == SIG_ERR) {    //SIGNIT é executado para tratar a interrupção do ctrc+c
                                                        //signit_handler é uma função criada para fechar todos os processos que estão sendo executados e avisar que o SIGNIT foi acionado.
        perror("signal");                               //Verifica se aconteceu algum erro ao receber o sinal.
        return 1;
    }

    if (inicializar_gpios(&fd_gpio60, &fd_gpio112) != 0) {  //Chama as funções para inicialização das GPIOS
        fprintf(stderr, "Falha ao inicializar GPIOs. Execute como root.\n");    //verifica se aconteceu algum erro na inicalização.
        return 1;
    }

    if (start_server(port) < 0) {                       //Inicializa o servidor na porta inicializada ao executar o programa,  //Verifica se aconteceu algum erro
        cleanup_and_exit(1);                            //Se algum erro ocorrer, fecha todos os processos
    }

    /* loop principal (síncrono) */
    while (1) {                                          // Loop infinito que mantém o servidor rodando
        struct sockaddr_in client_addr;                  // Cria uma estrutura sockaddr_in para armazenar as variáveis do client que irá se conectar, como endereço ip e porta de origem
        socklen_t client_len = sizeof(client_addr);      // Cria uma variável (api) para armazenar o tamanho de client_addr
        int client = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);    // Accept -->  aguarda uma nova conexão de client, passando o socket principal do servidor, retorna o socket criado para aquele client
                                                                                        //(struct sockaddr*)&client_addr --> onde o kernel armazena as informações do client
                                                                                        // &client_len -->  tamanho da estrutura
        if (client < 0) {                               //Se client < 0 ocorreu um erro na função accept
            if (errno == EINTR) continue;
            perror("accept");
            break;
        }
        printf("Conexão de %s:%d\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port)); //Se nenhum erro ocorreu printa qual foi o client que estabeleceu conexão, ip e porta.
        handle_client(client, port);            // chama a conexão com o client.
    }

    cleanup_and_exit(0);
    return 0;
}

```
{% endtab %}
{% endtabs %}
