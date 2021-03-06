---
title: Arquivo de inclusão
description: Arquivo de inclusão
services: vpn-gateway
author: cherylmc
ms.service: vpn-gateway
ms.topic: include
ms.date: 03/21/2018
ms.author: cherylmc
ms.custom: include file
ms.openlocfilehash: c06e69dd9d1997500589659e936dc25ee01ed145
ms.sourcegitcommit: 59914a06e1f337399e4db3c6f3bc15c573079832
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 04/20/2018
---
Para as SKUs atuais (VpnGw1, VpnGw2 e VPNGW3) para o qual você deseja redimensionar sua SKU de gateway para atualizar para uma mais avançada, você pode usar o `Resize-AzureRmVirtualNetworkGateway` cmdlet do PowerShell. Você também pode fazer o downgrade do tamanho do SKU de gateway usando esse cmdlet. Se você estiver usando a SKU de gateway básica [use estas instruções](../articles/vpn-gateway/vpn-gateway-about-skus-legacy.md#resize) para redimensionar o gateway.

O exemplo de PowerShell a seguir mostra um SKU de gateway sendo redimensionado para VpnGw2.

```powershell
$gw = Get-AzureRmVirtualNetworkGateway -Name vnetgw1 -ResourceGroupName testrg
Resize-AzureRmVirtualNetworkGateway -VirtualNetworkGateway $gw -GatewaySku VpnGw2
```

Você também pode redimensionar um gateway no portal do Azure, vá para o **Configuração** página do seu gateway de rede virtual e selecione uma SKU diferente na lista suspensa.