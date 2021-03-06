---
title: Recuperação de desastre para aplicativos SaaS usando backups com redundância geográfica do Banco de Dados SQL do Azure | Microsoft Docs
description: Saiba como usar backups com redundância geográfica do Banco de Dados SQL do Azure para recuperar um aplicativo SaaS multilocatário no caso de uma interrupção
keywords: tutorial do banco de dados SQL
services: sql-database
author: stevestein
manager: craigg
ms.service: sql-database
ms.custom: saas apps
ms.topic: article
ms.date: 04/16/2018
ms.author: ayolubek
ms.openlocfilehash: a677e6eb583e293f83df824804aa4cd6f8f5d778
ms.sourcegitcommit: 1362e3d6961bdeaebed7fb342c7b0b34f6f6417a
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/18/2018
---
# <a name="recover-a-multi-tenant-saas-application-using-geo-restore-from-database-backups"></a>Recuperar um aplicativo SaaS multilocatário usando a restauração geográfica de backups de banco de dados

Neste tutorial, você pode explorar um cenário de recuperação de desastre completo para um aplicativo SaaS multilocatário implementado usando o modelo de banco de dados por locatário. Você usa a [_restauração geográfica_](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-recovery-using-backups) para recuperar os bancos de dados do catálogo e de locatário de backups com redundância geográfica automaticamente mantidos em uma região de recuperação alternativa. Depois que a interrupção for resolvida, você usará a [_replicação geográfica_](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-geo-replication-overview) para repatriar bancos de dados alterados para a região original deles.

![geo-restore-architecture](media/saas-dbpertenant-dr-geo-restore/geo-restore-architecture.png)

A restauração geográfica é a solução de recuperação de desastre de menor custo para o Banco de Dados SQL.  No entanto, a restauração de backups com redundância geográfica pode resultar em perda de dados de até uma hora e pode levar um tempo considerável, dependendo do tamanho de cada banco de dados. **Para recuperar os aplicativos com RTO e RPO mais baixos possíveis, use a replicação geográfica em vez da restauração geográfica**.

Este tutorial explora os fluxos de trabalho de restauração e repatriação. Você aprenderá como:
> [!div class="checklist"]

>* Sincronizar as informações de configuração de pool elástico e de banco de dados no catálogo de locatário
>* Configurar um ambiente de espelho em uma região de ‘recuperação’, que inclui aplicativos, servidores e pools    
>* Recuperar bancos de dados de catálogo e de locatário usando a _restauração geográfica_
>* Repatriar o catálogo de locatário e os bancos de dados de locatário alterados usando a _replicação geográfica_ depois que a interrupção for resolvida
>* Atualizar o catálogo à medida que cada banco de dados é restaurado (ou repatriado) para controlar a localização atual da cópia ativa do banco de dados de cada locatário
>* Garantir que o banco de dados de aplicativos e de locatário estejam sempre co-localizados na mesma região do Azure para reduzir a latência  
 

