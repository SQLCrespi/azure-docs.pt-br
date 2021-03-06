---
title: Publicando um serviço de nuvem usando as ferramentas do Azure | Microsoft Docs
description: Saiba mais sobre como publicar projetos de serviço de nuvem do Azure usando o Visual Studio.
services: visual-studio-online
author: ghogen
manager: douge
assetId: 1a07b6e4-3678-4cbf-b37e-4520b402a3d9
ms.prod: visual-studio-dev15
ms.technology: vs-azure
ms.workload: azure
ms.topic: conceptual
ms.date: 11/11/2017
ms.author: ghogen
ms.openlocfilehash: bde00dfbf4a7ffde90d1a9a3d57d3a2decf74cad
ms.sourcegitcommit: fa493b66552af11260db48d89e3ddfcdcb5e3152
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/23/2018
---
# <a name="publishing-a-cloud-service-using-visual-studio"></a>Publicação de um serviço de nuvem usando o Visual Studio

O Visual Studio pode publicar um aplicativo diretamente no Azure, com suporte para ambientes de preparo e produção de um serviço de nuvem. Durante a publicação, selecione o ambiente de implantação e uma conta de armazenamento usada temporariamente para o pacote de implantação.

Durante o desenvolvimento e teste de um aplicativo do Azure, use a Implantação da Web para publicar alterações de forma incremental nas funções web. Depois de publicar seu aplicativo para um ambiente de implantação, a Implantação da Web permite implantar alterações diretamente na máquina virtual que está executando a função Web. Você não precisa empacotar e publicar o aplicativo do Azure inteiro sempre que desejar atualizar a função Web para testar as alterações. Com essa abordagem, você pode ter suas alterações de função da Web disponíveis na nuvem para testes sem aguardar a publicação do aplicativo em um ambiente de implantação.

Use os procedimentos a seguir para publicar seu aplicativo do Azure e atualizar uma função Web usando a implantação da Web:

- Publicar ou empacotar um aplicativo do Azure do Visual Studio
- Atualizar uma função Web como parte do ciclo de desenvolvimento e teste

## <a name="publish-or-package-an-azure-application-from-visual-studio"></a>Publicar ou empacotar um aplicativo do Azure do Visual Studio

Ao publicar seu aplicativo do Azure, você pode executar uma das seguintes tarefas:

