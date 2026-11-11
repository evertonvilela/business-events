# FrioTrack: recepção e consulta de eventos de dispositivos

> Estudo de caso fictício para discutir arquitetura AWS, serverless e segurança
> aplicada. Empresas, volumes e metas são premissas do exercício. Este documento
> descreve o problema e os requisitos antes da escolha de serviços.

## 1. Contexto de negócio

A FrioTrack oferece uma plataforma para redes de supermercados acompanharem a
temperatura e o funcionamento de seus equipamentos de refrigeração. Cada loja
possui sensores instalados em câmaras frias, freezers e geladeiras.

Os dispositivos enviam medições pela internet, mas a conexão das lojas pode ficar
indisponível por várias horas. Quando a conexão retorna, medições antigas e atuais
podem chegar ao sistema ao mesmo tempo.

A plataforma atende a duas necessidades:

- **Acompanhamento operacional:** verificar as últimas condições conhecidas dos
  equipamentos e identificar dispositivos sem comunicação recente.
- **Investigação:** consultar o histórico após problemas como perda de mercadoria
  ou falha de refrigeração.

O objetivo deste caso é projetar a recepção, a preservação e a consulta dos eventos,
com isolamento entre clientes e comportamento definido diante de falhas.

## 2. Dispositivos e eventos

Cada dispositivo monitora um equipamento e envia uma medição a cada **60 segundos**.
O tamanho médio considerado é **1 KB por evento**, incluindo seus metadados.

| Campo | Significado |
| --- | --- |
| Identificador do evento | Gerado pelo dispositivo e preservado nos reenvios. |
| Identificador do dispositivo | Identifica a origem da medição. |
| Número de sequência | Ajuda a perceber lacunas e mudanças na ordem dos eventos. |
| Instante da medição | Horário informado pelo dispositivo. |
| Temperatura | Medição em graus Celsius. |
| Estado da porta | Aberta ou fechada. |
| Nível da bateria | Percentual restante. |
| Versão do firmware | Identifica o software em execução no dispositivo. |
| Versão do formato | Identifica a estrutura do evento enviado. |

O sistema também registra o instante em que recebeu cada evento. O horário de
recebimento e o horário da medição têm significados diferentes e precisam ser
preservados.

### Condições de operação

- Um dispositivo pode ficar **até 24 horas sem conexão**, armazenando medições
  localmente para envio posterior.
- Ao reconectar, pode reenviar eventos já entregues porque perdeu a confirmação
  de recebimento.
- Os eventos podem chegar fora de ordem.
- O relógio do dispositivo pode estar incorreto. Seu horário não pode ser tratado
  como fonte confiável para todas as decisões.
- No laboratório, os dispositivos podem ser representados por um simulador.

## 3. Usuários e consultas

### Perfis de acesso

| Perfil | Necessidade e limite de acesso |
| --- | --- |
| Operador da loja | Consulta os equipamentos da própria unidade e identifica quais enviaram dados recentemente. |
| Analista da rede | Consulta as unidades de sua empresa e investiga o histórico de dispositivos e lojas. |
| Suporte da plataforma | Investiga problemas de entrega e processamento; o acesso aos dados dos clientes deve ser restrito e rastreável. |

### Consultas do primeiro escopo

1. Último estado conhecido dos dispositivos de uma unidade.
2. Histórico de um dispositivo em um intervalo de até sete dias.
3. Eventos de uma unidade em um intervalo de até 24 horas, com paginação.
4. Detalhes de um evento específico, respeitando as permissões do solicitante.

Uma medição antiga recebida após a reconexão não deve substituir uma medição mais
recente no estado apresentado ao usuário. A definição de “mais recente”, diante
de relógios incorretos e números de sequência, precisa ser explicitada na solução.

Os usuários precisam distinguir o último valor conhecido de uma indicação de que o
equipamento continua se comunicando. Exibir uma temperatura sem indicar que a
medição está desatualizada pode levar a uma interpretação incorreta.

## 4. Escala e perfil de carga

| Dimensão | Premissa |
| --- | ---: |
| Empresas clientes | 50 |
| Unidades monitoradas | 500 |
| Dispositivos por unidade, em média | 20 |
| Total de dispositivos | 10.000 |
| Frequência habitual | 1 evento por dispositivo por minuto |
| Taxa média aproximada | 167 eventos por segundo |
| Volume diário | 14,4 milhões de eventos |
| Tamanho médio do evento | 1 KB |
| Dados recebidos por dia | Aproximadamente 14,4 GB |
| Pico a suportar | 2.500 eventos por segundo por até 20 minutos |
| Retenção disponível para consulta | 90 dias |
| Usuários consultando simultaneamente | Até 100 |

As estimativas de volume usam unidades decimais e não incluem índices, réplicas,
backups ou outros custos de armazenamento. O volume diário considera a frequência
habitual; reenvios geram tráfego adicional.

O pico representa a reconexão de muitas lojas após uma interrupção regional.
Durante esse período, o sistema recebe o acúmulo de eventos e as novas medições.

A carga não é distribuída uniformemente: uma única empresa pode concentrar **30%
dos dispositivos**.

