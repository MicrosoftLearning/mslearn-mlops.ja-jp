---
lab:
  title: GitHub Actions を使用してモデル トレーニングを自動化する
  description: GitHub と Azure Machine Learning を安全に統合し、GitHub Actions ワークフローを使用してモデル トレーニングを自動化します。
  level: 300
  duration: 45 minutes
---

# GitHub Actions を使用してモデル トレーニングを自動化する

機械学習ソリューションが成熟するにつれて、Azure Machine Learning でのアドホック実験の実行から、反復可能なトレーニング ワークフローの自動化に移行します。 GitHub Actions を使用すると、既存のソース管理プラクティスに適合する安全でトレース可能なワークフローを使用して、必要なときにいつでも Azure Machine Learning ジョブを実行できます。

この演習では、GitHub Actions を使用して次の 3 つのフェーズでモデル トレーニングを自動化します。

- サービス プリンシパルと GitHub シークレットを使用して、GitHub から Azure Machine Learning ワークスペースへの安全なアクセスを構成する。
- 手動でトリガーされた GitHub Actions ワークフローから Azure Machine Learning コマンド ジョブを実行する。
- 機能ベースの開発とブランチ保護を使用して、モデル トレーニングが pull request ワークフローの一部として実行されるようにする。

この過程で、ワークスペースのネットワークとソース管理の設定が、モデル トレーニング用の安全な自動化を設計する方法にどのように影響するかを確認します。

## 開始する前に

必要なもの:

- 管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free?azure-portal=true)。
- リポジトリを作成し、GitHub Actions を構成するアクセス許可を持つ [GitHub](https://github.com/) アカウント。

## Azure Machine Learning ワークスペースをプロビジョニングする

まず、GitHub ワークフローから使用する Azure Machine Learning ワークスペースとコンピューティング リソースを作成します。

1. ブラウザーで Azure portal (`https://portal.azure.com/`) を開き、Microsoft アカウントを使用してサインインします。
1. ページの上部にある **[>_]** (**Cloud Shell**) ボタンを選択して Cloud Shell を開き、ダイアログが表示されたら **[Bash]** を選択します。
1. 正しいサブスクリプションと、**[ストレージ アカウントは不要]** が選択されていることを確認します。 次に、**[適用]** を選びます。
1. Cloud Shell ターミナルで、元のラボ リポジトリをクローンし、セットアップ スクリプトを実行します。

    ```azurecli
    rm -r mslearn-mlops -f
    git clone https://github.com/MicrosoftLearning/mslearn-mlops.git mslearn-mlops
    cd mslearn-mlops/infra
    ./setup.sh
    ```

    > 拡張機能がインストールできなかったというメッセージは無視してください。

1. スクリプトが完了するまで待ちます。 リソース グループ、Azure Machine Learning ワークスペース、コンピューティング リソースが作成されます。
1. Azure portal で、**[リソース グループ]** に移動し、作成された `rg-ai300-...` リソース グループを開きます。
1. Azure Machine Learning ワークスペース (`mlw-ai300-...` など) を選択し、**[スタジオの起動]** を選択して Azure Machine Learning スタジオを開きます。

ワークスペースの準備ができたので、独自の GitHub リポジトリを作成し、安全なアクセスを構成できるようになりました。

## テンプレートから GitHub リポジトリを作成する

次に、GitHub Actions を使用できるように、元のラボ リポジトリから独自の GitHub リポジトリを作成します。

1. ブラウザーで `https://github.com/MicrosoftLearning/mslearn-mlops` に移動します。
1. 右上隅の **[このテンプレートを使用する]** を選択し **[新しいリポジトリの作成]** を選択します。
1. **[所有者]** フィールドで、ご自分の GitHub アカウントを選択します。 **[リポジトリ名前]** フィールドに、`mslearn-mlops` などの名前を入力します。
1. **[Create repository from template](テンプレートからリポジトリを作成する)** を選択します。
1. テンプレートから作成した新しいリポジトリで、**[Actions]** タブに移動し、ダイアログが表示されたら GitHub Actions を有効にします。
1. 新しいリポジトリのクローン URL (`https://github.com/<your-alias>/mslearn-mlops.git` など) をメモします。 この URL は、リポジトリをローカルで、または開発環境から操作する場合に使用します。

テンプレート ベースのリポジトリの準備ができたので、GitHub を Azure に安全に接続できるようになりました。

## GitHub と Azure Machine Learning の統合を構成する

Azure Machine Learning に対する GitHub Actions の認証を行うには、サービス プリンシパルを使用します。 このサービス プリンシパルの資格情報は、暗号化されたシークレットとして GitHub リポジトリに格納されます。

1. Azure portal で、ページの上部にある **[>_]** (**Cloud Shell**) ボタンを選択し、Cloud Shell を開きます。
1. シェルの種類を選択するように求めるダイアログが表示されたら、**[Bash]** を選択します。
1. Azure Machine Learning ワークスペースに対して正しいサブスクリプションが選択されていることを確認します。
1. Cloud Shell で、Azure Machine Learning ワークスペースを含むリソース グループへの **[共同作成者]** アクセス権を持つサービス プリンシパルを作成します。 スクリプトを実行する前に、`<service-principal-name>`、`<subscription-id>`、`<your-resource-group-name>` を実際の値に置き換えます。 `sp-mslearn-mlops-github` などのわかりやすい名前を使用します。

    ```azurecli
    az ad sp create-for-rbac --name "<service-principal-name>" --role contributor \
            --scopes /subscriptions/<subscription-id>/resourceGroups/<your-resource-group-name> \
            --sdk-auth
    ```

1. コマンドの完全な JSON 出力を安全な場所にコピーします。 この値は、次の手順と後の課題で使用します。
1. テンプレートから作成した GitHub リポジトリから、**[設定]** > **[シークレットと変数]** > **[Actions]** の順に移動します。
1. **[New repository secret]\(新しいリポジトリ シークレット\)** を選択します。
1. シークレットの **[名前]** に「`AZURE_CREDENTIALS`」を入力します。
1. `az ad sp create-for-rbac` コマンドからの JSON 出力を **[値]** フィールドに貼り付け、**[シークレットを追加する]** を選択します。
1. **[変数]** を選択し、**[新しいリポジトリ変数]** を選択します。
1. **[名前]** に「`AZURE_RESOURCE_GROUP`」を入力し、**[値]** にリソース グループ名 (`rg-ai300-l<suffix>` など) を入力します。 **変数の追加**を選択します。
1. もう一度 **[新しいリポジトリ変数]** を選択します。 **[名前]** に「`AZURE_WORKSPACE_NAME`」を入力し、**[値]** に Azure Machine Learning ワークスペース名 (`mlw-ai300-l<suffix>` など) を入力します。 **変数の追加**を選択します。

GitHub リポジトリに暗号化されたシークレットが追加されました。これは、GitHub でホストされるランナーが Azure にサインインし、Azure Machine Learning ワークスペースにジョブを送信するために使用できます。

> [!NOTE]
> サービス プリンシパルのスコープは、Azure Machine Learning リソース グループに設定されています。 運用シナリオでは、アクセス許可をさらに制限し、プライベート エンドポイントやセルフホステッド ランナーなどのネットワーク制御と組み合わせて、ジョブを送信する方法と場所を制限します。

## ワークスペースのネットワーク アクセス オプションを確認する

GitHub からのトレーニングを自動化する前に、Azure Machine Learning ワークスペースでネットワーク アクセスを制御する方法を確認します。

1. Azure Machine Learning ワークスペースで **Azure portal** の、左側のナビゲーションで **[設定]** を選択し、**[ネットワーク]** を選択します。
1. **[パブリック アクセス]** と **[プライベート エンドポイント]** 設定を確認します。 次の方法に注目します: - すべてのネットワークからのパブリック ネットワーク アクセスを許可する。
        - 特定の IP 範囲へのアクセスを制限する。
        - パブリック ネットワーク アクセスを無効にして、プライベート エンドポイントを使用する。
1. このラボでは、GitHub でホストされるランナーがワークスペースに到達できるように、既定のパブリック アクセス設定をそのまま使用します。 運用環境では、通常、次の内容を実行します:- プライベート エンドポイントと仮想ネットワークを使用する。
        - これらのプライベート ネットワークに到達できるセルフホステッド ランナーで GitHub Actions を実行する。
        - ロールベースのアクセス制御 (RBAC) とネットワーク ルールを組み合わせて、ジョブを送信できるユーザーを厳密に制御する。

ネットワーク オプションを理解したので、GitHub からトレーニング ジョブを自動化する準備ができました。

## 手動でトリガーされたワークフローを使用してモデル トレーニングを自動化する

このセクションでは、GitHub ワークフローを Azure Machine Learning に接続し、コマンド ジョブを実行してモデルをトレーニングします。 ワークフローでは、先ほど作成した `AZURE_CREDENTIALS` シークレットを使用します。

1. ファイルを編集して GitHub にプッシュバックできる開発環境に、テンプレートから作成した `mslearn-mlops` リポジトリをクローンします。
1. クローンされたリポジトリで、`src/job.yml` を開き、`training_data` 入力のプレースホルダー値を置き換えて、コマンド ジョブでは、セットアップ スクリプトで作成された 1 つのファイル データ資産が使用されるようにします。

    ```yml
    inputs:
      training_data:
        type: uri_file
        path: azureml:diabetes-data@latest
    ```
1. クローンされたリポジトリで、`.github/workflows/manual-trigger-job.yml` ワークフロー ファイルを見つけます。
1. `manual-trigger-job.yml` を開き、既存の手順を確認します。 ワークフローでは、次の動作が行われる必要があります: - リポジトリ コードをチェックアウトする。
        - Azure Machine Learning CLI 拡張機能をインストールする。
        - `AZURE_CREDENTIALS` シークレットを使用して、`azure/login@v2` 経由で Azure にサインインする。
1. ワークフローの最後に、`src/job.yml` で定義された Azure Machine Learning ジョブを送信する新しいステップを追加します。 このコマンドには、作成した GitHub Actions 変数から提供される明示的な `--resource-group` と `--workspace-name` フラグが必要です。

    ```yml
    - name: Run Azure Machine Learning training job
        run: az ml job create -f src/job.yml --stream --resource-group ${{vars.AZURE_RESOURCE_GROUP}} --workspace-name ${{vars.AZURE_WORKSPACE_NAME}}
    ```

1. 変更を保存し、ローカル リポジトリにコミットし、フォークの **main** ブランチに変更をプッシュします。
1. GitHub で、リポジトリの **[Actions]** タブに移動します。
1. `manual-trigger-job.yml` で定義されたワークフローを選択し、**実行ワークフロー**を使用して手動で開始します。
1. ワークフローの実行が完了するのを待ちます。 **Azure Machine Learning トレーニング ジョブの実行**ステップが正常に完了したことを確認します。
1. Azure Machine Learning スタジオで **[ジョブ]** を選択し、`src/job.yml` に基づく新しいジョブが正常に実行されたことを確認します。 ジョブの入力、メトリック、ログを確認します。

これで、オンデマンドで実行できる GitHub Actions ワークフローを使用してトレーニング ジョブを自動化できました。

## 機能ベースの開発を使用してワークフローをトリガーする

ワークフローを手動で実行することは初期テストに役立ちますが、チーム環境では通常、誰かが変更を提案したときにトレーニング ワークフローを自動的に実行する必要があります。 次は、pull request に対して実行されるように既存のワークフローを更新し、機能ブランチとブランチ保護ルールを使用してワークフローの実行タイミングを制御します。

1. GitHub リポジトリで、`.github/workflows/manual-trigger-job.yml` ワークフロー ファイルを開きます。
1. `on` セクションを更新して、ワークフローが手動の場合と、pull request が **main** ブランチをターゲットにしたときの両方の場合で実行できるようにします。 次に例を示します。

    ```yml
    on:
        workflow_dispatch:
        pull_request:
            branches:
                - main
    ```

1. 更新されたワークフロー ファイルをコミットし、リポジトリの **main** ブランチにプッシュします。
1. GitHub で、**[設定]** > **[ブランチ]** に移動し、**[ブランチ保護ルールの追加]** を選択します。
1. 直接プッシュを防ぐ **main** ブランチのルールを構成します。 少なくとも、次を選択します: - **ブランチ名パターン**: `main`。
        - **一致するブランチを保護する**。
1. ブランチ保護ルールを保存します。
1. リポジトリのローカル クローンで、機能変更用の新しいブランチを作成します。 次に例を示します。

    ```bash
    git checkout -b feature/update-parameters
    ```

1. トレーニング構成に、小さく安全な変更を加えます。 たとえば、`src/train-model-parameters.py` または `src/job.yml` のハイパーパラメーターの値を調整します。
1. 機能ブランチへの変更をコミットし、GitHub にブランチをプッシュします。

    ```bash
    git add .
    git commit -m "Adjust training parameters"
    git push --set-upstream origin feature/update-parameters
    ```

1. GitHub で、機能ブランチから **main** への pull request を作成します。
1. [pull request] ページで、`manual-trigger-job.yml` で定義されているワークフローが、追加した `pull_request` トリガーにより自動的に実行されることを確認します。
1. ワークフローが正常に完了したら、結果を確認し、pull request を完了して変更を **main** にマージします。

同じトレーニング ワークフロー定義で機能ブランチ、ブランチ保護ルール、pull request によってトリガーされるワークフローを使用すると、モデル トレーニングの自動化がソース管理の制御された変更に確実に関連付けられます。

## Azure リソースをクリーンアップする

Azure Machine Learning と GitHub Actions を調べ終わったら、不要な Azure コストを避けるために作成したリソースを削除する必要があります。

1. [Azure Machine Learning スタジオ] タブを閉じて、Azure portal に戻ります。
1. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
1. Azure Machine Learning ワークスペースを含む **rg-ai300-...** リソース グループを選択します。
1. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
1. リソース グループ名を入力して、削除することを確認し、**[削除]** を選択します。
1. GitHub では、ワークフローやサンプル コードが不要になった場合は、`mslearn-mlops` テンプレートから作成したリポジトリを削除することもできます。

