---
title: Configurar clusters HDInsight ingressados no domínio usando o AAD-DS
description: Saiba como instalar e configurar clusters HDInsight ingressados no domínio usando o Azure Active Directory Domain Services
services: hdinsight
author: omidm1
manager: jhubbard
editor: cgronlun
ms.service: hdinsight
ms.topic: conceptual
ms.date: 04/17/2018
ms.author: omidm
ms.openlocfilehash: 060ca8040f514ec1df48c2ca4568cbbb2a529267
ms.sourcegitcommit: 59914a06e1f337399e4db3c6f3bc15c573079832
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/19/2018
---
# <a name="configure-domain-joined-hdinsight-clusters-using-azure-active-directory-domain-services"></a>Configurar clusters HDInsight ingressados no domínio usando o Azure Active Directory Domain Services

Clusters de domínio ingressado fornecem à empresa multiusuário recursos de segurança no HDInsight. Clusters de HDInsight ingressados no domínio são conectados aos domínios do active directory, para que os usuários do domínio possam usar suas credenciais de domínio para autenticar com os clusters e executar trabalhos de grandes dados. 

Neste artigo, você aprenderá como instalar e configurar clusters HDInsight ingressados no domínio usando o Azure Active Directory Domain Services.

> [!NOTE]
> Não há suporte para o Active Directory em VMs de IaaS do Azure.

## <a name="create-azure-adds"></a>Criar Azure ADDS

Você precisa criar um Azure AD DS para poder criar um cluster HDInsight. Para obter mais informações, consulte [Habilitar o Azure Active Directory Domain Services usando o Portal do Azure](../../active-directory-domain-services/active-directory-ds-getting-started.md). 

> [!NOTE]
> Somente os administradores de locatário tem os privilégios para criar serviços de domínio. Se você usar o ADLS (Azure Data Lake Storage) como o armazenamento padrão para o HDInsight, verifique se o locatário padrão do Azure AD para o ADLS é o mesmo do domínio para o cluster HDInsight. Para que essa configuração funcione com o Azure Data Lake Store, a autenticação multifator deve ser desabilitada para os usuários que terão acesso ao cluster.

Depois que o serviço de domínio AAD tiver sido configurado, você precisa criar uma conta de serviço no ADD (que será sincronizada com o AAD-DS) com as permissões corretas para criar o cluster HDInsight. Se essa conta de serviço já existir, você precisará redefinir sua senha e aguardar até que seja sincronizado ao AAD-DS (essa redefinição resultará na criação do hash da senha kerberos e pode levar até 30 min). Essa conta de serviço deve ter os seguintes privilégios:

- Adicionar computadores ao domínio e colocar entidades de serviço na UO que você especificar durante a criação do cluster.
- Criar entidades de serviço na UO que você especificar durante a criação do cluster.

É preciso habilitar o LDAP Seguro para o domínio gerenciado do Azure AD Domain Services. Para habilitar o LDAP Seguro, veja [Configurar o LDAP seguro (LDAPS) para um domínio gerenciado do Azure AD Domain Services](../../active-directory-domain-services/active-directory-ds-admin-guide-configure-secure-ldap.md).

## <a name="create-a-domain-joined-hdinsight-cluster"></a>Crie um cluster HDInsight associado a um domínio

A próxima etapa é criar o cluster HDInsight usando o AAD DS e a conta de serviço criada na seção anterior.

É mais fácil de colocar o serviço de domínio do Azure AD e o cluster HDInsight na mesma network virtual (VNet) do Azure. Caso eles estejam em VNets diferentes, você deve parear ambas VNets. Para obter mais informações, consulte [Emparelhamento de rede virtual do Azure](../../virtual-network/virtual-network-peering-overview.md).

Quando você cria um cluster HDInsight ingressado no domínio, você deve fornecer os seguintes parâmetros:

- **Nome de domínio**: O nome de domínio que está associado com o Azure AD DS. Por exemplo, contoso.onmicrosoft.com
- **Nome de usuário de domínio**: A conta de serviço no controlador de domínio do Azure AD que é criada na seção anterior. Por exemplo, hdiadmin@contoso.onmicrosoft.com. Esse usuário de domínio será o administrador deste cluster HDInsight ingressado no domínio.
- **Senha do domínio**: A senha da conta de serviço.
- **Unidade Organizacional**: Insira o nome diferenciado da UO que você deseja usar com o cluster HDInsight. Por exemplo: OU = HDInsightOU, DC = contoso, DC = onmicrosohift, DC = com. Se essa UO não existir, o cluster HDInsight tentará criá-la. 
- **URL LDAPS**: Por exemplo, ldaps://contoso.onmicrosoft.com:636
- **Acessar grupo de usuários**: Especifique o grupo de segurança cujos usuários você deseja sincronizar com o cluster. Por exemplo, HiveUsers. Se você quiser especificar vários grupos de usuários, separe-os por vírgula ','.
 
> [!NOTE]
> Como o Apache Zeppelin usa o nome de domínio para autenticar a conta de serviço administrativo, a conta de serviço DEVE ter o mesmo nome de domínio que o sufixo UPN para que o Apache Zeppelin funcione corretamente.
 
A seguinte captura de tela mostra a configuração no Portal do Azure:

![Configuração do Active Directory Domain Services ingressado no domínio do Azure HDInsight](./media/apache-domain-joined-configure-using-azure-adds/hdinsight-domain-joined-configuration-azure-aads-portal.png).


## <a name="next-steps"></a>Próximas etapas
* Para configurar políticas do Hive e executar consultas do Hive, confira [Configurar políticas do Hive para clusters HDInsight associados ao domínio](apache-domain-joined-run-hive.md).
* Para usar SSH para conectar-se a clusters do HDInsight Ingressados no Domínio, confira [Usar SSH com Hadoop baseado em Linux no HDInsight do Linux, Unix ou OS X](../hdinsight-hadoop-linux-use-ssh-unix.md#domainjoined).

