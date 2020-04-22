# Cognitive Services and real-time Serverless with Signal R

Let's start with a workshop overview. 
The main idea is to build a real-time serverless application with Translation and "Speech to text" Cognitive services from Microsoft.
Your goal is to use this tutorial to build your idea. The tutorial provides links to Cognitive services documentation, scenarios, etc.

Requirements for this tutorial are Visual Studio 2019 community or VS Code. Basic knowledge of C#

Below are three steps to build a SignalR real-time app, connect it to translation services, and then use Speech to Text.

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

	#add cors if you know your domains already.
	#az signalr cors add --name $signalName --resource-group $groupName --allowed-origins "http://example1.com" "https://example2.com"
	signalKey=$(az signalr key list --name $signalName --resource-group $groupName --query primaryKey -o tsv)

	speechName=Speech${groupName,,}
	textName=Text${groupName,,}
	az cognitiveservices account list-skus --location $location --kind SpeechServices

	az cognitiveservices account create --name $speechName --resource-group $groupName --location $location \
	--kind SpeechServices --sku S0 --yes

	speechKey1=$(az cognitiveservices account keys list --name $speechName --resource-group $groupName --query "key1" -o tsv)

	az cognitiveservices account create --name $textName --resource-group $groupName --location $location \
	--kind TextTranslation --sku S0 --yes

	textKey1=$(az cognitiveservices account keys list --name $textName --resource-group $groupName --query "key1" -o tsv)

	printf "\n\n Change <speechKey1> with:\n$speechKey1\n\n"

	printf "\n\n Change <textKey1> with:\n$textKey1\n\n"

	printf "\n Replace <signalKey> with:\n$signalKey\n\n"

	printf "\n\nReplace <signalConnString> with:\n$signalConnString\n\n"
```

FYI, the link below provides a reference list of Cognitive Services types with CLI reference
https://docs.microsoft.com/en-us/azure/cognitive-services/cognitive-services-apis-create-account-cli?tabs=windows

3. Copy SignalR connection string and Cognitive service keys to notepad.

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

13. You can deploy a function app to Azure via publishing profile if you want to.

14. Let's connect local and test remote applications.

15. There is a sample single page web application hosted in GitHub for your convenience. Open your browser to https://azure-samples.github.io/signalr-service-quickstart-serverless-chat/demo/chat-v2/.

16. When prompted for the function app base URL, enter http://localhost:7071.

17. Enter a username when prompted.

18. The web application calls the GetSignalRInfo function in the function app to retrieve the connection information to connect to Azure SignalR Service. When the connection is complete, the chat message input box appears.

19. Type a message and press enter. The application sends the message to the SendMessage function in the Azure Function app, which then uses the SignalR output binding to broadcast the message to all connected clients. If everything is working correctly, the message should appear in the application.

20. Open another instance of the web application in a different browser window. You will see that any messages sent will appear in all instances of the application.
    
Be aware of the cors issues and HTTP vs. HTTPS usage. My suggestion is to deploy the function to the Web app and setup CORS exceptions in the configuration.


# Cognitive services translations

The next step is to use Cognitive Services translation samples and integrate them with existing solutions. You can use advanced scenarios with "Speech to text" and then translation. But let's start with a simple case.
The main goal is to combine two tutorials inside one serverless application.

Text translator documentation(quite good)
https://docs.microsoft.com/en-us/azure/cognitive-services/translator/


1. We will be following a slightly adjusted version of this tutorial.
	https://docs.microsoft.com/en-us/azure/cognitive-services/translator/quickstart-translate?pivots=programming-language-csharp

2. Essentially you need to take source code from tutorial above and add it to the new Azure Function, so you can call translation API from code.

3. Get Translation subscription key from Azure CLI output and set it to global variables along with translator text endpoint. Use <textKey1> use setx.
	
	```
	setx TRANSLATOR_TEXT_SUBSCRIPTION_KEY "432432532532"
	setx TRANSLATOR_TEXT_ENDPOINT "https://api.cognitive.microsofttranslator.com/"
	```


4. Create an Azure Function with HTTP trigger in Visual studio.

5. Add namespaces and Json nuget to project
	using System;
	using System.Net.Http;
	using System.Text;
	using System.Threading.Tasks;
	// Install Newtonsoft.Json with NuGet
	using Newtonsoft.Json;

6. Create classes for the JSON response

```
	public class TranslationResult
	{
	    public DetectedLanguage DetectedLanguage { get; set; }
	    public TextResult SourceText { get; set; }
	    public Translation[] Translations { get; set; }
	}

	public class DetectedLanguage
	{
	    public string Language { get; set; }
	    public float Score { get; set; }
	}

	public class TextResult
	{
	    public string Text { get; set; }
	    public string Script { get; set; }
	}

	public class Translation
	{
	    public string Text { get; set; }
	    public TextResult Transliteration { get; set; }
	    public string To { get; set; }
	    public Alignment Alignment { get; set; }
	    public SentenceLength SentLen { get; set; }
	}

	public class Alignment
	{
	    public string Proj { get; set; }
	}

	public class SentenceLength
	{
	    public int[] SrcSentLen { get; set; }
	    public int[] TransSentLen { get; set; }
	}
