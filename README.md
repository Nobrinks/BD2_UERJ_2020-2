# Trabalho da disciplina banco de dados II - UERJ 2020/2

## Integrantes

* Dennis Ribeiro Paiva - 201610050611
* Igor Sousa Silva - 201610053411
* Paulo Victor Coelho - 201610049711
* Vinicius Sathler - 201610051611
  
## Introdução

O Amazon DynamoDB (ADDB) é um banco de dados de Chave-Valor (Key-Value) e de documentos. Cada chave só pode ter um valor.

Na ADDB as tabelas são guardadas em partições (Partitions) que seriam a parte fisica, como o ADDB lida com isso é totalmente transparente para a nossa aplicação. A Amazon Web Service (AWS) possui SSDs fazendo a alocação e replicando o banco automaticamente entre diferentes Data Centers disponíveis na região. Lembrando que ele é alocado inicialmente no Data center mais proximo e depois replicado.

## Componentes

No ADDB cada tabela funciona como uma coleção de elementos, e cada elemento funciona como uma coleção de atributos. com exceção da chave primária, os elementos não possuem um conjunto de atributos fixos, ou seja, cada item de uma tabela pode possuir atributos distintos. Também é possível criar atriutos aninhados com até 32 níveis.

<figure class="image">
    <img src="imagens\componentesADDB.png" alt='Exemplo da tabela "people" com 3 elementos'>
    <figcaption>Exemplo da tabela "People" com 3 elementos</figcaption>
</figure>

### Chaves Primárias

o ADDB possui dois tipos de chaves primárias, Partition Key e Sort Key. Cada elemento da tabela possui uma chave primária (partition key) única. É possível que uma tabela possua chave primária composta de 2 atributos (Partition Key e sort key).

<figure class="image">
    <img src="imagens\primaryKeysExample.png" alt='Exemplo da tabela "music" com chaves primárias compostas'>
    <figcaption>Exemplo da tabela "Music" com chaves primárias compostas</figcaption>
</figure>

## Arquitetura

O ADDB armazena todos os seus dados em "blocos" de memória chamados de "partitions" ou particões. O endereçamento desses dados funciona como um hash-map, onde cada elemento terá uma partition alvo e essas partitions são distribuídas e replicadas em diversos servidores da AWS da região.

### Partição

O AWS aloca máquinas o banco de dados de acordo com a demanda e cada máquina que participa do "cluster" recebe uma "tag" com um valor inteiro no intervalo [0,2^64). Quando ocorre uma requisição para inserção de dados no banco, a chave do elemento passa por uma função hash que retorna um inteiro no intervalo anterior, o dado é então guardado na primeira máquina encontrada, a busca pela máquina é realizada em um esquema de "relógio" de acordo com a imagem seguinte.

<figure class="image">
    <img src="imagens\arquitetura_dynamo.PNG" alt='Busca da máquina-alvo'>
    <figcaption>Busca da máquina-alvo</figcaption>
</figure>

No exemplo da imagem acima, se a função hash retornar 100000, o dado será armazenado na máquina A.

Caso uma máquina fique sobrecarregada de dados o AWS aloca máquinas adicionais para o banco. A nova máquina armazenará todos os dados do seu novo intervalo.

<figure class="image">
    <img src="imagens\arquitetura_dynamo_nova_maquina.PNG" alt='adicao de uma nova maquina'>
    <figcaption>Adição da máquina "D" ao cluster</figcaption>
</figure>

No exemplo acima todos os dados entre os endereços 10000 e 15000 seão transferidos para a máquina D. Caso uma máquina fique subutilizada, o banco pode remover uma máquina e transferir seus dados para a próxima máquina.

### Replicação

Para evitar perda de dados caso haja um problema em alguma máquina, quando ocorre uma requisição de inserção no banco, a AWS replica a inserção da máquina-alvo nas N-1 próximas máquinas.

