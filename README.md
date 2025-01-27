# Java on Azure Hands-on
## 構成
- Compute: Azure App Serice
- Database: Azure Database for PostgreSQL server
- Code repository: GitHub
- CI/CD: GitHub Actions
- Serverless computing: Azure Functions

## 前提条件
- Azureサブスクリプション上でのContributor以上権限
- GitHubのアカウント

### 参考: コンテンツ作成者の環境
- 作業用端末: Windows 10のPC
- ブラウザ: Microsofot Edge
- Azure Subscription: Owner権限
- コマンドラインツール: Azure Cloud Shell (Bash)
- GitHub: 個人のFreeプラン
- テキストエディタ: VS Code
- Gitツール: VS Code内のgit

## GitHub上でレポジトリをFork
GitHubにログインして、以下のレポジトリをFork
https://github.com/rukasakurai/20211027javahandson
(https://github.com/rukasakurai/azure-spring-workshop = Java 8バージョン)

## Azure PortalへのログインとCloud Shellのセットアップ
Azure Portalにログイン
https://portal.azure.com

言語設定を行う

Azure Cloud Shellを起動し、Bashを選択（Azure Cloud Shell はファイルを保持するために、ストレージ アカウントを作成する必要あり）

## リソースグループの作成
### GUI
https://portal.azure.com/#create/Microsoft.ResourceGroup

### コマンドライン
以下をAzure Cloud Shell内のBashで実行

変数を設定
```
POSTFIX=<firstnamelastname>
```
```
AZ_RESOURCE_GROUP=rg-$POSTFIX
AZ_LOCATION=japaneast
```
```
az group create \
    --name $AZ_RESOURCE_GROUP \
    --location $AZ_LOCATION \
    | jq
```
Azure Portalからリソースグループが作成されていることを確認

### 参考: 命名規則について
https://docs.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations

## Azure Database for PostgreSQL のインスタンスを作成する
### GUI
https://portal.azure.com/#create/hub
`Azure Database for PostgreSQL`を選択
### コマンドライン
変数を設定
```
AZ_POSTGRE_NAME=psql-$POSTFIX
AZ_PORTAL_IP_ADDRESS=$(curl http://whatismyip.akamai.com/)
AZ_POSTGRESQL_USERNAME=spring
```
```
AZ_POSTGRESQL_PASSWORD=<任意のパスワード>
```

Azure Database for Postgre作成
```
az postgres server create --resource-group $AZ_RESOURCE_GROUP --name $AZ_POSTGRE_NAME  --location $AZ_LOCATION --admin-user $AZ_POSTGRESQL_USERNAME --admin-password $AZ_POSTGRESQL_PASSWORD --sku-name GP_Gen5_2
```

Azure Cloud Shellからの通信の穴あけ
```
az postgres server firewall-rule create --resource-group $AZ_RESOURCE_GROUP --server $AZ_POSTGRE_NAME --name AllowMyIP --start-ip-address $AZ_PORTAL_IP_ADDRESS --end-ip-address $AZ_PORTAL_IP_ADDRESS
```

Azure Portalから確認。以下のコマンドで確認することも可能
```
az postgres server show --resource-group $AZ_RESOURCE_GROUP --name $AZ_POSTGRE_NAME -o yaml
```

Azure Cloud Shellから以下を実行し、DB接続

```
psql --host=$AZ_POSTGRE_NAME.postgres.database.azure.com --port=5432 --username=$AZ_POSTGRESQL_USERNAME@$AZ_POSTGRE_NAME --dbname=postgres
```

以下を実行し、テーブル作成
```
CREATE TABLE todo (
  id int, description varchar(255), details varchar(255), done bit, PRIMARY KEY (id)
);
```
```
\dt
``` 
```
exit
```

### 参考
- Creating PostgresSQL: https://docs.microsoft.com/azure/postgresql/quickstart-create-server-database-azure-cli
- 価格表: https://azure.microsoft.com/pricing/details/postgresql/server/

## Web Appの作成
### GUI
https://portal.azure.com/#create/hub
`Web アプリ`を選択
### コマンドライン
変数を設定
```
PLAN_NAME=plan-$POSTFIX
APP_NAME=app-$POSTFIX
```
App Service Planの作成
```
az appservice plan create --name $PLAN_NAME --resource-group $AZ_RESOURCE_GROUP --sku FREE
```
Web Appの作成
```
# az webapp list-runtimes
az webapp create --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --plan $PLAN_NAME --runtime "java|11|Java SE|8"
#az webapp create --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --plan $PLAN_NAME --runtime "java|1.8|Java SE|8"
```
DB接続の設定
```
az webapp config appsettings set --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --settings spring.datasource.password=$AZ_POSTGRESQL_PASSWORD
az webapp config appsettings list --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP -o table
```

### 作成されたWeb Appの確認
Portal上からWeb AppのURLを取得し、ブラウザからアクセス

### 参考 
- 価格表: https://azure.microsoft.com/pricing/details/app-service/

## GitHub Actions設定
- Azure Portalから対象のApp Service→Deployment Center→GitHub→ログイン→Organization選択→Repository選択→Branch=main
- GitHubのActionsタブでCI/CDが実行されていることを確認
- Web AppのURLをブラウザで確認

## APIにデータをPOSTし、データがDBに格納されていることを確認
### APIにデータをPOST
以下のをAzure Cloud Shellから実行
```
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"description":"configuration","details":"congratulations, you have set up your Spring Boot application correctly!","done": "true"}' \
    http://$APP_NAME.azurewebsites.net/
```
### データがDBに格納されていることを確認
Portal上からWeb AppのURLを取得し、ブラウザからアクセス

## GitHub上のコード変更
- GitHub上でコードを変更（src/main/java/com/example/demo/Todo.javaのgetDescription内の文字列等）
- GitHub上のコードを変更すると、GitHub Actionsが走り、新しいコードが自動的に展開されるので、GitHub Actionsが実行されていることを確認
- Web AppのURLをブラウザで確認

## Azure Functions
Azure Functionsのハンズオンについては、以下のモジュールを活用
https://docs.microsoft.com/learn/modules/create-serverless-logic-with-azure-functions

```
FUNC_NAME=func-$POSTFIX
echo $FUNC_NAME
```
## Resourceのロックや削除について
- リソースのロックについて
- 不要なリソースを削除