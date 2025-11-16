# Sensor de temperatura

## Sensor de temperatura para controle de temperatura do motor

### **Problema Identificado**

Durante o desenvolvimento do projeto, foi identificada a necessidade de adicionar uma ventoinha para o resfriamento da JIGA. No entanto, essa estratégia se mostrou pouco eficaz, pois a ventoinha não consegue reduzir adequadamente a temperatura do motor durante o funcionamento.

Observou-se que o principal ponto de aquecimento é o motor; os demais componentes da JIGA mantêm temperaturas estáveis mesmo durante operação contínua (24/7).

O teste de carga realizado para gerar 20 milhões de eventos evidenciou que a ventoinha não é suficiente para resfriar o motor em uso.

### **Análise e Solução Proposta**

Para evitar o superaquecimento do motor, foi definido que a JIGA deve operar em uma sala com temperatura controlada por ar-condicionado ajustado para 16 °C.

Caso ocorra alguma falha no ar-condicionado, o sistema deverá identificar o problema por meio de um sensor de temperatura instalado no motor e interromper automaticamente o funcionamento até que o ar-condicionado seja reparado.

{% hint style="info" %}
O motor NEMA 17 tem uma faixa de operação de temperatura de -20ºC até 40ºC
{% endhint %}

Dessa forma, o escopo do projeto foi atualizado para incluir:

* Instalação de um sensor de temperatura no motor modelo DSB1820, responsável por monitorar a temperatura durante o funcionamento.
* Caso o sensor detecte temperatura superior a 30 °C, o sistema deve interromper o funcionamento do motor e enviar um aviso automático ao usuário.
* Implementação de um menu na interface web para exibir, em tempo real, a temperatura do motor.

Observação: a ponta do motor é acoplada a uma estrutura 3D responsável pela fixação do cartão, utilizando cola quente. Por esse motivo, temperaturas acima de 30 °C podem comprometer a fixação, justificando o limite térmico definido.
