---
title: 変数の代入
description: ソリューション開発のための、Azure SQL Data Warehouse での T-SQL 変数の代入に関するヒント。
services: synapse-analytics
author: XiaoyuMSFT
manager: craigg
ms.service: synapse-analytics
ms.topic: conceptual
ms.subservice: ''
ms.date: 04/17/2018
ms.author: xiaoyul
ms.reviewer: igorstan
ms.custom: seo-lt-2019
ms.openlocfilehash: 0adcd9bdf92b7ec649b7d91ca0e655fc006b3549
ms.sourcegitcommit: 8a9c54c82ab8f922be54fb2fcfd880815f25de77
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/27/2020
ms.locfileid: "80351658"
---
# <a name="assigning-variables-in-azure-sql-data-warehouse"></a>Azure SQL Data Warehouse での変数の代入

ソリューション開発のための、Azure SQL Data Warehouse での T-SQL 変数の代入に関するヒント。

## <a name="setting-variables-with-declare"></a>DECLARE を使用した変数の設定

SQL Data Warehouse の変数は、`DECLARE` ステートメントまたは `SET` ステートメントを使用して設定します。 DECLARE を使用した変数の初期化は、SQL Data Warehouse で変数値を設定する最も柔軟性の高い方法の 1 つです。

```sql
DECLARE @v  int = 0
;
```

DECLARE を使用して、一度に複数の変数を設定することもできます。 SELECT または UPDATE を使用して次を行うことはできません。

```sql
DECLARE @v  INT = (SELECT TOP 1 c_customer_sk FROM Customer where c_last_name = 'Smith')
,       @v1 INT = (SELECT TOP 1 c_customer_sk FROM Customer where c_last_name = 'Jones')
;
```

同じ DECLARE ステートメントで変数を初期化し、使用することはできません。 要点を示すために、次の例では、同じ DECLARE ステートメントで  **を初期化し、使用しています。このようなステートメントは使用**できません@p1。 次の例は、エラーが発生します。

```sql
DECLARE @p1 int = 0
,       @p2 int = (SELECT COUNT (*) FROM sys.types where is_user_defined = @p1 )
;
```

## <a name="setting-values-with-set"></a>SET を使用した値の設定

SET は、1 つの変数を設定する際によく使用されるメソッドです。

次のステートメントはすべて、SET を使用して変数を設定する有効な方法です。

```sql
SET     @v = (Select max(database_id) from sys.databases);
SET     @v = 1;
SET     @v = @v+1;
SET     @v +=1;
```

SET を使用して設定できる変数は、一度に 1 つに限られます。 ただし、複合演算子は許容されます。

## <a name="limitations"></a>制限事項

UPDATE は変数代入には使用できません。

## <a name="next-steps"></a>次のステップ

開発についてのその他のヒントは、[開発の概要](sql-data-warehouse-overview-develop.md)に関するページをご覧ください。