Antes de iniciar este tutorial, verifique se todos os pré-requisitos a seguir foram atendidos:
* O aplicativo de banco de dados SAAS Wingtip Tickets por locatário é implantado. Para implantá-lo em menos de cinco minutos, veja [Implantar e explorar o aplicativo de banco de dados por locatário SaaS Wingtip Tickets](saas-dbpertenant-get-started-deploy.md)  
* O Azure PowerShell está instalado. Para obter detalhes, consulte [Introdução ao Azure PowerShell](https://docs.microsoft.com/powershell/azure/get-started-azureps)

## <a name="introduction-to-the-geo-restore-recovery-pattern"></a>Introdução ao padrão de recuperação de restauração geográfica

A recuperação de desastre (DR) é uma consideração importante para muitos aplicativos, seja por motivos de conformidade ou de continuidade de negócios. Se for necessária uma interrupção prolongada do serviço, um plano de recuperação de desastre bem preparado pode minimizar a interrupção dos negócios. Um plano de recuperação de desastre na restauração geográfica deve realizar várias metas:
 * Reserve toda a capacidade necessária na região de recuperação escolhida assim que possível para garantir que ela esteja disponível para restaurar bancos de dados de locatário.
 * Estabelecer um ambiente de recuperação de imagem espelhada que reflete a configuração de pool e de banco de dados original 
 * Deve ser possível cancelar o processo de restauração em funcionamento se a região original ficar online novamente.
 * Habilitar rapidamente o provisionamento de locatário de forma que a integração do novo locatário possa reiniciar assim que possível  
 * Ser otimizado para restauração de locatários em ordem de prioridade
 * Ser otimizado para colocar os locatários online assim que possível ao seguir as etapas em paralelo quando for prático
 * Ser resiliente a falhas, reiniciável e idempotente
 * Repatrie bancos de dados para a região original deles com impacto mínimo nos locatários quando a interrupção for resolvida.  

> [!Note]
> O aplicativo é recuperado para o _região emparelhada_ da região em que o aplicativo é implantado. Para saber mais, veja [Regiões emparelhadas do Azure](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions).   



Neste tutorial, esses desafios são resolvidos usando os recursos do Banco de Dados SQL do Azure e a plataforma Azure:

* [Modelos do Azure Resource Manager](https://docs.microsoft.com/azure/azure-resource-manager/resource-manager-create-first-template) para reservar toda a capacidade necessária assim que possível. Os modelos do Azure Resource Manager são usados para provisionar uma imagem espelho dos servidores originais e pools elásticos na região de recuperação. Um servidor e um pool separados também são criados para provisionar novos locatários. 
* [Biblioteca de Cliente de Banco de Dados Elástico](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-elastic-database-client-library) (EDCL) para criar e manter um catálogo de banco de dados de locatário.  O catálogo é estendido para incluir as informações de configuração periodicamente atualizadas do banco de dados e do pool.
* [Recursos de recuperação de gerenciamento de fragmentos](https://docs.microsoft.com/azure/sql-database/sql-database-elastic-database-recovery-manager) da EDCL para manter as entradas de localização de banco de dados no catálogo durante a recuperação e a repatriação.  
* A [restauração geográfica](https://docs.microsoft.com/azure/sql-database/sql-database-disaster-recovery) para recuperar os bancos de dados do catálogo e de locatário de backups com redundância geográfica mantidos automaticamente. 
* [Operações de restauração assíncrona](https://docs.microsoft.com/azure/azure-resource-manager/resource-manager-async-operations) enviadas na ordem de prioridade de locatário, que são enfileiradas para cada pool pelo sistema e processadas em lotes para o pool não ficar sobrecarregado. Essas operações podem ser canceladas antes ou durante a execução, se necessário.    
* [Replicação geográfica](https://docs.microsoft.com/azure/sql-database/sql-database-geo-replication-overview) para repatriar os bancos de dados para a região original após a interrupção. O uso da replicação geográfica garante que não haja nenhuma perda de dados e tenha um impacto mínimo sobre o locatário.
* [Aliases DNS do SQL server](https://docs.microsoft.com/azure/sql-database/dns-alias-overview) para permitir que o processo de sincronização de catálogo para se conectar ao catálogo ativo, independentemente de sua localização.  

## <a name="get-the-disaster-recovery--scripts"></a>Obter os scripts de recuperação de desastre 

Os scripts de DR usados neste tutorial estão disponíveis no [banco de dados SaaS Wingtip Tickets por repositório do GitHub](https://github.com/Microsoft/WingtipTicketsSaaS-DbPerTenant). Confira as [diretrizes gerais](saas-tenancy-wingtip-app-guidance-tips.md) para obter as etapas a fim de baixar e desbloquear os scripts de gerenciamento do Wingtip Tickets.
> [!IMPORTANT]
> Como todos os scripts de gerenciamento da Wingtip Tickets, os scripts de recuperação de desastre têm qualidade de exemplo e não devem ser usados em produção.   

## <a name="review-the-healthy-state-of-the-application"></a>Examinar o estado de integridade do aplicativo
Antes de iniciar o processo de recuperação, examine o estado de integridade normal do aplicativo.
1. No navegador da Web, abra o Hub de Eventos da Wingtip Tickets (http://events.wingtip-dpt.&lt;usuário&gt;.trafficmanager.net - replace &lt;usuário&gt; com o valor do usuário da sua implantação).
    * Role até a parte inferior da página e observe o nome do servidor de catálogo e a localização no rodapé.  A localização é a região em que você implantou o aplicativo.    
        DICA: passe o mouse sobre a localização para ampliar a exibição.

    ![Estado de integridade do hub de eventos na região original](media/saas-dbpertenant-dr-geo-restore/events-hub-original-region.png)

1. Clique no locatário Contoso Concert Hall e abra sua página de eventos.
    * No rodapé, observe o nome do servidor de locatários. A localização será igual à localização do servidor de catálogo.

    ![Região original do Contoso Concert Hall](media/saas-dbpertenant-dr-geo-restore/contoso-original-location.png)    
1. No [portal do Azure](https://portal.azure.com), examine e abra o grupo de recursos no qual você implantou o aplicativo.
    * Observe os recursos e a região na qual os componentes de serviço de aplicativo e os servidores do Banco de Dados SQL são implantados.

## <a name="sync-tenant-configuration-into-catalog"></a>Sincronizar a configuração de locatário com o catálogo

Nesta tarefa, você inicia um processo para sincronizar a configuração dos servidores, dos pools elásticos e dos bancos e dados com o catálogo de locatário.  Essas informações são usadas posteriormente para configurar um ambiente de espelho na região de recuperação.

> [!IMPORTANT]
> Para simplificar, o processo de sincronização e outros processos de recuperação e de repatriação de longa execução são implementados nesses exemplos como trabalhos ou sessões locais do Powershell executados em seu logon de usuário de cliente. Os tokens de autenticação emitidos quando seu logon expira após várias horas e então os trabalhos falham. Em um cenário de produção, os processos de execução longa devem ser implementados como serviços do Azure confiáveis de algum tipo, em execução sob uma entidade de serviço. Consulte [Usar o Azure PowerShell para criar uma entidade de serviço com um certificado](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal). 

1. No _ISE do PowerShell_, abra o arquivo ...\Learning Modules\UserConfig.psm1. Substitua `<resourcegroup>` e `<user>` nas linhas 10 e 11 pelo valor usado quando você implantou o aplicativo.  Salve o arquivo!

2. No *ISE do PowerShell*, abra o script Modules\Business continuidade e desastres ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1.
    *  Durante este tutorial, você executará cada um dos cenários neste script do PowerShell, portanto, mantenha esse arquivo aberto.

3. Defina o seguinte:
    * **$DemoScenario = 1**, Iniciar um trabalho em segundo plano que sincroniza o servidor de locatário e informações de configuração do pool com o catálogo

3. Pressione **F5** para executar o script de sincronização. 
    *  Essas informações serão usadas posteriormente para garantir que a recuperação crie uma imagem espelhada de pools, servidores e bancos de dados na região de recuperação.  
![Processo de sincronização](media/saas-dbpertenant-dr-geo-restore/sync-process.png)

Deixe a janela do PowerShell em execução em segundo plano e prossiga com o restante deste tutorial.

> [!Note]
> O processo de sincronização se conecta ao catálogo por um alias DNS. O alias é modificado durante a restauração e a repatriação para apontar para o catálogo ativo. O processo de sincronização mantém o catálogo atualizado com todas as alterações do banco de dados ou do pool feitas na região de recuperação.  Durante a repatriação, essas alterações são aplicadas aos recursos equivalentes na região original.

## <a name="geo-restore-recovery-process-overview"></a>Visão geral do processo de recuperação de restauração geográfica

O processo de recuperação de restauração geográfica implanta o aplicativo e restaura os bancos de dados de backups para a área de recuperação.

O processo de recuperação faz o seguinte:

1. Desabilita o ponto de extremidade do Gerenciador de Tráfego para o aplicativo Web na região original. Desabilitar o ponto de extremidade impede que os usuários se conectem ao aplicativo em um estado inválido caso a região original fique online durante a recuperação.

1. Provisiona um servidor de catálogo de recuperação na região de recuperação, restaura geograficamente o banco de dados do catálogo e atualiza o alias _activecatalog_ para apontar para o servidor de catálogo restaurado.  
    * A alteração do alias do catálogo garante que o processo de sincronização de catálogo sempre sincronize com o catálogo ativo.

1. Marca todos os locatários existentes no catálogo de recuperação como offline para impedir o acesso a bancos de dados de locatário antes da restauração.

1. Provisiona uma instância do aplicativo na região de recuperação e o configura para usar o catálogo restaurado nessa região.
    * Para manter a latência mínima, o aplicativo de exemplo foi projetado para sempre se conectar a um banco de dados de locatário na mesma região.

1. Provisiona um pool elástico e de servidores no qual novos locatários serão provisionados. A criação desses recursos garante que o provisionamento de novos locatários não interfira na recuperação de locatários existentes.

1. Atualiza o alias new-tenant para apontar para o servidor para novos bancos de dados de locatário na região de recuperação. A alteração desse alias garante que os bancos de dados para quaisquer novos locatários sejam provisionados na região de recuperação.
        
1. Provisiona pools elásticos e de servidores na região de recuperação para restaurar os bancos de dados de locatário. Esses servidores e pools são uma imagem espelhada da configuração na região original.  O provisionamento antecipado de pools reserva a capacidade necessária para restaurar todos os bancos de dados.
    * Uma interrupção em uma região pode gerar uma pressão significativa nos recursos disponíveis na região emparelhada.  Se você confia na restauração geográfica para recuperação de desastre, então é recomendável reservar recursos rapidamente. Considere o uso de replicação geográfica se for muito importante que um aplicativo seja recuperado em uma região específica. 

1. Habilita o ponto de extremidade do Gerenciador de Tráfego para o aplicativo Web na região de recuperação. Habilitar este ponto de extremidade permite que o aplicativo provisione novos locatários. Neste estágio, os locatários existentes ainda estão offline.

1. Envia os lotes de solicitações para restaurar bancos de dados em ordem de prioridade. 
    * Os lotes são organizados para que os bancos de dados sejam restaurados em paralelo entre todos os pools.  
    * As solicitações de restauração são enviadas de forma assíncrona para que sejam enviadas rapidamente e enfileiradas para execução em cada pool.
    * Como as solicitações de restauração são processadas em paralelo entre todos os pools, é melhor distribuir locatários importantes entre vários pools. 

1. Monitora o serviço de banco de dados SQL para determinar quando os bancos de dados são restaurados. Quando um banco de dados do locatário for restaurado, ele será marcado como online no catálogo e uma soma rowversion do banco de dados de locatário é registrada. 
    * Os bancos de dados de locatário podem ser acessados pelo aplicativo assim que são marcados como online no catálogo.
    * A soma dos valores de rowversion no banco de dados de locatário é armazenada no catálogo. Essa soma atua como uma impressão digital, que permite que o processo de repatriação determine se o banco de dados foi atualizado na região de recuperação.      

## <a name="run-the-recovery-script"></a>Executar o script de recuperação

> [!IMPORTANT]
> Este tutorial restaura os bancos de dados de backups com redundância geográfica. Embora esses backups geralmente fiquem disponíveis em 10 minutos, pode demorar até uma hora para que sejam disponibilizados. O script fará uma pausa até que eles estejam disponíveis.  É hora do intervalo!

Agora imagine que haja uma interrupção na região em que o aplicativo é implantado e execute o script de recuperação:

1. No *ISE do PowerShell*, no script ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1, defina os seguintes valores:
    * **$DemoScenario = 2**, Recuperar o aplicativo para uma região de recuperação por meio da restauração de backups com redundância geográfica

1. Pressione **F5** para executar o script.  
    * O script é aberto em uma nova janela do PowerShell e, em seguida, inicia um conjunto de trabalhos do PowerShell executados em paralelo.  Esses trabalhos restauram servidores, pools e bancos de dados para a região de recuperação. 
    * A região de recuperação é a _região emparelhada_ associada à região do Azure na qual você implantou o aplicativo. Para saber mais, veja [Regiões emparelhadas do Azure](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions). 

1. Monitore o status do processo de recuperação na janela do PowerShell.

    ![processo de recuperação](media/saas-dbpertenant-dr-geo-restore/dr-in-progress.png)

>Para explorar o código para os trabalhos de recuperação, examine os scripts do PowerShell na pasta ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\RecoveryJobs.

## <a name="review-the-application-state-during-recovery"></a>Examinar o estado do aplicativo durante a recuperação
Enquanto o ponto de extremidade do aplicativo estiver desabilitado no Gerenciador de Tráfego, o aplicativo não estará disponível. O catálogo é restaurado e todos os locatários são marcados como offline.  O ponto de extremidade do aplicativo na região de recuperação é então habilitado e o aplicativo volta a ficar online. Embora o aplicativo esteja disponível, os locatários aparecem offline no hub de eventos até que seus bancos de dados sejam restaurados. É importante projetar seu aplicativo para que ele lide com bancos de dados de locatário offline.

1. Depois que o banco de dados do catálogo tiver sido recuperado, mas antes de os locatários ficarem online novamente, atualize o Hub de Eventos da Wingtip Tickets no navegador da Web.
    * No rodapé, observe que o nome do servidor de catálogo agora tem um sufixo _-recovery_ e está localizado na região de recuperação.
    * Observe que os locatários que ainda não foram restaurados estão marcados como offline e não são selecionáveis.   
 
    ![processo de recuperação](media/saas-dbpertenant-dr-geo-restore/events-hub-tenants-offline-in-recovery-region.png)    

    * Se você abrir a página Eventos de um locatário diretamente enquanto o locatário estiver offline, a página exibirá uma notificação de 'locatário offline'. Por exemplo, se Contoso Concert Hall estiver offline, tente abrir http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net/contosoconcerthall ![processo de recuperação](media/saas-dbpertenant-dr-geo-restore/dr-in-progress-offline-contosoconcerthall.png)

## <a name="provision-a-new-tenant-in-the-recovery-region"></a>Provisionar um novo locatário na região de recuperação
Antes mesmo da restauração dos bancos de dados de locatário, você poderá provisionar novos locatários na região de recuperação. Os novos bancos de dados de locatário provisionados na região de recuperação serão repatriados com os bancos de dados recuperados posteriormente.   

1. No *ISE do PowerShell*, no script ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1, defina esta propriedade:
    * **$DemoScenario = 3**, Provisionar um novo locatário na região de recuperação

1. Pressione **F5** para executar o script. 

1. A página de eventos de Hawthorn Hall abre no navegador quando o provisionamento é concluído. 
    * Observe que o banco de dados Hawthorn Hall está localizado na região de recuperação.
    ![Hawthorn Hall provisionado na região de recuperação](media/saas-dbpertenant-dr-geo-restore/hawthorn-hall-provisioned-in-recovery-region.png)

1. No navegador, atualize a página do Hub de Eventos da Wingtip Tickets para ver Hawthorn Hall incluído. 
    * Se você tiver provisionado Hawthorn Hall sem aguardar a restauração dos outros locatários, outros locatários ainda poderão estar offline.

## <a name="review-the-recovered-state-of-the-application"></a>Examinar o estado recuperado do aplicativo

Quando o processo de recuperação for concluída, o aplicativo e todos os locatários estarão totalmente funcionais na região de recuperação. 

1. Depois que a exibição na janela do console do PowerShell indicar que todos os locatários foram recuperados, atualize o Hub de Eventos.  Os locatários aparecerão online, inclusive o novo locatário, Hawthorn Hall.

    ![locatários novos e recuperados no hub de eventos](media/saas-dbpertenant-dr-geo-restore/events-hub-with-hawthorn-hall.png)

1. Clique em Contoso Concert Hall e abra sua página Eventos.
    * No rodapé, observe que o banco de dados está localizado no servidor de recuperação localizado na região de recuperação.

    ![Contoso na região de recuperação](media/saas-dbpertenant-dr-geo-restore/contoso-recovery-location.png)

1. No [portal do Azure](https://portal.azure.com), abra a lista de grupos de recursos.  
    * Observe o grupo de recursos que você implantou, mais o grupo de recursos de recuperação, com o sufixo _-recovery_.  O grupo de recursos de recuperação contém todos os recursos criados durante o processo de recuperação, além de novos recursos criados durante a interrupção.  

1. Abra o grupo de recursos de recuperação e observe os seguintes itens:
    * As versões de recuperação dos servidores de catálogo e locatários1 com o sufixo _-recovery_.  Os bancos de dados restaurados de catálogo e de locatário nesses servidores têm os nomes usados na região original.

    * O servidor SQL _tenants2-dpt-&lt;user&gt;-recovery_.  Este servidor é usado para provisionar novos locatários durante a interrupção.
    *   O Serviço de Aplicativo chamado _events-wingtip-dpt-&lt;recoveryregion&gt;-&lt;usuário&gt_;, que é a instância de recuperação do aplicativo Eventos. 

    ![Contoso na região de recuperação](media/saas-dbpertenant-dr-geo-restore/resources-in-recovery-region.png)   
    
1. Abra o servidor SQL _tenants2-dpt-&lt;usuário&gt;-recovery_.  Observe que ele contém o banco de dados _hawthornhall_ e o pool elástico, _Pool1_.  O banco de dados _hawthornhall_ está configurado como um banco de dados elástico no pool elástico _Pool1_.

## <a name="change-tenant-data"></a>Alterar dados de locatário 
Nesta tarefa, você atualiza bancos de dados de locatário restaurados. O processo de repatriação copiará bancos de dados restaurados que foram alterados para a região original. 

1. No navegador, encontre a lista de eventos para o Contoso Concert Hall e anote o nome do último evento, _Seriously Strauss_.

1. No *ISE do PowerShell*, no script ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1, defina o seguinte valor:
    * **$DemoScenario = 4**, Excluir um evento de um locatário na região de recuperação
1. Pressione **F5** para executar o script.
1. Atualize a página de eventos de Contoso Concert Hall (http://events.wingtip-dpt.&lt;usuário&gt;.trafficmanager.net/contosoconcerthall) e observe que o evento Seriously Strauss está ausente.

Nesse ponto no tutorial, você recuperou o aplicativo, que está em execução na região de recuperação.  Você provisionou um novo locatário na região de recuperação e modificou dados de um dos locatários restaurados.  

> [!Note]
> Outros tutoriais no exemplo não são projetados para funcionar com o aplicativo no estado de recuperação. Se você quiser explorar outros tutoriais, repatrie o aplicativo primeiro.

## <a name="repatriation-process-overview"></a>Visão geral do processo de repatriação

O processo de repatriação reverte o aplicativo e seus bancos de dados para sua região original após uma interrupção for resolvida.

![Repatriação da restauração geográfica](media/saas-dbpertenant-dr-geo-restore/geo-restore-repatriation.png) 


O processo:

1. Interrompe qualquer atividade de restauração em andamento e cancela quaisquer solicitações de restauração de banco de dados em andamento ou pendentes.

1. Reativa os bancos de dados de locatário da região original que não foram alterados desde a interrupção.  Isso inclui bancos de dados que ainda não foram recuperados e os bancos de dados que foram recuperados, mas não alterados posteriormente. Os bancos de dados reativados estão exatamente da mesma forma como da última vez em que foram acessados por seus locatários.

1. Provisiona uma imagem espelhada do novo pool elástico e de servidores de locatários na região original. Após a conclusão dessa ação, o alias new-tenant é atualizado para apontar para esse servidor. Atualizar o alias faz com que a nova integração de locatário ocorra na região original em vez de na região de recuperação.

1. Usando a replicação geográfica, move o catálogo para a região original desde a região de recuperação.

1. Atualiza a configuração do pool na região original para que ela fique consistente com as alterações feitas na região de recuperação durante a interrupção.

1. Cria os servidores e os pools necessários para hospedar qualquer banco de dados novo criado durante a interrupção.

1. Usando a replicação geográfica, repatria bancos de dados de locatário restaurados que foram atualizados após a restauração e todos os novos bancos de dados de locatário provisionados durante a interrupção. 

1. Limpa os recursos criados na região de recuperação durante o processo de restauração.

Para limitar o número de bancos de dados de locatário que precisam ser repatriados, as etapas 1 e 3 são concluídas imediatamente.  

A etapa 4 só será concluída se o catálogo da região de recuperação tiver sido modificado durante a interrupção. O catálogo é atualizado se novos locatários são criados ou se qualquer configuração de banco de dados ou do pool é alterada na região de recuperação.

É importante que a etapa 7 cause o mínimo de interrupção para os locatários e nenhum dado seja perdido. Para alcançar essa meta, o processo usa a _replicação geográfica_.

Antes da replicação geográfica de cada banco de dados, o banco de dados correspondente na região original é excluído. É feita então a replicação geográfica do banco de dados da região de recuperação, criando uma réplica secundária na região original. Após a conclusão da replicação, o locatário é marcado como offline no catálogo, o que interrompe todas as conexões com o banco de dados na região de recuperação. Então é feito o failover do banco de dados, fazendo com que qualquer transação pendente seja processada no secundário, para que nenhum dado seja perdido. No failover, as funções de banco de dados são revertidas.  O secundário na região original se torna o banco de dados principal de leitura/gravação, e o banco de dados na região de recuperação se torna um secundário somente leitura. A entrada de locatário no catálogo é atualizada para referenciar o banco de dados na região original e o locatário é marcado como online.  Neste ponto, a repatriação do banco de dados foi concluída. 

Os aplicativos devem ser escritos com lógica de repetição para garantir que eles se reconectem automaticamente quando as conexões forem interrompidas.  Quando eles se reconectam usando o catálogo para interromper a conexão, se conectam ao banco de dados repatriado na região original. Embora essa breve desconexão geralmente não seja observada, você pode optar por repatriar bancos de dados fora do horário comercial. 

Depois que um banco de dados é repatriado, o banco de dados secundário na região de recuperação pode ser excluído. O banco de dados na região original, então, depende da restauração geográfica para ter a proteção de recuperação de desastre novamente.

Na etapa 8, os recursos na região de recuperação, incluindo os servidores de recuperação e pools, são excluídos.

## <a name="run-the-repatriation-script"></a>Executar o script repatriação
Agora vamos imaginar que a interrupção foi resolvida e o script de repatriação está sendo executado.

Se você seguiu o tutorial, o script reativa imediatamente o Fabrikam Jazz club e o Dogwood Dojo na região original porque eles não foram modificados. Em seguida, ele repatria o novo locatário, Hawthorn Hall, e o Contoso Concert Hall porque ele foi modificado. O script também repatria o catálogo, que foi atualizado quando Hawthorn Hall foi provisionado.
  
1. No *ISE do PowerShell*, no script ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1.

1. Verifique se o processo de Sincronização de Catálogo ainda está em execução em sua instância do PowerShell.  Se necessário, reinicie-o ao definir:
    * **$DemoScenario = 1**, Iniciar a sincronização do servidor locatário, o pool e informações de configuração do banco de dados para o catálogo
    * Pressione **F5** para executar o script.
1.  Então, para iniciar o processo de repatriação, defina:
    * **$DemoScenario = 5**, Repatriar o aplicativo para sua região original
    * Pressione **F5** para executar o script de recuperação em uma nova janela do PowerShell.  A repatriação levará vários minutos e pode ser monitorada na janela do PowerShell.
1. Durante a execução do script, atualize a página de Hub de eventos (http://events.wingtip-dpt.&lt;usuário&gt;.trafficmanager.net)
    * Observe que todos os locatários estão online e acessíveis durante este processo.
1. Clique no Fabrikam Jazz Club para abri-lo. Se você não tiver modificado esse locatário, observe no rodapé de página que o servidor já foi revertido para o servidor original.
1. Abra ou atualize a página de eventos do Contoso Concert Hall e observe no rodapé que o banco de dados é ainda está no servidor _-recovery_ inicialmente.  
1. Atualize a página de eventos do Contoso Concert Hall quando concluir o processo de repatriação e observe que o banco de dados está em sua região original.
1. Atualize o Hub de Eventos novamente para abrir o Hawthorn Hall e observe que o seu banco de dados também está localizado na região original. 

## <a name="clean-up-recovery-region-resources-after-repatriation"></a>Limpar os recursos de região de recuperação após a repatriação
Após a conclusão da repatriação, é seguro excluir os recursos da região de recuperação. 

> [!IMPORTANT]
> Exclua esses recursos imediatamente para interromper todas as cobranças sobre eles.

O processo de restauração cria todos os recursos de recuperação em um grupo de recursos de recuperação. O processo de limpeza excluirá esse grupo de recursos e removerá todas as referências aos recursos do catálogo. 

1. No *ISE do PowerShell*, no script ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1, defina:
    * **$DemoScenario = 6**, Excluir recursos obsoletos da região de recuperação

1. Pressione **F5** para executar o script.

Depois de limpar os scripts, o aplicativo estará de volta ao início.  Você pode executar o script novamente ou experimentar outros tutoriais neste ponto.

## <a name="designing-the-application-to-ensure-app-and-database-are-colocated"></a>Como projetar o aplicativo para garantir que ele e o banco de dados sejam colocados 
O aplicativo foi projetado para sempre se conectar de uma instância na mesma região do banco de dados de locatário. Esse design reduz a latência entre o aplicativo e o banco de dados. Essa otimização assume que a interação do aplicativo no banco de dados é mais ativa que a interação do usuário com o aplicativo.  

Os bancos de dados de locatário podem ser distribuídos por regiões originais e de recuperação por algum tempo durante a repatriação.  Para cada banco de dados, o aplicativo procura a região na qual o banco de dados está localizado, fazendo uma pesquisa de DNS no nome do servidor de locatário. No Banco de Dados SQL, o nome do servidor é um alias. O nome de servidor com alias contém o nome da região. Se o aplicativo não estiver na mesma região do que o banco de dados, ele redirecionará para a instância na mesma região que o servidor de banco de dados.  Redirecionar a instância na mesma região que o banco de dados minimiza a latência entre o aplicativo e o banco de dados.  

## <a name="next-steps"></a>Próximas etapas

Neste tutorial, você aprendeu a:
> [!div class="checklist"]

>* Use o catálogo de locatário para manter informações de configuração atualizadas periodicamente, permitindo que um ambiente de recuperação de imagem espelhada seja criado em outra região
>* Use a restauração geográfica para recuperar bancos de dados SQL do Azure para a área de recuperação
>* Atualizar o catálogo de locatário para refletir os locais do banco de dados de locatário restaurados  
>* Use um alias DNS para permitir que um aplicativo se conecte ao catálogo de locatário sem reconfiguração
>* Use a replicação geográfica para repatriar os bancos de dados recuperados para a região original deles após a resolução da interrupção

Agora, experimente a [Recuperação de desastre para um aplicativo SaaS multilocatário usando a replicação geográfica de banco de dados](saas-dbpertenant-dr-geo-replication.md) para saber como reduzir drasticamente o tempo necessário para recuperar um aplicativo multilocatário em grande escala usando a replicação geográfica.

## <a name="additional-resources"></a>Recursos adicionais

* [Tutoriais adicionais que aproveitam o aplicativo de SaaS do Wingtip](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-wtp-overview#sql-database-wingtip-saas-tutorials)