- Criar um pacote de serviço: você pode usar esse pacote e o arquivo de configuração de serviço para publicar seu aplicativo para um ambiente de implantação do [portal do Azure](https://portal.azure.com).

- Publicar seu projeto do Azure do Visual Studio: para publicar seu aplicativo diretamente no Azure, use o Assistente de Publicação. Para obter informações, consulte [Assistente de Publicação de Aplicativo do Azure](vs-azure-tools-publish-azure-application-wizard.md).

### <a name="to-create-a-service-package-from-visual-studio"></a>Para criar um pacote de serviço do Visual Studio

1. Quando estiver pronto para publicar seu aplicativo, abra o Gerenciador de Soluções, abra o menu de atalho do projeto do Azure que contém suas funções e selecione Publicar.

1. Para criar um pacote de serviço, siga estas etapas:

   a. No menu de atalho do projeto do Azure, escolha **Pacote**.

   b. Na caixa de diálogo **Empacotar Aplicativo do Azure** , escolha a configuração de serviço para a qual você deseja criar um pacote e, em seguida, escolha a configuração de compilação.

   c. (Opcional) Para ativar a Área de Trabalho Remota para o serviço de nuvem depois de publicá-lo, selecione **Habilitar Área de Trabalho Remota para todas as Funções** e, em seguida, **Configurações** para configurar as credenciais da Área de Trabalho Remota. Para saber mais, veja [Habilitar Conexão de Área de Trabalho Remota para uma função nos Serviços de Nuvem do Azure usando o Visual Studio](cloud-services/cloud-services-role-enable-remote-desktop-visual-studio.md).

      Se você quiser depurar seu serviço de nuvem após publicá-lo, ative a depuração remota, selecionando **Ativar o Depurador Remoto para todas as Funções**.

   d. Para criar o pacote, escolha o link **pacote** .

      O Explorador de Arquivos mostra o local do arquivo do pacote recém-criado. Você pode copiar esse local para que possa usá-lo do portal do Azure.

   e. Para publicar esse pacote em um ambiente de implantação, você deve usar esse local como o local do pacote ao criar um serviço de nuvem e implantar esse pacote em um ambiente com o portal do Azure.

1. (Opcional) Para cancelar o processo de implantação, no menu de atalho do item de linha no log de atividades, escolha **Cancelar e remover**. Esse comando interrompe o processo de implantação e exclui o ambiente de implantação do Azure. Para remover o ambiente após a implantação, use o portal do Azure.

1. (Opcional) Após ter iniciado as instâncias de função, o Visual Studio mostra automaticamente o ambiente de implantação no nó **Serviços de Nuvem** no Gerenciador de Servidores. Nele, você pode ver o status das instâncias de função individuais. Confira [Gerenciando recursos de Azure com o Cloud Explorer](vs-azure-tools-resources-managing-with-cloud-explorer.md). A ilustração a seguir mostra as instâncias de função enquanto elas ainda estão no estado Inicializando:

    ![VST_DeployComputeNode](./media/vs-azure-tools-publishing-a-cloud-service/IC744134.png)

## <a name="update-a-web-role-as-part-of-the-development-and-testing-cycle"></a>Atualizar uma função Web como parte do ciclo de desenvolvimento e teste

Se a infraestrutura de back-end do aplicativo for estável, mas as funções Web precisarem de atualizações mais frequentes, você poderá usar a Implantação da Web para atualizar apenas uma função Web em seu projeto. A Implantação da Web é útil quando você não deseja recriar e reimplantar as funções de trabalho de back-end ou se tem várias funções Web e deseja atualizar somente uma delas.

### <a name="requirements-for-using-web-deploy"></a>Requisitos para o uso da Implantação da Web

- **Apenas para fins de desenvolvimento e teste:** as alterações são feitas diretamente na máquina virtual em que a função web está sendo executada. Se essa máquina virtual precisar ser reciclada, as alterações serão perdidas porque o pacote original que você publicou é usado para recriar a máquina virtual para a função. Republique seu aplicativo para obter as alterações mais recentes para a função Web.

- **Apenas funções web podem ser atualizadas:** funções de trabalho não podem ser atualizadas. Além disso, não é possível atualizar o `RoleEntryPoint` em `web role.cs`.

- **Só há suporte a uma única instância de uma função web:** não pode haver várias instâncias de uma função web no ambiente de implantação. No entanto, há suporte para várias funções Web, cada com apenas uma instância.

- **Habilitar conexões de área de trabalho remota:** esse requisito permite que a Implantação da Web use o usuário e a senha para se conectar à máquina virtual para implantar as alterações no servidor que executa o IIS (Serviços de Informações da Internet). Além disso, talvez seja necessário conectar-se à máquina virtual para adicionar um certificado confiável ao IIS nessa máquina virtual. (Esse certificado garante que a conexão remota ao IIS usada pela Implantação da Web é segura).

O procedimento a seguir supõe que você esteja usando o assistente para **Publicar Aplicativo do Azure** .

### <a name="enable-web-deploy-when-you-publish-your-application"></a>Habilitar a Implantação da Web ao publicar seu aplicativo

1. Para habilitar a opção **Habilitar Implantação da Web para todas as funções web**, primeiro, é necessário configurar conexões de área de trabalho remota. Escolha **Habilitar Área de Trabalho Remota** para todas as funções e, em seguida, forneça as credenciais que serão usadas na conexão remota na caixa **Configuração da Área de Trabalho Remota** que é mostrada. Consulte [Habilitar a Conexão de Área de Trabalho Remota para uma função nos Serviços de Nuvem do Azure usando o Visual Studio](cloud-services/cloud-services-role-enable-remote-desktop-visual-studio.md).

1. Para habilitar a Implantação da Web para todas as funções Web em seu aplicativo, selecione **Habilitar Implantação da Web para todas as funções Web**.

    Um triângulo amarelo de aviso será exibido. A Implantação da Web usa um certificado não confiável, autoassinado por padrão, o que não é recomendado para carregar dados confidenciais. Se precisar proteger esse processo para dados confidenciais, você poderá adicionar um certificado SSL a ser usado para conexões de Implantação da Web. Esse deve ser um certificado confiável. Para saber mais, veja [Tornar a implantação Web segura](#make-web-deploy-secure).

1. Escolha **Avançar** para mostrar a tela **Resumo** e, em seguida, escolha **Publicar** para implantar o serviço de nuvem.

    O serviço de nuvem é publicado. A máquina virtual que é criada tem conexões remotas habilitadas para o IIS para que a Implantação da Web possa ser usada para atualizar suas funções Web sem as publicar novamente.

   > [!NOTE]
   > Se você tiver mais de uma instância configurada para uma função Web, será exibida uma mensagem de aviso informando que cada função Web será limitada a uma instância somente no pacote que é criado para publicar seu aplicativo. Selecione **OK** para continuar. Conforme mencionado na seção Requisitos, você pode ter mais de uma função Web, mas apenas uma instância de cada função.

### <a name="update-your-web-role-by-using-web-deploy"></a>Para atualizar sua função Web usando a Implantação da Web

1. Para usar a Implantação da Web, faça alterações de código no projeto para qualquer uma das funções Web no Visual Studio que você deseja publicar, clique com o botão direito no nó do projeto na solução e aponte para **Publicar**. A caixa de diálogo **Publicar Web** é exibida.

1. (Opcional) Se adicionou um certificado SSL confiável a ser usado para conexões remotas para o IIS, você pode desmarcar a caixa de seleção **Permitir certificado não confiável** . Para saber mais sobre como adicionar um certificado para proteger a Implantação da Web, veja a seção **Para tornar a Implantação da Web segura**, mais adiante neste artigo.

1. Para usar a Implantação da Web, o mecanismo de publicação precisa do nome de usuário e da senha que você configurou para a conexão de Área de Trabalho Remota quando publicou o pacote pela primeira vez.

   a. Em **Nome de usuário**, digite o nome de usuário.

   b. Em **Senha**, digite a senha.

   c. (Opcional) Se você quiser salvar esta senha nesse perfil, escolha **Salvar senha**.

1. Para publicar as alterações em sua função Web, escolha **Publicar**.

    A linha de status exibe **Publicação iniciada**. Quando a publicação for concluída, a mensagem **Publicação bem-sucedida** será exibida. Agora as alterações foram implantadas na função Web em sua máquina virtual. Você pode então iniciar o aplicativo do Azure no ambiente do Azure para testar as alterações.

### <a name="make-web-deploy-secure"></a>Para tornar a Implantação da Web segura

1. A Implantação da Web usa um certificado não confiável, autoassinado por padrão, o que não é recomendado para carregar dados confidenciais. Se precisar proteger esse processo para dados confidenciais, você poderá adicionar um certificado SSL a ser usado para conexões de Implantação da Web. Esse deve ser um certificado confiável, que você obtém de uma CA (autoridade de certificação).

    Para tornar a Implantação da Web segura para cada máquina virtual de cada uma das funções web, é necessário carregar o certificado de confiança que você deseja usar para a implantação da Web no portal do Azure. Esse certificado garante que o certificado seja adicionado à máquina virtual que é criada para a função Web quando você publicar seu aplicativo.

1. Para adicionar um certificado SSL confiável ao IIS a ser usado em conexões remotas, siga estas etapas:

   a. Para se conectar à máquina virtual que está executando a função web, selecione a instância da função web em **Cloud Explorer** ou **Gerenciador de Servidores** e escolha o comando **Conectar-se usando a Área de Trabalho Remota**. Para obter etapas detalhadas de como se conectar à máquina virtual, consulte [Habilitar a Conexão de Área de Trabalho Remota para uma função nos Serviços de Nuvem do Azure usando o Visual Studio](cloud-services/cloud-services-role-enable-remote-desktop-visual-studio.md). O navegador solicitará que você baixe um arquivo `.rdp`.

   b. Para adicionar um certificado SSL, abra o serviço de gerenciamento no Gerenciador do IIS. No Gerenciador do IIS, habilite o SSL abrindo o link **Ligações** no painel **Ação**. A caixa de diálogo **Adicionar Ligação do Site** é exibida. Escolha **Adicionar** e então escolha HTTPS na lista suspensa **Tipo**. Na lista **Certificado SSL**, escolha o certificado SSL que foi assinado por uma AC e que você carregou no portal do Azure. Para obter mais informações, consulte [Definir configurações de conexão para o serviço de gerenciamento](http://go.microsoft.com/fwlink/?LinkId=215824).

      > [!NOTE]
      > Se você adicionar um certificado SSL confiável, o triângulo amarelo de aviso não será mais exibido no **Assistente de Publicação**.

## <a name="include-files-in-the-service-package"></a>Incluir arquivos no pacote de serviço

Talvez seja necessário incluir arquivos específicos em seu pacote de serviço para que eles estejam disponíveis na máquina virtual que é criada para uma função. Por exemplo, você talvez queira adicionar um arquivo `.exe` ou um `.msi` que é usado por um script de inicialização para o pacote de serviço. Ou talvez seja necessário adicionar um assembly que uma função Web ou um projeto de função de trabalho requer. Para incluir arquivos, eles devem ser adicionados à solução para seu aplicativo do Azure.

1. Para adicionar um assembly a um pacote de serviço, use as seguintes etapas:

   a. No **Gerenciador de Soluções**, abra o nó do projeto que está sem o assembly referenciado.
   b. Para adicionar o assembly ao projeto, abra o menu de atalho da pasta **Referências** e escolha **Adicionar Referência**. A caixa de diálogo Adicionar Referência é exibida.
   c. Escolha a referência que você deseja adicionar e, em seguida, escolha o botão **OK**. A referência é adicionada à lista na pasta **Referências** .
   d. Abra o menu de atalho do assembly que você adicionou e escolha **Propriedades**. A janela **Propriedades** é exibida.

      Para incluir esse assembly no pacote de serviço, na lista **Local da Cópia**, escolha **Verdadeiro**.
1. No **Gerenciador de Soluções** , abra o nó do projeto que está sem o assembly referenciado.

1. Para adicionar o assembly ao projeto, abra o menu de atalho da pasta **Referências** e escolha **Adicionar Referência**. A caixa de diálogo **Adicionar Referência** é exibida.

1. Escolha a referência que você deseja adicionar e, em seguida, escolha o botão **OK** .

    A referência é adicionada à lista na pasta **Referências** .

1. Abra o menu de atalho do assembly que você adicionou e escolha **Propriedades**. A janela Propriedades é exibida.

1. Para incluir esse assembly no pacote de serviço, na lista **Local da Cópia**, escolha **Verdadeiro**.

1. Para incluir arquivos no pacote de serviço que foram adicionados ao projeto de função Web, abra o menu de atalho do arquivo e escolha **Propriedades**. Na janela **Propriedades**, escolha **Conteúdo** na caixa de listagem **Criar Ação**.

1. Para incluir arquivos no pacote de serviço que foram adicionados ao projeto de função de trabalho, abra o menu de atalho do arquivo e escolha **Propriedades**. Na janela **Propriedades**, escolha **Copiar se for mais recente** na caixa de listagem **Copiar para diretório de saída**.

## <a name="next-steps"></a>Próximas etapas

Para saber mais sobre a publicação do Azure por meio do Visual Studio, consulte o [Assistente de Publicação de Aplicativo do Azure](vs-azure-tools-publish-azure-application-wizard.md).
