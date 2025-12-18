# Resultados e conclusão

O objetivo principal do projeto — viabilizar, de forma remota, a geração de eventos para usuários sem acesso físico ao dispositivo — foi plenamente alcançado. A solução implementada permite ao desenvolvedor validar funcionalidades do código que dependem de leituras por cartão, assegurando a realização de testes remotos de forma reprodutível, controlada e organizada.

O objetivo secundário, que consistia em popular a base de dados com um grande volume de eventos para testar sua capacidade máxima (meta de 20 milhões de registros), não foi integralmente atingido nas condições atuais. Observou-se que o motor de geração contínua opera a uma taxa aproximada de 1 acesso a cada 2 segundos.

Nesse ritmo, considerando operação contínua (24/7), a produção estimada é de aproximadamente 43.200 acessos por dia. Mantida essa taxa, o tempo necessário para alcançar 20 milhões de eventos seria de cerca de 463 dias, o que se mostra inviável para testes de carga e stress.

* **Taxa observada:** 1 acesso a cada 2 s = 0,5 acessos/s
* **Acessos por dia:** 0,5 × 86.400 s = 43.200
* **Tempo para 20.000.000 registros:** 20.000.000 / 43.200 ≈ 463 dias

#### Soluções para resolver o teste de carga

Para reduzir o tempo total necessário e tornar viáveis os testes de carga, é possíve:

**1. Otimização do código**

* Melhorar o tempo de rotação do motor entre os acessos, diminuir de 2 s para 0,5s

**2. Ajuste da cadência de leitura / controladora**

* A controladora de acesso atualmente impõe um limite físico aproximado de 0,5 s por leitura de cartão.
* **Novo cenário:**\
  2 acessos/s × 86.400 s = 172.800 acessos/dia → tempo para 20 milhões ≈ 116 dias.

**3. Paralelização com múltiplas JIGAs/controladoras**

* Escalar horizontalmente o sistema por meio da criação de múltiplas instâncias (JIGAs) operando em paralelo.
* O tempo total necessário passa a ser aproximadamente 116 dias / N, assumindo que cada JIGA atinja 2 acessos/s.
* Exemplos:
  * 5 JIGAs em paralelo → \~23 dias
  * 10 JIGAs em paralelo → \~11,6 dias

#### Conclusão consolidada

* O requisito de acesso remoto e validação funcional foi atendido com sucesso.
* Para popular a base de dados com 20 milhões de eventos em um prazo aceitável, é necessária a combinação de otimizações de software, redução da latência por leitura (quando possível) e/ou aumento do paralelismo por meio da adição de JIGAs/controladoras.
