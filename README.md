# AzureSynapseAnalyticsNYTaxi

<p>Implementa uma solução de Engenharia de dados para análise e geração de relatórios das viagens dos táxis de Nova York. O projeto utiliza exclusivamente a Azure e o Synapse Analytics como principal componente. Os dados são fornecidos pela NYC Taxi & Limousine Commission, que grava os históricos das viagens. O estudo também simula uma campanha de incentivo ao uso de cartões de crédito, que será mostrada em um relatório de Power BI.
As transformações, conversões e incrementos são feitos basicamente em scripts SQL dentro do Synapse Analytics conectados ao motor Serverless do recurso. Um pool SQL dedicado e o Spark pool também foram utilizados para agregações e geração de tabela externa para staging.
O Synapse Analytics também é responsável pela orquestração, automatização e monitoramento dos pipelines do projeto.
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/warehouse.png"/>
</p>
Houve também integração com o Cosmos DB utilizando Synapse Link para simular um sistema que hospeda dados de evento, como um gerenciador de corridas.<p>
Arquivos utilizados em diversos formatos, conforme distribuição a seguir: TRIP DATA (Parquet, Delta e CSV); TAXI ZONE e CALENDAR (CSV); TRIP TYPE (TSV); RATE CODE e PAYMENT TYPE (JSON); VENDOR (CSV e Quoted).

## Arquitetura
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/arq.png"/>
</p>

## Recursos utilizados no Azure:
* Data Lake Gen2 para armazenar dados brutos, processados e de apresentação. Serão gerados dois conteiners, um com os dados em si, e o outro que será utilizado pelo próprio Synapse para guardar informações dos recursos, como os Pools de SQL, por exemplo. 
* Synapse Analytics para ingestão, transformação, conversão, automatização e orquestração. 
* Serverless SQL Pool, Dedicated SQL Pool e Spark Pool como ferramentas de computação.
* Cosmos DB como um banco de dados NoSQL que sincroniza dados de dispositivos simulados.
* Key Vault para gerenciamento na segurança de acesso.
* Power BI para exibição de relatórios.



## Preparo preliminar do ambiente
* Criar uma conta Azure, uma subscription e um resource group usado como um contêiner do projeto.
* Criar conta de armazenamento.
* Criar um Synapse Analytics workspace

## Overview dos arquivos a serem utilizados
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/format_files.png"/>
</p>

## Estrutura e construção do projeto
O contêiner foi criado manualmente utilizando o Microsoft Azure Storage Explorer. Como não haverá manutenção dos arquivos, eles foram inseridos também manualmente numa pasta denominada “raw”. 
Se criado no mesmo grupo de recursos, ao entrar no Synapse Studio/Data/Linked será possível avistar o data lake e a pasta “raw” criada. Dentro dela todos os arquivos que serão utilizados. Com um clique no botão direito do mouse é possível criar um novo script SQL. Os dados são obtidos através do OPENROWSET usando o pool de SQL sem servidor. A função permite acessar arquivos no armazenamento do Azure e ler o conteúdo de uma fonte de dados remota. Dessa forma é criada uma estrutura de dados em SQL conforme a seguir.

### Estrutura
Os arquivos em si possuem uma estrutura imprópria, que prejudicam o desempenho, sendo necessário fazer alterações de nomenclatura e formatos. Por isso, cada um deles terá seu correspondente em SQL. Todas as anotações de alteração se encontram no projeto. São criadas também base de dados de diferentes momentos do projeto. A “nyc_taxi_discovery” contempla o correspondente dos sete arquivos, suas variações de formato e outros arquivos que verificam duplicatas, a qualidade dos dados, joins (para obter conclusões que não são possíveis com arquivos isolados), transformações (que também verificam qualidade dos dados) e descobertas. Tudo é colocado na pasta Discovery.

### Camada Relacional (LDW)
Uma outra pasta, de nome ldw, também tem sua própria base de dados. Aqui a ênfase está nas tabelas externas. Uma tabela externa aponta para dados localizados no Azure Storage Blobs ou Azure Data Lake Storage. Para importar dados de tabelas externas para pools de SQL dedicados é utilizada a instrução CREATE TABLE AS SELECT (CETAS). 

