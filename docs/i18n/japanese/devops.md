# DevOps ハンドブック

このガイドは、インフラストラクチャスタックとプラットフォームをどのように維持するかを理解するのに役立ちます。 このガイドで、すべての操作について詳しく説明しているわけではありませんが、システムを理解する上での参考になります。

ご意見やご質問があれば、どうぞご連絡ください。喜んでご説明いたします。

# フライトマニュアル - コードデプロイ

このリポジトリは、継続的に構築され、テストされ、**インフラストラクチャの個別のセット (サーバー、データベース、CDNなど)** にデプロイされます。

これには3つのステップが含まれます。

1. 新規変更 (修正および機能変更の両方を含む) は、プルリクエストによりプライマリ開発ブランチ (`main`) にマージされます。
2. これらの変更は、一連の自動テストで実行されます。
3. テストに合格すると、インフラストラクチャ上でのデプロイメントに対して変更をリリースします(または必要に応じて更新します)。

#### コードベースのビルド - Git ブランチのデプロイメントへのマッピング

通常、[`main`](https://github.com/freeCodeCamp/freeCodeCamp/tree/main) (デフォルトの開発ブランチ) は、[`prod-staging`](https://github.com/freeCodeCamp/freeCodeCamp/tree/prod-staging) ブランチに 1 日 1 回マージされ、分離されたインフラストラクチャにリリースされます。

これは開発者とボランティアのコントリビューター用の中間リリースです。 「ステージング」または「ベータ」リリースとも呼ばれます。

それは `freeCodeCamp.org` のライブプロダクション環境と同じで、データベース、サーバー、Web プロキシなどの別々のセットを使用しています。 この分離により、freeCodeCamp.org の main プラットフォームの正規ユーザーに影響を与えることなく、「本番」のようなシナリオで継続的な開発と機能をテストすることができます。

開発者チーム [`@freeCodeCamp/dev-team`](https://github.com/orgs/freeCodeCamp/teams/dev-team/members) が、ステージングプラットフォームでの変更に満足したら、これらの変更は数日ごとに [`prod-current`](https://github.com/freeCodeCamp/freeCodeCamp/tree/prod-current) ブランチに移されます。

これが freeCodeCamp.org で本番プラットフォームに変更を加えた最終リリースです。

#### 変更のテスト - 統合テストとユーザー承認テスト

私たちは、コードの品質を確認するために、様々なレベルの統合と受け入れテストを採用しています。 すべてのテストは、[GitHub Actions CI](https://github.com/freeCodeCamp/freeCodeCamp/actions) や [Azure Pipelines](https://dev.azure.com/freeCodeCamp-org/freeCodeCamp) のようなソフトウェアにより実行されます。

私たちは、チャレンジソリューション、Server API、クライアントユーザーインターフェースをテストするための単体テストを行っています。 これらは、異なるコンポーネント間の統合をテストするのに役立ちます。

> [!NOTE] また、メールの更新や API やサードパーティサービスへの呼び出しなど、現実世界のシナリオを再現するのに役立つエンドユーザーテストを作成中です。

これらのテストを組み合わせることで、問題が繰り返されるのを防ぎ、別のバグや機能の作業中にバグが発生しないようにします。

#### 変更のデプロイ - 変更をサーバーにプッシュする

開発サーバーと本番サーバーに変更をプッシュする継続的デリバリーソフトウェアを設定しています。

保護されたリリースブランチに変更がプッシュされると、そのブランチに対してビルドパイプラインが自動的にトリガーされます。 ビルドパイプラインは、アーティファクトを構築し、後で使用するためにコールドストレージに保管する責任があります。

実行が正常に完了すると、ビルドパイプラインは対応するリリースパイプラインをトリガーします。 リリースパイプラインは、ビルドアーティファクトを収集し、それらをサーバーに移動し、稼働させる責任があります。

ビルドとリリースのステータスは [こちら](#build-test-and-deployment-status) からご確認いただけます。

## ビルドをトリガー・テスト・デプロイする

現時点では、開発チームのメンバーのみが本番ブランチにプッシュできます。 `production-*` ブランチへの変更は、[`upstream`](https://github.com/freeCodeCamp/freeCodeCamp) への早送りマージによってのみ可能です。

> [!NOTE] 今後、アクセス管理と透明性を向上させるために、プルリクエストを介してこのフローを改善します。

### ステージングアプリケーションに変更をプッシュする

1. リモートを正しく構成します。

   ```sh
   git remote -v
   ```

   **結果:**

   ```
   origin   git@github.com:raisedadead/freeCodeCamp.git (fetch)
   origin   git@github.com:raisedadead/freeCodeCamp.git (push)
   upstream git@github.com:freeCodeCamp/freeCodeCamp.git (fetch)
   upstream git@github.com:freeCodeCamp/freeCodeCamp.git (push)
   ```

2. `main` ブランチが初期状態であり、アップストリームと同期していることを確認してください。

   ```sh
   git checkout main
   git fetch --all --prune
   git reset --hard upstream/main
   ```

3. GitHub CI がアップストリームの `main` ブランチを渡していることを確認してください。

   [継続的インテグレーション](https://github.com/freeCodeCamp/freeCodeCamp/actions) テストは、`main` ブランチに関して、緑色であり PASSING でなければなりません。 `main` ブランチコードを表示する際、コミットハッシュの横にある緑色のチェックマークをクリックします。

    <details> <summary> GitHub Actionsでステータスを確認する (スクリーンショット) </summary>
      <br>
      ![Check build status on GitHub Actions](https://raw.githubusercontent.com/freeCodeCamp/freeCodeCamp/main/docs/images/devops/github-actions.png)
    </details>

   これに失敗した場合は、停止してエラーの確認をします。

4. リポジトリをローカルにビルドできることを確認します。

   ```
   npm run clean-and-develop
   ```

5. 早送りマージにより、変更を `main` から `prod-staging` に移行します。

   ```
   git checkout prod-staging
   git merge main
   git push upstream
   ```

   > [!NOTE] 強制的にプッシュすることはできません。履歴を書き直した場合、これらのコマンドはエラーになります。
   > 
   > エラーになったとしたら、誤った操作をしたかもしれませんので、やり直します。

上記手順では、`prod-staging` ブランチのビルドパイプラインで自動的に実行がトリガーされます。 ビルドが完了すると、アーティファクトは `.zip` ファイルとしてコールドストレージで保存され、後で取り出され使用されます。

接続されたビルドパイプラインから新たなアーティファクトが利用可能になると、リリースパイプラインが自動的にトリガーされます。 ステージングプラットフォームでは、このプロセスに手動での承認は含まれません。また、アーティファクトは クライアント CDN および API サーバーにプッシュされます。

### 本番アプリケーションに変更をプッシュする

プロセスはほとんどステージングプラットフォームと同じですが、いくつかの追加のチェックが行われます。 これは、何百人ものユーザーが常に使用している freeCodeCamp.org 上で何も壊さないようにするためです。

| すべてがステージングプラットフォームで動作していることを確認しない限り、これらのコマンドを実行しないでください。 先に進む前に、ステージング上のテストを回避またはスキップしないでください。 |
|:---------------------------------------------------------------------------------------------- |
|                                                                                                |

1. `prod-staging` ブランチが初期状態であり、アップストリームと同期していることを確認してください。

   ```sh
   git checkout prod-staging
   git fetch --all --prune
   git reset --hard upstream/prod-staging
   ```

2. 早送りマージにより、変更を `prod-staging` から `prod-current` に移行します。

   ```
   git checkout prod-current
   git merge prod-staging
   git push upstream
   ```

   > [!NOTE] 強制的にプッシュすることはできません。履歴を書き直した場合、これらのコマンドはエラーになります。
   > 
   > エラーになったとしたら、誤った操作をしたかもしれませんので、やり直します。

上記手順では、`prod-current` ブランチのビルドパイプラインで自動的に実行がトリガーされます。 ビルドアーティファクトの準備が完了すると、リリースパイプラインで実行がトリガーされます。

**スタッフアクションの追加手順**

リリースの実行がトリガーされると、開発者スタッフチームのメンバーは自動的に手動介入メールを受け取ります。 彼らはリリース実行を _承認_、または _拒否_ することができます。

変更がうまく動作し、ステージングプラットフォームでテストされている場合は、承認することができます。 承認は、自動的に拒否される前に、リリースがトリガーされてから4時間以内に行われる必要があります。 拒否された実行が拒否された場合、スタッフは手動でリリース実行を再トリガーするか、リリースの次のサイクルを待つことになります。

スタッフ用:

| ビルドの実行が完了したら、直接リンクについて E メールを確認するか、[リリースダッシュボードにアクセス](https://dev.azure.com/freeCodeCamp-org/freeCodeCamp/_release) してください。 |
|:--------------------------------------------------------------------------------------------------------------------------- |
|                                                                                                                             |

スタッフがリリースを承認すると、パイプラインは freeCodeCamp.org の本番用 CDN および API サーバーにその変更を反映させます。

## ビルド、テスト、デプロイスのテータス

ここでは、コードベースの現在のテスト、ビルド、およびデプロイの状況を示します。

| ブランチ                                                                             | 単体テスト                                                                                                                                                                                                                            | 統合テスト                                                                                                                                                                                                                    | ビルド & デプロイ                                                                                                                        |
|:-------------------------------------------------------------------------------- |:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |:--------------------------------------------------------------------------------------------------------------------------------- |
| [`main`](https://github.com/freeCodeCamp/freeCodeCamp/tree/main)                 | [![Node.js CI](https://github.com/freeCodeCamp/freeCodeCamp/workflows/Node.js%20CI/badge.svg?branch=main)](https://github.com/freeCodeCamp/freeCodeCamp/actions?query=workflow%3A%22Node.js+CI%22)                               | [![Cypress E2E Tests](https://img.shields.io/endpoint?url=https://dashboard.cypress.io/badge/simple/ke77ns/main&style=flat&logo=cypress)](https://dashboard.cypress.io/projects/ke77ns/analytics/runs-over-time)         | -                                                                                                                                 |
| [`prod-staging`](https://github.com/freeCodeCamp/freeCodeCamp/tree/prod-staging) | [![Node.js CI](https://github.com/freeCodeCamp/freeCodeCamp/workflows/Node.js%20CI/badge.svg?branch=prod-staging)](https://github.com/freeCodeCamp/freeCodeCamp/actions?query=workflow%3A%22Node.js+CI%22+branch%3Aprod-staging) | [![Cypress E2E Tests](https://img.shields.io/endpoint?url=https://dashboard.cypress.io/badge/simple/ke77ns/prod-staging&style=flat&logo=cypress)](https://dashboard.cypress.io/projects/ke77ns/analytics/runs-over-time) | [Azure Pipelines](https://dev.azure.com/freeCodeCamp-org/freeCodeCamp/_dashboards/dashboard/d59f36b9-434a-482d-8dbd-d006b71713d4) |
| [`prod-current`](https://github.com/freeCodeCamp/freeCodeCamp/tree/prod-staging) | [![Node.js CI](https://github.com/freeCodeCamp/freeCodeCamp/workflows/Node.js%20CI/badge.svg?branch=prod-current)](https://github.com/freeCodeCamp/freeCodeCamp/actions?query=workflow%3A%22Node.js+CI%22+branch%3Aprod-current) | [![Cypress E2E Tests](https://img.shields.io/endpoint?url=https://dashboard.cypress.io/badge/simple/ke77ns/prod-current&style=flat&logo=cypress)](https://dashboard.cypress.io/projects/ke77ns/analytics/runs-over-time) | [Azure Pipelines](https://dev.azure.com/freeCodeCamp-org/freeCodeCamp/_dashboards/dashboard/d59f36b9-434a-482d-8dbd-d006b71713d4) |
| `prod-next` (試験的、予定)                                                             | -                                                                                                                                                                                                                                | -                                                                                                                                                                                                                        | -                                                                                                                                 |

## 早期アクセスとベータテスト

皆さんがこれらのリリースを **"パブリックベータテスト"** モードでテストし、プラットフォームの今後の機能に早期アクセスできるようにします。 これらの機能 / 変更は、**次の、ベータ、ステージング** などと呼ばれます。

フィードバックや Issue 報告を通じた貢献は、**復元力**、**一貫性** および **安定性** のある `freeCodeCamp.org` 本番プラットフォームを構築するのに役立ちます。

見つけたバグの報告や、freeCodeCamp.orgをより良くする支援に感謝します。 素晴らしい皆さんです！

### プラットフォームの今後のバージョンを特定する

現在、パブリックベータテストバージョンは次の場所で利用できます。

| アプリケーション | 言語    | URL                                      |
|:-------- |:----- |:---------------------------------------- |
| 学習       | 英語    | <https://www.freecodecamp.dev>           |
|          | スペイン語 | <https://www.freecodecamp.dev/espanol>   |
|          | 中国語   | <https://chinese.freecodecamp.dev>       |
| ニュース     | 英語    | <https://www.freecodecamp.dev/news>      |
| フォーラム    | 英語    | <https://forum.freecodecamp.dev>         |
|          | 中国語   | <https://chinese.freecodecamp.dev/forum> |
| API      | -     | `https://api.freecodecamp.dev`           |

> [!NOTE] ドメイン名は **`freeCodeCamp.org`** とは異なります。 これは、検索エンジンのインデックス作成を防止し、プラットフォームの通常ユーザーの混乱を避けるための、意図的なものです。
> 
> 上記リストは、提供するアプリケーションを包括したものではありません。 また、リソースを節約するために、すべての言語バリエーションがステージングにデプロイされるわけではありません。

### プラットフォームの現在のバージョンを特定する

**プラットフォームの現在のバージョンは [`freeCodeCamp.org`](https://www.freecodecamp.org) で常に利用できます。**

開発者チームは、リリース変更時に、`prod-stageing` ブランチから `prod-current` への変更をマージします。 トップコミットは、サイト上で表示されるもののはずです。

状況セクションにあるデプロイログおよびビルドにアクセスして、デプロイされた正確なバージョンを確認できます。 あるいは、[contributors チャットルーム](https://chat.freecodecamp.org/channel/contributors) で確認することもできます。

### 既知の制限

プラットフォームのベータ版を使用する場合、いくつかの既知の制限とトレードオフがあります。

- #### これらのベータプラットフォーム上のデータ / 個人的な進捗は、保存されたり本番環境に移行されることはありません。

  **ベータ版のユーザーは本番とは異なるアカウントを持つことになります。** ベータ版は本番と物理的に分離されたデータベースを使用します。 これにより、偶発的なデータ損失や変更を防ぐことができます。 開発チームは、必要に応じてこのベータ版のデータベースを削除する可能性があります。

- #### ベータ版プラットフォームの稼働時間と信頼性については保証はありません。

  デプロイは頻繁に行われ、時には非常に速いペースで 1 日に複数回行われることになります。 その結果、ベータ版において、不測のダウンタイムが発生したり機能が壊れることがあります。

- #### 修正を確認する手段として、このサイトに一般ユーザーを送らないでください。

  ベータサイトは、ローカルの開発とテストを強化するためのものでしたし、今もそうです。 それはこれから起こることを約束するものではありませんが、取り組まれていることを垣間見るものです。

- #### サインインページが本番環境とは異なる場合があります。

  Auth0 上で freeCodeCamp 開発用のテストテナントを使用しているため、カスタムドメインを設定することはできません。 そのため、すべてのリダイレクトコールバックとログインページが `https://freecodecamp-dev.auth0.com/` のようなデフォルトドメインに表示されます。 This does not affect the functionality and is as close to production as we can get.

## Issue の報告とフィードバック

ディスカッションやバグ報告をする場合、新しい Issue を開いてください。

ご質問があれば、`dev[at]freecodecamp.org` にメールをご送信ください。 セキュリティ脆弱性は、公開トラッカーやフォーラムではなく、`security[at]freecodecamp.org` に報告する必要があります。

# フライトマニュアル - サーバーメンテナンス

> [!WARNING]
> 
> 1. ガイドは、**freeCodeCamp スタッフのみ** に適用されます。
> 2. インストラクションは包括的なものではありませんので、ご注意ください。

スタッフの一員として、Azure、Digital Ocean などのクラウドサービスプロバイダーへのアクセスが許可されている可能性があります。

仮想マシン (VM) で作業するために使用できる便利なコマンドをいくつか紹介します。例えばメンテナンスの更新や一般的なハウスキーピングの実行です。

## VM のリストを取得する

> [!NOTE] 既に VM へ SSH アクセスできるかもしれませんが、クラウドポータルへのアクセスが許可されていない限り VMを一覧表示することはできません。

### Azure

Azure CLI のインストール `az`: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

> **(一回のみ) [`homebrew`](https://brew.sh) で macOS にインストールします。**

```
brew install azure-cli
```

> **(一回のみ) ログインします。**

```
az login
```

> **VM 名と P アドレスのリストを取得します。**

```
az vm list-ip-addresses --output table
```

### Digital Ocean

Digital Ocean CLI のインストール `doctl`: https://github.com/digitalocean/doctl#installing-doctl

> **(一回のみ) [`homebrew`](https://brew.sh) で macOS にインストールします。**

```
brew install doctl
```

> **(一回のみ) ログインします。**

認証とコンテキストの切り替え: https://github.com/digitalocean/doctl#authenticating-with-digitalocean

```
doctl auth init
```

> **VM 名と IP アドレスのリストを取得します。**

```
doctl compute droplet list --format "ID,Name,PublicIPv4"
```

## 新しいリソースをスピンする

私たちは IaC 設定の作成に取り組んでいます。そして、その作業中は Azure ポータルまたは Azure CLI を使用して、新しい仮想マシンやその他のリソースをスピンさせることができます。

> [!TIP] スピニングリソースの選択に関係なく、docker のインストールや SSH キーの追加など基本的なプロビジョニングを行うのに役立つ [便利な cloud-init 設定ファイル](https://github.com/freeCodeCamp/infra/tree/main/cloud-init) がいくつかあります。

## VM を最新に保つ

アップデートとアップグレードを行うことで、VM を最新の状態に保つ必要があります。 これにより、仮想マシンが最新のセキュリティ修正でパッチされるようになります。

> [!WARNING] これらのコマンドを実行する前に下記を実行します。
> 
> - VM が完全にプロビジョニングされており、インストール後の手順が実行されていないことを確認してください。
> - アプリケーションを既に提供している VM 上で、パッケージを更新する場合は、アプリが停止 / 保存されていることを確認してください。 パッケージ更新により、ネットワーク帯域幅や、メモリ、CPU の使用率が急増し、 実行中のアプリケーションが停止します。

パッケージ情報を更新する

```console
sudo apt update
```

インストール済みパッケージをアップグレードする

```console
sudo apt upgrade -y
```

未使用のパッケージをクリーンアップする

```console
sudo apt autoremove -y
```

## Web サーバーでの作業 (プロキシ)

Web サーバーのために、負荷分散 (Azure Load Balancer) インスタンスを実行しています。 これらのサーバーは NGINX を実行しています。NGINX は、独自インフラストラクチャで実行される様々なアプリケーションから freeCodeCamp.org へと、トラフィックを中継するリバースプロキシとして使用されます。

NGINX 設定は [このリポジトリ](https://github.com/freeCodeCamp/nginx-config) で確認できます。

### 最初のインストール

コードを使用して VM をプロビジョニング

1. NGINX をインストールし、リポジトリから設定します。

   ```console
   sudo su

   cd /var/www/html
   git clone https://github.com/freeCodeCamp/error-pages

   cd /etc/
   rm -rf nginx
   git clone https://github.com/freeCodeCamp/nginx-config nginx

   cd /etc/nginx
   ```

2. Cloudflare のオリジン証明書とアップストリームアプリケーション設定をインストールします。

   安全なストレージから Cloudflare のオリジン証明書を取得し、 必要な場所にインストールします。

   **または**

   既存の証明書上に移動します。

   ```console
   # Local
   scp -r username@source-server-public-ip:/etc/nginx/ssl ./
   scp -pr ./ssl username@target-server-public-ip:/tmp/

   # Remote
   rm -rf ./ssl
   mv /tmp/ssl ./
   ```

   アップストリーム設定を更新します。

   ```console
   vi configs/upstreams.conf
   ```

   ソース / オリジンアプリケーションの IP アドレスを追加 / 更新します。

3. ネットワーキングとファイアウォールを設定します。

   必要に応じて、イングレスオリジンアドレスに Azure ファイアウォールと `ufw` を設定します。

4. VM をロードバランサーバックエンドプールに追加します。

   必要に応じて、ロードバランサーにルールを設定し追加します。 バランサーバックエンドプールをロードするために、VM を追加する必要があるかもしれません。

### ログとモニタリング

1. 以下のコマンドを使用して NGINX サービスのステータスを確認します。

   ```console
   sudo systemctl status nginx
   ```

2. サーバーのログとモニタリングは以下で行います。

   現行の基本的なモニタリングダッシュボードは、NGINX Amplify ([https://amplify.nginx.com]('https://amplify.nginx.com')) です。 監視向上のため、より細かいメトリックに取り組んでいます。

### インスタンスの更新 (メンテナンス)

NGINX インスタンスへの設定変更は、GitHub 上でメンテナンスされています。これらは、以下のように各インスタンスにデプロイされる必要があります。

1. SSH でインスタンスに接続し、sudo と入力します。

```console
sudo su
```

2. 最新の設定コードを取得します。

```console
cd /etc/nginx
git fetch --all --prune
git reset --hard origin/main
```

3. 設定 [with Signals](https://docs.nginx.com/nginx/admin-guide/basic-functionality/runtime-control/#controlling-nginx) をテストして再度読み込みます。

```console
nginx -t
nginx -s reload
```

## API インスタンスでの作業

1. ノードバイナリのビルドツール (`node-gyp`) をインストールします。

```console
sudo apt install build-essential
```

### 最初のインストール

コードを使用して VM をプロビジョニング

1. ノード LTS をインストールします。

2. Update `npm` and install PM2 and setup `logrotate` and startup on boot

   ```console
   npm i -g npm@8
   npm i -g pm2
   pm2 install pm2-logrotate
   pm2 startup
   ```

3. Clone freeCodeCamp, setup env and keys.

   ```console
   git clone https://github.com/freeCodeCamp/freeCodeCamp.git
   cd freeCodeCamp
   git checkout prod-current # or any other branch to be deployed
   ```

4. Create the `.env` from the secure credentials storage.

5. Create the `google-credentials.json` from the secure credentials storage.

6. Install dependencies

   ```console
   npm ci
   ```

7. Build the server

   ```console
   npm run create:config && npm run build:curriculum && npm run build:server
   ```

8. Start Instances

   ```console
   cd api-server
   pm2 start ./lib/production-start.js -i max --max-memory-restart 600M --name org
   ```

### Logging and Monitoring

```console
pm2 logs
```

```console
pm2 monit
```

### Updating Instances (Maintenance)

Code changes need to be deployed to the API instances from time to time. It can be a rolling update or a manual update. The later is essential when changing dependencies or adding environment variables.

> [!ATTENTION] The automated pipelines are not handling dependencies updates at the minute. We need to do a manual update before any deployment pipeline runs.

#### 1. Manual Updates - Used for updating dependencies, env variables.

1. Stop all instances

```console
pm2 stop all
```

2. Install dependencies

```console
npm ci
```

3. Build the server

```console
npm run create:config && npm run build:curriculum && npm run build:server
```

4. Start Instances

```console
pm2 start all --update-env && pm2 logs
```

#### 2. Rolling updates - Used for logical changes to code.

```console
pm2 reload all --update-env && pm2 logs
```

> [!NOTE] We are handling rolling updates to code, logic, via pipelines. You should not need to run these commands. These are here for documentation.

## Work on Client Instances

1. Install build tools for node binaries (`node-gyp`) etc.

```console
sudo apt install build-essential
```

### First Install

Provisioning VMs with the Code

1. Install Node LTS.

2. Update `npm` and install PM2 and setup `logrotate` and startup on boot

   ```console
   npm i -g npm@8
   npm i -g pm2
   npm install -g serve
   pm2 install pm2-logrotate
   pm2 startup
   ```

3. Clone client config, setup env and keys.

   ```console
   git clone https://github.com/freeCodeCamp/client-config.git client
   cd client
   ```

   Start placeholder instances for the web client, these will be updated with artifacts from the Azure pipeline.

   > Todo: This setup needs to move to S3 or Azure Blob storage 
   > 
   > ```console
   >    echo "serve -c ../../serve.json www -p 50505" >> client-start-primary.sh
   >    chmod +x client-start-primary.sh
   >    pm2 delete client-primary
   >    pm2 start  ./client-start-primary.sh --name client-primary
   >    echo "serve -c ../../serve.json www -p 52525" >> client-start-secondary.sh
   >    chmod +x client-start-secondary.sh
   >    pm2 delete client-secondary
   >    pm2 start  ./client-start-secondary.sh --name client-secondary
   > ```

### Logging and Monitoring

```console
pm2 logs
```

```console
pm2 monit
```

### Updating Instances (Maintenance)

Code changes need to be deployed to the API instances from time to time. It can be a rolling update or a manual update. The later is essential when changing dependencies or adding environment variables.

> [!ATTENTION] The automated pipelines are not handling dependencies updates at the minute. We need to do a manual update before any deployment pipeline runs.

#### 1. Manual Updates - Used for updating dependencies, env variables.

1. Stop all instances

   ```console
   pm2 stop all
   ```

2. Install or update dependencies

3. Start Instances

   ```console
   pm2 start all --update-env && pm2 logs
   ```

#### 2. Rolling updates - Used for logical changes to code.

```console
pm2 reload all --update-env && pm2 logs
```

> [!NOTE] We are handling rolling updates to code, logic, via pipelines. You should not need to run these commands. These are here for documentation.

## Work on Chat Servers

Our chat servers are available with a HA configuration [recommended in Rocket.Chat docs](https://docs.rocket.chat/installation/docker-containers/high-availability-install). The `docker-compose` file for this is [available here](https://github.com/freeCodeCamp/chat-config).

We provision redundant NGINX instances which are themselves load balanced (Azure Load Balancer) in front of the Rocket.Chat cluster. The NGINX configuration file are [available here](https://github.com/freeCodeCamp/chat-nginx-config).

### First Install

Provisioning VMs with the Code

**NGINX Cluster:**

1. Install NGINX and configure from repository.

   ```console
   sudo su

   cd /var/www/html
   git clone https://github.com/freeCodeCamp/error-pages

   cd /etc/
   rm -rf nginx
   git clone https://github.com/freeCodeCamp/chat-nginx-config nginx

   cd /etc/nginx
   ```

2. Install Cloudflare origin certificates and upstream application config.

   Get the Cloudflare origin certificates from the secure storage and install at required locations.

   **OR**

   Move over existing certificates:

   ```console
   # Local
   scp -r username@source-server-public-ip:/etc/nginx/ssl ./
   scp -pr ./ssl username@target-server-public-ip:/tmp/

   # Remote
   rm -rf ./ssl
   mv /tmp/ssl ./
   ```

   Update Upstream Configurations:

   ```console
   vi configs/upstreams.conf
   ```

   Add/update the source/origin application IP addresses.

3. Setup networking and firewalls.

   Configure Azure firewalls and `ufw` as needed for ingress origin addresses.

4. Add the VM to the load balancer backend pool.

   Configure and add rules to load balancer if needed. You may also need to add the VMs to load balancer backend pool if needed.

**Docker Cluster:**

1. Install Docker and configure from the repository

   ```console
   git clone https://github.com/freeCodeCamp/chat-config.git chat
   cd chat
   ```

2. Configure the required environment variables and instance IP addresses.

3. Run rocket-chat server

   ```console
   docker-compose config
   docker-compose up -d
   ```

### Logging and Monitoring

1. Check status for NGINX service using the below command:

   ```console
   sudo systemctl status nginx
   ```

2. Check status for running docker instances with:

   ```console
   docker ps
   ```

### Updating Instances (Maintenance)

**NGINX Cluster:**

Config changes to our NGINX instances are maintained on GitHub, these should be deployed on each instance like so:

1. SSH into the instance and enter sudo

   ```console
   sudo su
   ```

2. Get the latest config code.

   ```console
   cd /etc/nginx
   git fetch --all --prune
   git reset --hard origin/main
   ```

3. Test and reload the config [with Signals](https://docs.nginx.com/nginx/admin-guide/basic-functionality/runtime-control/#controlling-nginx).

   ```console
   nginx -t
   nginx -s reload
   ```

**Docker Cluster:**

1. SSH into the instance and navigate to the chat config path

   ```console
   cd ~/chat
   ```

2. Get the latest config code.

   ```console
   git fetch --all --prune
   git reset --hard origin/main
   ```

3. Pull down the latest docker image for Rocket.Chat

   ```console
   docker-compose pull
   ```

4. Update the running instances

   ```console
   docker-compose up -d
   ```

5. Validate the instances are up

   ```console
   docker ps
   ```

6. Cleanup extraneous resources

   ```console
   docker system prune --volumes
   ```

   Output:

   ```console
   WARNING! This will remove:
     - all stopped containers
     - all networks not used by at least one container
     - all volumes not used by at least one container
     - all dangling images
     - all dangling build cache

   Are you sure you want to continue? [y/N] y
   ```

   Select yes (y) to remove everything that is not in use. This will remove all stopped containers, all networks and volumes not used by at least one container, and all dangling images and build caches.

## Work on Contributor Tools

### Deploy updates

ssh into the VM (hosted on Digital Ocean).

```console
cd tools
git pull origin master
npm ci
npm run build
pm2 restart contribute-app
```

## Updating Node.js versions on VMs

List currently installed node & npm versions

```console
nvm -v
node -v
npm -v

nvm ls
```

Install the latest Node.js LTS, and reinstall any global packages

```console
nvm install --lts --reinstall-packages-from=default
```

Verify installed packages

```console
npm ls -g --depth=0
```

Alias the `default` Node.js version to the current LTS (pinned to latest major version)

```console
nvm alias default 16
```

(Optional) Uninstall old versions

```console
nvm uninstall <version>
```

> [!ATTENTION] For client applications, the shell script can't be resurrected between Node.js versions with `pm2 resurrect`. Deploy processes from scratch instead. This should become nicer when we move to a docker based setup.
> 
> If using PM2 for processes you would also need to bring up the applications and save the process list for automatic recovery on restarts.

Get the uninstall instructions/commands with the `unstartup` command and use the output to remove the systemctl services

```console
pm2 unstartup
```

Get the install instructions/commands with the `startup` command and use the output to add the systemctl services

```console
pm2 startup
```

Quick commands for PM2 to list, resurrect saved processes, etc.

```console
pm2 ls
```

```console
pm2 resurrect
```

```console
pm2 save
```

```console
pm2 logs
```

## Installing and Updating Azure Pipeline Agents

See: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops and follow the instructions to stop, remove and reinstall agents. Broadly you can follow the steps listed here.

You would need a PAT, that you can grab from here: https://dev.azure.com/freeCodeCamp-org/_usersSettings/tokens

### Installing agents on Deployment targets

Navigate to [Azure Devops](https://dev.azure.com/freeCodeCamp-org) and register the agent from scratch in the requisite [deployment groups](https://dev.azure.com/freeCodeCamp-org/freeCodeCamp/_machinegroup).

> [!NOTE] You should run the scripts in the home directory, and make sure no other `azagent` directory exists.

### Updating agents

Currently updating agents requires them to be removed and reconfigured. This is required for them to correctly pick up `PATH` values and other system environment variables. We need to do this for instance updating Node.js on our deployment target VMs.

1. Navigate and check status of the service

   ```console
   cd ~/azagent
   sudo ./svc.sh status
   ```

2. Stop the service

   ```console
   sudo ./svc.sh stop
   ```

3. Uninstall the service

   ```console
   sudo ./svc.sh uninstall
   ```

4. Remove the agent from the pipeline pool

   ```console
   ./config.sh remove
   ```

5. Remove the config files

   ```console
   cd ~
   rm -rf ~/azagent
   ```

Once You have completed the steps above, you can repeat the same steps as installing the agent.

# Flight Manual - Email Blast

We use [a CLI tool](https://github.com/freecodecamp/sendgrid-email-blast) to send out the weekly newsletter. To spin this up and begin the process:

1. Sign in to DigitalOcean, and spin up new droplets under the `Sendgrid` project. Use the Ubuntu Sendgrid snapshot with the most recent date. This comes pre-loaded with the CLI tool and the script to fetch emails from the database. With the current volume, three droplets are sufficient to send the emails in a timely manner.

2. Set up the script to fetch the email list.

   ```console
   cd /home/freecodecamp/scripts/emails
   cp sample.env .env
   ```

   You will need to replace the placeholder values in the `.env` file with your credentials.

3. Run the script.

   ```console
   node get-emails.js emails.csv
   ```

   This will save the email list in an `emails.csv` file.

4. Break the emails down into multiple files, depending on the number of droplets you need. This is easiest to do by using `scp` to pull the email list locally and using your preferred text editor to split them into multiple files. Each file will need the `email,unsubscribeId` header.

5. Switch to the CLI directory with `cd /home/sendgrid-email-blast` and configure the tool [per the documentation](https://github.com/freeCodeCamp/sendgrid-email-blast/blob/main/README.md).

6. Run the tool to send the emails, following the [usage documentation](https://github.com/freeCodeCamp/sendgrid-email-blast/blob/main/docs/cli-steps.md).

7. When the email blast is complete, verify that no emails have failed before destroying the droplets.

# Flight Manual - Adding news instances for new languages

### Theme Changes

We use a custom [theme](https://github.com/freeCodeCamp/news-theme) for our news publication. Adding the following changes to the theme enables the addition of new languages.

1. Include an `else if` statement for the new [ISO language code](https://www.loc.gov/standards/iso639-2/php/code_list.php) in [`setup-locale.js`](https://github.com/freeCodeCamp/news-theme/blob/main/assets/config/setup-locale.js)
2. Create an initial config folder by duplicating the [`assets/config/en`](https://github.com/freeCodeCamp/news-theme/tree/main/assets/config/en) folder and changing its name to the new language code. (`en` —> `es` for Spanish)
3. Inside the new language folder, change the variable names in `main.js` and `footer.js` to the relevant language short code (`enMain` —> `esMain` for Spanish)
4. Duplicate the [`locales/en.json`](https://github.com/freeCodeCamp/news-theme/blob/main/locales/en.json) and rename it to the new language code.
5. In [`partials/i18n.hbs`](https://github.com/freeCodeCamp/news-theme/blob/main/partials/i18n.hbs), add scripts for the newly created config files.
6. Add the related language `day.js` script from [cdnjs](https://cdnjs.com/libraries/dayjs/1.10.4) to the [freeCodeCamp CDN](https://github.com/freeCodeCamp/cdn/tree/main/build/news-assets/dayjs/1.10.4/locale)

### Ghost Dashboard Changes

Update the publication assets by going to the Ghost dashboard > settings > general and uploading the publications's [icon](https://github.com/freeCodeCamp/design-style-guide/blob/master/assets/fcc-puck-500-favicon.png), [logo](https://github.com/freeCodeCamp/design-style-guide/blob/master/downloads/fcc_primary_large.png), and [cover](https://github.com/freeCodeCamp/design-style-guide/blob/master/assets/fcc_ghost_publication_cover.png).
