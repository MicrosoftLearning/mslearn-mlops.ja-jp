---
lab:
  title: Azure Machine Learning を使用して MLOps ソリューションを計画および準備する
  description: 開発および運用環境を設計し、ワークスペース、レジストリ、データ資産をプロビジョニングするための Azure CLI 自動化を計画します。
  level: 300
  duration: 45 minutes
---

# Azure Machine Learning を使用して MLOps ソリューションを計画および準備する

機械学習ソリューションが成熟するにつれて、1 回限りの実験から、モデルのトレーニング、評価、デプロイのための反復可能なプロセスに移行します。 このプロセスをサポートするには、依存している Azure リソースを確実にプロビジョニングできる環境戦略と自動化が必要です。

この演習では、次の 3 つのフェーズで MLOps 対応環境の計画と準備に重点を置きます。

- 開発用 Azure Machine Learning ワークスペースと関連リソースをプロビジョニングする既存の Azure CLI スクリプトを確認します。
- モデルに共有 Azure Machine Learning レジストリを使用するが、開発データから運用データを分離する運用環境を設計します。
- Azure CLI コマンドを使用して、開発および運用環境をサポートするために既存の `infra/setup.sh` スクリプトを拡張する方法を計画します。

このラボで設計するすべてのコマンドを実行するわけではありません。 代わりに、実際の MLOps のセットアップを自動化するために使用するアーキテクチャ、名前付け、スクリプト パターンについて理解することに重点を置きます。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free?azure-portal=true)が必要です。

## 既存の開発環境スクリプトを確認する

ポータルを使用して Azure Machine Learning を操作するために必要なリソースと資産を手動で作成できます。 ただし、作業を簡単に追跡して自動化する場合は、CLI コマンドを使用する方が簡単です。 シェル スクリプトで CLI コマンドを組み合わせることができます。 このセクションでは、シェル スクリプトを確認し、それがプロビジョニングする内容を特定します。

1. ブラウザーで、Azure portal (`https://portal.azure.com/`) を開き、Microsoft アカウントでサインインします。
1. ページ上部の検索ボックスの右側にある \[>_] (*Cloud Shell*) ボタンを選びます。 これにより、ポータルの下部に Cloud Shell ペインが開きます。
1. メッセージが表示されたら、 **[Bash]** を選択します。 Cloud Shell を初めて開いたときに、使用するシェルの種類 (*Bash* または *PowerShell*) を選択するように求められます。
1. 正しいサブスクリプションが指定されていることと、**[ストレージ アカウントは不要]** が選択されていることを確認します。 **適用**を選択します。
1. ターミナルで、次のコマンドを入力して、このリポジトリをクローンします。

    ```azurecli
    rm -r mslearn-mlops -f
    git clone https://github.com/MicrosoftLearning/mslearn-mlops.git mslearn-mlops
    ```

    > コピーしたコードを Cloud Shell に貼り付けるには、`SHIFT + INSERT` を使用します。

1. リポジトリがクローンされたら、次のコマンドを入力して `infra` フォルダーに変更し、セットアップ スクリプトを実行します。

    ```azurecli
    cd mslearn-mlops/infra
    code setup.sh
    ```

    > [!NOTE]
    > `code` コマンドを使用できない場合は、新しい Cloud Shell エクスペリエンスを使用しています。 ツール バーの **[クラシック Cloud Shell に切り替える]** を選択し、**[確認]** を選択して、クラシック Cloud Shell に切り替えます。 次に、コマンドをもう一度実行します。

1. スクリプトを確認し、現在の**開発**環境用に作成されたリソースを特定します。
    - ランダム化されたサフィックスを持つリソース グループ (`rg-ai300-l...`など)。
    - Azure Machine Learning ワークスペース (`mlw-ai300-l...` など)。
    - 対話型作業用のコンピューティング インスタンス。
    - トレーニング ジョブ用のコンピューティング クラスター。
    - `data/diabetes-data` フォルダーにある糖尿病のトレーニング データのデータ資産。

1. スクリプトで次の手順が行われる方法に注目してください。
    - 名前の競合を回避するために、ランダムなサフィックスを生成する。
    - **Microsoft.MachineLearningServices** リソース プロバイダーを登録する。
    - リソース グループとワークスペースの既定値を設定して、後続の `az ml` コマンドで自動的に使用されるようにする。

このスクリプトが開発のために行う内容を理解することで、運用環境で変更または追加することを考える準備が整いました。

## MLOps 用の開発および運用環境を設計する

さらにコマンドを追加する前に、環境をどのようにするかを明確に把握しておく必要があります。 一般的な MLOps パターンは次のとおりです。

- 開発と運用で**個別のワークスペース**を使用して、運用環境のワークロードから実験を分離する。
- **共有 Azure Machine Learning レジストリ**を使用してワークスペース間でモデルや環境などの再利用可能な資産を格納し昇格させる。
- 開発環境に運用データが表示されないように**個別のデータ資産**を使用する。

このラボでは、次のターゲット アーキテクチャを想定してください。

