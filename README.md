# CognitiveServicesWorkshop


subscriptionID=$(az account list --query "[?contains(name,'Microsoft')].[id]" -o tsv)
echo "Test subscription ID is = " $subscriptionID
az account set --subscription $subscriptionID
az account show

location=northeurope
postfix=$RANDOM
groupName=AIServerlessUA$postfix

az group create --name $groupName --location $location

location=northeurope
accountSku=Standard_LRS
accountName=${groupName,,}
echo "accountName  = " $accountName

az storage account create --name $accountName --location $location --kind StorageV2 \
--resource-group $groupName --sku $accountSku --access-tier Hot  --https-only true

runtime=dotnet
applicationName=${groupName,,}
accountName=${groupName,,}
echo "applicationName  = " $applicationName

az functionapp create --resource-group $groupName \
--name $applicationName --storage-account $accountName --runtime $runtime \
--consumption-plan-location $location --functions-version 3

signalName=${groupName,,}
az signalr create --name $signalName --resource-group $groupName --sku Standard_S1 --unit-count 1 --service-mode Serverless

signalConnString=$(az signalr key list --name $(az signalr list \
--resource-group $groupName --query [0].name -o tsv) \
--resource-group $groupName --query primaryConnectionString -o tsv)

printf "\n\nReplace <signalConnString> with:\n$signalConnString\n\n"

#add cors if you know your domains already.
#az signalr cors add --name $signalName --resource-group $groupName --allowed-origins "http://example1.com" "https://example2.com"
signalKey=$(az signalr key list --name $signalName --resource-group $groupName --query primaryKey -o tsv)

printf "\n Replace <signalKey> with:\n$signalKey\n\n"

speechName=${groupName,,}
az cognitiveservices account list-skus --location $location --kind SpeechServices
az cognitiveservices account create --name $speechName --resource-group $groupName --location $location \
--kind SpeechServices --sku S0 --yes

speechKey1=$(az cognitiveservices account keys list --name $speechName --resource-group $groupName --query "key1" -o tsv)

printf "\n\n Change <speechKey1> with:\n$speechKey1\n\n"


