# Implementação

### Controle do motor de passo

Para o controle do motor de passo, foi utilizado o driver DRV8825. Conforme documentado na seção [materiais-utilizados](../materiais-utilizados/ "mention"), o controle foi implementado por meio do pino STEP, responsável pelo envio dos pulsos de comando ao driver.

Considerações de funcionamento do motor

* A alimentação do motor foi configurada para 24 V.
* Foi adicionado um capacitor de acoplamento com o objetivo de suprimir ruídos elétricos e garantir maior estabilidade no funcionamento do circuito.
* As conexões das bobinas foram realizadas utilizando os pinos A1, A2, B1 e B2 do driver. O motor utilizado é do tipo bipolar, portanto, uma bobina foi conectada entre os pinos A1–A2, e a outra entre B1–B2.

O driver foi configurado para operar no modo half-step, o que significa que o motor, originalmente com um passo de 1,8° (equivalente a 200 passos para uma volta completa em modo full-step), passa a requerer 400 passos para uma rotação de 360° em half-step.

No projeto, foi implementada uma rotação de 90° para realizar o movimento de acesso. Após um intervalo de 5 segundos, o pino DIR foi acionado para inverter o sentido de rotação — de horário para anti-horário — retornando o motor à posição inicial, com o mesmo número de passos (−90°).

Os pinos M0, M1 e M2 foram utilizados para configurar o modo de operação do driver. Para o modo half-step, foi definida a seguinte configuração:

* M0 = HIGH
* M1 = LOW
* M2 = LOW

Além disso, foram utilizados:

* O pino STEP, responsável pelo envio dos pulsos de controle;
* O pino DIR, responsável pela definição da direção de rotação (horário/anti-horário).

## Montagem física da Jiga

A montagem física da jiga foi executada com base no projeto previamente definido, utilizando cinco placas de MDF e cantoneiras metálicas para garantir a fixação estrutural entre as peças. Foram integrados ao conjunto uma controladora de acesso, um suporte de cartão produzido por impressão 3D e dois cartões, devidamente instalados no suporte.

Adicionalmente, foi incorporado um suporte dedicado ao motor de passo, permitindo sua fixação adequada à estrutura de madeira. O driver do motor foi instalado em uma protoboard localizada no interior da jiga, assegurando organização dos componentes eletrônicos.

As conexões do motor de passo, bem como a alimentação da controladora de acesso, foram roteadas por uma abertura realizada na madeira com o uso de serra-copo, garantindo passagem limpa e segura dos cabos.

A seguir, apresentam-se as imagens referentes ao processo de montagem e ao modelo final da jiga.

<div align="center"><figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.39.jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.40 (2).jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.41 (1).jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.41.jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.42 (1).jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-11-16 at 09.17.43.jpeg" alt=""><figcaption></figcaption></figure></div>

