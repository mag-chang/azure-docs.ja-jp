---
title: ロード バランサーでパブリック IP アドレスを管理する
titleSuffix: Azure Virtual Network
description: Azure Load Balancer でパブリック IP アドレスを使用する方法と、構成を変更する方法について説明します。
author: asudbring
ms.author: allensu
ms.service: virtual-network
ms.subservice: ip-services
ms.topic: how-to
ms.date: 06/28/2021
ms.custom: template-how-to
ms.openlocfilehash: 1d3f8f07412e55da49c8502fde57b9e45dc4a786
ms.sourcegitcommit: 692382974e1ac868a2672b67af2d33e593c91d60
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 10/22/2021
ms.locfileid: "130217413"
---
# <a name="manage-a-public-ip-address-with-a-load-balancer"></a>ロード バランサーでパブリック IP アドレスを管理する

パブリック ロード バランサーは、TCP および UDP トラフィックをバックエンド プールに分散するためのレイヤー 4 ソリューションです。 ロード バランサーでは、Basic および Standard SKU を使用できます。 これらの SKU は、パブリック IP アドレスの Basic および Standard SKU に対応しています。

ロード バランサーに関連付けられたパブリック IP は、インターネットに接続するフロントエンド IP 構成として機能します。 フロントエンドは、バックエンド プール内のリソースにアクセスするために使用されます。 バックエンド プールのメンバーがインターネットにエグレスするために、フロントエンド IP を使用できます。 

Basic SKU Azure Load Balancer は、可用性オプションと機能セットでは制限があります。 Standard SKU ロード バランサーと IP アドレスの組み合わせは、運用ワークロードで推奨されるデプロイです。 Standard SKU IP アドレスでは、サポートされているリージョンの可用性ゾーンがサポートされます。 

この記事では、サブスクリプション内の既存のパブリック IP アドレスを使用してロード バランサーを作成する方法について学習します。 

ロード バランサーに関連付けられている現在のパブリック IP を変更する方法について説明します。 

アウトバウンド バックエンド プールのフロントエンド構成をパブリック IP プレフィックスに変更する方法について説明します。  

最後に、この記事では、ロード バランサーでパブリック IP とパブリック IP プレフィックスを使用する固有の側面について確認します。 

> [!NOTE]
> この記事の例では、Standard SKU ロード バランサーとパブリック IP を使用します。 Basic SKU ロード バランサーでは、ロード バランサーとパブリック IP リソースの作成時に SKU を選択する場合を除き、手順は同じです。 Basic ロード バランサーでは、アウトバウンド規則またはパブリック IP プレフィックスはサポートされていません。 

## <a name="prerequisites"></a>前提条件

- アクティブなサブスクリプションが含まれる Azure アカウント。 [無料で作成できます](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio)。
- サブスクリプション内の 2 つの Standard SKU のパブリック IP アドレス。 この IP アドレスは、どのリソースにも関連付けることができません。 Standard SKU のパブリック IP アドレスの作成の詳細については、[パブリック IP の作成 - Azure portal](./create-public-ip-portal.md) に関するページを参照してください。
    - この記事の例では、新しいパブリック IP アドレスに **myStandardPublicIP-1** および **myStandardPublicIP-2** という名前を付けます。
- サブスクリプション内のパブリック IP プレフィックス。 パブリック IP プレフィックスの作成の詳細については、「[Azure portal を使用してパブリック IP アドレス プレフィックスを作成する](./create-public-ip-prefix-portal.md)」を参照してください。
    - この記事の例では、新しいパブリック IP プレフィックスに **myPublicIPPrefixOutbound** という名前を付けます。

## <a name="create-load-balancer-existing-public-ip"></a>ロード バランサーの既存のパブリック IP を作成する

このセクションでは、Standard SKU ロード バランサーを作成します。 前提条件で作成した IP アドレスをロード バランサーのフロントエンド IP として選択します。