<figure class="image">
    <img src="imagens\arquitetura_dynamo_replicacao.PNG" alt='Replicação da requisição nas N-1 próximas máquinas (N=3)'>
    <figcaption>Replicação da requisição nas N-1 próximas máquinas (N=3)</figcaption>
</figure>

Para evitar perda de desempenho na operação de escrita, o banco enviará a resposta de conclusão da escrita caso uma quantidade W de máquinas termine a operação.

Um problema desse método é que uma requisição de leitura pode tentar recuperar dados de uma máquina que não tenha completado o processo de escrita, retornando dados inválidos ou desatualizados. Para resolver este problema o banco lê R cópias do dado entre as máquinas do cluster, retornando o valor mais recente.

Para garantir que a operação seja um sucesso, o valor de R+W deve ser superior ao valor de N, dessa maneira garante-se que o dado mais recente será recuperado.

## Otimização (GLI e SLI)

Como explicado anteriormente, todo elemento de uma tabela possui uma chave primária chamada ``partition key``, e essa chave passa por uma função hash para definir em qual partição o elemento será guardado.

<figure class="image">
    <img src="imagens\DataDistributionPK.png" alt='Exemplo de alocação de um elemento em uma partition'>
    <figcaption>Exemplo de alocação de um elemento em uma partição</figcaption>
</figure>

Se o elemento possuir uma chave secundária, ou ``Sort Key``, serão alocados para a mesma partição todos os elementos com a mesma ``partition key`` e estes estarão ordenados pela sua ``sort key``.

<figure class="image">
    <img src="imagens\DataDistributionSK.png" alt='Exemplo de alocação de um elemento em uma partição e ordenado pela sua sort key'>
    <figcaption>Exemplo de alocação de um elemento em uma partição e ordenado pela sua sort key</figcaption>
</figure>

No ADDB, pode-se criar uma _secondary index_, que é diferente do conceito de index de um banco relacional. Quando você cria a *secondary index* deve-se criar uma *partition key* e uma sort key e defini-las. Após a criação, pode-se fazer uma query ou um scan igual como seria feito em uma tabela. ADDB não tem um otimizador de queries, então o _secondary index_ é usado apenas quando se faz uma _query_ ou um _scan_ nele mesmo.
Quando é gerado uma *secondary index*, uma outra tabela é criada com as chaves definidas, além da *primary key* da tabela original, ou seja, no final, mesmo tendo definido apenas duas chaves, a nova tabela terá 3.

No Dynamo pode-se usar dois tipos de indexes:

* *secondary indexes* globais: a *primary key* do index deve ter dois atributos da tabela (qualquer atributo). Para projetar posteriormente, apenas será mostrado os atributos definidos no parâmetro ``Projection`` no momento da criação do índex. Esse parâmetro será explicado.

* *secondary indexes* locais: a *partition key* do index deve ser a mesma *partition key* da tabela. Entretanto, a *sort key* pode ser qualquer atributo da tabela. Além disso, podemos projetar qualquer atributo a posteriori da criação do índex, mesmo que não tenha sido definido no ``Projection``.

Suas diferenças:

