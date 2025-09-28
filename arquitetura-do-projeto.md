---
description: Arquitetura utilizada para o projeto
---

# Arquitetura do projeto

A arquitetura proposta para o projeto utiliza um microcomputador embarcado BeagleBone como unidade central de processamento e controle dos sinais GPIO (General Purpose Input/Output) dos equipamentos envolvidos.A BeagleBone será o componente principal, responsável por enviar comandos aos pinos GPIO que estarão integrados ao driver de motor, o qual controlará o motor de passo. Abaixo estão ilustradas as especificações dos pinos disponíveis na BeagleBone:







A arquitetura do projeto consistirá em utilizar um microcomputador embarcado a Beaglebone para processamento e comandos GPIOS nos equipamentos utilizados.

A Beaglebone é o componente principar e será responsável por comandar os GPIOS que serão integrados ao drivre de motor que controlorá o motor de passo, abaixo está a descrição dos pinos utilizáveis da Beaglebone

<figure><img src=".gitbook/assets/black_hardware_details.png" alt="" width="563"><figcaption></figcaption></figure>

<p align="center">Imagem 1 - Detalhes do hardware</p>

<p align="center"></p>

<figure><img src=".gitbook/assets/cape-headers.png" alt="" width="563"><figcaption></figcaption></figure>

<p align="center">Imagem 2 - Pinos I/O</p>

<figure><img src=".gitbook/assets/cape-headers-digital (1).png" alt="" width="563"><figcaption></figcaption></figure>

<p align="center">Imagem 3 - Pinos I/O digitais</p>

<p align="center"></p>

**Recursos Utilizados**

Para este projeto, serão utilizados os seguintes pinos GPIO:

* **3 pinos GPIO** dedicados ao driver de motor de passo
* **1 pino GPIO** destinado ao controle da ventoinha

Além do controle via GPIO, a BeagleBone também será responsável por hospedar um **servidor web** para acesso remoto e gerenciamento do motor. Esse servidor será disponibilizado em um **endereço IP privado (IP x)**, acessível apenas por meio de conexão à **VPN** da rede. Dessa forma, o desenvolvedor poderá operar remotamente o sistema com segurança, sem expor o serviço a acessos externos não autorizados.
