# Integrate Open AI Services with Cosmos DB: RAG pattern
This repository provides a demo showcasing the usage of the RAG pattern for integrating Open AI services with custom data in Cosmos DB. The goal is to limit the responses from Open AI services based on recipes stored in Cosmos DB.

## Prerequsites
- Azure Cosmos DB for MongoDB vCore Account
- Azure Open AI Service
    - Deploy text-davinci-003 model for Embeding
    - Deploy gpt-35-turbo model for Chat Completion

  
## Dependencies
This demo is built using .NET and utilizes the following SDKs:
-	MongoDB Driver
-	Azure Open AI Services SDK

## Getting Started
When you run the application for the first time, it will connect to Cosmos DB and report that there are no recipes available, as we have not uploaded any recipes yet.
To begin, follow these steps:
1)	**Upload Documents to Cosmos DB:** Select the first option in the application and hit enter. This option reads documents from the local machine and uploads the JSON files to the Cosmos DB NoSQL account.
    #### Parsing files from Local Folder
    ``` C#
    public static List<Recipe> ParseDocuments(string Folderpath)
        {
            List<Recipe> ret = new List<Recipe>();

            Directory.GetFiles(Folderpath).ToList().ForEach(f =>
                {
                    var jsonString= System.IO.File.ReadAllText(f);
                    Recipe recipe = JsonConvert.DeserializeObject<Recipe>(jsonString);
                    recipe.id = recipe.name.ToLower().Replace(" ", "");
                    ret.Add(recipe);

                }
            );


            return ret;

        }
    
    ```

    ####  Upsert Documents to Azure Cosmos DB for MongoDB vCore
    ```C#
    public async Task UpsertVectorAsync(Recipe recipe)
        {

            BsonDocument document = recipe.ToBsonDocument();

            if (!document.Contains("_id"))
            {
                Console.WriteLine("UpsertVectorAsync: Document does not contain _id.");
                throw new ArgumentException("UpsertVectorAsync: Document does not contain _id.");
            }

            string? _idValue = document["_id"].ToString();


            try
            {
                var filter = Builders<BsonDocument>.Filter.Eq("_id", _idValue);
                var options = new ReplaceOptions { IsUpsert = true };
                await _recipeCollection.ReplaceOneAsync(filter, document, options);

            }
            catch (Exception ex)
            {

                Console.WriteLine($"Exception: UpsertVectorAsync(): {ex.Message}");
                throw;
            }

        }
    ```
3)	**Vectorize and Upload Recipes to Azure Cosmos DB for MongoDB vCore:** The JSON data uploaded to Cosmos DB is not yet ready for efficient integration with Open AI. To use the RAG pattern, we need to find relevant recipes from Cosmos DB. Embeddings help us achieve this. To accomplish the task, we will utilize the vector search capability in Azure Cosmos DB for MongoDB vCore to search for embeddings. Firstly, create the required vector search index in Azure Cosmos DB for MongoDB vCore. Then, vectorize the recipes and upload the vectors to Azure Cosmos DB for MongoDB vCore. Selecting the second option in the application will perform all these activities.

      
    ####  Build the Vector Index in Azure Cosmos DB for MongoDB vCore
    ```C#
    public void CreateVectorIndexIfNotExists(string vectorIndexName)
        {

            try
            {

                //Find if vector index exists in vectors collection
                using (IAsyncCursor<BsonDocument> indexCursor = _recipeCollection.Indexes.List())
                {
                    bool vectorIndexExists = indexCursor.ToList().Any(x => x["name"] == vectorIndexName);
                    if (!vectorIndexExists)
                    {
                        BsonDocumentCommand<BsonDocument> command = new BsonDocumentCommand<BsonDocument>(
                        BsonDocument.Parse(@"
                            { createIndexes: 'Recipe', 
                              indexes: [{ 
                                name: 'vectorSearchIndex', 
                                key: { embedding: 'cosmosSearch' }, 
                                cosmosSearchOptions: { kind: 'vector-ivf', numLists: 5, similarity: 'COS', dimensions: 1536 } 
                              }] 
                            }"));

                        BsonDocument result = _database.RunCommand(command);
                        if (result["ok"] != 1)
                        {
                            Console.WriteLine("CreateIndex failed with response: " + result.ToJson());
                        }
                    }
                }

            }
            catch (MongoException ex)
            {
                Console.WriteLine("MongoDbService InitializeVectorIndex: " + ex.Message);
                throw;
            }

        }
    
    ```

    #### Initialize the Azure Open AI SDK
  	``` C#
    public OpenAIService(string endpoint, string key, string embeddingsDeployment, string CompletionDeployment, string maxTokens)
    {



        _openAIEmbeddingDeployment = embeddingsDeployment;
        _openAICompletionDeployment = CompletionDeployment;
        _openAIMaxTokens = int.TryParse(maxTokens, out openAIMaxTokens) ? openAIMaxTokens : 8191;


        OpenAIClientOptions clientOptions = new OpenAIClientOptions()
        {
            Retry =
            {
                Delay = TimeSpan.FromSeconds(2),
                MaxRetries = 10,
                Mode = RetryMode.Exponential
            }
        };

        try
        {

            //Use this as endpoint in configuration to use non-Azure Open AI endpoint and OpenAI model names
            if (endpoint.Contains("api.openai.com"))
                _openAIClient = new OpenAIClient(key, clientOptions);
            else
                _openAIClient = new(new Uri(endpoint), new AzureKeyCredential(key), clientOptions);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"OpenAIService Constructor failure: {ex.Message}");
        }
    }
    ```   
   
    #### Generate Embedings using Open AI Service
  	``` C#
    public async Task<float[]?> GetEmbeddingsAsync(dynamic data)
    {
        try
        {
            EmbeddingsOptions options = new EmbeddingsOptions(data)
            {
                Input = data
            };

            var response = await _openAIClient.GetEmbeddingsAsync(openAIEmbeddingDeployment, options);

            Embeddings embeddings = response.Value;

            float[] embedding = embeddings.Data[0].Embedding.ToArray();

            return embedding;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"GetEmbeddingsAsync Exception: {ex.Message}");
            return null;
        }
    }
    ```

    