Os arquivos têm as seguintes funções:
* create_databases – responsável pela criação da base de dados, sua configuração e esquemas.
* create_external_data_sources – para a criação de tabelas externas é necessário criar recursos externos, função deste arquivo.
* create_external_file_formats - 	também requisito para criação de tabelas externas.
* create_bronze_tables – cria tabelas externas de sete arquivos localizados no “raw”.
* create_bronze_views – cria views dos restantes: rate code, payment type e trip_data_green, sendo este último montado com partições. Ainda não disponíveis em tabelas externas, apenas em views. 
* create_silver_taxi_zone e demais arquivos da pasta “raw”, desta vez as tabelas externas já contêm transformações
* create_silver_views – cria a view transformada e particionada do arquivo trip_data_green
* create_gold_trip_data_green – cria a tabela com a tabela pronta para apresentação. Neste caso é utilizado um Stored Procedure que será mencionado a seguir. 
* create_gold_views – cria a view pronta e particionada do arquivo trip_data_green.

#### Stored Procedure
Como cada arquivo possui procedimentos repetitivos, uma solução interessante é o stored procedures (procedimentos armazenados), um conjunto de comandos em SQL que podem ser executados de uma só vez, como em uma função. Ele armazena tarefas repetitivas e aceita parâmetros de entrada para que a tarefa seja efetuada de acordo com a necessidade individual. São construídos arquivos de origem da pasta “silver” com essas funções.

### Staging
É uma tabela temporária que guarda todos os dados que serão usados para fazer alterações na tabela de destino, incluindo atualizações e inserções, otimizando o desempenho. Ela é distribuída em Round Robin, ideal para este tipo de prática. Para pools de SQL sem servidor, usamos (CTAS) para salvar o resultado da consulta em uma tabela externa no Armazenamento do Microsoft Azure. Conforme o caso. A base de dados “nyt_dwh” tem a tabela staging.
O motor utilizado nesta tabela é diferente do anterior. Até o momento apenas o Serverless SQL pool tinha sido usado, porém desta vez é utilizado o Dedicated SQL pool. Um pool dedicado de local físico, sendo assim, só poderemos ter acesso a esta tabela se o pool estiver ligado, conforme abaixo. 

<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/dedicatedpool_round_robin.png"/>
</p>
Para utilizar, é necessário cria-lo no workspace do Synapse.

### Synapse Link e Cosmos DB
O Synapse Analytics tem conexão direta com o Cosmos DB através do Synapse Link. No projeto, foi simulado que todos os veículos possuíam dispositivos que enviavam informações ao Cosmos DB. O Cosmos mandava esses dados diretamente para o Synapse Analytics, onde se pode fazer consultas em tempo real, como no print abaixo.
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/query_cosmos.png"/>
</p>
A pasta synapse_link possui as instruções que migram esses eventos do Cosmos para o Synapse Analytics. Já a pasta cosmos demonstra a query vista acima. 
Essas informações simuladas foram construídas em json e inseridas no Cosmos DB, após a criação do recurso.

### Notebooks
Além dos Scripts SQL há duas pastas com notebooks. Para executar é necessário utilizar o spark pools, também criado no workspace do Synapse Analytics. O terceiro motor de computação do projeto. 
O arquivo de consulta do heartbeat cria dataframes e lê dados do repositório analítico do Cosmos DB. Já o outro faz agregações no dataframe e os salva na pasta gold.

### Pipelines e triggers
Para automatização dos trabalhos, foram criados pipelines. O primeiro deles é o create_silver_tables que executa o script equivalente.
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/pl1.png"/>
</p>
O próximo é o create_silver_trip_data_green que executa os arquivos com os stored procedures criando a silver view.
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/pl2.png"/>
</p>
O próximo é o create_gold_trip_data_green que faz exatamente o mesmo que o pipeline anterior, só que no gold. 
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/pl3.png"/>
</p>
Por fim um pipeline pai que executa todos os pipelines anteriores. Importante ressaltar que o pipeline gold só é executado se o silvers tiverem sucesso. Pois a construção do gold depende disso. 
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/pl4.png"/>
</p>
Para orquestração, um trigger de agendamento. Neste exemplo o pipeline pai é executado a cada 5 minutos. Em caso de falha, um e-mail é enviado a quem for determinado.
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/tr.png"/>
</p>

### Relatório
O Power BI foi utilizado para confeccionar um relatório de apresentação. Nele foi simulada uma campanha de incentivo para pagamento via cartão de crédito. Verifica-se a correlação entre o dia da semana e o método de pagamento; o método de pagamento pela localidade e a soma dos valores de tarifa através do tempo, onde nota-se uma queda acentuada no valor em função de ondas da COVID19. 
<p align="center">
<img src="https://github.com/LeandroRFausto/AzureSynapseAnalyticsNYTaxi/blob/main/nyt/prints/campanha_pbi.png"/>
</p>








