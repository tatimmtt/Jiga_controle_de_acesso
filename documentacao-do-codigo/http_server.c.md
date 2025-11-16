# http\_server.c

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file http_server.c
 * @brief Implementação do servidor HTTP e handlers
 */

#include "http_server.h"
#include "web_content.h"
#include "gpio.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <ctype.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

/* ---------- Funções auxiliares de resposta HTTP ---------- */

void respond_raw(int client_fd, const char *status_line, 
                 const char *content_type, const char *body, size_t body_len) {
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

void respond_text(int client_fd, int status_code, 
                  const char *status_text, const char *body) {
    char status_line[64];
    snprintf(status_line, sizeof(status_line), "HTTP/1.1 %d %s", 
             status_code, status_text);
    respond_raw(client_fd, status_line, "text/plain; charset=utf-8", 
                body ? body : "", body ? strlen(body) : 0);
}

void respond_html(int client_fd, int status_code, 
                  const char *status_text, const char *html) {
    char status_line[64];
    snprintf(status_line, sizeof(status_line), "HTTP/1.1 %d %s", 
             status_code, status_text);
    respond_raw(client_fd, status_line, "text/html; charset=utf-8", 
                html ? html : "", html ? strlen(html) : 0);
}

void respond_json(int client_fd, int status_code, const char *json) {
    char status_line[64];
    snprintf(status_line, sizeof(status_line), "HTTP/1.1 %d %s", 
             status_code, (status_code == 200) ? "OK" : "Error");
    respond_raw(client_fd, status_line, "application/json; charset=utf-8", 
                json ? json : "", json ? strlen(json) : 0);
}

void respond_404(int client_fd) {
    respond_text(client_fd, 404, "Not Found", "Endpoint não encontrado\n");
}

/* ---------- Parser de formulário ---------- */

int parse_form_temperature(const char *body, int body_len) {
    // Procura por "temperature=" no corpo da requisição
    const char *p = strstr(body, "temperature=");
    if (!p) return -1;

    p += strlen("temperature=");

    // Extrai os dígitos (incluindo sinal negativo)
    char tmp[32];
    int i = 0;
    while ((p < body + body_len) && i < (int)(sizeof(tmp) - 1) && 
           (isdigit((unsigned char)*p) || *p == '-')) {
        tmp[i++] = *p++;
    }
    tmp[i] = '\0';

    if (i == 0) return -1;
    return atoi(tmp);
}

/* ---------- Handlers de rotas ---------- */

/**
 * @brief Handler para GET /api/status
 * 
 * Retorna JSON com:
 * - temperature: temperatura atual
 * - status: "Normal" ou "Anormal" (superaquecido)
 * - motor_running: true/false
 */
static void handle_api_status(int client_fd, motor_state_t *state) {
    int temp = motor_get_temperature(state);
    int overheat = motor_is_overheated(state);
    int running = motor_is_running(state);

    char json[256];
    snprintf(json, sizeof(json), 
             "{\"temperature\":%d,\"status\":\"%s\",\"motor_running\":%s}",
             temp, 
             overheat ? "Anormal" : "Normal",
             running ? "true" : "false");

    respond_json(client_fd, 200, json);
    printf("  ← API Status: temp=%d°C, status=%s, motor=%s\n", 
           temp, overheat ? "Anormal" : "Normal", running ? "Rodando" : "Parado");
}

/**
 * @brief Handler para POST /api/temperature
 * 
 * Recebe parâmetro "temperature" via form-urlencoded
 * e atualiza a temperatura do sistema.
 */
static void handle_api_temperature(int client_fd, motor_state_t *state, 
                                   const char *request, ssize_t req_len) {
    // Localizar Content-Length
    const char *cl = strstr(request, "\r\nContent-Length:");
    int content_length = 0;

    if (!cl) cl = strstr(request, "\nContent-Length:");
    if (cl) {
        cl = strchr(cl, ':');
        if (cl) {
            cl++;
            while (*cl == ' ') cl++;
            content_length = atoi(cl);
        }
    }

    // Localizar início do corpo
    const char *body = strstr(request, "\r\n\r\n");
    if (!body) body = strstr(request, "\n\n");
    if (body) body += (body[1] == '\r') ? 4 : 2;
    else body = request + strlen(request);

    int body_len = (int)(req_len - (body - request));

    // Parser a temperatura
    int temp = parse_form_temperature(body, body_len);
    if (temp == -1 || temp < -50 || temp > 200) {
        respond_text(client_fd, 400, "Bad Request", 
                    "Formato inválido. Use temperature=N (-50 a 200)\n");
        printf("  ✗ Temperatura inválida recebida\n");
        return;
    }

    // Atualizar temperatura
    motor_set_temperature(state, temp);
    respond_text(client_fd, 200, "OK", "Temperatura atualizada\n");
    printf("  ✓ Temperatura atualizada: %d°C\n", temp);
}

/**
 * @brief Handler para POST /api/rotate
 * 
 * Inicia o movimento de rotação do motor (90° ida e volta).
 */
static void handle_api_rotate(int client_fd, motor_state_t *state) {
    int result = motor_start_rotation(state);

    if (result == 0) {
        respond_text(client_fd, 200, "OK", 
                    "Movimento iniciado em background.\n");
    } else {
        // Verifica o motivo da falha
        if (motor_is_overheated(state)) {
            respond_text(client_fd, 409, "Conflict", 
                        "Motor superaquecido. Comando não executado.\n");
        } else if (motor_is_running(state)) {
            respond_text(client_fd, 409, "Conflict", 
                        "Motor já está em movimento.\n");
        } else {
            respond_text(client_fd, 500, "Internal Server Error", 
                        "Falha ao iniciar o motor\n");
        }
    }
}

/**
 * @brief Handler para POST /api/stop
 * 
 * Solicita parada imediata do motor.
 */
static void handle_api_stop(int client_fd, motor_state_t *state) {
    motor_stop(state);

    if (motor_is_running(state)) {
        respond_text(client_fd, 200, "OK", "Solicitada parada do motor.\n");
    } else {
        respond_text(client_fd, 200, "OK", "Motor já parado.\n");
    }
}

/* ---------- Roteador de requisições ---------- */

void handle_client(int client_fd, motor_state_t *state, int port) {
    char buf[RECV_BUF];
    ssize_t r = recv(client_fd, buf, sizeof(buf) - 1, 0);

    if (r <= 0) {
        close(client_fd);
        return;
    }
    buf[r] = '\0';

    // Parsear método e path
    char method[16] = {0}, path[512] = {0};
    sscanf(buf, "%15s %511s", method, path);

    printf("→ Requisição: %s %s\n", method, path);

    /* ----- ROTAS ESTÁTICAS ----- */

    // GET / -> Página principal
    if (strcmp(method, "GET") == 0 && strcmp(path, "/") == 0) {
        respond_html(client_fd, 200, "OK", get_home_html());
        close(client_fd);
        return;
    }

    // GET /style.css
    if (strcmp(method, "GET") == 0 && strcmp(path, "/style.css") == 0) {
        respond_raw(client_fd, "HTTP/1.1 200 OK", "text/css; charset=utf-8", 
                   get_style_css(), strlen(get_style_css()));
        close(client_fd);
        return;
    }

    // GET /script.js
    if (strcmp(method, "GET") == 0 && strcmp(path, "/script.js") == 0) {
        respond_raw(client_fd, "HTTP/1.1 200 OK", 
                   "application/javascript; charset=utf-8",
                   get_script_js(), strlen(get_script_js()));
        close(client_fd);
        return;
    }

    /* ----- API ENDPOINTS ----- */

    // GET /api/status
    if (strcmp(method, "GET") == 0 && strcmp(path, "/api/status") == 0) {
        handle_api_status(client_fd, state);
        close(client_fd);
        return;
    }

    // POST /api/temperature
    if (strcmp(method, "POST") == 0 && strcmp(path, "/api/temperature") == 0) {
        handle_api_temperature(client_fd, state, buf, r);
        close(client_fd);
        return;
    }

    // POST /api/rotate
    if (strcmp(method, "POST") == 0 && strcmp(path, "/api/rotate") == 0) {
        handle_api_rotate(client_fd, state);
        close(client_fd);
        return;
    }

    // POST /api/stop
    if (strcmp(method, "POST") == 0 && strcmp(path, "/api/stop") == 0) {
        handle_api_stop(client_fd, state);
        close(client_fd);
        return;
    }

    // Rota não encontrada
    respond_404(client_fd);
    close(client_fd);
}

/* ---------- Funções do servidor ---------- */

int http_server_init(http_server_t *server, int port, motor_state_t *motor_state) {
    struct sockaddr_in addr;

    server->port = port;
    server->motor_state = motor_state;

    // Criar socket
    server->listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server->listen_fd < 0) {
        perror("socket");
        return -1;
    }

    // Permitir reutilização rápida da porta
    int opt = 1;
    if (setsockopt(server->listen_fd, SOL_SOCKET, SO_REUSEADDR, 
                   &opt, sizeof(opt)) < 0) {
        perror("setsockopt");
        close(server->listen_fd);
        return -1;
    }

    // Configurar endereço
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    // Bind
    if (bind(server->listen_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind");
        close(server->listen_fd);
        return -1;
    }

    // Listen
    if (listen(server->listen_fd, BACKLOG) < 0) {
        perror("listen");
        close(server->listen_fd);
        return -1;
    }

    printf("✓ Servidor HTTP escutando em http://0.0.0.0:%d\n", port);
    printf("  Acesse pelo navegador: http://localhost:%d\n\n", port);

    return 0;
}

void http_server_run(http_server_t *server) {
    printf("=== Servidor pronto para aceitar conexões ===\n\n");

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        int client = accept(server->listen_fd, 
                           (struct sockaddr*)&client_addr, &client_len);

        if (client < 0) {
            if (errno == EINTR) continue;  // Interrompido por sinal
            perror("accept");
            break;
        }

        printf("📡 Conexão de %s:%d\n", 
               inet_ntoa(client_addr.sin_addr), 
               ntohs(client_addr.sin_port));

        handle_client(client, server->motor_state, server->port);
    }
}

void http_server_close(http_server_t *server) {
    if (server->listen_fd >= 0) {
        close(server->listen_fd);
        server->listen_fd = -1;
    }
}


```
{% endtab %}
{% endtabs %}
