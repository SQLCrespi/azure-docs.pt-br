---
title: Soluções de Armazenamento do Azure para R Server no HDInsight – Azure | Microsoft Docs
description: Conheça as diferentes opções de armazenamento disponíveis com o Microsoft R Server no HDInsight
services: hdinsight
documentationcenter: ''
author: nitinme
manager: jhubbard
editor: cgronlun
ms.assetid: 1cf30096-d3ca-45ea-b526-aa3954402f66
ms.service: hdinsight
ms.custom: hdinsightactive
ms.devlang: R
ms.topic: conceptual
ms.date: 03/22/2018
ms.author: nitinme
ms.openlocfilehash: 23e32a913fb73d2207f7cf37ce6230e428fbe95c
ms.sourcegitcommit: 9cdd83256b82e664bd36991d78f87ea1e56827cd
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/16/2018
---
# <a name="azure-storage-solutions-for-r-server-on-azure-hdinsight"></a>Soluções de Armazenamento do Microsoft Azure para o Microsoft R Server no Azure HDInsight

O Microsoft R Server no HDInsight tem uma variedade de soluções de armazenamento para manter os dados, código ou objetos que contêm resultados da análise. Elas incluem as seguintes opções:

- [Blob do Azure](https://azure.microsoft.com/services/storage/blobs/)
- [Armazenamento do Azure Data Lake](https://azure.microsoft.com/services/data-lake-store/)
- [Armazenamento de Arquivos do Azure](https://azure.microsoft.com/services/storage/files/)

Você também tem a opção de acessar várias contas de armazenamento ou contêineres do Azure com o cluster HDInsight. O Armazenamento de Arquivos do Azure é uma opção de armazenamento de dados conveniente para uso no nó de borda, que permite a você montar um compartilhamento de arquivos do Armazenamento do Azure para, por exemplo, o sistema de arquivos do Linux. Mas os compartilhamentos de arquivos do Azure podem ser montados e usados por qualquer sistema que tenha um sistema operacional com suporte, como Windows ou Linux. 

Ao criar um cluster Hadoop no HDInsight, você especifica uma conta de **armazenamento do Azure** ou um **Data Lake Store**. Um contêiner de armazenamento específico dessa conta mantém o sistema de arquivos do cluster que você cria (por exemplo, o Sistema de Arquivos Distribuído Hadoop). Para obter mais informações e diretrizes, consulte:

- [Usar o Armazenamento do Azure com o HDInsight](../hdinsight-hadoop-use-blob-storage.md)
- [Usar o Data Lake Store com clusters Azure HDInsight](../hdinsight-hadoop-use-data-lake-store.md)

## <a name="use-azure-blob-storage-accounts-with-r-server-cluster"></a>Usar várias contas de Armazenamento de Blobs do Azure com o cluster do Microsoft R Server

Se você especificou mais de uma conta de armazenamento ao criar o cluster de Microsoft R Server, as instruções a seguir explicam como usar uma conta secundária para acesso e operações de dados no cluster R Server. Suponha as contas de armazenamento e contêiner a seguir: **storage1** e um contêiner padrão chamado **container1**, e **storage2** com **container2**.

> [!WARNING]
> Para fins de desempenho, o cluster HDInsight é criado no mesmo datacenter que a conta de armazenamento primária especificada. Não há suporte para o uso de uma conta de armazenamento em um local diferente do cluster HDInsight.

### <a name="use-the-default-storage-with-r-server-on-hdinsight"></a>Usar o armazenamento padrão com o Microsoft R Server no HDInsight

1. Usando um cliente SSH, conecte-se ao nó de borda do cluster. Para saber mais sobre como usar o SSH com clusters HDInsight, confira [Usar SSH com HDInsight](../hdinsight-hadoop-linux-use-ssh-unix.md).
  
2. Copie um arquivo de exemplo, mysamplefile.csv, para o /diretório de compartilhamento. 

        hadoop fs –mkdir /share
        hadoop fs –copyFromLocal mycsv.scv /share  

3. Alterne para o R Studio ou outro console R e grave o código R para definir o nome do nó para **padrão** e o local do arquivo que você deseja acessar.  

        myNameNode <- "default"
        myPort <- 0

        #Location of the data:  
        bigDataDirRoot <- "/share"  

        #Define Spark compute context:
        mySparkCluster <- RxSpark(nameNode=myNameNode, consoleOutput=TRUE)

        #Set compute context:
        rxSetComputeContext(mySparkCluster)

        #Define the Hadoop Distributed File System (HDFS) file system:
        hdfsFS <- RxHdfsFileSystem(hostName=myNameNode, port=myPort)

        #Specify the input file to analyze in HDFS:
        inputFile <-file.path(bigDataDirRoot,"mysamplefile.csv")

Todas as referências de diretório e arquivo apontam para a conta de armazenamento `wasb://container1@storage1.blob.core.windows.net`. Essa é a **conta de armazenamento padrão** associada ao cluster do HDInsight.

### <a name="use-the-additional-storage-with-r-server-on-hdinsight"></a>Use o armazenamento adicional com o Microsoft R Server no HDInsight

Agora, suponha que você queira processar um arquivo chamado mysamplefile1.csv, localizado no /diretório particular do **container2** no **storage2**.

No código R, aponte a referência de nó de nome para a conta de armazenamento **storage2** .

    myNameNode <- "wasb://container2@storage2.blob.core.windows.net"
    myPort <- 0

    #Location of the data:
    bigDataDirRoot <- "/private"

    #Define Spark compute context:
    mySparkCluster <- RxSpark(consoleOutput=TRUE, nameNode=myNameNode, port=myPort)

    #Set compute context:
    rxSetComputeContext(mySparkCluster)

    #Define HDFS file system:
    hdfsFS <- RxHdfsFileSystem(hostName=myNameNode, port=myPort)

    #Specify the input file to analyze in HDFS:
    inputFile <-file.path(bigDataDirRoot,"mysamplefile1.csv")

Todas as referências de diretório e arquivo agora apontam para a conta de armazenamento `wasb://container2@storage2.blob.core.windows.net`. Esse é o **Nome de Nó** que você especificou.

Você precisa configurar o diretório /user/RevoShare/<SSH username> no **storage2** da seguinte maneira:


    hadoop fs -mkdir wasb://container2@storage2.blob.core.windows.net/user
    hadoop fs -mkdir wasb://container2@storage2.blob.core.windows.net/user/RevoShare
    hadoop fs -mkdir wasb://container2@storage2.blob.core.windows.net/user/RevoShare/<RDP username>

## <a name="use-an-azure-data-lake-store-with-r-server-cluster"></a>Usar um Azure Data Lake Store com o cluster do Microsoft R Server 

Para usar Data Lake Store com seu cluster HDInsight, você precisa dar ao cluster acesso a cada repositório do Azure Data Lake que deseja usar. Para obter instruções sobre como usar o Portal do Azure pra criar um cluster HDInsight com uma conta do Azure Data Lake Store como armazenamento padrão ou um repositório adicional, veja [Criar um cluster HDInsight com o Data Lake Store usando o Portal do Azure](../../data-lake-store/data-lake-store-hdinsight-hadoop-use-portal.md).

Você usa o repositório no script R de forma semelhante a como você fez com uma conta de armazenamento do Azure secundária, conforme descrito no procedimento anterior.

### <a name="add-cluster-access-to-your-azure-data-lake-stores"></a>Adicionar acesso de cluster aos repositórios Azure Data Lake
Você pode acessar um repositório Data Lake usando uma Entidade de Serviço do Azure AD (Azure Active Directory) que está associada ao cluster do HDInsight.

1. Ao criar o cluster do HDInsight, selecione **Identidade do Cluster AAD** na guia **Fonte de Dados**.

2. Na caixa de diálogo **Identidade de Cluster AAD**, em **Selecionar Entidade de Serviço do AD**, selecione **Criar novo**.

Depois de dar um nome à Entidade de Serviço e criar uma senha para ela, clique em **Gerenciar Acesso de ADLS** para associar a Entidade de Serviço ao seu Data Lake Store.

Também é possível adicionar o acesso do cluster a uma ou mais contas Data Lake Store após a criação do cluster. Abra a entrada do Portal do Azure para um Data Lake Store e vá para **Data Explorer > Acessar > Adicionar**. 

### <a name="how-to-access-the-data-lake-store-from-r-server-on-hdinsight"></a>Como acessar o Data Lake Store do Microsoft R Server no HDInsight

Após obter acesso a um repositório Data Lake, você pode usar o repositório no cluster do Microsoft R Server no HDInsight da forma como faria com uma conta de armazenamento do Azure secundária. A única diferença é que o prefixo **wasb://** se altera para **adl://** como segue:


    # Point to the ADL store (e.g. ADLtest)
    myNameNode <- "adl://rkadl1.azuredatalakestore.net"
    myPort <- 0

    # Location of the data (assumes a /share directory on the ADL account)
    bigDataDirRoot <- "/share"  

    # Define Spark compute context
    mySparkCluster <- RxSpark(consoleOutput=TRUE, nameNode=myNameNode, port=myPort)

    # Set compute context
    rxSetComputeContext(mySparkCluster)

    # Define HDFS file system
    hdfsFS <- RxHdfsFileSystem(hostName=myNameNode, port=myPort)

    # Specify the input file in HDFS to analyze
    inputFile <-file.path(bigDataDirRoot,"mysamplefile.csv")

Os comandos a seguir são usados para configurar a conta de armazenamento Data Lake com o diretório RevoShare e adicionar o arquivo .csv de exemplo do exemplo anterior:


    hadoop fs -mkdir adl://rkadl1.azuredatalakestore.net/user
    hadoop fs -mkdir adl://rkadl1.azuredatalakestore.net/user/RevoShare
    hadoop fs -mkdir adl://rkadl1.azuredatalakestore.net/user/RevoShare/<user>

    hadoop fs -mkdir adl://rkadl1.azuredatalakestore.net/share

    hadoop fs -copyFromLocal /usr/lib64/R Server-7.4.1/library/RevoScaleR/SampleData/mysamplefile.csv adl://rkadl1.azuredatalakestore.net/share

    hadoop fs –ls adl://rkadl1.azuredatalakestore.net/share


## <a name="use-azure-file-storage-with-r-server-on-hdinsight"></a>Usar o Armazenamento de Arquivos do Azure com o Microsoft R Server no HDInsight

Também há uma opção de armazenamento de dados conveniente para uso no nó de borda chamado [Azure Files]((https://azure.microsoft.com/services/storage/files/). Esse recurso permite montar um compartilhamento de arquivos do Armazenamento do Azure para o sistema de arquivos do Linux. Essa opção pode ser útil para armazenar arquivos de dados, scripts R e objetos de resultado que podem ser necessários posteriormente, especialmente quando fizer sentido usar o sistema de arquivos nativo no nó de borda em vez do HDFS. 

Uma grande vantagem dos Arquivos do Azure é que os compartilhamentos de arquivos podem ser montados e usados por qualquer sistema que tenha um sistema operacional com suporte, como Windows ou Linux. Por exemplo, ele pode ser usado por outro cluster do HDInsight que você ou alguém em sua equipe tenha, por uma VM do Azure ou até mesmo por um sistema local. Para obter mais informações, consulte:

- [Como utilizar o Armazenamento de arquivos do Azure com Linux](../../storage/files/storage-how-to-use-files-linux.md)
- [Como utilizar o Armazenamento de Arquivos do Azure no Windows](../../storage/files/storage-dotnet-how-to-use-files.md)


## <a name="next-steps"></a>Próximas etapas

* [Visão geral do cluster Microsoft R Server no HDInsight](r-server-overview.md)
* [Iniciar com o cluster do Microsoft R Server no Hadoop](r-server-get-started.md)
* [Opções de contexto de computação para cluster do Microsoft R Server no HDInsight](r-server-compute-contexts.md)