| Características  | Global Secondary Index | Local Secondary Index |
| ----------- | ----------- | ----------- |
| *Key Schema* | A chave primária pode ser simples (*partition key*) ou composta (*partition* e *sort key*). | A chave primária deve ser composta (*partition key* e *sort key*).
| *Key Attributes* | A *partition* e a *sort key* (se definida) podem ser qualquer atributo da tabela base do tipo String, Número ou Binário. | A *partition key* do índice é a mesma da tabela base. A *sort key* pode ser qualquer atributo da tabela base do tipo String, Número ou Binário.
| Restrições de tamanho por valores de *Partition Key* | Não há restrições | Para cada valor de *partition key*, o tamanho total de todos os itens indexados deve ser de <10 GB.
| Operações Online de índices | Podem ser criados ao mesmo tempo da criação de uma tabela. Pode ser adicionado um novo índice a uma tabela existente ou excluir um índice existente. | São criados ao mesmo tempo em que se cria uma tabela. Não é possível ser adicionado a uma tabela existente nem excluir índices secundários locais existentes.
| *Queries* e *Partitions* | Permite a consulta da tabela inteira, em todas as partições. | Permite consultar uma única partição, conforme especificado pelo valor da *partition key* na consulta.
| *Provisioned Throughput Consumption* | Tem suas próprias configurações de *provisioned throughput* para operações de leitura e gravação. *Queries* ou *scans* consomem unidades de capacidade do índice, e não da tabela base. O mesmo vale para atualizações de índice secundário global devido a gravações de tabelas. | *Queries* ou *scans* consomem unidades de capacidade de leitura da tabela base. Quando você escreve em uma tabela, seus índices secundários locais também são atualizados. Essas atualizações consomem unidades de capacidade de gravação da tabela base.
| Projected Attributes | Após ser definido o *projection*, não pode-se alterar os atributos que serão projetados na *query/scan* do índice. | Pode ser definido atributos a posteriori da criação do índice

Pode-se adicionar uma index global em uma tabela existente, usando a ação UpdateTable e especificando GlobalSecondaryIndexUpdates

``` JSON
{
  TableName: "Music",
  AttributeDefinitions:[
    {AttributeName: "Genre", AttributeType: "S"},
    {AttributeName: "Price", AttributeType: "N"}
  ],
  GlobalSecondaryIndexUpdates: [
    {
      Create: {
        IndexName: "GenreAndPriceIndex",
        KeySchema: [
          {AttributeName: "Genre", KeyType: "HASH"}, //Partition key
          {AttributeName: "Price", KeyType: "RANGE"}, //Sort key
        ],
        Projection: {
          "ProjectionType": "ALL"
        },
        ProvisionedThroughput: { // Only specified if using provisioned mode
          "ReadCapacityUnits": 1,"WriteCapacityUnits": 1
        }
      }
    }
  ]
}
```

Você deve prover os seguintes parâmetros para a `UpdateTable`

* ``TableName`` - A tabela que o index vai ser associado

* ``AttributeDefinitions`` - os tipos de dados para a *key schema*

* ``GlobalSecondaryIndexUpdates`` - detalhes sobre o índice que você deseja criar:

  * `IndexName` - um nome para o índice.

  * ``KeySchema`` - os atributos que são usados para a chave primária do índice.

  * ``Projection`` - atributos da tabela que são copiados para o índice. Neste caso, ALL significa que todos os atributos são copiados. Pode ser INCLUDE e definir certos atributos ou KEYS_ONLY para ser apenas chaves.

  * ``ProvisionedThroughput`` (para tabelas definidas)- o número de leituras e gravações por segundo que você precisa para este índice. (Isso é separado das configurações de throughput provisionado da tabela.)

otimização da otimização
  https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-gsi-aggregation.html

## Transações

[Transaction](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)

Com as transações da Amazon DynamoDB, você pode agrupar várias ações e submetê-las como uma única operação de tudo-ou-nada com a TransactWriteItems ou TransactGetItems. As seções seguintes descrevem operações da API, gerenciamento de capacidade e outros detalhes sobre o uso de operações transacionais no DynamoDB. 

### TransactWriteItems 
TransactWriteItems é uma operação de escrita síncrona e idempotente que agrupa até 25 ações de escrita em uma única operação de tudo-ou-nada. Essas ações podem ter como alvo até 25 itens distintos em uma ou mais tabelas DynamoDB dentro da mesma conta AWS e na mesma Região. O tamanho agregado dos itens na transação não pode exceder 4 MB. As ações são completadas atomicamente de forma que ou todas elas sejam bem sucedidas ou nenhuma delas seja bem sucedida. 