- 実験用の**開発ワークスペース** :
    - リソース グループ: `rg-ai300-dev-<suffix>`
    - ワークスペース: `mlw-ai300-dev-<suffix>`
    - データ資産: `data/diabetes-data` フォルダー内のサンプル データを指す `diabetes-dev-folder`。
- 運用トレーニングとデプロイ用の**運用ワークスペース** :
    - リソース グループ: `rg-ai300-prod-<suffix>`
    - ワークスペース: `mlw-ai300-prod-<suffix>`
    - データ資産: `production/data` フォルダー内の大きなデータセットを指す `diabetes-prod-folder`。
- 再利用可能な資産の**共有レジストリ**:
    - レジストリ: 中央リソース グループの `mlr-ai300-shared-<suffix>`。
    - どちらのワークスペースも、このレジストリからモデルと環境をプッシュおよびプルできます。

> [!NOTE]
> 運用環境では、通常、開発、テスト、運用向けに個別のワークスペース (および、場合によっては個別のサブスクリプション) を作成します。このラボでは、**設計**と**スクリプト**に焦点を当てます。 追加の Azure コストが生じないようにする場合は、実際に複数のワークスペースを作成する必要はありません。

この設計に留意することで、自動化に追加する Azure CLI コマンドを計画する準備ができます。

## 運用ワークスペースと共有レジストリの Azure CLI コマンドを計画する

次に、ターゲット アーキテクチャを Azure CLI コマンドにマップします。 すぐに実行するのではなく、このセクションを使って、使用するリソースと名前付け規則について検討します。

1. Cloud Shell エディターで、既存のスクリプトに基づいて新しいファイルを作成し、安全に実験を行えるようにします。

    ```bash
    cp setup.sh setup-prod-design.sh
    code setup-prod-design.sh
    ```

1. 新しいファイルの先頭に、環境と共有レジストリの両方の変数を追加します。 次に例を示します。

    ```bash
    # Existing random suffix
    guid=$(cat /proc/sys/kernel/random/uuid)
    suffix=${guid//[-]/}
    suffix=${suffix:0:18}

    # Dev environment
    DEV_RESOURCE_GROUP="rg-ai300-dev-${suffix}"
    DEV_WORKSPACE_NAME="mlw-ai300-dev-${suffix}"

    # Prod environment
    PROD_RESOURCE_GROUP="rg-ai300-prod-${suffix}"
    PROD_WORKSPACE_NAME="mlw-ai300-prod-${suffix}"

    # Shared registry (one per subscription/region)
    REGISTRY_RESOURCE_GROUP="rg-ai300-reg-${suffix}"
    REGISTRY_NAME="mlr-ai300-shared-${suffix}"
    ```

1. `infra` フォルダーで、`registry.yml` を開き、共有レジストリを定義する値を確認します。 Azure CLI は、この YAML ファイルを文字通り読み取るので、Bash スクリプトでは、create コマンドを実行する前に、動的レジストリ名とプライマリ リージョンをファイルに挿入する必要があります。 このラボでは、次のように `registry.yml` のプレースホルダーを使用します。

    ```yml
    name: REGISTRY_NAME_PLACEHOLDER
    tags:
      description: Shared registry for approved machine learning assets across workspaces
    location: PRIMARY_REGION_PLACEHOLDER
    replication_locations:
      - location: PRIMARY_REGION_PLACEHOLDER
    ```

1. 独自のリソース グループに**共有レジストリ**を作成するコマンドを計画します。 次に例を示します。

    ```azurecli
    # Create a resource group for the shared registry
    az group create --name $REGISTRY_RESOURCE_GROUP --location $RANDOM_REGION

    # Render registry.yml with the dynamic values from the script
    sed \
        -e "s|REGISTRY_NAME_PLACEHOLDER|$REGISTRY_NAME|g" \
        -e "s|PRIMARY_REGION_PLACEHOLDER|$RANDOM_REGION|g" \
        registry.yml > registry.generated.yml

    # Create an Azure Machine Learning registry from the rendered YAML file
    az ml registry create \
        --file registry.generated.yml \
        --resource-group $REGISTRY_RESOURCE_GROUP
    ```

    プライマリ レジストリ リージョンは、YAML 定義に 2 回表示されます (`location` に 1 回、`replication_locations` にもう 1 回表示されます)。 Bash 変数から YAML をレンダリングすると、それらの値の一貫性が保たれます。

1. **運用**リソース グループとワークスペースを作成するコマンドを計画します。 既存の開発ワークスペースと同じパターンに従いますが、運用名を使用します。

    ```azurecli
    # Create the production resource group
    az group create --name $PROD_RESOURCE_GROUP --location $RANDOM_REGION

    # Create the production Azure Machine Learning workspace
    az ml workspace create \
        --name $PROD_WORKSPACE_NAME \
        --resource-group $PROD_RESOURCE_GROUP \
        --location $RANDOM_REGION
    ```

