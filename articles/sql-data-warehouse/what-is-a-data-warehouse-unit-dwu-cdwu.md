---
title: Unidades do Data Wharehouse (DWUs, cDWUs) no SQL Data Warehouse do Azure | Microsoft Docs
description: Recomendações sobre como escolher o número ideal de unidades do data warehouse (DWUs, cDWUs) para otimizar o preço e o desempenho, e como alterar o número unidades.
services: sql-data-warehouse
author: ronortloff
manager: craigg-msft
ms.service: sql-data-warehouse
ms.topic: conceptual
ms.component: implement
ms.date: 04/17/2018
ms.author: rortloff
ms.reviewer: igorstan
ms.openlocfilehash: a83a9f9332d81e02a83efc019ad56027316301ab
ms.sourcegitcommit: 1362e3d6961bdeaebed7fb342c7b0b34f6f6417a
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/18/2018
---
# <a name="data-warehouse-units-dwus-and-compute-data-warehouse-units-cdwus"></a>DWUs (Unidades do Data Warehouse) cDWUs (Unidades do Data Warehouse de computação)
Recomendações sobre como escolher o número ideal de unidades do data warehouse (DWUs, cDWUs) para otimizar o preço e o desempenho, e como alterar o número unidades. 

## <a name="what-are-data-warehouse-units"></a>O que são Unidades de Data Warehouse?
Com a CPU do SQL Data Warehouse, a memória e e/s são empacotados em unidades de escala de computação, chamadas de Unidades de Data Warehouse (DWUs). Uma DWU representa uma medida abstrata normalizada de recursos de computação e desempenho. Ao alterar o nível de serviço, você altera a quantidade de DWUs que estão alocadas para o sistema, o que, por sua vez, ajusta o desempenho e o custo do seu sistema. 

Para pagar por um melhor desempenho, você pode aumentar o número de unidades de data warehouse. Para pagar por um desempenho menor, reduza as unidades de data warehouse. Os custos de armazenamento e computação são cobrados separadamente, para que as mudanças de unidades de data warehouse não afetem os custos de armazenamento.

O desempenho de unidades de data warehouse baseia-se nestas métricas de carga de trabalho do data warehouse:

- Com que velocidade uma consulta de data warehouse padrão verifica um grande número de linhas e, em seguida, executa uma agregação complexa? Essa operação tem uso intensivo de E/S e CPU.
- Com que velocidade o data warehouse pode incluir dados do Azure Storage Blobs ou do Azure Data Lake? Essa operação tem uso intensivo de rede e CPU. 
- Com que velocidade o comando T-SQL [CREATE TABLE AS SELECT](/sql/t-sql/statements/create-table-as-select-azure-sql-data-warehouse) copia uma tabela? Essa operação envolve a leitura de dados do armazenamento, distribuindo-os em todos os nós do dispositivo e gravando novamente no armazenamento. Essa operação é de uso intensivo de CPU, rede e E/S.

Aumento de DWUs:
- Altera de maneira linear o desempenho do sistema para verificações, agregações e instruções de CTAS
- Aumenta o número de leitores e gravadores para operações de carregamento do PolyBase
- Aumenta o número máximo de consultas simultâneas e slots de simultaneidade.

## <a name="service-level-objective"></a>Objetivo de nível de serviço
O Objetivo de nível de serviço (SLO) é a configuração de escalabilidade que determina o nível de custo e desempenho do data warehouse. Os níveis de serviço para a escala de nível de desempenho Otimizado para computação são medidos em unidades de Computação do data warehouse (cDWU), por exemplo DW2000c. O nível de serviço Otimizado para Elasticidade é medido em DWUs, por exemplo DW2000. 

No T-SQL, a configuração SERVICE_OBJECTIVE determina o nível de serviço e o nível de desempenho para o data warehouse.

```sql
--Optimized for Elasticity
CREATE DATABASE myElasticSQLDW
WITH
(    SERVICE_OBJECTIVE = 'DW1000'
)
;

--Optimized for Compute
CREATE DATABASE myComputeSQLDW
WITH
(    SERVICE_OBJECTIVE = 'DW1000c'
)
;
```

## <a name="performance-tiers-and-data-warehouse-units"></a>Níveis de desempenho e unidades de data warehouse

Cada nível de desempenho usa uma unidade de medida ligeiramente diferente para suas unidades de data warehouse. Essa diferença é refletida na fatura à medida que a unidade de escala é convertida diretamente para cobrança.

- O nível de desempenho otimizado para elasticidade é medido em unidades de data warehouse (DWUs).
- O nível de desempenho otimizado para computação é medido em unidades de data warehouse de computação (cDWUs). 

