# Desenvolvimento da placa

Para a entrega final, foi desenvolvido o layout da placa de circuito impresso (PCB) com o objetivo de migrar o protótipo da protoboard para uma solução final com componentes soldados e disposição organizada na JIGA. O projeto foca em confiabilidade de montagem, facilidade de manutenção e conformidade com boas práticas de layout e fabricação.

### Esquemático

O esquemático foi confeccionado no KiCad e contempla todos os componentes e conexões descritos na documentação prévia [arquitetura-do-projeto.md](../arquitetura-do-projeto.md "mention"). Os símbolos e footprints foram verificados quanto à compatibilidade mecânica e elétrica, adotando nomenclatura clara e referências de pinos padronizadas para facilitar revisão e manutenção

&#x20;&#x20;

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### Layout da placa

&#x20;O layout da PCB foi realizado no KiCad seguindo boas práticas de roteamento e distribuição de sinais. As principais decisões de projeto incluíram:

* Planeamento de planos de terra contínuos para minimizar impedâncias e loops de corrente.
* Dimensionamento de trilhas conforme corrente prevista e regras de manufatura (larguras de trilha padronizadas em 1.2mm).
* Posicionamento de componentes para facilitar a montagem, testes e a dissipação térmica.

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

### Fabricação da placa

A manufatura da placa foi executada por meio de processo caseiro de transferência térmica e corrosão química aplicado a placa fenolite, seguido de perfuração e soldagem manual:

* Transferência do padrão: impressão do layout em papel carbono para transferência térmica ao cobre.
* Ataque químico: corrosão do cobre exposto utilizando solução apropriada para remoção do cobre não desejado, garantindo definição dos traços.
* Perfuração: furos mecânicos realizados com broca de precisão para terminais e fixações.
* Soldagem: soldagem manual com estanho, verificando-se qualidade das junções e possíveis pontes.

<div><figure><img src="../.gitbook/assets/WhatsApp Image 2025-12-18 at 17.12.10 (3).jpeg" alt=""><figcaption></figcaption></figure> <figure><img src="../.gitbook/assets/WhatsApp Image 2025-12-18 at 17.12.10 (1).jpeg" alt=""><figcaption></figcaption></figure></div>
