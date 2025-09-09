---
icon: terminal
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: false
  metadata:
    visible: true
---

# JIGA DE CONTROLE DE ACESSO

## Resumo

Este projeto tem como objetivo otimizar os processos de teste e desenvolvimento do software _Defense IA_, por meio da identificação e implementação de soluções para desafios específicos enfrentados pela equipe, especialmente no módulo de controle de acesso.

## Introdução

### Definição do Problema

O módulo de controle de acesso do _Defense IA_ integra-se a dispositivos físicos, como controladoras de acesso. Um dos principais desafios enfrentados pela equipe de desenvolvimento é a limitação de testes físicos, especialmente para colaboradores em regime remoto, que não conseguem realizar a leitura de cartões nos dispositivos para gerar eventos no sistema. Além disso, há a necessidade de realizar testes de carga, simulando acessos em massa — com volumes superiores a 1 milhão de registros — para avaliar a performance e a capacidade do banco de dados em armazenar e processar eventos de acesso. Este projeto concentra-se na resolução de dois problemas principais:

* Simulação de eventos de acesso por meio da ativação remota da passagem de cartões nos dispositivos,  através de uma interface web, permitindo a execução de testes mesmo em ambientes fora do laboratório, como no trabalho remoto.
* Testes de performance voltados para a validação da escalabilidade e da robustez do banco de dados, por meio da simulação de grandes volumes de eventos, superiores a 1 milhão de registros, com o objetivo de avaliar a capacidade de armazenamento, processamento e resposta do sistema sob carga intensa.



### Motivação do Projeto

A motivação central é o desenvolvimento de uma JIGA de testes que possa ser integrada à esteira de desenvolvimento do _Defense IA_. Essa JIGA utilizará dispositivos eletrônicos como microcontroladores e motores de passo para simular fisicamente a passagem de cartões nos leitores, gerando eventos reais no sistema.O projeto representa uma aplicação prática de conhecimentos acadêmicos e técnicos, com potencial de impacto direto na eficiência dos testes e na qualidade do produto final. A integração entre hardware e software permitirá a automação de testes e a ampliação da cobertura de cenários, contribuindo para a evolução contínua da solução.

## Objetivos

### Objetivo Geral

Desenvolver uma JIGA de testes de controle de acesso.

### Objetivos específicos

1. Analisar dados existentes para compreender as principais dificuldades.
2. Identificar padrões e tendências que informem as decisões estratégicas.
3. Desenvolver e implementar estratégias que melhorem os processos visados.
4. Medir e avaliar o impacto das soluções implementadas nos resultados organizacionais.

## Metodologia

Para alcançar os objetivos especificados, será adotada uma metodologia iterativa e ágil, promovendo ciclos curtos de desenvolvimento e feedback contínuo. A abordagem permitirá ajustar estratégias rapidamente e garantir que as soluções atendam às necessidades práticas da equipe e da organização.

#### Etapas do processo

1. **Descoberta e Planejamento**: Envolver todas as partes interessadas para prioritizar tarefas e definir metas claras.
2. **Desenvolvimento e Implementação**: Prototipar e desenvolver em iterações, com avaliação regular do progresso.
3. **Testes e Avaliações**: Verificar a eficiência das soluções por meio de testes rigorosos.
4. **Otimização Contínua**: Ajustar as estratégias com base nos dados coletados e no feedback recebido.

Essa estrutura permitirá uma execução coordenada e eficiente, garantindo resultados alcançáveis e mensuráveis.

## Arquitetura do projeto

A arquitetura do projeto é composta por diversos componentes interligados que facilitam a operação e o desenvolvimento contínuo do sistema. A seguir, detalhamos os principais elementos:

1. **Frontend**: Desenvolvido com frameworks modernos, o frontend é responsável pela interface de usuário, garantindo uma experiência fluida e responsiva.
2. **Backend**: Constituído por uma robusta API que gerencia lógica de negócios e toda integração com o banco de dados.
3. **Banco de Dados**: Utiliza um sistema de gerenciamento de banco de dados relacional para armazenar e organizar dados de forma eficiente.
4. **Infraestrutura de Deploy**: Automação através de pipelines que asseguram a integração contínua e o delivery das atualizações.

Esses componentes são integrados de maneira a proporcionar uma solução escalável e de fácil manutenção.

## Materiais e equipamentos utilizados

A lista de materiais e equipamentos empregados no desenvolvimento e operação do sistema inclui servidores, dispositivos de rede, ferramentas de monitoramento e computadores para desenvolvimento. Esses recursos garantem a eficiência, segurança e escalabilidade da infraestrutura de tecnologia.