Uma operação TransactWriteItems difere de uma operação BatchWriteItem em que todas as ações nela contidas devem ser completadas com sucesso, ou nenhuma alteração é feita. Com uma operação BatchWriteItem, é possível que apenas algumas das ações do lote sejam bem sucedidas enquanto as outras não.
Não é possível visar o mesmo item com múltiplas operações dentro da mesma transação. Por exemplo, você não pode realizar um ConditionCheck e também uma ação de atualização sobre o mesmo item na mesma transação. 

Você pode adicionar os seguintes tipos de ações a uma transação: 

* Put - Inicia uma operação PutItem para criar um novo item ou substituir um item antigo por um novo item, condicionalmente ou sem especificar qualquer condição. 

* Update - Inicia uma operação UpdateItem para editar os atributos de um item existente ou adicionar um novo item à tabela se ele ainda não existir. Use esta ação para adicionar, excluir ou atualizar atributos em um item existente condicionalmente ou sem uma condição. 

* Delete - Inicia uma operação DeleteItem para excluir um único item em uma tabela identificada por sua chave primária. 

* ConditionCheck - Verifica se um item existe ou verifica a condição de atributos específicos do item. 

Uma vez concluída uma transação, as mudanças feitas dentro dessa transação são propagadas para índices secundários globais (GSIs), fluxos e backups. Como a propagação não é imediata ou instantânea, se uma tabela for restaurada do backup para um ponto médio de propagação, ela pode conter algumas, mas não todas, as mudanças feitas durante uma transação recente. 

#### Idempotência

Opcionalmente, você pode incluir um _token_ de acesso ao fazer uma chamada TransactWriteItems para garantir que a solicitação seja idempotente. Fazer suas transações idempotentes ajuda a evitar erros de aplicação se a mesma operação for submetida várias vezes devido a um timeout de conexão ou outro problema de conectividade. 

Se a chamada TransactWriteItems original foi bem sucedida, as chamadas TransactWriteItems subsequentes com o mesmo _token_ de acesso retornam com sucesso sem fazer nenhuma alteração. Se o parâmetro ReturnConsumedCapacity for definido, a chamada TransactWriteItems inicial retorna o número de unidades de capacidade de gravação consumidas na realização das mudanças. Chamadas TransactWriteItems subsequentes com o mesmo _token_ de acesso retornam o número de unidades de capacidade de leitura consumidas na leitura do item. 

Pontos importantes sobre a idempotência: 

* Uma ficha de cliente é válida por 10 minutos após o término da solicitação que a utiliza. Após 10 minutos, qualquer pedido que utilize o mesmo _token_ de acesso é tratado como um novo pedido. Não se deve reutilizar o mesmo _token_ de acesso para a mesma solicitação após 10 minutos. 

* Se você repetir uma solicitação com o mesmo _token_ de acesso dentro da janela de idempotência de 10 minutos, mas alterar algum outro parâmetro de solicitação, a DynamoDB retorna uma exceção IdempotentParameterMismatch. 

#### Tratamento de erros de escrita 

As transações de escrita não são bem sucedidas nas seguintes circunstâncias: 

* Quando uma condição em uma das expressões de condição não é satisfeita. 

* Quando um erro de validação de transação ocorre porque mais de uma ação na mesma TransactWriteItems operação visa o mesmo item.  

* Quando uma solicitação TransactWriteItems entra em conflito com uma operação TransactWriteItems em andamento em um ou mais itens da solicitação TransactWriteItems. Neste caso, a solicitação falha com uma TransactionCanceledException.  

* Quando não há capacidade provisionada suficiente para que a transação seja concluída. 

* Quando um item se torna muito grande (maior que 400 KB), ou um índice secundário local (LSI) torna-se muito grande, ou um erro de validação similar ocorre devido a mudanças feitas pela transação. 

* Quando há um erro do usuário, tal como um formato de dados inválido. 

 

### TransactGetItems API 

