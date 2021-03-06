---
title: Azure Event Grid のトピックまたはドメインに対して IP ファイアウォールを構成する (プレビュー)
description: この記事では Event Grid のトピックまたはドメインに対してファイアウォール設定を構成する方法について説明します。
services: event-grid
author: spelluru
ms.service: event-grid
ms.topic: conceptual
ms.date: 03/11/2020
ms.author: spelluru
ms.openlocfilehash: b195872ca1002970fa96ae133d5eb47a9267796d
ms.sourcegitcommit: 2ec4b3d0bad7dc0071400c2a2264399e4fe34897
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/28/2020
ms.locfileid: "79300510"
---
# <a name="configure-ip-firewall-for-azure-event-grid-topics-or-domains-preview"></a>Azure Event Grid のトピックまたはドメインに対して IP ファイアウォールを構成する (プレビュー)
既定では、要求が有効な認証と承認を受けている限り、トピックおよびドメインにはインターネットからアクセスできます。 IP ファイアウォールを使用すると、[CIDR (クラスレス ドメイン間ルーティング)](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 表記の一連の IPv4 アドレスまたは IPv4 アドレス範囲のみにアクセスを制限できます。 他の IP アドレスから発信するパブリッシャーは拒否され、403 (禁止) の応答が返されます。 Event Grid でサポートされるネットワーク セキュリティ機能の詳細については、[Event Grid のネットワーク セキュリティ](network-security.md)に関する記事を参照してください。

この記事では Azure Event Grid のトピックまたはドメインに対して IP ファイアウォール設定を構成する方法について説明します。

## <a name="use-azure-portal"></a>Azure Portal の使用
このセクションでは、Azure portal を使用して受信 IP ファイアウォール規則を作成する方法について説明します。 このセクションで示す手順は、トピックに関するものです。 同様の手順を使用して、**ドメイン**の受信 IP 規則を作成できます。 

1. [Azure portal](https://portal.azure.com) 内で、イベント グリッドのトピックまたはドメインに移動し、 **[ネットワーク]** タブに切り替えます。
2. **[パブリック ネットワーク]** を選択して、インターネットを含むすべてのネットワークがリソースにアクセスできるようにします。 

    IP ベースのファイアウォール規則を使用して、トラフィックを制限できます。 クラスレス ドメイン間ルーティング (CIDR) 表記で、単一の IPv4 アドレスまたは IP アドレスの範囲を指定します。 

    ![[パブリック ネットワーク] ページ](./media/configure-firewall/public-networks-page.png)
3. プライベート エンドポイント接続のみにこのリソースへのアクセスを許可するには、 **[Private endpoints only]\(プライベート エンドポイントのみ\)** を選択します。 このページの **[プライベート エンドポイント接続]** タブを使用して、接続を管理します。 

    ![[パブリック ネットワーク] ページ](./media/configure-firewall/private-endpoints-page.png)
4. ツールバーの **[保存]** を選択します。 



## <a name="use-azure-cli"></a>Azure CLI の使用
このセクションでは、Azure CLI コマンドを使用して受信 IP 規則を含むトピックを作成する方法について説明します。 このセクションで示す手順は、トピックに関するものです。 同様の手順を使用して、**ドメイン**の受信 IP 規則を作成できます。 


### <a name="enable-public-network-access-for-an-existing-topic"></a>既存のトピックに対してパブリック ネットワーク アクセスを有効にする
既定では、トピックとドメインに対してパブリック ネットワーク アクセスが有効になっています。 受信 IP ファイアウォール規則を構成することで、トラフィックを制限できます。 

```azurecli-interactive
az rest --method patch --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" --body "{\""properties\"": {\""publicNetworkAccess\"": \""Enabled\""}}"
```

### <a name="disable-public-network-access-for-an-existing-topic"></a>既存のトピックに対してパブリック ネットワーク アクセスを無効にする
トピックまたはドメインに対してパブリック ネットワーク アクセスが無効になっている場合、パブリック インターネット経由のトラフィックは許可されません。 プライベート エンドポイント接続のみが、これらのリソースへのアクセスを許可されます。 

```azurecli-interactive
az rest --method patch --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" --body "{\""properties\"": {\""publicNetworkAccess\"": \""Disabled\""}}"
```

### <a name="create-topic-with-inbound-ip-rules"></a>受信 IP 規則を含むトピックを作成する
次の CLI コマンド例では、1 ステップで受信 IP 規則を含むイベント グリッド トピックを作成します。 

```azurecli-interactive
az rest --method put \
    --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" \
    --body {\""location\"":\""<LOCATION>\", \""properties\"" :{\""publicNetworkAccess\"":\""enabled\"",\""InboundIpRules\"": [ {\""ipMask\"": \""<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>\"", \""action\"": \""allow\""} ]}}
```

### <a name="create-topic-first-and-then-add-inbound-ip-rules"></a>まずトピックを作成し、次に受信 IP 規則を追加する
この例では、まずイベント グリッド トピックを作成し、次に別のコマンドでそのトピックの受信 IP 規則を追加します。 また、2 番目のコマンドで設定された受信 IP 規則も更新されます。 

```azurecli-interactive

# create the event grid topic first
az rest --method put \
    --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" \
    --body {\""location\"":\""<LOCATION>\""}

# add inbound IP rules
az rest --method put \
    --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" 
    --body {\""location\"":\""<LOCATION>\", \""properties\"" :{\""publicNetworkAccess\"":\""enabled\"", \""InboundIpRules\"": [ {\""ipMask\"": \""<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>\"", \""action\"": \""allow\""} ]}}

# later, update topic with additional ip rules or remove them. 
az rest --method put \
    --uri "/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" 
    --body {\""location\"":\""<LOCATION>\", \""properties\"" :{\""publicNetworkAccess\"":\""enabled\"", \""InboundIpRules\"": [ {\""ipMask\"": \""<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>\"", \""action\"": \""allow\""}, {\""ipMask\"": \""<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>\"", \""action\"": \""allow\""} ]}}
```


## <a name="use-powershell"></a>PowerShell の使用
このセクションでは、Azure PowerShell コマンドを使用して、受信 IP ファイアウォール規則を含む Azure Event Grid トピックを作成する方法について説明します。 このセクションで示す手順は、トピックに関するものです。 同様の手順を使用して、**ドメイン**の受信 IP 規則を作成できます。 

### <a name="prerequisite"></a>前提条件
「[方法:リソースにアクセスできる Azure AD アプリケーションとサービス プリンシパルをポータルで作成する](../active-directory/develop/howto-create-service-principal-portal.md)」の手順に従って、Azure Active Directory アプリケーションを作成し、以下の値を書き留めます。

- ディレクトリ (テナント) ID
- アプリケーション (クライアント) ID
- アプリケーション (クライアント) シークレット

### <a name="prepare-token-and-headers-for-rest-api-calls"></a>REST API 呼び出し用のトークンとヘッダーを準備する 
次の前提条件コマンドを実行して、REST API 呼び出しで使用する認証トークンを取得し、Authorization やその他のヘッダー情報を設定します。 

```azurepowershell-interactive
# replace <CLIENT ID> and <CLIENT SECRET>
$body = "grant_type=client_credentials&client_id=<CLIENT ID>&client_secret=<CLIENT SECRET>&resource=https://management.core.windows.net"

# get the authentication token. Replace <TENANT ID>
$Token = Invoke-RestMethod -Method Post `
    -Uri https://login.microsoftonline.com/<TENANT ID>/oauth2/token `
    -Body $body `
    -ContentType 'application/x-www-form-urlencoded'

# set authorization and content-type headers
$Headers = @{}
$Headers.Add("Authorization","$($Token.token_type) "+ " " + "$($Token.access_token)")
$Headers.Add("Content-Type","application/json")
```

### <a name="enable-public-network-access-for-an-existing-topic"></a>既存のトピックに対してパブリック ネットワーク アクセスを有効にする
既定では、トピックとドメインに対してパブリック ネットワーク アクセスが有効になっています。 受信 IP ファイアウォール規則を構成することで、トラフィックを制限できます。 

```azurepowershell-interactive
$body = @{"properties"=@{"publicNetworkAccess"="enabled"}} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Method 'Patch' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    -Body $body `
    | ConvertTo-Json -Depth 5
```

### <a name="disable-public-network-access-for-an-existing-topic"></a>既存のトピックに対してパブリック ネットワーク アクセスを無効にする
トピックまたはドメインに対してパブリック ネットワーク アクセスが無効になっている場合、パブリック インターネット経由のトラフィックは許可されません。 プライベート エンドポイント接続のみが、これらのリソースへのアクセスを許可されます。 

```azurepowershell-interactive
$body = @{"properties"=@{"publicNetworkAccess"="disabled"}} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Method 'Patch' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    -Body $body `
    | ConvertTo-Json -Depth 5
```

### <a name="create-an-event-grid-topic-with-inbound-rules-in-one-step"></a>受信規則を含むイベント グリッド トピックを 1 ステップで作成する

```azurepowershell-interactive

# prepare the body for the REST PUT method. Notice that inbound IP rules are included. 
$body = @{"location"="<LOCATION>"; "sku"= @{"name"="basic"}; "properties"=@{"publicNetworkAccess"="enabled"; "inboundIpRules"=@(@{"ipmask"="<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>";"action"="allow"})}} | ConvertTo-Json -Depth 5

# create the event grid topic with inbound IP rules
Invoke-RestMethod -Method 'Put' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    -Body $body

# verify that the topic was created
Invoke-RestMethod -Method 'Get' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    | ConvertTo-Json -Depth 5
```


### <a name="create-event-grid-topic-first-and-then-add-inbound-ip-rules"></a>まずイベント グリッド トピックを作成し、次に受信 IP 規則を追加する

```azurepowershell-interactive

# prepare the body for the REST PUT method. Notice that no inbound IP rules are specified. 
$body = @{"location"="<LOCATION>"; "sku"= @{"name"="basic"}; "properties"=@{"publicNetworkAccess"="enabled";}} | ConvertTo-Json -Depth 5

# create the Event Grid topic
Invoke-RestMethod -Method 'Put' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    -Body $body`

# verify that the topic was created
Invoke-RestMethod -Method 'Get' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    | ConvertTo-Json -Depth 5

# prepare the body for REST PUT method. Notice that it includes inbound IP rules now. This feature available in both basic and premium tiers.
$body = @{"location"="<LOCATION>"; "sku"= @{"name"="basic"}; "properties"=@{"publicNetworkAccess"="enabled"; "inboundIpRules"=@(@{"ipmask"="<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>";"action"="allow"}, @{"ipmask"="<IP ADDRESS or IP ADDRESS RANGE in CIDR notation>";"action"="allow"})}} | ConvertTo-Json -Depth 5

# update the topic with inbound IP rules
Invoke-RestMethod -Method 'Put' `
    -Uri "https://management.azure.com/subscriptions/<AZURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    -Body $body

# verify that the topic was updated
Invoke-RestMethod -Method 'Get' 
    -Uri "https://management.azure.com/subscriptions/<AzURE SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP NAME>/providers/Microsoft.EventGrid/topics/<EVENT GRID TOPIC NAME>?api-version=2020-04-01-preview" `
    -Headers $Headers `
    | ConvertTo-Json -Depth 5

```

## <a name="next-steps"></a>次のステップ

* イベント配信の監視について詳しくは、「[Event Grid メッセージ配信の監視](monitor-event-delivery.md)」をご覧ください。
* 認証キーについて詳しくは、「[Event Grid のセキュリティと認証](security-authentication.md)」をご覧ください。
* Azure Event Grid サブスクリプションの作成の詳細については、[Event Grid サブスクリプション スキーマ](subscription-creation-schema.md)に関する記事を参照してください。
