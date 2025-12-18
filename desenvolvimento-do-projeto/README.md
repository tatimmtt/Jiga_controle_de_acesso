---
description: Descrição das etapas de desenvolvimento do projeto
---

# Desenvolvimento do projeto



## Comando do motor através da GPIO

Para realizar o controle do motor na BeagleBone, foi desenvolvido um código em linguagem C destinado ao gerenciamento de dois pinos GPIO: um associado ao pino STEP e outro ao pino DIR.

O código está documentado em:

* [gpio.c.md](../documentacao-do-codigo/gpio.c.md "mention")
* [gpio.h.md](../documentacao-do-codigo/gpio.h.md "mention")
* [motor.c.md](../documentacao-do-codigo/motor.c.md "mention")
* [motor.h.md](../documentacao-do-codigo/motor.h.md "mention")



## Interface web para controle remoto do motor

Com o objetivo de possibilitar o controle remoto do motor e, consequentemente, gerar eventos na controladora de acesso, foi desenvolvida uma interface web baseada em um servidor HTTP.

O servidor implementa APIs REST responsáveis por acionar a função de rotação do motor. Ao ser invocada, essa função executa o movimento do motor de passo e, simultaneamente, envia o evento de acionamento ao dispositivo de controle de acesso, atendendo assim ao propósito central do projeto.

O código está documentado em:

* [http\_server.c.md](../documentacao-do-codigo/http_server.c.md "mention")
* [http\_server.h.md](../documentacao-do-codigo/http_server.h.md "mention")
* [web\_content.c.md](../documentacao-do-codigo/web_content.c.md "mention")
* [web\_content.h.md](../documentacao-do-codigo/web_content.h.md "mention")

## Sensor de temperatura para controle de temperatura do motor

Para monitorar a temperatura do motor, que é o componente mais crítico, foi adicionado ao projeto o sensor de temperatura GY-906 que é reponsável por monitorar a temperaturar de minuto em minuto e parar o sistema se acontecer um aquecimento excessivo.

O código está disponível em:

* [temperature\_sensor.c.md](../documentacao-do-codigo/temperature_sensor.c.md "mention")
* [temperature\_sensor.h.md](../documentacao-do-codigo/temperature_sensor.h.md "mention")