A TransactGetItems é uma operação de leitura síncrona que agrupa até 25 ações. Essas ações podem direcionar até 25 itens distintos em uma ou mais tabelas DynamoDB dentro da mesma conta AWS e Região. O tamanho agregado dos itens na transação não pode exceder 4 MB.  

As ações Get são realizadas atomicamente para que ou todas elas tenham sucesso ou todas elas falhem: 

* Get - Inicia uma operação GetItem para recuperar um conjunto de atributos para o item com a chave primária dada. Se nenhum item correspondente for encontrado, Get não retorna nenhum dado. 

#### Tratamento de erros de leitura 

As transações lidas não têm sucesso sob as seguintes circunstâncias: 

* Quando uma solicitação TransactGetItems entra em conflito com uma operação TransactWriteItems em andamento em um ou mais itens da solicitação TransactGetItems. Neste caso, a solicitação falha com uma TransactionCanceledException. 
* Quando não há capacidade provisionada suficiente para que a transação seja concluída. 
* Quando há um erro do usuário, tal como um formato de dados inválido. 

### Níveis de Isolamento para Transações DynamoDB 

Os níveis de isolamento das operações transacionais (TransactWriteItems ou TransactGetItems) e outras operações são os seguintes. 


### Serializável

Isolamento serializável garante que os resultados de múltiplas operações simultâneas sejam os mesmos como se nenhuma operação tivesse começado até que a anterior tivesse terminado. 

Existe um isolamento serializável entre os seguintes tipos de operação: 

* Entre qualquer operação transacional e qualquer operação de escrita padrão (PutItem, UpdateItem, ou DeleteItem). 

* Entre qualquer operação transacional e qualquer operação de leitura padrão (GetItem). 

* Entre uma operação de TransactWriteItems e uma operação de TransactGetItems. 

Embora haja um isolamento serializável entre as operações transacionais, e cada padrão individual escreve em uma operação BatchWriteItem, não há isolamento serializável entre a transação e a operação BatchWriteItem como uma unidade. 

Da mesma forma, o nível de isolamento entre uma operação transacional e uma GetItems individual em uma operação BatchGetItem é serializável. Mas o nível de isolamento entre a transação e a operação BatchGetItem como uma unidade é _read-committed_. 

Uma única solicitação GetItem é serializável com relação a uma solicitação TransactWriteItems em uma das duas formas, antes ou depois da solicitação TransactWriteItems. Múltiplas solicitações GetItem, contra chaves em uma solicitação TransactWriteItems concorrente, podem ser executadas em qualquer ordem e, portanto, os resultados são readcomissionados. 

Por exemplo, se as solicitações GetItem para o item A e item B forem executadas simultaneamente com uma solicitação TransactWriteItems que modifica tanto o item A quanto o item B, há quatro possibilidades:  

* Ambas as solicitações GetItem são executadas antes da solicitação TransactWriteItems. 
* Ambas as solicitações GetItem são executadas após a solicitação TransactWriteItems. 
* A solicitação GetItem para o item A é executada antes da solicitação TransactWriteItems. Para o item B, a solicitação GetItem é executada após a solicitação TransactWriteItems. 
* A solicitação GetItem para o item B é executada antes da solicitação TransactWriteItems. Para o item A, a solicitação GetItem é executada após a TransactWriteItems. 

Se o nível de isolamento serializável for preferível para múltiplas solicitações GetItem, então use TransactGetItems. 

### Read-Commited 

O isolamento do _read-committed_ garante que as operações lidas sempre retornem valores comprometidos para um item. O isolamento de leitura-compromisso não impede modificações do item imediatamente após a operação lida.O nível de isolamento é _read-committed_ entre qualquer operação transacional e qualquer operação de leitura que envolva múltiplas leituras padrão (BatchGetItem, Query, ou Scan). Se uma transação escrita atualiza um item no meio de uma operação BatchGetItem, Query ou Scan, a operação lida retorna o novo valor comprometido.

## Teoria: descrever como ocorre o controle de concorrência no BD;

### Optimistic locking

