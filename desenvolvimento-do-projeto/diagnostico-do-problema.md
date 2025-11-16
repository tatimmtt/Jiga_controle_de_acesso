# Diagnóstico do problema

## Diagnóstico do problema

Nesta seção são apresentados e detalhados os principais problemas observados.

### Gerar evento de acesso de forma remota

A equipe de desenvolvimento está responsável pela criação de um módulo do software “P001”, voltado ao controle de acesso. Para os profissionais que atuam nessa área, é essencial validar os eventos de acesso, os quais ocorrem da seguinte forma: um dispositivo de controle de acesso é conectado ao software, e todo evento gerado nesse dispositivo é enviado e processado pelo sistema.

Como essa integração precisa ser constantemente validada — tanto por desenvolvedores quanto por analistas de qualidade (QAs), é imprescindível que os profissionais que trabalham de forma remota também consigam gerar eventos de acesso no dispositivo para validá-los diretamente no software.

### Testes de performance de banco de dados

Outro problema identificado refere-se aos testes de performance do banco de dados, que são essenciais durante o processo de validação das novas versões do software.

O sistema foi projetado para armazenar até 20 milhões de eventos de acesso no banco de dados. No entanto, a geração manual desses eventos é impraticável e ineficiente, uma vez que cada evento requer a aproximação de um cartão com credencial a uma controladora de acesso, para então o evento ser enviado ao software e armazenado.

Considerando que cada evento é gerado em média a cada um segundo, seriam necessárias aproximadamente 234 dias de operação contínua para registrar manualmente apenas 20 milhões eventos, tornando inviável a execução desse processo em larga escala.

Essa limitação compromete a realização de testes de desempenho e estresse no banco de dados, dificultando a análise da eficiência, escalabilidade e estabilidade do sistema sob condições de uso real.
