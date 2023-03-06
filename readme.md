# Spring PetClinic App on Azure 

This is a slightly modified version of the famous [PetClinic App](https://speakerdeck.com/michaelisvy/spring-petclinic-sample-application), the app configuration has been modified to run with MySQL as its default profile instead of H2. 

In this demo we will spin up an Azure MySQL database, build a run the application locally, then deploy it to Azure App Service as a Jar file, after this we will build a container image and deploy the App to Azure Container Apps and Azure Kubernetes Service to demonstrate the choice and differences across these 3 platforms. 

**Note** the following is just a demonstration to reflect on the Azure Platform choices, this is not how things should be deployed in production i.e. you will restrict access to VNETs, you will use managed identities and such. 

# Deploy Azure MySQL Flexible Server 

We Start by deploying a MySQL DB in Azure so we test against it.

```

#define the variables 
export RESOURCE_GROUP=spring-mysql
export DATABASE_NAME=spring-mysql-$RANDOM
export LOCATION=northeurope
export MYSQL_ADMIN_USERNAME=moadmin
export MYSQL_ADMIN_PASSWORD=mysql$RANDOM$RANDOM
#needed for the spring application
export MYSQL_NON_ADMIN_USERNAME=petclinic
export MYSQL_NON_ADMIN_PASSWORD=petclinic$RANDOM$RANDOM

#create resource group 
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION \
    --output tsv

#create the databse server 
az mysql flexible-server create \
    --resource-group $RESOURCE_GROUP \
    --name $DATABASE_NAME \
    --location $LOCATION \
    --admin-user $MYSQL_ADMIN_USERNAME \
    --admin-password $MYSQL_ADMIN_PASSWORD \
    --yes \
    --output tsv


#create the petclinic database inside the database server 
az mysql flexible-server db create \
    --resource-group $RESOURCE_GROUP \
    --database-name petclinic \
    --server-name $DATABASE_NAME \
    --output tsv

#get your IP address so we can test locally 
export MY_IP_ADDRESS=`curl ifconfig.co/`

#add firewall rule for your IP address 
az mysql flexible-server firewall-rule create \
    --resource-group $RESOURCE_GROUP \
    --name $DATABASE_NAME \
    --start-ip-address $MY_IP_ADDRESS \
    --end-ip-address $MY_IP_ADDRESS \
    --rule-name allowmyip \
    --output tsv

#add the below rule to allow services from Azure to connect to the database 
az mysql flexible-server firewall-rule create \
    --resource-group $RESOURCE_GROUP \
    --name $DATABASE_NAME \
    --start-ip-address 0.0.0.0


#the below script will create the user name and password for the petclinic databse 
cat << EOF > create_user.sql
CREATE USER '$MYSQL_NON_ADMIN_USERNAME'@'%' IDENTIFIED BY '$MYSQL_NON_ADMIN_PASSWORD';
GRANT ALL PRIVILEGES ON petclinic.* TO '$MYSQL_NON_ADMIN_USERNAME'@'%';
FLUSH PRIVILEGES;
EOF

#run the script 
mysql -h $DATABASE_NAME.mysql.database.azure.com --user $MYSQL_ADMIN_USERNAME --enable-cleartext-plugin --password=$MYSQL_ADMIN_PASSWORD < create_user.sql

#just in case you want to validate 
mysql -h $DATABASE_NAME.mysql.database.azure.com --user $MYSQL_NON_ADMIN_USERNAME --enable-cleartext-plugin --password=$MYSQL_NON_ADMIN_PASSWORD

#remove the file 
rm create_user.sql

# **dont skip this** the below variables will be needed from the application to function 
export MYSQL_URL=jdbc:mysql://$DATABASE_NAME.mysql.database.azure.com:3306/petclinic?serverTimezone=UTC
export MYSQL_USER=$MYSQL_NON_ADMIN_USERNAME
export MYSQL_PASS=$MYSQL_NON_ADMIN_PASSWORD

```

## Run the application locally 

```
git clone https://github.com/mohmdnofal/spring-petclinic.git
cd spring-petclinic
./mvnw package
java -jar target/*.jar
```

You can then access petclinic at http://localhost:8080/



## Deploy the application to App Service 

```
#define the variables 
APP_SERVICE_PLAN=spring-mysql
WEB_APP_NAME=spring-mysql-$RANDOM


#build the application
mvn package

#create the app service plan (I used P1V2 to demenstrate the deployment slots and premium features, B1 is more than suffecient to run the demo)

az appservice plan create \
    --name $APP_SERVICE_PLAN \
    --resource-group $RESOURCE_GROUP \
    --sku P1V2 \
    --is-linux

#check the possible run times 
az webapp list-runtimes

#create the web app 
az webapp create \
    --name $WEB_APP_NAME \
    --resource-group $RESOURCE_GROUP \
    --plan $APP_SERVICE_PLAN \
    --runtime JAVA:17-java17

#add the required environment variables as application settings 
az webapp config appsettings set \
    -g $RESOURCE_GROUP \
    -n $WEB_APP_NAME \
    --settings "MYSQL_URL=$MYSQL_URL" "MYSQL_USER=$MYSQL_USER" "MYSQL_PASS=$MYSQL_PASS"


#deploy the application 
az webapp deploy \
    --resource-group $RESOURCE_GROUP \
    --name $WEB_APP_NAME \
    --src-path ./target/spring-petclinic-3.0.0-SNAPSHOT.jar \
    --type jar

#get the application FQDN 
az webapp show -g $RESOURCE_GROUP -n $WEB_APP_NAME --query defaultHostName -o tsv
```

## Create a docker image 
**Azure Container APPs and AKS would require a container image, so this step is mandatory to run the ACA and AKS demos**


```
#define variables
IMAGE_NAME=petclinic-mysql
DOCKER_HUB_ID=mohamman

#build the image 
docker build --tag $IMAGE_NAME . 
docker tag $IMAGE_NAME $DOCKER_HUB_ID/$IMAGE_NAME

#push the image 
docker push $DOCKER_HUB_ID/$IMAGE_NAME
```

# Deploy the Application to Azure Container Apps 

```
#add container apps extension if not exist 
az extension add --name containerapp --upgrade

#register the resource providers if not already registered 
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights


#define variables 
CONTAINERAPPS_ENVIRONMENT=java-aca-env
CONTAINER_APP_NAME=spring-mysql-$RANDOM


#create ACA environment
az containerapp env create \
  --name $CONTAINERAPPS_ENVIRONMENT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

#deploy the app 
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --image $DOCKER_HUB_ID/$IMAGE_NAME \
  --target-port 8080 \
  --secret "mysql-url=$MYSQL_URL" "mysql-user=$MYSQL_USER" "mysql-pass=$MYSQL_PASS" \
  --env-vars "MYSQL_URL=secretref:mysql-url" "MYSQL_USER=secretref:mysql-user" "MYSQL_PASS=secretref:mysql-pass" \
  --ingress 'external' \
  --query properties.configuration.ingress.fqdn
```

# Deploy the application to Azure Kubernetes Service 
Following we will be creating an AKS cluster and deploy the application to it 

```
#define variables 
AKS_CLUSTER_NAME=java-aks-$RANDOM

#create a single node cluster 
az aks create \
    -g $RESOURCE_GROUP \
    -n $AKS_CLUSTER_NAME \
    --enable-managed-identity \
    --node-count 1 \
    --enable-addons monitoring \
    --enable-msi-auth-for-monitoring \
    --generate-ssh-keys

#get the credinitals so you can access the cluster 
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME

#validate 
kubectl get nodes


#create a Kubernetes secret so we can access the databse 
kubectl create secret generic mysql-secret \
    --from-literal=mysql_url=$MYSQL_URL \
    --from-literal=mysql_user=$MYSQL_USER \
    --from-literal=mysql_pass=$MYSQL_PASS

#a deployment file was created already, we will use it to deploy the app 
kubectl apply -f k8s/spring-mysql-deployment.yaml

#validate 
kubectl get pods

#get the public ip and access the service 
kubectl get service spring-mysql-svc
```

Thank you if you made it this far :) 