_Optimistic locking_ é uma estratégia que assegura que o item do lado do cliente que se está atualizando (ou excluindo) seja o mesmo que o item na _Amazon DynamoDB_. Se usar esta estratégia, as gravações de seu banco de dados serão protegidas de serem sobrescritas pelas gravações de outros, e vice versa

Com o _Optimistic locking_, cada item tem um atributo que funciona como um número de versão. Se retornar um item de uma tabela, a aplicação grava o número de versão daquele item. Pode-se atualizar o item, mas apenas se o número de versão no lado do servidor não tiver sido alterado. Se houver uma incompatibilidade de versão, significa que alguém modificou este item ateriormente. A tentativa de atualização falha, porque se tem uma versão desatualizada do item. Caso isto aconteça, simplesmente tente de novo recuperando o item e, em seguida, tentando atualizá-lo.

_Optimistic locking_ tem o seguinte impacto nestes métodos do ```DynamoDBWrapper```:

* ```save```, ```put```, ```update```  — Para um novo item, o ```DynamoDBWrapper``` atribui um número de versão inicial 1. Caso recupere um item, atualize um ou mais de suas propriedade e tente salvar as alterações, a operação ```save```, ```put``` e ```update``` terão sucesso apenas se o número de versão no lado do cliente e no lado do servidor forem correspondentes.

* ```delete``` — o método ```delete``` necessita de um objeto como parametro e o ```DynamoDBMapper``` realiza a verificação da versão antes de remover o item. A verificação da versão pode ser desabilitada se ```DynamoDBMapperConfig.SaveBehavior.CLOBBER``` for especificado na requisição.

### Distributed Locking

_Distributed Locking_ é uma técnica a qual, quando implementada corretamente, garante que dois processos não podem acessar um dado compartilhado ao mesmo tempo. Os processos dependem do protocolo de bloqueio para garantir que apenas um tenha permissão para prosseguir uma vez que se tenha um bloqueio.

Implementações comuns para _distributed locking_ geralmente involvem um gerenciador de bloqueios (_lock manager_) com o qual os processos se comunicam. O gerenciador de bloqueios controla o estado dos bloqueios.

A implementação de um _distributed locking_ é difícil de se conseguir porque há casos extremos a considerar tais como:

* Condições de corrida

* _Deadlocks_

* _Stale locks_

Uma solução para o _distributed lock_ involve gravar o atual detentor do bloqueio (_holder lock_), o processo que está impedindo os demais processos de executarem. Enquanto processos concorrentes tentam ser os próximos detentores do bloqueio. Uma desvantagem desta abordagem é que um processo poderia estar sem sorte enquanto outros seriam muito sortudos. O primeiro processo que não puder se tornar o _holder lock_, aguarda por um certo tempo, e após este tempo ele percebe que outro processo vai tomar a sua frente, impedindo esse processo inicial. Isto pode continuar indefinidamente, levando o processo a sofrer de starvation.