## 5. Riscos e prioridades do negócio

A ordem de preocupação é:

1. Expor dados de uma empresa a outra.
2. Confirmar o recebimento de um evento e depois perdê-lo.
3. Mostrar um equipamento como atualizado quando ele está sem comunicação.
4. Permitir que um dispositivo comprometido prejudique os demais.
5. Exibir medições duplicadas ou apresentar eventos antigos como estado atual.

Atrasos curtos são aceitáveis. Durante incidentes, a interface deve deixar claro
quando os dados estão atrasados.

Os eventos não contêm nomes de pessoas. Localização das unidades, identificadores
e padrões de funcionamento são considerados informações comerciais restritas.

## 6. Requisitos iniciais

Os requisitos abaixo compõem a primeira versão do contrato do sistema. Os pontos
em aberto da seção seguinte devem ser resolvidos antes de considerar o contrato
completo.

| ID | Área | Requisito |
| --- | --- | --- |
| R01 | Confirmação de recebimento | Informar sucesso somente quando o sistema tiver assumido a responsabilidade de preservar o evento. |
| R02 | Disponibilização | Em operação normal, 99% dos eventos aceitos ficam consultáveis em até 10 segundos após o recebimento. |
| R03 | Consulta | 95% das consultas previstas retornam a primeira página em até 2 segundos, com até 100 usuários simultâneos. |
| R04 | Duplicação | Reenviar o mesmo evento não cria uma segunda medição visível. |
| R05 | Eventos atrasados | Aceitar medições feitas nas últimas 24 horas, preservando a distinção entre horário da medição e do recebimento. |
| R06 | Validação | Rejeitar eventos com formato inválido e informar o motivo ao emissor. |
| R07 | Isolamento | Cada usuário acessa somente as empresas e unidades para as quais possui autorização. |
| R08 | Identidade dos dispositivos | Identificar e bloquear um dispositivo individualmente. |
| R09 | Abuso | Limitar o impacto de envio excessivo sobre os demais dispositivos e clientes. |
| R10 | Rastreabilidade | Permitir investigar o recebimento, a rejeição e a disponibilização de um evento. |
| R11 | Retenção | Eventos deixam de estar disponíveis nas consultas após 90 dias; cópias e backups terão uma política de expiração própria a definir. |
| R12 | Estado atual | Eventos antigos recebidos posteriormente não substituem medições mais recentes no último estado conhecido. |
| R13 | Atualidade dos dados | Permitir ao usuário reconhecer quando um dispositivo está sem comunicação recente. |

## 7. Pontos que exigem decisão

Estas questões permanecem abertas para a revisão dos requisitos e da arquitetura:

- Qual é a meta de disponibilidade mensal e quais falhas ela deve abranger?
- Quais são os objetivos de recuperação e de perda de dados aceitável após falhas?
  Como eles se relacionam com a confirmação de recebimento de R01?
- O que caracteriza “operação normal” em R02? Qual prazo vale durante os picos e
  quanto tempo o sistema pode levar para eliminar o acúmulo de eventos?
- Como avaliar a janela de 24 horas quando o relógio do dispositivo está errado?
  O que acontece com uma medição no limite dessa janela que sofre atraso de envio?
- Como determinar o último estado se o relógio estiver incorreto ou o número de
  sequência reiniciar após uma reinicialização do dispositivo?
- Qual é o escopo de unicidade do identificador do evento e por quanto tempo o
  sistema precisa reconhecer reenvios? O que fazer se o mesmo identificador chegar
  com conteúdo diferente?
- Como vincular a identidade de um dispositivo à empresa e à unidade autorizadas?
- Quanto tempo pode decorrer entre o bloqueio de um dispositivo e a interrupção
  efetiva de seus envios?
- Qual intervalo sem comunicação caracteriza um dispositivo como desatualizado?
- A retenção é contada a partir da medição ou do recebimento? Qual é o prazo para
  expiração das cópias e dos backups?
- Qual ordenação e qual tamanho de página serão usados para avaliar as consultas?
- Qual é o orçamento de operação estimado e qual o limite de gasto do laboratório?

## 8. Fora do primeiro escopo

- Acionamento ou controle remoto dos equipamentos de refrigeração.
- Envio de notificações.
- Detecção automática de anomalias.

## 9. Validação da futura solução

A proposta de arquitetura deve explicar como atende aos requisitos, quais decisões
tomou nos pontos em aberto e quais limitações permanecem.

Os primeiros experimentos serão pequenos e reproduzíveis:

1. **Reenvio duplicado:** entregar o mesmo evento mais de uma vez e verificar que
   apenas uma medição fica visível.
2. **Chegada fora de ordem:** entregar uma medição antiga depois de uma mais recente
   e verificar o histórico e o último estado conhecido.
3. **Acesso entre clientes:** tentar consultar um evento de outra empresa, mesmo
   conhecendo seus identificadores, e verificar a recusa de acesso.

O laboratório pode usar uma amostra reduzida da frota. Os resultados devem indicar
a carga efetivamente testada e distinguir o comportamento demonstrado das
estimativas de capacidade para a escala completa.
