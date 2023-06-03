**Índice**

- [**O problema**](#o-problema)
- [**A solução**](#a-solução)
- [E agora, como migrar os dados para nova estrutura?](#e-agora-como-migrar-os-dados-para-nova-estrutura)
- [Sobre a aplicação para ETL](#sobre-a-aplicação-para-etl)
- [Resultado do ETL](#resultado-do-etl)

--------

Opa, espero que estejam bem!

Bom... vou compartilhar uma história de alguns meses mas tentarei ser o mais breve possível.

### **O problema**

Em um dos sistemas do cliente da empresa que trabalho, existia um chat de atendimento legado onde era centralizado toda comunicação entre a equipe de atendimento que fazem parte de filiais da empresa do grupo e usuários do sistema, e por ser um sistema legado sempre havia issues de bugs que surgiam por uma falta de boa estrutura de dados, consequentemente também várias issues de performance e escalabilidade levando em consideração que apenas tabelas diretas da funcionalidade que registravam `conversas` e `mensagens` e outras que compunham a consulta destas duas tinham juntas por volta de ~7.5MM de registros, indiretas de `usuários` e `atendentes` envolvidos e outras por volta de mais ~1MM, naquela estrutura e com as consultas necessárias esses números já era um problema.

Dito isso, passei alguns meses atendendo issues que para correção ou implementação de algo simples era necessário refatorar queries, procedures, manejo de indices, fazer malabarismo de colunas e dados para atender melhor os relacionamentos e consultas (não usam ORM, era tudo na unha).

E a discussão de uma refatoração de toda a funcionalidade sempre estava presente na equipe por causa da forma que estava posto a estrutura de dados, só que isso não era uma discussão tão simples pois existia duas equipes a interna do cliente e a externa que eu faço parte.

### **A solução**

Então alguns meses na luta, a recomendação da refatoração foi atendida pelo cliente, para alegria de todos que esbarravam na funcionalidade e principalmente a minha que estava pegando a maioria das issues.

Como eu já estava familiarizado com parte da estrutura da funcionalidade, fiquei responsável pela refatoração.

Não vou me aprofundar muito do que foi feito pois não é o objetivo a refatoração em si.

Após 2 meses de trabalho para entender toda estrutura da funcionalidade, dezenas de procedures visitadas (maioria das regras de negócio dos sistemas do cliente são em procedures), refatorando e atendendo outras issues de outras partes do sistema, finalizei a modelagem das tabelas, os endpoints da funcionalidade, com uma nova estrutura de dados, eliminando tabelas e colunas desnecessárias, separando responsabilidades, camada de repositório para lidar com o banco, procedures criadas, validações de payload para manter consistência, tudo documentado para não ter desculpa para não seguir a padronização, coisas que na visão de boa parte é o mínimo mas até então não tinha (isso é muito comum).

Testado endpoints, dupliquei a funcionalidade nos fronts (interface do atendente / interface web do usuário / interface mobile seria feita depois por outro membro do time) apliquei os requisitos de melhoria, criada interface de serviço, implementa, tudo testado e aprovado no ambiente de desenvolvimento. Ufa.... done!

### E agora, como migrar os dados para nova estrutura?

*Disclaimer: Lembrando que não sou DBA ou especialista em banco de dados.*

Chegou o grande momento, migrar os dados, e esse momento levantava um questionamento, qual a melhor forma de fazer isso? E um ponto importante, em menor tempo possível, pois a data de entrega se aproximava, para a equipe de QA do cliente começarem os testes no ambiente de staging.

Cheguei a avaliar a utilização de ferramentas prontas no mercado para isso, algumas me pareciam complexas demais para o caso em específico, e a principal que aparece para esse cenário é a SSIS da própria Microsoft visto que o banco é MSSQL.

Mas o banco estava na AWS e precisaria configura-lo no servidor RDS e naquele momento não gostaria de envolver a equipe de infra, e pela minha falta de experiência com a ferramenta talvez demorasse mais do que fazer por conta própria pro caso específico, pelas transformações necessárias dos dados.

Um exemplo de muitos motivos que me fez pensar assim: a coluna que contia o conteúdo da mensagem, tinha 3 tipos de dados diferentes, e um deles era JSON Prosemirror, e todos os tipos deveriam ser convertidos para HTML ou text plain. E existia várias outras ponderações dessas.

### Sobre a aplicação para ETL

Decidi então seguir para o desenvolvimento de uma ferramenta simples de ETL, já imaginando um conceito de sistema distribuído para ganharmos em velocidade processamento usando vários núcleos para processar a fila de registros.

> **Por que usar núcleos ao invés de threads neste caso?**
>
> Neste caso específico, a escolha de utilizar núcleos em vez de threads foi feita visando evitar problemas de concorrência e assegurar um ambiente isolado para o processamento de cada lote de dados. Com o uso de threads, os recursos do núcleo seriam compartilhados, o que poderia resultar em conflitos entre os diferentes lotes de processamento. Ao optar por núcleos, cada lote é executado em um ambiente separado, com seus próprios recursos dedicados, como registradores, caches e unidades de execução. Isso proporciona uma divisão física dos recursos, minimizando a interferência entre os lotes e impedindo que o desempenho de um lote afete negativamente os outros. Além disso, levando em consideração esses fatores, utilizar threads exigiria um gerenciamento mais complexo, e demandaria um tempo que eu não tinha disponível.

Aplicação foi feita em Node, pensada para funcionar num conceito modular, onde poderá futuramente ser utilizada como lib também, pois todo o core de gerenciamento é separado das definições do ETL que ficam em `modules` é onde fica toda definição das tabelas de origem e destino, o mapeamento das colunas de origem e destino, e a função de transformação dos dados, dentro de cada modulo. No banco usado para o ETL, será criado uma tabela chamada `etl_queue` onde estará os registros da fila do processamento.

Na função `transformData` é possível fazer consultas adicionais, no caso desta funcionalidade de chat de atendimento na tabela que registrava as conversas, foi necessário fazer 8 consultas adicionais no banco para o carregamento das informações na tabela nova de conversas.

Não vou me aprofundar sobre a aplicação do ETL, pois vocês mesmos podem saber mais e/ou contribuir no repositório. :)

> [Github - Database-ETL](https://github.com/martinshumberto/database-etl)

*Essa versão foi feita pensando em solucionar esse objetivo descrito, dentro de um prazo apertado.*

Estrutura:

```bash
├── .env
├── .env.example
├── .eslintrc.js
├── .gitignore
├── .prettierrc
├── .vscode
│   └── settings.json
├── README.md
├── index.js
├── logs
│   ├── etl_combined.log
│   ├── etl_error.log
│   ├── etl_info.log
│   ├── etl_sql.log
│   └── etl_warn.log
├── package.json
├── src
│   ├── config
│   │   ├── db.js
│   │   ├── etl.js
│   │   └── log.js
│   ├── lib
│   │   ├── commands
│   │   │   ├── create-module.js
│   │   │   ├── reset-state.js
│   │   │   └── templates
│   │   │       └── module-index.js
│   │   ├── db.js
│   │   ├── etl.js
│   │   ├── helpers
│   │   │   ├── index.js
│   │   │   └── sql.js
│   │   ├── log.js
│   │   ├── queue.js
│   └── modules
│       └── .gitkeep
└── yarn.lock
```

Tentei deixar o mais reutilizável possível sem impactar na entrega no trabalho, mas certamente tem muito o que melhorar. E gostaria muito que vocês contribuíssem apontando essas melhorias e dando opinião se possível.

E certamente darei continuidade a ela dentro do possível, abrangendo mais banco de dados, e promovendo melhorias já previstas ou que me indiquem, para ter uma ferramenta pronta, versátil e rápida de ETL.

### Resultado do ETL

Abaixo é um recorte da estimativa e defesa enviada para a gerencia do projeto para solicitação de uma maquina para o processamento do ETL com um número considerável de núcleos para cumprirmos o prazo.

> ...
>
> **Mensagens:**
> Total de registros: 1.5MM
> Finalizar montagem da fila: ˜10min `*`
> Segundos de processamento por 100 registros: 45 seg. (0.45 por registro)
> Total de segundos para processamento: 683.622s
>
> *Estimativas:*
> 1 núcleo = 683.622 segs. = ~7 dias `**`
> 20 núcleos = 683.622/20 segs. = ~9 horas `**`
> 68 núcleos = lotes de 5000 = 239.267/68 segs. = ~1 hora `**`
>
> --------
> `*` A montagem da fila é feita pelo cluster master (0) e de acordo com o valor setado no batchSize (default: 100) vai distribuindo os lotes de forma igualitária para os outros clusters irem processando a fila, e após finalizar o lançamento dos lotes para processamento o cluster master é liberado para processar a fila também.
> `**` Estimativa feita na minha maquina, e ao rodar no AWS será mais rápido em torno de ~30/40% por estar na mesma rede interna e na mesma região que o database, a estimativa da AWS já está completando a porcentagem.
>
> ...

E o resultado final foi de: **~1h30m** para todos os módulos!

A estimativa total enviada junto ao recorte acima foi de 3hrs.

A maquina utilizada na AWS:
`x2gd.16xlarge com 64 núcleos por $5.344 a hora`

--------

*Importante ressaltar que aqui é uma simplificação de todo o processo para ser o mais breve possível e compartilhar com vocês. Sempre há muitas discussões, testes, tentativas, erros, acertos e melhorias a serem feitas.*

--------

Obrigado.

Deixe seu comentário! 😄
