---
description: Arquitetura utilizada para o projeto
---

# Arquitetura do projeto

A arquitetura do projeto prevê o uso de uma BeagleBone como unidade central de processamento. Nela estarão concentrados tanto o servidor HTTP quanto o processamento dos comandos GPIO responsáveis pelo controle do driver de motor.

O driver DRV8825 será responsável pela integração com o motor NEMA 17, um motor bipolar configurado para operar no modo half step (M0 = HIGH, M1 = LOW, M2 = LOW). O driver utilizará um pino GPIO para receber os pulsos de controle do motor (pino STEP), que determinam os passos de movimentação, e um pino DIR, responsável por definir o sentido de rotação.

O motor estará acoplado a um suporte de cartão de acesso, de forma que sua movimentação permitirá o acionamento físico do mecanismo de liberação no dispositivo de controle de acesso.

Além disso, será implementado um servidor web HTTP, que possibilitará o envio de comandos via API por meio de uma interface web cliente. Essa interface permitirá gerar requisições ao servidor para acionar o pino STEP, promovendo a rotação do motor e, consequentemente, a liberação do acesso.
