---
description: Arquitetura utilizada para o projeto
---

# Arquitetura do projeto

{% tabs %}
{% tab title="BeagleBone" %}
\[Cliente Web (PC/Celular)]\
⇅ Ethernet/Wi-Fi\
\[BeagleBone]\
├── Servidor Web (Apache/Nginx + Backend Python/C++)\
├── Interface com GPIOs (via PRU ou libgpiod)\
├── Ventoinha\
└── Driver de Motor (ex: DRV8824)\
↓\
\[Motor de Passo]



**Esquema de ligação entre BeagleBone e driver de motor**

| BeagleBone | DRV8824 |
| ---------- | ------- |
| GPIO1\_6   | STEP    |
| GPIO1\_7   | DIR     |
| GPIO1\_2   | ENABLE  |

**Esquema de ligação entre BeagleBone e ventoinha**

+12V (Fonte externa)\
│\
├────────────┐\
│                                   │\
\[Ventoinha] -  \[Diodo 1N4007]\
│                                  │\
│                               ┌┴┐\
└────┬──────┘  └──┐\
&#x20;              │                              │\
\[Dreno do MOSFET]          │\
&#x20;              │                              │\
\[Fonte do MOSFET] ───┘\
&#x20;              │\
&#x20;            GND (comum com BeagleBone)



BeagleBone GPIO\_Z ──┬─\[Resistor 1kΩ]─┬─> \[Gate do MOSFET]\
&#x20;                                            │                               │\
&#x20;                                         GND                         GND
{% endtab %}
{% endtabs %}
