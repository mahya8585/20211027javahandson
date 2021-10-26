# azure-spring-workshop
## 構成
- Compute: Azure App Serice
- Database: Azure Database for PostgreSQL server
- Code repository: GitHub
- CI/CD: GitHub Actions

https://docs.microsoft.com/en-us/learn/modules/deploy-java-spring-boot-app-service-mysql/5-exercise-deploy

## 前提条件
- Azureサブスクリプション上でのContributor権限
- Java JDK (1.8 以降)、Maven (3.0 以降)、Azure CLI (2.12 以降)、Bash、Curl のローカルでのインストール

### 参考: コンテンツ作成者の環境
- Windows 10
- Bashのターミナル:
  - WSL2 (確認方法: wsl -l -v)
　- GitBash (CurlのPOSTがWLS2だと実行できなかったので)
- java version "1.8.0_301" (確認方法: java -version)
- (確認方法: mvn -v)
- (確認方法: az -v)

# こちらのサンプルコードの展開方法
# 

## GitHub上でレポジトリをFork
https://github.com/rukasakurai/20211027javahandson
(https://github.com/rukasakurai/azure-spring-workshop = Java 8バージョン)

## リソースグループの作成
以下のAzure Cloud Shell内のBashで実行
```
POSTFIX=<firstnamelastname>
AZ_RESOURCE_GROUP=rg-$POSTFIX
AZ_LOCATION=japaneast
az group create \
    --name $AZ_RESOURCE_GROUP \
    --location $AZ_LOCATION \
    | jq
```
Azure Portalからリソースグループが作成されていることを確認

### Azure Database for PostgreSQL のインスタンスを作成する
参考ドキュメンテーション: Creating PostgresSQL
https://docs.microsoft.com/en-us/azure/postgresql/quickstart-create-server-database-azure-cli

```
AZ_POSTGRE_NAME=postgresql-$POSTFIX
AZ_POSTGRESQL_USERNAME=spring
AZ_POSTGRESQL_PASSWORD=P@ssword123
AZ_PORTAL_IP_ADDRESS=$(curl http://whatismyip.akamai.com/)
AZ_LOCAL_IP_ADDRESS=<パソコンのIPアドレス>
```

Azure Database for PostgreSQL のインスタンスを作成する
```
az postgres server create --resource-group $AZ_RESOURCE_GROUP --name $AZ_POSTGRE_NAME  --location $AZ_LOCATION --admin-user $AZ_POSTGRESQL_USERNAME --admin-password $AZ_POSTGRESQL_PASSWORD --sku-name GP_Gen5_2
```

ローカル環境/Azure Portalからの穴あけ
```
az postgres server firewall-rule create --resource-group $AZ_RESOURCE_GROUP --server $AZ_POSTGRE_NAME --name AllowMyIP --start-ip-address $AZ_LOCAL_IP_ADDRESS --end-ip-address $AZ_LOCAL_IP_ADDRESS
az postgres server firewall-rule create --resource-group $AZ_RESOURCE_GROUP --server $AZ_POSTGRE_NAME --name AllowMyIP --start-ip-address $AZ_PORTAL_IP_ADDRESS --end-ip-address $AZ_PORTAL_IP_ADDRESS
```

Azure Portalから確認。以下のコマンドで確認することも可能
```
az postgres server show --resource-group $AZ_RESOURCE_GROUP --name $AZ_POSTGRE_NAME -o yaml
```

Alllow access to Azure services: Yes

Azure Cloud Shellから以下を実行し、DB接続
```
psql --host=$AZ_POSTGRE_NAME.postgres.database.azure.com --port=5432 --username=$AZ_POSTGRESQL_USERNAME@$AZ_POSTGRE_NAME --dbname=postgres
```

以下を実行し、テーブル作成
```
CREATE TABLE todo (
  id int, description varchar(255), details varchar(255), done bit, PRIMARY KEY (id)
);
exit
```

## App Serviceの作成
PLAN_NAME=plan-$POSTFIX
APP_NAME=app-$POSTFIX
GIT_REPO=https://github.com/rukasakurai/azure-spring-workshop

az appservice list-locations --sku {B1, B2, B3, D1, F1, FREE, I1, I1v2, I2, I2v2, I3, I3v2, P1V2, P1V3, P2V2, P2V3, P3V2, P3V3, PC2, PC3, PC4, S1, S2, S3, SHARED}
az appservice list-locations --sku B1
az appservice plan create --name $PLAN_NAME --resource-group $AZ_RESOURCE_GROUP --sku FREE
az appservice plan create --name $PLAN_NAME --resource-group $AZ_RESOURCE_GROUP --sku FREE

az webapp list-runtimes
az webapp create --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --plan $PLAN_NAME --runtime "java|11|Java SE|8"
az webapp create --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --plan $PLAN_NAME --runtime "java|1.8|Java SE|8"

az webapp config appsettings list --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP -o table
az webapp config appsettings set --name $APP_NAME --resource-group $AZ_RESOURCE_GROUP --settings spring.datasource.password=P@ssword123

# GitHub Actions設定
1. 対象のApp Service→Deployment Center→GitHub→ログイン→Organization選択→Repository選択→Branch=main



## GitHub上のコード変更
GitHub上のコードを変更すると、GitHub Actionsが走り、新しいコードが自動的に展開される

## アプ
### GET
http://$APP_NAME.azurewebsites.net/
### POST
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"description":"configuration","details":"congratulations, you have set up your Spring Boot application correctly!","done": "true"}' \
    https://demo-1633072672760.azurewebsites.net/

curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"description":"configuration","details":"congratulations, you have set up your Spring Boot application correctly!","done": "true"}' \
    http://app-rukasakuraitemp.azurewebsites.net/

