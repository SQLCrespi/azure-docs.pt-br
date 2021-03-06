---
title: Expressões de filtro de exemplo possíveis com a preparação de dados do Azure Machine Learning | Microsoft Docs
description: Este documento fornece um conjunto de exemplos de expressões de filtro possíveis com a preparação de dados do Azure Machine Learning
services: machine-learning
author: euangMS
ms.author: euang
manager: lanceo
ms.reviewer: jmartens, jasonwhowell, mldocs
ms.service: machine-learning
ms.workload: data-services
ms.custom: ''
ms.devlang: ''
ms.topic: article
ms.date: 02/01/2018
ms.openlocfilehash: 973c56b8b2821c8e3d63161e6a233243639c74f4
ms.sourcegitcommit: 59914a06e1f337399e4db3c6f3bc15c573079832
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/19/2018
---
# <a name="sample-of-filter-expressions-python"></a>Exemplo de expressões de filtro (Python) 
Antes de ler este apêndice, leia [Visão geral da extensibilidade do Python](data-prep-python-extensibility-overview.md).

## <a name="filter-with-equivalence-test"></a>Filtrar com teste de equivalência
Filtre somente as linhas em que o valor de Col2 (numérico) for maior que 4. 

```python
    row["Col2"] > 4
```

## <a name="filter-with-multiple-columns"></a>Filtrar com várias colunas 
Filtrar somente as linhas em que Col1 contém o valor **Bom** e Col2 contém o valor 1 (numérico). 
```python
    row["Col1"] == 'Good' and row["Col2"] == 1
```

## <a name="test-filter-against-null"></a>Filtro de teste com null
Filtre somente as linhas em que Col1 tem um valor nulo. 
```python
    pd.isnull(row["Col1"])
```