[Concurrency using optimistic locking](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.OptimisticLocking.html)
[Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
## Teoria: descrever sobre a segurança no BD, controle de acesso, concessão e revogação de privilégio, existência ou não de criptografia de dados;
[Database security](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/security.html)

<br>
===================================================
## Backup e Restauração
o ADDB oferece duas formas de back-up a partir da AWS, Point-in-time backup e On-demand backup, ambos descritos a seguir.

### Point-in-time backup

Este modo de backup oferece a opção de restaurar o banco de dados para qualquer ponto a partir do momento em que ele é ativado (até no máximo 35 dias). O processo restaura a tabela antiga em uma nova tabela. Vale ressaltar que todas as configurações da tabela se manterão iguais a tabela mais recente.

No exemplo abaixo ativa-se o Point-in-time backup para a tabela music:

```txt
aws dynamodb update-continuous-backups \
    --table-name Music \
    --point-in-time-recovery-specification PointInTimeRecoveryEnabled=True 
```
Logo após recupera-se o backup mais recente disponível(usualmente retorna os ultimos 5 minutos):

```txt
aws dynamodb restore-table-to-point-in-time \
    --source-table-name Music \
    --target-table-name MusicMinutesAgo \
    --use-latest-restorable-time
```

Também é possível recuperar para um horário específico:

```txt
aws dynamodb restore-table-to-point-in-time \
    --source-table-name Music \
    --target-table-name MusicEarliestRestorableDateTime \
    --no-use-latest-restorable-time \
    --restore-date-time 1519257118.0
```

### On-demand backup

É possível criar backups sob demanda com comandos como:

```txt
aws dynamodb create-backup --table-name Music \
 --backup-name MusicBackup
```

O backup on-demand é criado de maneira assíncrona, ò usuário ainda pode executar querys nas tabelas que estão sendo salvas, mas não é possível pausar ou cancelar a operação e remover a tabela.<br><br>
Após a criação do backup é possível listar todos backups:
```txt
aws dynamodb list-backups
```
E por fim é possível restaurar a tabela Music:

```txt
aws dynamodb restore-table-from-backup \
--target-table-name Music \
--backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Music/backup/01489173575360-b308cd7d 
```

<!--
Temos duas formas de fazer [backup](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BackupRestore.html). A [on-demand](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/backuprestore_HowItWorks.html) e a [point-in-time](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery_Howitworks.html).
A on-demand tem que fazer o [backup](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Backup.Tutorial.html) e o [restore](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Restore.Tutorial.html).
<<<<<<< HEAD
A point-int-time o backup é automatico e o [restore](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery.Tutorial.html) pode ser por console ou AWS CLI -->
=======

## Prática: instalação do BD localmente;


para instalar o aws-cli:

1) Tem que instalar o java jdk e jre>8
2) https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html

para configurar o aws cli
``` txt
aws configure
    AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
    AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    Default region name [None]: us-west-2
    Default output format [None]: json
```
pra rodar o servidor do banco localmente:
``` txt
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb
```

para listar as tabelas:
``` txt
aws dynamodb list-tables --endpoint-url http://localhost:8000
```

esse localhost tem que ter em todas as queries pq ta local

o banco ja tem um default que ele usa quando voce omite o sharedDb

para criar o tabela:
``` txt
aws dynamodb create-table
    --table-name Music
    --attribute-definitions
        AttributeName=Artist,AttributeType=S
        AttributeName=SongTitle,AttributeType=S
    --key-schema
        AttributeName=Artist,KeyType=HASH
        AttributeName=SongTitle,KeyType=RANGE
```

para fazer queries
``` txt
aws dynamodb batch-write-item --request-items file://Forum.json
```

``` JSON
{
    "Forum": [
        {
            "PutRequest": {
                "Item": {
                    "Name": {"S":"Amazon DynamoDB"},
                    "Category": {"S":"Amazon Web Services"},
                    "Threads": {"N":"2"},
                    "Messages": {"N":"4"},
                    "Views": {"N":"1000"}
                }
            }
        },
        {
            "PutRequest": {
                "Item": {
                    "Name": {"S":"Amazon S3"},
                    "Category": {"S":"Amazon Web Services"}
                }
            }
        }
    ]
}
```

## Referências

<ol>
<li>Amazon DynamoDB. What is Amazon DynamoDB. Disponível em: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html. Acessado em Abril 2021</li>
<li>Amazon DynamoDB. How it works. Disponível em: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html. Acessado em Abril 2021</li>
<li>Core Components of Amazon DynamoDB. Disponível em: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html. Acessado em Maio 2021</li>
<li>Medium. The Architecture of Amazon’s DynamoDB and Why Its Performance Is So High. Disponível em: https://medium.com/swlh/architecture-of-amazons-dynamodb-and-why-its-performance-is-so-high-31d4274c3129#:~:text=DynamoDB. Acesso em Abril 2021</li>
</ol>
