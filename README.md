# CognitiveServicesWorkshop

Lets start with a workshop overview. It is based on two workflows from microsoft. 
Cognitive service sample with voice to text and translations and real-time application clients with Azure SignalR service and Azure Functions. Translation will be performed via sv-en configuration of cognitive services. This kind of application solve support communication issues in the near realtime.

Requirements for this tutorial is Visual Studio 2019 community or VS Code. Basic knowledge of C#

# Step 1. Infrastructure

1. The first step is to create a demo Azure subscription 
https://azure.microsoft.com/en-us/free/
Take script below to deploy all needed infrastructure 

```
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
```

3. Copy SignalR connection string and Cognitive service key to notepad.

4. Signal IR and Azure Functions. Steps below following official MS tutorial with important details
	https://docs.microsoft.com/en-us/azure/azure-signalr/signalr-quickstart-azure-functions-csharp

5. Proceed with a demo chat application from this tutorial clone tutorial repository.
	https://github.com/Azure-Samples/signalr-service-quickstart-serverless-chat.git
	6. Azure functions application located here
	/src/chat/csharp

7. The local web application to edit is located here.
	/docs/demo/chat-v2

8. Copy SignalR primary connection string from notepad 

9. In Visual Studio, in Solution Explorer, rename local.settings.sample.json to local.settings.json.

10. In local.settings.json, paste the SignalR connection string into the value of the AzureSignalRConnectionString setting. Save the file.

11. Open Functions.cs. There are two HTTP triggered functions in this function app:
		a. GetSignalRInfo - Uses the SignalRConnectionInfo input binding to generate and return valid connection information.
		b. SendMessage - Receives a chat message in the request body and uses the SignalR output binding to broadcast the message to all connected client applications.

12. Visual Studio: In the Debug menu, select Start debugging to run the application.

13. You can deploy function app to Azure via publishing profile, if you want to.

14. Lets connect local and test remote applications.

15. There is a sample single page web application hosted in GitHub for your convenience. Open your browser to https://azure-samples.github.io/signalr-service-quickstart-serverless-chat/demo/chat-v2/.

16. When prompted for the function app base URL, enter http://localhost:7071.

17. Enter a username when prompted.

18. The web application calls the GetSignalRInfo function in the function app to retrieve the connection information to connect to Azure SignalR Service. When the connection is complete, the chat message input box appears.

19. Type a message and press enter. The application sends the message to the SendMessage function in the Azure Function app, which then uses the SignalR output binding to broadcast the message to all connected clients. If everything is working correctly, the message should appear in the application.

20. Open another instance of the web application in a different browser window. You will see that any messages sent will appear in all instances of the application.
    
Be aware of the cors issues and http vs https usage. My suggestion is to deploy function to the Web app and setup CORS exceptions in configuration.


# Cognitive services translations

The next step is to use Cognitive Services translation samples and integrate them with existing solutions. You can use advanced scenario with speech to text and then translation. But lets start with simple case.
The interesting point is that you need to combine two tutorials inside one serverless application.

1. We will be following this tutorial
	https://docs.microsoft.com/en-us/azure/cognitive-services/translator/quickstart-translate?pivots=programming-language-csharp

2. Essentialy you need to take source code from tutorial above and add it to the new Azure Function.

3. Create Azure Function in Visual studio and add calls to the translations.
	
  
# Cognitive services Speech to text and translation

	Lets take a look at general service drscription of Speech to text. 
	https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/get-started

1. Lets take a look at the demo description of the workshop.
	https://github.com/Azure-Samples/cognitive-services-speech-sdk/tree/master/quickstart/csharp/dotnetcore/translate-speech-to-text

2. We already have deployed infrastructure and secret code output from bash console.

3. Lets proceed with steps