```

7. Get subscription information from environment variables
	
	```
	string subscriptionKey = Environment.GetEnvironmentVariable("TRANSLATOR_TEXT_SUBSCRIPTION_KEY");
	string endpoint = Environment.GetEnvironmentVariable("TRANSLATOR_TEXT_ENDPOINT");
	```

8. Add translate request class. And change it to output translated values instead of console app.

```
	static public async Task TranslateTextRequest(string subscriptionKey, string endpoint, string route, string inputText)
	{
		using (var client = new HttpClient())
		using (var request = new HttpRequestMessage())
		{
		request.Method = HttpMethod.Post;
		request.RequestUri = new Uri(endpoint + route);
		request.Content = new StringContent(requestBody, Encoding.UTF8, "application/json");
		request.Headers.Add("Ocp-Apim-Subscription-Key", subscriptionKey);

		HttpResponseMessage response = await client.SendAsync(request).ConfigureAwait(false);
		string result = await response.Content.ReadAsStringAsync();
		TranslationResult[] deserializedOutput = JsonConvert.DeserializeObject<TranslationResult[]>(result);
		foreach (TranslationResult o in deserializedOutput)
		{
		    // Print the detected input language and confidence score.
		    Console.WriteLine("Detected input language: {0}\nConfidence score: {1}\n", o.DetectedLanguage.Language, o.DetectedLanguage.Score);
		    // Iterate over the results and print each translation.
		    foreach (Translation t in o.Translations)
		    {
			Console.WriteLine("Translated to {0}: {1}", t.To, t.Text);
		    }
		}
		}

	}
```

9. Call it from function

```
[FunctionName("Function1")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            string textToTranslate = req.Query["text"];
            string subscriptionKey = Environment.GetEnvironmentVariable("TRANSLATOR_TEXT_SUBSCRIPTION_KEY");
            string endpoint = Environment.GetEnvironmentVariable("TRANSLATOR_TEXT_ENDPOINT");
            string route = "/translate?api-version=3.0&to=de&to=it&to=ja&to=th";

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            textToTranslate = textToTranslate ?? data?.name;

            string translationResult = await TranslateTextRequest(subscriptionKey, endpoint, route, textToTranslate);

            string responseMessage = string.IsNullOrEmpty(translationResult)
                ? "This HTTP triggered function executed successfully. Pass a Text in the query string or in the request body for a personalized response."
                : $"Hello, {translationResult}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }
```

10. Fix bugs :).
	
  
# Cognitive services Speech to text and translation

Let's take a look at general service description of Speech to text. This step covers console application under Windows 10.

https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/get-started
https://github.com/Azure-Samples/cognitive-services-speech-sdk - list of examples.

1. Lets take a look at the demo description of the workshop and GitHub repository.
https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/get-started

2. Get console application from this GitHub repository
https://github.com/Azure-Samples/cognitive-services-speech-sdk/tree/master/quickstart/csharp/dotnetcore/translate-speech-to-text

3. Replace values with northeurope region and <speechKey1> from Azure CLI output.
	
```
	var config = SpeechTranslationConfig.FromSubscription("YourSubscriptionKey", "YourServiceRegion");
```

4. Run(F5) console application, fix bugs, and observe results. Update language configuration to the needed source.

5. Now you need to send "Speech to text" results to your translator function.

6. Now its time to broadcast translation text to Azure function with the help of the following code.
https://github.com/aspnet/AzureSignalR-samples/tree/master/samples/Management/Sig



# Thanks :)