DWUs e cDWUs oferecem suporte ao dimensionamento vertical ou horizontal de computação e à pausa da computação quando você não precisar usar o data warehouse. Todas essas operações são sob demanda. O nível de desempenho otimizado para computação também usa um cache local baseado em disco em nós de computação para melhorar o desempenho. Quando você dimensiona ou pausa o sistema, o cache é invalidado e, portanto, é necessário um período de aquecimento de cache antes que o desempenho ideal seja obtido.  

À medida que você aumenta as unidades de data warehouse, estará aumentando linearmente os recursos de computação. O nível de desempenho otimizado para computação fornece o melhor desempenho de consulta e a escala mais alta, mas tem um preço de entrada mais alto. Ele é destinado a empresas que têm uma demanda constante por desempenho. Esses sistemas fazem o melhor uso do cache. 

### <a name="capacity-limits"></a>Limites de capacidade
Cada servidor SQL (por exemplo, myserver.database.windows.net) tem uma cota de [Unidade de Transação de Banco de Dados (DTU)](../sql-database/sql-database-what-is-a-dtu.md) que permite um número específico de unidades de depósito de dados. Para mais informações, consulte o [limites de capacidade de gerenciamento de carga de trabalho](sql-data-warehouse-service-capacity-limits.md#workload-management).


## <a name="how-many-data-warehouse-units-do-i-need"></a>De quantas unidades de data warehouse eu preciso?
O número ideal de unidades de data warehouse depende muito de sua carga de trabalho e da quantidade de dados carregados no sistema.

Etapas para encontrar a melhor DWU para sua carga de trabalho:

1. Durante o desenvolvimento, comece selecionando uma DWU menor usando o nível de desempenho otimizado para elasticidade.  Como a preocupação neste estágio é a validação funcional, o nível de desempenho otimizado para elasticidade é uma opção razoável. Um bom ponto de partida é DW200. 
2. Monitore o desempenho do seu aplicativo à medida que testa as cargas de dados no sistema, observando o número de DWUs selecionadas comparado ao desempenho que você observar.
3. Identifique quaisquer requisitos adicionais para períodos de atividade de pico temporários. Se a carga de trabalho demonstrar picos e ciclos significativos na atividade e houver um bom motivo para dimensionar com frequência, então, favoreça o nível de desempenho otimizado para elasticidade.
4. Se você precisar de mais de 1000 DWUs, então favoreça o nível de desempenho otimizado para computação já que ele oferece o melhor desempenho.

O SQL Data Warehouse é um sistema de dimensionamento horizontal que pode provisionar várias quantidades de dados dimensionáveis de computação e consulta. Para ver seus verdadeiros recursos para dimensionamento, especialmente em DWUs maiores, é recomendável dimensionar o conjunto de dados ao dimensionar para garantir que você tenha dados suficientes para alimentar as CPUs. Para testar o dimensionamento, é recomendável usar pelo menos 1 TB.

> [!NOTE]
>
> O desempenho de consulta só aumentará com mais paralelização se o trabalho puder ser dividido entre nós de computação. Se você achar que o dimensionamento não está alterando o desempenho, será necessário ajustar o design da tabela e/ou de suas consultas. Para obter diretrizes de ajuste de consulta, consulte [Gerenciar consultas de usuário](sql-data-warehouse-overview-manage-user-queries.md). 

## <a name="permissions"></a>Permissões

Alterar as unidades de data warehouse exige as permissões descritas em [ALTERAR BANCO DE DADOS](/sql/t-sql/statements/alter-database-transact-sql). 

## <a name="view-current-dwu-settings"></a>Exibir configurações atuais de DWU

Para exibir a configuração atual de DWU:

1. Abra o Pesquisador de Objetos do SQL Server no Visual Studio.
2. Conecte-se ao banco de dados mestre associado ao servidor lógico do Banco de Dados SQL.
3. Selecione do modo de exibição de gerenciamento dinâmico sys.database_service_objectives. Veja um exemplo: 

```sql
SELECT  db.name [Database]
,       ds.edition [Edition]
,       ds.service_objective [Service Objective]
FROM    sys.database_service_objectives   AS ds
JOIN    sys.databases                     AS db ON ds.database_id = db.database_id
;
```

## <a name="change-data-warehouse-units"></a>Alterar unidades do data warehouse

### <a name="azure-portal"></a>Portal do Azure
Para alterar DWUs ou cDWUs:

1. Abra o [Portal do Azure](https://portal.azure.com), abra seu banco de dados e clique em **Escala**.

2. Em **Escala**, mova o controle deslizante para a esquerda ou direita para alterar a configuração de DWU.

3. Clique em **Salvar**. Será exibida uma mensagem de confirmação. Clique em **sim** para confirmar ou em **não** para cancelar.

### <a name="powershell"></a>PowerShell
Para alterar as DWUs ou cDWUs, use o cmdlet do PowerShell [Set-AzureRmSqlDatabase](/powershell/module/azurerm.sql/set-azurermsqldatabase). O exemplo a seguir define o objetivo de nível de serviço como DW1000 para o banco de dados MySQLDW, que está hospedado no servidor MyServer.

```Powershell
Set-AzureRmSqlDatabase -DatabaseName "MySQLDW" -ServerName "MyServer" -RequestedServiceObjectiveName "DW1000"
```

Para obter mais informações, consulte [cmdlets do PowerShell para SQL Data Warehouse](sql-data-warehouse-reference-powershell-cmdlets.md)

### <a name="t-sql"></a>T-SQL
Com o T-SQL, é possível exibir as configurações atuais de DWU ou cDWU, alterar as configurações e verificar o progresso. 

Para alterar as DWUs ou as cDWUs:

1. Conecte-se ao banco de dados mestre associado ao seu servidor lógico do Banco de Dados SQL.
2. Use a instrução TSQL [ALTER DATABASE](/sql/t-sql/statements/alter-database-transact-sql). O exemplo a seguir define o objetivo de nível de serviço como DW1000 para o banco de dados MySQLDW. 

```Sql
ALTER DATABASE MySQLDW
MODIFY (SERVICE_OBJECTIVE = 'DW1000')
;
```

### <a name="rest-apis"></a>APIs REST

Para alterar as DWUs, use a API REST [Criar ou Atualizar Banco de Dados](/rest/api/sql/databases/createorupdate). O exemplo a seguir define o objetivo de nível de serviço como DW1000 para o banco de dados MySQLDW, que está hospedado no servidor MyServer. O servidor está em um grupo de recursos do Azure chamado ResourceGroup1.

```
PUT https://management.azure.com/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.Sql/servers/{server-name}/databases/{database-name}?api-version=2014-04-01-preview HTTP/1.1
Content-Type: application/json; charset=UTF-8

{
    "properties": {
        "requestedServiceObjectiveName": DW1000
    }
}
```

Para obter mais exemplos de API REST, consulte [APIs REST para o SQL Data Warehouse](sql-data-warehouse-manage-compute-rest-api.md).

## <a name="check-status-of-dwu-changes"></a>Verificar o status das alterações de DWU

As alterações de DWU podem levar vários minutos para concluir. Se você estiver dimensionando automaticamente, considere implementar a lógica para assegurar que determinadas operações tenham sido concluídas antes de prosseguir com outra ação. 

Verificar o estado do banco de dados por meio de vários pontos de extremidade permitirá que você implemente corretamente a automação. O portal fornece uma notificação após a conclusão de uma operação e o estado atual do bancos de dados, mas não permite a verificação de estado programática. 

Não é possível verificar o estado do banco de dados para operações de dimensionamento horizontal com o portal do Azure.

Para verificar o status de alterações de DWU:

1. Conecte-se ao banco de dados mestre associado ao seu servidor lógico do Banco de Dados SQL.
2. Envie a seguinte consulta para verificar o estado do banco de dados.


```sql
SELECT    *
FROM      sys.databases
;
```

3. Envie a seguinte consulta para verificar o estado da operação

```sql
SELECT    *
FROM      sys.dm_operation_status
WHERE     resource_type_desc = 'Database'
AND       major_resource_id = 'MySQLDW'
;
```

Esta DMV retorna informações sobre várias operações de gerenciamento no SQL Data Warehouse, como a operação e o estado da operação, que será IN_PROGRESS ou COMPLETED.

## <a name="the-scaling-workflow"></a>O fluxo de trabalho do dimensionamento

Quando você inicia uma operação de dimensionamento, o sistema primeiro elimina todas as sessões abertas, revertendo todas as transações abertas para garantir um estado consistente. Para operações de dimensionamento, o dimensionamento ocorrerá somente após a conclusão dessa reversão de transação.  

- Para uma operação de dimensionamento vertical, o sistema provisiona a computação adicional e, em seguida, anexa novamente à camada de armazenamento. 
- Para uma operação de redução vertical, os nós desnecessários são desanexados do armazenamento e anexados novamente nos nós restantes.

## <a name="next-steps"></a>Próximas etapas
Para saber mais sobre como gerenciamento do desempenho, consulte [Classes de recursos para gerenciamento de carga de trabalho](resource-classes-for-workload-management.md) e [Limites de simultaneidade e memória](memory-and-concurrency-limits.md).