1. 最後に、開発および運用データを分離したままにするデータ資産を計画します。 実験には **dev** フォルダーを使用し、運用トレーニングには **production** フォルダーを使用します。

    ```azurecli
    # In the dev workspace: data asset that points to experimentation data
    az configure --defaults group=$DEV_RESOURCE_GROUP workspace=$DEV_WORKSPACE_NAME
    az ml data create \
        --type uri_folder \
        --name diabetes-dev-folder \
        --path ../data/diabetes-data

    # In the prod workspace: data asset that points to production data
    az configure --defaults group=$PROD_RESOURCE_GROUP workspace=$PROD_WORKSPACE_NAME
    az ml data create \
        --type uri_folder \
        --name diabetes-prod-folder \
        --path ../production/data
    ```

> [!IMPORTANT]
> このラボでは、追加のリソース グループとワークスペースを作成する新しいコマンドを実行する**必要はありません**。 開発および運用リソースが明確に分離され、運用データが開発環境から区別されたままになるようにスクリプトを構成する方法を理解することに重点を置きます。 スクリプトを実行する場合は、次のセクションの省略可能な手順に従ってください。

これらのコマンドを計画して、アーキテクチャを、シェル スクリプトで自動化できる具体的な Azure CLI 操作に変換することができました。

## 省略可能: 設計のスクリプトを実行する

設計の動作を確認する場合は、参照ソリューションに対してスクリプトを検証してから実行できます。 この手順はオプションであり、リソースが存在している間は Azure のコストが発生します。

1. Cloud Shell で、`infra` フォルダーにいることを確認します。

    ```bash
    cd mslearn-mlops/infra
    ```

1. リポジトリには、完全なスクリプトの外観を示す参照スクリプト `infra/setup-mlops-envs.sh` が含まれています。 ご自分の `setup-prod-design.sh` と比較して、作業内容を確認してみましょう。

    ```bash
    diff setup-prod-design.sh setup-mlops-envs.sh
    ```

    違いを確認し、必要に応じてスクリプトを更新します。

1. スクリプトに問題がなければ、スクリプトを実行可能にして実行します。

    ```bash
    chmod +x setup-prod-design.sh
    ./setup-prod-design.sh
    ```

1. スクリプトが完了したら、Azure portal で次のリソースを確認します。
    - 開発、運用、共有レジストリ向けの新しいリソース グループ。
    - 開発と運用向けの分離されたワークスペース。
    - それぞれのワークスペースにあるデータ資産 `diabetes-dev-folder` と `diabetes-prod-folder`。

1. 調査が完了したら、料金が発生し続けないように、作成した追加のリソース グループを必ず削除してください。

## 複数の環境のセットアップ スクリプトを拡張する方法を計画する

実際の MLOps プロジェクトでは、オンデマンドで適切な環境をプロビジョニングできる単一の自動化エントリ ポイントが必要です。 このセクションでは、この目標をサポートするために既存のスクリプトを進化させる方法について考えてみます。

1. スクリプトに**ターゲット環境**を渡す方法を決定します。 たとえば、`dev` や `prod` などのパラメーターを受け取ることができます。

    ```bash
    ENVIRONMENT=${1:-dev}
    ```

1. 環境に基づいて、リソース グループとワークスペース変数を設定する方法を計画します。 次に例を示します。

    ```bash
    if [ "$ENVIRONMENT" = "prod" ]; then
        RESOURCE_GROUP=$PROD_RESOURCE_GROUP
        WORKSPACE_NAME=$PROD_WORKSPACE_NAME
    else
        RESOURCE_GROUP=$DEV_RESOURCE_GROUP
        WORKSPACE_NAME=$DEV_WORKSPACE_NAME
    fi
    ```

1. リソースの中で**共有**すべきものと、**分離**すべきものを検討します。
    - レジストリは開発と運用の間で共有されるため、1 回作成して再利用します。
    - ワークスペース、コンピューティング、データ資産は環境固有であるため、さまざまなセキュリティとアクセス制御を適用できます。

1. このスクリプトが、より広範な MLOps ワークフローにどのように適合するかを考慮します。
    - **GitHub Actions** では、承認済みのモデルをデプロイする場合、pull request と `prod` を検証するときに、`dev` を使用してスクリプトを呼び出します。
    - **ローカル開発**では、データ サイエンティストは、`dev` を使用してスクリプトを呼び出して、実験環境をゼロから再作成できます。

以前のラボの動作を変更せずに、既存のスクリプトをより柔軟なプロビジョニング ツールに進化させる方法について、明確なプランができました。

## Azure リソースをクリーンアップする

Azure Machine Learning を調べ終わったら、不要な Azure のコストを避けるために作成したリソースを削除する必要があります。

1. [Azure Machine Learning スタジオ] タブを閉じて、Azure portal に戻ります。
1. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
1. このラボの実験中に追加のリソース グループ (`rg-ai300-dev-...`、`rg-ai300-prod-...`、`rg-ai300-reg-...`など) を作成した場合は、それらのリソース グループも削除します。
1. 元の `setup.sh` スクリプトによって作成された **[rg-ai300-...]** リソース グループを選択します。
1. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
1. リソース グループ名を入力して、削除することを確認し、**[削除]** を選択します。
