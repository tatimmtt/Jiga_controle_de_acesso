# http\_server.h

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file http_server.h
 * @brief Interface do servidor HTTP e handlers de API
 * 
 * Este módulo implementa um servidor HTTP simples que serve:
 * - Arquivos estáticos (HTML, CSS, JS)
 * - API REST para controle do motor
 */

#ifndef HTTP_SERVER_H
#define HTTP_SERVER_H

#include "motor.h"

/* Configurações do servidor */
#define DEFAULT_PORT 8081
#define BACKLOG 8           // Fila de conexões pendentes
#define RECV_BUF 8192       // Tamanho do buffer de recepção

/**
 * @brief Contexto do servidor HTTP
 */
typedef struct {
    int listen_fd;              // Socket de escuta
    int port;                   // Porta do servidor
    motor_state_t *motor_state; // Referência ao estado do motor
} http_server_t;

/**
 * @brief Inicializa o servidor HTTP
 * 
 * Cria o socket, faz bind e começa a escutar na porta especificada.
 * 
 * @param server Ponteiro para a estrutura do servidor
 * @param port Porta para escutar (ex: 8081)
 * @param motor_state Ponteiro para o estado do motor
 * @return 0 em sucesso, -1 em erro
 */
int http_server_init(http_server_t *server, int port, motor_state_t *motor_state);

/**
 * @brief Loop principal do servidor
 * 
 * Aceita conexões e processa requisições HTTP.
 * Bloqueia até que o servidor seja encerrado.
 * 
 * @param server Ponteiro para a estrutura do servidor
 */
void http_server_run(http_server_t *server);

/**
 * @brief Fecha o servidor e libera recursos
 * 
 * @param server Ponteiro para a estrutura do servidor
 */
void http_server_close(http_server_t *server);

/**
 * @brief Processa uma requisição HTTP de um cliente
 * 
 * Roteia a requisição para o handler apropriado baseado no método e path.
 * 
 * @param client_fd Socket do cliente
 * @param motor_state Estado do motor
 * @param port Porta do servidor (para logs)
 */
void handle_client(int client_fd, motor_state_t *motor_state, int port);

/* ---------- Funções auxiliares de resposta HTTP ---------- */

/**
 * @brief Envia resposta HTTP genérica
 * 
 * @param client_fd Socket do cliente
 * @param status_line Linha de status (ex: "HTTP/1.1 200 OK")
 * @param content_type Tipo de conteúdo (ex: "text/html")
 * @param body Corpo da resposta
 * @param body_len Tamanho do corpo
 */
void respond_raw(int client_fd, const char *status_line, 
                 const char *content_type, const char *body, size_t body_len);

/**
 * @brief Envia resposta de texto simples
 * 
 * @param client_fd Socket do cliente
 * @param status_code Código HTTP (ex: 200, 404)
 * @param status_text Texto do status (ex: "OK", "Not Found")
 * @param body Corpo da resposta
 */
void respond_text(int client_fd, int status_code, 
                  const char *status_text, const char *body);

/**
 * @brief Envia resposta HTML
 * 
 * @param client_fd Socket do cliente
 * @param status_code Código HTTP
 * @param status_text Texto do status
 * @param html Conteúdo HTML
 */
void respond_html(int client_fd, int status_code, 
                  const char *status_text, const char *html);

/**
 * @brief Envia resposta JSON
 * 
 * @param client_fd Socket do cliente
 * @param status_code Código HTTP
 * @param json String JSON
 */
void respond_json(int client_fd, int status_code, const char *json);

/**
 * @brief Envia resposta 404 Not Found
 * 
 * @param client_fd Socket do cliente
 */
void respond_404(int client_fd);

/**
 * @brief Extrai valor de um parâmetro form-urlencoded
 * 
 * @param body Corpo da requisição POST
 * @param body_len Tamanho do corpo
 * @return Valor da temperatura ou -1 se inválido
 */
int parse_form_temperature(const char *body, int body_len);

#endif /* HTTP_SERVER_H */

```
{% endtab %}
{% endtabs %}