1. [Azure portal](https://portal.azure.com) にサインインします。

2. ポータルの上部にある検索ボックスに、「**ロード バランサー**」と入力します。

3. 検索結果で **[ロード バランサー]** を選択します。

4. **[+ 作成]** を選択します。

5. **[ロード バランサーの作成]** で、次の情報を入力または選択します。

    | 設定 | 値 |
    | ------- | ----- |
    | **プロジェクトの詳細** |   |
    | サブスクリプション | サブスクリプションを選択します。 |
    | Resource group | **[新規作成]** を選択します。 </br> 「**myResourceGroupIP**」と入力します。 </br> **[OK]** を選択します。 |
    | **インスタンスの詳細** |   |
    | 名前 | 「**myLoadBalancer**」と入力します。 |
    | リージョン | **[(米国) 米国西部 2]** を選択します。 |
    | Type | 既定値の **[パブリック]** のままにします。 |
    | SKU | 規定値である **[標準]** のままにします。 |
    | レベル | 既定値の **[地域]** のままにします。 |
    | **パブリック IP アドレス** |   |
    | パブリック IP アドレス | **[既存のものを使用]** を選択します。 |
    | パブリック IP アドレスの選択 | **[myStandardPublicIP-1]** を選択します。 |
    | パブリック IPv6 アドレスを追加する | 既定値の **[いいえ]** のままにします。 |

6. **[確認と作成]** タブを選択するか、青い **[確認と作成]** ボタンを選択します。 

7. **［作成］** を選択します

> [!NOTE]
> これは、ロード バランサーの単純なデプロイです。 詳細な構成と設定については、「[クイックスタート: Azure portal を使用して、VM の負荷分散を行うパブリック ロード バランサーを作成する](../../load-balancer/quickstart-load-balancer-standard-public-portal.md)」を参照してください
>
> Azure Load Balancer の詳細については、「[Azure Load Balancer の概要](../../load-balancer/load-balancer-overview.md)」を参照してください

## <a name="change-or-remove-public-ip-address"></a>パブリック IP アドレスを変更または削除する

このセクションでは、Azure portal にサインインし、ロード バランサーの IP アドレスを変更します。 

Azure Load Balancer では、IP アドレスがフロントエンドに関連付けられている必要があります。 イングレスおよびエグレスのトラフィックのフロントエンドとして、個別のパブリック IP アドレスを使用できます。 

IP を変更するには、以前に作成した新しいパブリック IP アドレスをロード バランサーのフロントエンドに関連付けます。

1. [Azure portal](https://portal.azure.com) にサインインします。

2. ポータルの上部にある検索ボックスに、「**ロード バランサー**」と入力します。

3. 検索結果で **[ロード バランサー]** を選択します。

4. **[ロード バランサー]** で、 **[myLoadBalancer]** か、変更するロード バランサーを選択します。

5. **[myLoadBalancer]** の設定で、 **[フロントエンド IP の構成]** を選択します。

6. **[フロントエンド IP の構成]** で、 **[LoadBalancerFrontend]** またはお使いのロード バランサーのフロントエンドを選択します。

7. ロード バランサーのフロントエンド構成の **[パブリック IP アドレス]** で、 **[myStandardPublicIP-2]** を選択します。

8. **[保存]** を選択します。

9. ロード バランサーのフロントエンドに **myStandardPublicIP-2** という名前の新しい IP アドレスが表示されていることを確認します。

    > [!NOTE]
    > これらの手順は、リージョン間ロード バランサーで有効です。 リージョン間ロード バランサーの詳細については、「 **[リージョン間のロード バランサー](../../load-balancer/cross-region-overview.md)** 」を参照してください。


## <a name="add-public-ip-prefix"></a>パブリック IP プレフィックスを追加する

Standard Load Balancer では、送信元ネットワーク アドレス変換 (SNAT) のアウトバウンド規則がサポートされています。 SNAT を使用すると、バックエンド プールのメンバーによるインターネットへのエグレスが可能になります。 パブリック IP プレフィックスでは、アウトバウンド接続に複数の IP アドレスを許可することで、SNAT の拡張性を向上させます。 

複数の IP を使用すると、SNAT ポートの枯渇を回避できます。 各フロントエンド IP には、ロード バランサーで使用できる 64,000 個のエフェメラル ポートが用意されています。 詳しくは、[アウトバウンド規則](../../load-balancer/outbound-rules.md)に関するページをご覧ください。

このセクションでは、パブリック IP プレフィックスを使用するためにアウトバウンド接続に使用されるフロントエンド構成を変更します。

> [!IMPORTANT]
> このセクションを完了するには、アウトバウンド フロントエンド構成とアウトバウンド規則がデプロイされたロード バランサーが必要です。

1. [Azure portal](https://portal.azure.com) にサインインします。

2. ポータルの上部にある検索ボックスに、「**ロード バランサー**」と入力します。

3. 検索結果で **[ロード バランサー]** を選択します。

4. **[ロード バランサー]** で、 **[myLoadBalancer]** か、変更するロード バランサーを選択します。

5. **[myLoadBalancer]** の設定で、 **[フロントエンド IP の構成]** を選択します。

6. **[フロントエンド IP 構成]** で、 **[LoadBalancerFrontendOutbound]** またはお使いのロード バランサーのアウトバウンド フロントエンドを選択します。

7. **[IP の種類]** で、 **[パブリック IP プレフィックス]** を選びます。

8. **[パブリック IP プレフィックス]** で、以前に作成したパブリック IP プレフィックス **myPublicIPPrefixOutbound** を選択します。

9. **[保存]** を選択します。

10. **[フロントエンド IP 構成]** で、IP プレフィックスがアウトバウンド フロントエンド構成に追加されたことを確認します。

## <a name="more-information"></a>詳細情報

* リージョン間ロード バランサーは、複数のリージョンにまたがることがある特殊な種類の Standard パブリック ロード バランサーです。 リージョン間ロード バランサーのフロントエンドは、Standard SKU パブリック IP のグローバル階層オプションでのみ使用できます。 リージョン間ロード バランサーのフロントエンド IP に送信されるトラフィックは、リージョンのパブリック ロード バランサー全体に分散されます。 リージョン フロントエンド IP は、リージョン間ロード バランサーのバックエンド プールに含まれます。 詳細については、「[リージョン間ロード バランサー](../../load-balancer/cross-region-overview.md)」を参照してください。

* 既定では、パブリック ロード バランサーでは、同じバックエンド ポートを使用する複数の負荷分散規則を使用することはできません。 同じバックエンド ポートに対する複数の規則構成が必要な場合は、負荷分散規則のフローティング IP オプションを有効にします。 この設定により、バックエンド プールに送信されるトラフィックの宛先 IP アドレスが上書きされます。 フローティング IP が有効になっていない場合、宛先はバックエンド プールのプライベート IP になります。 フローティング IP が有効になっている場合、宛先 IP はロード バランサーのフロントエンド パブリック IP になります。 このトラフィックを正しく受信するには、バックエンド インスタンスのネットワーク構成でこのパブリック IP が構成されている必要があります。 インスタンスでフロントエンド IP アドレスが構成されたループバック インターフェイス。 詳細については、「[Azure Load Balancer のフローティング IP の構成](../../load-balancer/load-balancer-floating-ip.md)」を参照してください

* ロード バランサーの設定では、多くの場合、バックエンド プールのメンバーにインスタンス レベルのパブリック IP も割り当てられることがあります。 このアーキテクチャを使用している場合、これらの IP にトラフィックを直接送信すると、ロード バランサーはバイパスされます。 

* Standard パブリック ロード バランサーとパブリック IP アドレスの両方に、キープアライブが聞こえるまでに接続を開いたままにする時間に関する TCP タイムアウト値を割り当てることができます。 パブリック IP がロード バランサーのフロントエンドとして割り当てられている場合、IP のタイムアウト値が優先されます。 この設定は、ロード バランサーへのインバウンド接続にのみ適用されることに注意してください。 詳細については、[ロード バランサーの TCP リセット](../../load-balancer/load-balancer-tcp-reset.md)に関するページを参照してください

## <a name="caveats"></a>注意事項

* Standard パブリック ロード バランサーでは、フロントエンド パブリック IP またはパブリック IP プレフィックスとして Standard SKU の静的 IPv6 アドレスを使用できます。  すべてのデプロイは、IPv4 と IPv6 の両方のフロントエンドを備えるデュアル スタックである必要があります。 NAT64 変換は使用できません。 詳細については、「[Azure で IPv6 デュアル スタック アプリケーションを展開する - PowerShell](../../load-balancer/virtual-network-ipv4-ipv6-dual-stack-standard-load-balancer-powershell.md)」を参照してください (Basic パブリック ロード バランサーは、フロントエンドのパブリック IP として Basic SKU の動的 IPv6 アドレスを使用できます)。

* パブリック ロード バランサーに複数のフロントエンドが割り当てられている場合、特定のバックエンド インスタンスからのフローを特定の IP 上のエグレスに割り当てる方法は存在しません。  詳細については、「[Azure Load Balancer の複数のフロントエンド](../../load-balancer/load-balancer-multivip-overview.md)」を参照してください
## <a name="next-steps"></a>次のステップ

この記事では、ロード バランサーを作成し、既存のパブリック IP を使用する方法について説明しました。 

ロード バランサーのフロントエンド構成で IP アドレスを置き換えました。 

最後に、パブリック IP プレフィックスを使用するようにアウトバウンド フロントエンド構成を変更しました。

- Azure Load Balancer の詳細については、[Azure Load Balancer とは何か](../../load-balancer/load-balancer-overview.md)に関する記事を参照してください。
- Azure のパブリック IP アドレスの詳細については、「[パブリック IP アドレス](./public-ip-addresses.md)」を参照してください。