---
title: Azure Event Grid のセキュリティと認証
description: この記事では、Event Grid リソース (WebHook、サブスクリプション、カスタム トピック) へのアクセスを認証するためのさまざまな方法について説明します。
services: event-grid
author: banisadr
manager: timlt
ms.service: event-grid
ms.topic: conceptual
ms.date: 03/06/2020
ms.author: babanisa
ms.openlocfilehash: 0b7c5b42ac6291c6687337ba8d6a9d35830b9bda
ms.sourcegitcommit: 2ec4b3d0bad7dc0071400c2a2264399e4fe34897
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/28/2020
ms.locfileid: "79236251"
---
# <a name="authenticating-access-to-event-grid-resources"></a>Event Grid リソースへのアクセスの認証

Azure Event Grid には、3 種類の認証があります。

* webhook のイベント配信
* イベントのサブスクリプション
* カスタム トピックの発行

## <a name="webhook-event-delivery"></a>webHook のイベント配信

Webhook は、Azure Event Grid からイベントを受信する多数ある方法の 1 つです。 新しいイベントの準備ができるたびに、Event Grid サービスは、本文にイベントが含まれる HTTP 要求を構成済み HTTP エンドポイントに POST します。

Webhook をサポートする他の多くのサービスと同様に、Event Grid を使用するには、Webhook エンドポイントへのイベントの配信を開始する前に、そのエンドポイントの所有権を証明する必要があります。 この要件により、悪意のあるユーザーはエンドポイントをイベントで氾濫させることができなくなります。 以下に示す 3 つの Azure サービスのいずれかを使用すると、Azure インフラストラクチャはこの検証を自動的に処理します。

