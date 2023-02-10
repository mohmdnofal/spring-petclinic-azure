export AZ_RESOURCE_GROUP=java
export AZ_DATABASE_NAME=spring-mysql
export AZ_LOCATION=northeurope
AZ_MYSQL_ADMIN_USERNAME=moadmin
AZ_MYSQL_ADMIN_PASSWORD=Password123$
AZ_MYSQL_NON_ADMIN_USERNAME=petclinic
AZ_MYSQL_NON_ADMIN_PASSWORD=petclinic
export AZ_USER_IDENTITY_NAME=mysql-identity
export CURRENT_USERNAME=$(az ad signed-in-user show --query userPrincipalName -o tsv)
export CURRENT_USER_OBJECTID=$(az ad signed-in-user show --query id -o tsv)



az group create \
    --name $AZ_RESOURCE_GROUP \
    --location $AZ_LOCATION \
    --output tsv



az mysql flexible-server create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --location $AZ_LOCATION \
    --admin-user $AZ_MYSQL_ADMIN_USERNAME \
    --admin-password $AZ_MYSQL_ADMIN_PASSWORD \
    --yes \
    --output tsv


az mysql flexible-server db create \
    --resource-group $AZ_RESOURCE_GROUP \
    --database-name petclinic \
    --server-name $AZ_DATABASE_NAME \
    --output tsv


MY_IP_ADDRESS=`curl ifconfig.co/`


az mysql flexible-server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --start-ip-address $MY_IP_ADDRESS \
    --end-ip-address $MY_IP_ADDRESS \
    --rule-name allowiprange \
    --output tsv

#allow applications from azure to connect 
az mysql flexible-server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --start-ip-address 0.0.0.0


cat << EOF > create_user.sql
CREATE USER '$AZ_MYSQL_NON_ADMIN_USERNAME'@'%' IDENTIFIED BY '$AZ_MYSQL_NON_ADMIN_PASSWORD';
GRANT ALL PRIVILEGES ON demo.* TO '$AZ_MYSQL_NON_ADMIN_USERNAME'@'%';
FLUSH PRIVILEGES;
EOF

mysql -h $AZ_DATABASE_NAME.mysql.database.azure.com --user $AZ_MYSQL_ADMIN_USERNAME --enable-cleartext-plugin --password=$AZ_MYSQL_ADMIN_PASSWORD < create_user.sql


# General 
RESOURCE_GROUP=$AZ_RESOURCE_GROUP
LOCATION=$AZ_LOCATION

# App service 
APP_SERVICE_PLAN=javaplan
WEB_APP_NAME=spring-mysql


mvn package

az appservice plan create --name $APP_SERVICE_PLAN --resource-group $RESOURCE_GROUP --sku P1V2 --is-linux
az webapp list-runtimes
az webapp create --name $WEB_APP_NAME --resource-group $RESOURCE_GROUP --plan $APP_SERVICE_PLAN --runtime JAVA:17-java17
az webapp config appsettings set -g $RESOURCE_GROUP -n $WEB_APP_NAME --settings "MYSQL_URL=$MYSQL_URL" "MYSQL_USER=$MYSQL_USER" "MYSQL_PASS=$MYSQL_PASS"
az webapp deploy --resource-group $RESOURCE_GROUP --name $WEB_APP_NAME --src-path ./target/spring-petclinic-3.0.0-SNAPSHOT.jar --type jar


#containerize the application 

docker build --tag spring-petclinic-mysql-profile . 
docker tag spring-petclinic-mysql-profile mohamman/spring-petclinic-mysql-profile

# Container Apps 

az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights






CONTAINERAPPS_ENVIRONMENT=java-env
CONTAINER_APP_NAME=spring-mysql
docker push mohamman/spring-petclinic-mysql-profile


az containerapp env create \
  --name $CONTAINERAPPS_ENVIRONMENT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION


az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image mohamman/spring-petclinic-mysql \
  --target-port 8080 \
  --ingress 'external' \
  --query properties.configuration.ingress.fqdn



#build with secrets 

CONTAINER_APP_NAME_ENV=spring-mysql-secrets
az containerapp create \
  --name $CONTAINER_APP_NAME_ENV \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image mohamman/spring-petclinic-mysql-var \
  --target-port 8080 \
  --secret "mysql-url=$MYSQL_URL" "mysql-user=$MYSQL_USER" "mysql-pass=$MYSQL_PASS" \
  --env-vars "MYSQL_URL=secretref:mysql-url" "MYSQL_USER=secretref:mysql-user" "MYSQL_PASS=secretref:mysql-pass" \
  --ingress 'external' \
  --query properties.configuration.ingress.fqdn


CONTAINER_APP_NAME_ENV=spring-mysql-porfile
az containerapp create \
  --name $CONTAINER_APP_NAME_ENV \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image mohamman/spring-petclinic-mysql-profile \
  --target-port 8080 \
  --secret "mysql-url=$MYSQL_URL" "mysql-user=$MYSQL_USER" "mysql-pass=$MYSQL_PASS" \
  --env-vars "MYSQL_URL=secretref:mysql-url" "MYSQL_USER=secretref:mysql-user" "MYSQL_PASS=secretref:mysql-pass" \
  --ingress 'external' \
  --query properties.configuration.ingress.fqdn




# AKS cluster 
AKS_CLUSTER_NAME=java-aks
az aks create -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --enable-managed-identity --node-count 1 --enable-addons monitoring --enable-msi-auth-for-monitoring  --generate-ssh-keys

az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

kubectl get nodes



kubectl create secret generic mysql-secret --from-literal=mysql_url=$MYSQL_URL --from-literal=mysql_user=$MYSQL_USER --from-literal=mysql_pass=$MYSQL_PASS

kubectl apply -f k8s/spring-mysql-deployment.yaml

kubectl get pods



# Azure Spring Apps 