5)	**Perform Search:** The third option in the application runs the search based on the user query. The user query is converted to an embedding using the Open AI service. The embedding is then sent to Azure Cosmos DB for MongoDB vCore to perform a vector search. The vector search attempts to find vectors that are close to the supplied vector and returns a list of documents from Azure Cosmos DB for MongoDB vCore. The serialized documents retrieved from Azure Cosmos DB for MongoDB vCore are passed as strings to the Open AI service completion service as prompts. During this process, we also include the instructions we want to provide to the Open AI service as prompt. The Open AI service processes the instructions and custom data provided as prompts to generate the response.

    

    #### Performing Vector Search in Azure Cosmos DB for MongoDB vCore
  	```C#
    public async Task<List<Recipe>> VectorSearchAsync(float[] queryVector)
        {
            List<string> retDocs = new List<string>();

            string resultDocuments = string.Empty;

            try
            {
                //Search Azure Cosmos DB for MongoDB vCore collection for similar embeddings
                //Project the fields that are needed
                BsonDocument[] pipeline = new BsonDocument[]
                {   
                    BsonDocument.Parse($"{{$search: {{cosmosSearch: {{ vector: [{string.Join(',', queryVector)}], path: 'embedding', k: {_maxVectorSearchResults}}}, returnStoredSource:true}}}}"),
                    BsonDocument.Parse($"{{$project: {{embedding: 0}}}}"),
                };

                var bsonDocuments = await _recipeCollection.Aggregate<BsonDocument>(pipeline).ToListAsync();

                var recipes = bsonDocuments.ToList().ConvertAll(bsonDocument => BsonSerializer.Deserialize<Recipe>(bsonDocument)); 
                return recipes;
            }
            catch (MongoException ex)
            {
                Console.WriteLine($"Exception: VectorSearchAsync(): {ex.Message}");
                throw;
            }

        }
    ```
    
   
    #### Prompt Engineering to make sure Open AI service limits the response to supplied recipes
    ```C#
    //System prompts to send with user prompts to instruct the model for chat session
    private readonly string _systemPromptRecipeAssistant = @"
        You are an intelligent assistant for Contoso Recipes. 
        You are designed to provide helpful answers to user questions about 
        recipes, cooking instructions provided in JSON format below.
  
        Instructions:
        - Only answer questions related to the recipe provided below,
        - Don't reference any recipe not provided below.
        - If you're unsure of an answer, you can say ""I don't know"" or ""I'm not sure"" and recommend users search themselves.        
        - Your response  should be complete. 
        - List the Name of the Recipe at the start of your response folowed by step by step cooking instructions
        - Assume the user is not an expert in cooking.
        - Format the content so that it can be printed to the Command Line console;
        - In case there are more than one recipes you find let the user pick the most appropiate recipe.";
     ```
    
    #### Generate Chat Completion based on Prompt and Custom Data 
    ``` C#
    public async Task<(string response, int promptTokens, int responseTokens)> GetChatCompletionAsync(string userPrompt, string documents)
    {

        try
        {

            ChatMessage systemMessage = new ChatMessage(ChatRole.System, _systemPromptRecipeAssistant + documents);
            ChatMessage userMessage = new ChatMessage(ChatRole.User, userPrompt);


            ChatCompletionsOptions options = new()
            {

                Messages =
                {
                    systemMessage,
                    userMessage
                },
                MaxTokens = openAIMaxTokens,
                Temperature = 0.5f, //0.3f,
                NucleusSamplingFactor = 0.95f, 
                FrequencyPenalty = 0,
                PresencePenalty = 0
            };

            Azure.Response<ChatCompletions> completionsResponse = await openAIClient.GetChatCompletionsAsync(openAICompletionDeployment, options);

            ChatCompletions completions = completionsResponse.Value;

            return (
                response: completions.Choices[0].Message.Content,
                promptTokens: completions.Usage.PromptTokens,
                responseTokens: completions.Usage.CompletionTokens
            );

        }
        catch (Exception ex)
        {

            string message = $"OpenAIService.GetChatCompletionAsync(): {ex.Message}";
            Console.WriteLine(message);
            throw;

        }
    }
    ```