* [Event Grid コネクタ](https://docs.microsoft.com/connectors/azureeventgrid/)を使用した Azure Logic Apps
* [Webhook](../event-grid/ensure-tags-exists-on-new-virtual-machines.md) を使用した Azure Automation
* [Event Grid トリガー](../azure-functions/functions-bindings-event-grid.md)を使用した Azure Functions

他の種類のエンドポイント (HTTP トリガー ベースの Azure 関数など) を使用する場合は、エンドポイントのコードが Event Grid を使用した検証ハンドシェイクに参加する必要があります。 Event Grid では、サブスクリプションを検証する 2 つの方法がサポートされています。

1. **ValidationCode ハンドシェイク (プログラムによる)** :この方法は、エンドポイントのソース コードを制御する場合にお勧めします。 イベント サブスクリプションの作成時に、Event Grid はサブスクリプション検証イベントをエンドポイントに送信します。 このイベントのスキーマは、他の Event Grid イベントに似ています。 このイベントのデータ部分には `validationCode` プロパティが含まれます。 アプリケーションは、検証の要求が想定されるイベント サブスクリプションに対するものであることを確認し、検証コードを Event Grid にエコーで返します。 このハンドシェイク メカニズムは、すべての Event Grid バージョンでサポートされます。

2. **ValidationURL ハンドシェイク (手動)** :場合によっては、エンドポイントのソース コードにアクセスして ValidationCode ハンドシェイクを実装することはできません。 たとえば、サード パーティのサービス ([Zapier](https://zapier.com) や [IFTTT](https://ifttt.com/) など) を使用する場合は、プログラムによって検証コードに応答できません。

   2018-05-01-preview バージョン以降では、手動の検証ハンドシェイクが Event Grid によってサポートされるようになりました。 API バージョン 2018-05-01-preview 以降を使用する SDK またはツールでイベント サブスクリプションを作成すると、Event Grid によって、サブスクリプション検証イベントのデータ部分で `validationUrl` プロパティが送信されます。 ハンドシェイクを完了するには、イベント データ内でその URL を探し、そこへ GET 要求を手動で送信します。 REST クライアントと Web ブラウザーのどちらでも使用できます。

   指定された URL は 5 分間有効です。 この期間、イベント サブスクリプションのプロビジョニング状態は `AwaitingManualAction` になります。 手動の検証を 5 分以内に完了しなかった場合、プロビジョニング状態は `Failed` に設定されます。 手動による検証を始める前に、もう一度イベント サブスクリプションを作成する必要があります。

    この認証メカニズムではまた、webhook エンドポイントが 200 の HTTP 状態コードを返すことも必要です。それにより、手動検証モードに設定される前に、検証イベントの POST が受け付けられたことを認識できるようになります。 つまり、エンドポイントが 200 を返しても、プログラムで検証の応答を戻さないと、モードは手動検証モードに移行されます。 5 分以内に検証 URL に GET が存在した場合、検証ハンドシェイクは成功したと見なされます。

> [!NOTE]
> 検証に自己署名証明書を使用することは、サポートされていません。 代わりに、証明機関 (CA) からの署名入り証明書を使用します。

### <a name="validation-details"></a>検証の詳細

* イベント サブスクリプションの作成時または更新時に、Event Grid はサブスクリプション検証イベントをターゲット エンドポイントに投稿します。 
* このイベントには、ヘッダー値 "aeg-event-type: SubscriptionValidation" が含まれています。
* イベント本文のスキーマは、他の Event Grid イベントと同じです。
* イベントの eventType プロパティは、`Microsoft.EventGrid.SubscriptionValidationEvent` です。
* イベントの data プロパティには、ランダムに生成された文字列を持つ `validationCode` プロパティが含まれています。 たとえば、"validationCode: acb13…" のようなプロパティです。
* イベント データには、`validationUrl` プロパティと、サブスクリプションを手動で検証するための URL も含まれます。
* 配列には、検証イベントのみが含まれています。 その他のイベントは、検証コードをエコーで返した後、別の要求で送信されます。
* EventGrid DataPlane SDK には、サブスクリプション検証イベント データとサブスクリプション検証の応答に対応するクラスがあります。

SubscriptionValidationEvent の例を以下に示します。

```json
[{
  "id": "2d1781af-3a4c-4d7c-bd0c-e34b19da4e66",
  "topic": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "subject": "",
  "data": {
    "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
    "validationUrl": "https://rp-eastus2.eventgrid.azure.net:553/eventsubscriptions/estest/validate?id=512d38b6-c7b8-40c8-89fe-f46f9e9622b6&t=2018-04-26T20:30:54.4538837Z&apiVersion=2018-05-01-preview&token=1A1A1A1A"
  },
  "eventType": "Microsoft.EventGrid.SubscriptionValidationEvent",
  "eventTime": "2018-01-25T22:12:19.4556811Z",
  "metadataVersion": "1",
  "dataVersion": "1"
}]
```

エンドポイントの所有権を証明するには、次の例に示すように、validationResponse プロパティで検証コードをエコーで返します。

```json
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

HTTP 200 OK 応答状態コードを返す必要があります。 HTTP 202 Accepted は有効な Event Grid サブスクリプション検証の応答として認識されません。 HTTP 要求は 30 秒以内に完了する必要があります。 操作が 30 秒以内に完了しない場合、操作は取り消され、5 秒後に再試行される場合があります。 すべての試行が失敗した場合、検証ハンドシェイク エラーとして扱われます。

または、検証 URL に GET 要求を手動で送信して、サブスクリプションを検証することができます。 イベント サブスクリプションは、検証されるまで保留状態にとどまります。 検証 URL ではポート 553 が使用されます。 ファイアウォール規則によってポート 553 がブロックされている場合、手動ハンドシェイクを正常に行うには、規則を更新する必要があります。

サブスクリプション検証ハンドシェイクの処理の例については、[C# のサンプル](https://github.com/Azure-Samples/event-grid-dotnet-publish-consume-events/blob/master/EventGridConsumer/EventGridConsumer/Function1.cs)に関するページをご覧ください。

### <a name="checklist"></a>チェック リスト

イベント サブスクリプションの作成時に表示される "指定されたエンドポイント https:\//your-endpoint-here の検証の試行に失敗しました。 詳細については、https:\//aka.ms/esvalidation を参照してください" のようなエラー メッセージは、検証ハンドシェイクでエラーが発生したことを示します。 このエラーを解決するには、次の点を確認します。

* ターゲット エンドポイントで実行されるアプリケーション コードを制御しているか。 たとえば、HTTP トリガー ベースの Azure 関数を記述する場合は、その関数を変更するためにアプリケーション コードにアクセスできるかどうか。
* アプリケーション コードにアクセスできる場合は、上記のサンプルに示すように ValidationCode ベースのハンドシェイクのメカニズムを実装します。

* アプリケーション コードにアクセスできない場合 (Webhook をサポートするサードパーティのサービスを使用している場合など) は、手動のハンドシェイクのメカニズムを使用できます。 検証イベントで validationUrl を受け取るために、2018-05-01-preview API バージョン以降を使用していることを確認してください (Event Grid の Azure CLI 拡張機能をインストールします)。 手動の検証ハンドシェイクを完了するには、`validationUrl` プロパティの値を取得し、Web ブラウザーでその URL にアクセスしてください。 検証が成功すると、そのことを示すメッセージが Web ブラウザーに表示されます。 イベント サブスクリプションの provisioningState には、"Succeeded" と表示されます。 

### <a name="event-delivery-security"></a>イベント配信のセキュリティ

#### <a name="azure-ad"></a>Azure AD

Event Grid の認証に Azure Active Directory を使用することで Webhook エンドポイントを保護し、エンドポイントにイベントを発行できます。 Azure Active Directory アプリケーションを作成すること、アプリケーションで Event Grid を認証するためのロールとサービス プリンシプルを作成すること、Azure AD アプリケーションを使用するようにイベント サブスクリプションを構成することが必要になります。 [Event Grid で AAD を構成する方法について学習してください](secure-webhook-delivery.md)。

#### <a name="query-parameters"></a>クエリ パラメーター
イベント サブスクリプションを作成するときに、webhook URL にクエリ パラメーターを追加することで、webhook エンドポイントをセキュリティで保護できます。 これらのクエリ パラメーターのいずれかを、[アクセス トークン](https://en.wikipedia.org/wiki/Access_token)などのシークレットに設定します。 Webhook はシークレットを使用して、イベントが有効なアクセス許可を持つ Event Grid からのものであることを認識できます。 Event Grid には、webhook へのすべてのイベント配信にこれらのクエリ パラメーターが含められます。

イベント サブスクリプションを編集すると、Azure [CLI](https://docs.microsoft.com/cli/azure?view=azure-cli-latest) で [--include-full-endpoint-url](https://docs.microsoft.com/cli/azure/eventgrid/event-subscription?view=azure-cli-latest#az-eventgrid-event-subscription-show) パラメーターを使用した場合を除き、クエリ パラメーターが表示されなくなるか、返されなくなります。

最後に、Azure Event Grid がサポートするのは HTTPS webhook エンドポイントのみであることにご注意ください。

## <a name="event-subscription"></a>イベント サブスクリプション

イベントにサブスクライブするには、イベント ソースとハンドラーへのアクセスがあることを証明する必要があります。 WebHook を所有していることの証明については、前のセクションで説明しました。 WebHook ではないイベント ハンドラー (イベント ハブ、キュー ストレージなど) を使用している場合は、そのリソースへの書き込みアクセスが必要です。 このアクセス許可のチェックにより、未認証のユーザーはリソースにイベントを送信できなくなります。

イベント ソースであるリソースに対する **Microsoft.EventGrid/EventSubscriptions/Write** アクセス許可を持っている必要があります。 リソースのスコープで新しいサブスクリプションを作成するため、このアクセス許可が必要です。 必要なリソースは、サブスクライブしているのがシステム トピックかカスタム トピックかによって異なります。 ここでは、この 2 つの種類について説明します。

### <a name="system-topics-azure-service-publishers"></a>システム トピック (Azure サービスの発行元)

システム トピックの場合、イベントを発行するリソースのスコープに新しいイベント サブスクリプションを書き込むアクセス許可が必要です。 リソースの形式は次のとおりです。`/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/{resource-provider}/{resource-type}/{resource-name}`

たとえば、**myacct** というストレージ アカウントでイベントにサブスクライブするには、Microsoft.EventGrid/EventSubscriptions/Write アクセス許可が必要です。`/subscriptions/####/resourceGroups/testrg/providers/Microsoft.Storage/storageAccounts/myacct`

### <a name="custom-topics"></a>カスタム トピック

カスタム トピックの場合、Event Grid トピックのスコープに新しいイベント サブスクリプションを書き込むアクセス許可が必要です。 リソースの形式は次のとおりです。`/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.EventGrid/topics/{topic-name}`

たとえば、**mytopic** というカスタム トピックにサブスクライブするには、Microsoft.EventGrid/EventSubscriptions/Write アクセス許可が必要です。`/subscriptions/####/resourceGroups/testrg/providers/Microsoft.EventGrid/topics/mytopic`

## <a name="custom-topic-publishing"></a>カスタム トピックの発行

カスタム トピックは、Shared Access Signature (SAS) またはキー認証を使用します。 SAS が推奨されますが、キー認証はプログラミングが簡単で、既存の多くの webhook 発行元と互換性があります。 

認証値は、HTTP ヘッダーに含めます。 SAS の場合は、ヘッダー値に **aeg-sas-token** を使用します。 キー認証の場合は、ヘッダー値に **aeg-sas-key** を使用します。

### <a name="key-authentication"></a>キー認証

キー認証は、最も単純な形式の認証です。 次の形式を使用します。`aeg-sas-key: <your key>`

たとえば、次のようにキーを渡します。

```
aeg-sas-key: VXbGWce53249Mt8wuotr0GPmyJ/nDT4hgdEj9DpBeRr38arnnm5OFg==
```

### <a name="sas-tokens"></a>SAS トークン

Event Grid の SAS トークンには、リソース、有効期限、および署名が含まれます。 SAS トークンの形式は `r={resource}&e={expiration}&s={signature}` です。

リソースは、イベントを送信する Event Grid グリッド トピックのパスです。 たとえば、有効なリソース パスは次のとおりです。`https://<yourtopic>.<region>.eventgrid.azure.net/eventGrid/api/events`

キーから署名を生成します。

たとえば、有効な **aeg-sas-token** 値は次のとおりです。

```http
aeg-sas-token: r=https%3a%2f%2fmytopic.eventgrid.azure.net%2feventGrid%2fapi%2fevent&e=6%2f15%2f2017+6%3a20%3a15+PM&s=a4oNHpRZygINC%2fBPjdDLOrc6THPy3tDcGHw1zP4OajQ%3d
```

次の例では、Event Grid で使用する SAS トークンを作成します。

```cs
static string BuildSharedAccessSignature(string resource, DateTime expirationUtc, string key)
{
    const char Resource = 'r';
    const char Expiration = 'e';
    const char Signature = 's';

    string encodedResource = HttpUtility.UrlEncode(resource);
    var culture = CultureInfo.CreateSpecificCulture("en-US");
    var encodedExpirationUtc = HttpUtility.UrlEncode(expirationUtc.ToString(culture));

    string unsignedSas = $"{Resource}={encodedResource}&{Expiration}={encodedExpirationUtc}";
    using (var hmac = new HMACSHA256(Convert.FromBase64String(key)))
    {
        string signature = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(unsignedSas)));
        string encodedSignature = HttpUtility.UrlEncode(signature);
        string signedSas = $"{unsignedSas}&{Signature}={encodedSignature}";

        return signedSas;
    }
}
```

### <a name="encryption-at-rest"></a>保存時の暗号化

Event Grid サービスによってディスクに書き込まれるすべてのイベントまたはデータは、保存時に確実に暗号化されるように、Microsoft マネージド キーによって暗号化されます。 また、[Event Grid の再試行ポリシー](delivery-and-retry.md)に従って、イベントまたはデータが保持される最大期間は 24 時間です。 Event Grid では、24時間またはイベントの Time-To-Live のいずれか少ない方が経過すると、すべてのイベントまたはデータが自動的に削除されます。

## <a name="next-steps"></a>次のステップ

* Event Grid の概要については、[Event Grid の紹介](overview.md)に関する記事を参照してください。
