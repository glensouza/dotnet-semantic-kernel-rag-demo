# Create a Semantic Search Plugin to query the AI Search Index

1. Install the following NuGet packages:

- `Microsoft.SemanticKernel`
- `Microsoft.SemanticKernel.Connectors.AzureAISearch`
- `Microsoft.SemanticKernel.Connectors.OpenAI`
- `Microsoft.SemanticKernel.Plugins.OpenApi`

1. Navigate back to the reference application and open the **Home.razor** file.
1. Add the using statements for the Semantic Kernel and the OpenAI Text Embedding Generation service.

    ```csharp
    @using Azure
    @using Azure.Search.Documents.Indexes
    @using Microsoft.SemanticKernel;
    @using Microsoft.SemanticKernel.ChatCompletion;
    @using Microsoft.SemanticKernel.Connectors.OpenAI;
    @using Microsoft.SemanticKernel.Plugins.OpenApi;
    ```

1. Inject the Configuraion service into the component.

    ```csharp
    [Inject]
    private IConfiguration Configuration { get; set; }
    ```

1. Inside the `OnInitializedAsync` method register the service for Azure OpenAI Text Embedding Generation with the Kernel Builder, the AI Search and the *Azure AI Search Vector Store connector* with the Kernel Builder.

    ```csharp
    protected override async Task OnInitializedAsync()
    {
        // Configure Semantic Kernel
        IKernelBuilder kernelBuilder = Kernel.CreateBuilder();

        // Add OpenAI Chat Completion
        kernelBuilder.AddAzureOpenAIChatCompletion(
            this.Configuration["AOI_DEPLOYMODEL"] ?? "gpt-35-turbo", 
            this.Configuration["AOI_ENDPOINT"]!, 
            this.Configuration["AOI_API_KEY"]!);

        // Register Azure OpenAI Text Embeddings Generation
        kernelBuilder.Services.AddAzureOpenAITextEmbeddingGeneration(
            this.Configuration["EMBEDDINGS_DEPLOYMODEL"]!, 
            this.Configuration["AOI_ENDPOINT"]!, 
            this.Configuration["AOI_API_KEY"]!);

        // Register Search Index
        kernelBuilder.Services.AddSingleton(
            _ => new SearchIndexClient(
                new Uri(this.Configuration["AI_SEARCH_URL"]!),
                new AzureKeyCredential(this.Configuration["AI_SEARCH_KEY"]!)));

        // Register Azure AI Search Vector Store
        kernelBuilder.AddAzureAISearchVectorStore();

        // This is used by Blazor to capture the user input for shortcut keys.
        this.KeyCodeService.RegisterListener(this.OnKeyDownAsync);
        await this.LoadChatHistory();
    }
    ...

1. Create a new class in the **Plugins** folder called **SearchPlugin.cs**. This is the Semantic Search Plugin to query the AI Search Index created earlier. This Plugin should take the users query and generate an embedding using the Text Embedding model. The embedding should then be used to query the AI Search Index containing the indexed file(s) and return the most relevant information.

    ```csharp
    using Azure.Search.Documents;
    using Azure;
    using Azure.Search.Documents.Indexes;
    using Azure.Search.Documents.Models;
    using Microsoft.SemanticKernel;
    using Microsoft.SemanticKernel.Embeddings;
    using System.ComponentModel;
    using System.Text.Json.Serialization;

    #pragma warning disable SKEXP0001 // Type is for evaluation purposes only and is subject to change or removal in future updates. Suppress this diagnostic to proceed.

    namespace Blazor.AI.Web.Plugins;

    public class SearchPlugin(ITextEmbeddingGenerationService textEmbeddingGenerationService, SearchIndexClient indexClient)
    {
        [KernelFunction("contoso_search")]
        [Description("Search documents for employer Contoso")]
        public async Task<string> SearchAsync([Description("The users optimized semantic search query")] string query)
        {
            // Convert string query to vector
            ReadOnlyMemory<float> embedding = await textEmbeddingGenerationService.GenerateEmbeddingAsync(query);

            // Set the index to use in AI Search
            SearchClient searchClient = indexClient.GetSearchClient("employeehandbook");

            // Configure request parameters
            VectorizedQuery vectorQuery = new(embedding);
            vectorQuery.Fields.Add("contentVector"); // name of the vector field from index schema

            SearchOptions searchOptions = new() { VectorSearch = new() { Queries = { vectorQuery } } };

            //var response = await searchClient.SearchAsync<SearchDocument>(searchOptions);

            // Perform search request
            Response<SearchResults<IndexSchema>> response = await searchClient.SearchAsync<IndexSchema>(searchOptions);

            //// Collect search results
            await foreach (SearchResult<IndexSchema> result in response.Value.GetResultsAsync())
            {
                return result.Document.Content; // Return text from first result
            }

            return string.Empty;
        }

        //This schema comes from the index schema in Azure AI Search
        private sealed class IndexSchema
        {
            [JsonPropertyName("content")]
            public string Content { get; set; }

            [JsonPropertyName("title")]
            public string Title { get; set; }

            [JsonPropertyName("url")]
            public string Url { get; set; }
        }
    }
    ```

1. Add the plugin to Semantic Kernel:

    ```csharp
    this.kernel.Plugins.AddFromType<SearchPlugin>("SearchPlugin", this.kernel.Services);
    ```

1. Update the `SendMessage()` method to use the Semantic Search Plugin when the user asks a question about the Contoso Handbook.

    ```csharp
    private async Task SendMessage()
    {
        if (!string.IsNullOrWhiteSpace(this.MessageInput) && this.chatCompletionService != null)
        {
            // This tells Blazor the UI is going to be updated.
            this.StateHasChanged();
            this.loading = true;

            // Copy the user message to a local variable and clear the MessageInput field in the UI
            string userMessage = this.MessageInput;
            this.MessageInput = string.Empty;
            this.StateHasChanged();
            
            //// Start Sending a message to the chat completion service

            this.chatHistory.AddUserMessage(userMessage);
            try
            {
                IReadOnlyList<ChatMessageContent> response = await this.chatCompletionService.GetChatMessageContentsAsync(this.chatHistory,
                    executionSettings: this.openAIPromptExecutionSettings,
                    kernel: this.kernel);
                foreach (ChatMessageContent aiResponse in response)
                {
                    if ((aiResponse.Role == AuthorRole.Assistant || aiResponse.Role == AuthorRole.Tool) && aiResponse.Content != null)
                    {
                        this.chatHistory.AddMessage(aiResponse.Role, aiResponse.Content);
                    }
                }
            }
            catch (HttpOperationException e)
            {
                if (e.ResponseContent != null) this.chatHistory.AddAssistantMessage(e.ResponseContent);
            }

            //// End Sending a message to the chat completion service

            this.loading = false;
        }
    }
    ```

1. Test the Plugin

    Test the plugin by running the applications and asking the Chatbot questions about the Contoso Handbook. The Chatbot should be able to answer questions similar to the following:

    - `What are the steps for the Contoso Performance Reviews?`
    - `What is Contoso's policy on Data Security?`
    - `Who do I contact at Contoso for questions regarding workplace safety?`
