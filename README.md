# Dotnet Semantic Kernel RAG Demo

## Retrieval-Augmented Generation (RAG)

The RAG (Retrieval-Augmented Generation) pattern is an AI model architecture that enhances the generation of text by integrating external, relevant information retrieved from a knowledge base or database, improving accuracy and context. It combines retrieval mechanisms with generative models to produce more informed and contextually relevant responses.

## Visual Studio New Project Setup

- Navigate to `File` > `New` > `Project`
- Select `.NET Aspire Starter App` template
- Click `Next`
- Enter name for solution and click `Next`
- Select `.NET 8 LTS` and click `Create`

## Add FluentUI Components

- Add the following NuGet packages:
  - Microsoft.FluentUI.AspNetCore.Components
  - Microsoft.FluentUI.AspNetCore.Components.Emoji
  - Microsoft.FluentUI.AspNetCore.Components.Icons
  - Microsoft.SemanticKernel
  - Markdig
  - Markdown.ColorCode
- Open the `_Imports.razor` file and add the following:

  ```csharp
  @using Microsoft.FluentUI.AspNetCore.Components
  @using Microsoft.FluentUI.AspNetCore.Components.DesignTokens
  @using Microsoft.FluentUI.AspNetCore.Components.Extensions
  @using Microsoft.SemanticKernel
  @using Microsoft.SemanticKernel.ChatCompletion;
  ```

- Add this line to `program.cs`:

  ```csharp
  builder.Services.AddFluentUIComponents();
  ```

## CSS

Add the following css to your project:

```css
.chat-header {
    display: flex;
    padding: 10px;
    border-bottom: 1px solid var(--neutral-outline-rest);
}

.chat-messages {
    flex-grow: 1;
    overflow-y: auto;
    padding: 10px;
    display: flex;
    flex-direction: column;
    overscroll-behavior-y: contain;
    scroll-snap-type: y;
    scroll-behavior: smooth;
}

.message-card {
    padding: 10px;
    margin-bottom: 8px;
    max-width: 70vw;
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.message-container {
    display: flex;
    align-items: flex-start;
}

    .message-container .message-content {
        padding-left: 10px;
        word-wrap: break-word;
    }

.user {
    align-self: flex-start;
    background-color: var(--neutral-layer-4);
}

.assistant {
    align-self: flex-end;
    background-color: var(--neutral-layer-2);
    display: flex;
    flex-direction: row-reverse;
}

.tool {
    align-self: flex-end;
    background-color: var(--neutral-layer-4);
    display: flex;
    flex-direction: row-reverse;
}

.assistant .message-container {
    flex-direction: row-reverse;
}

.tool .message-container {
    flex-direction: row-reverse;
}

.chat-input {
    padding: 10px;
    border-top: 1px solid var(--neutral-outline-rest);
}

.chat-input-area {
    width: 100%;
    border-radius: 5px;
}

.send-button {
    width: 80px;
    height: 55px;
}

pre {
    display: inline-block;
    width: 60vw;
    overflow-x: auto;
}
```

## Razor Component

Replace the Home.razor file with the following:

```csharp
@page "/"
@rendermode InteractiveServer

@using Markdig

<PageTitle>Dotnet RAG Chat</PageTitle>

<h1>Welcome to chat</h1>

<p>This Dotnet Blazor app uses Semantic Kernel orchestrating Azure AI Search and Azure OpenAI for RAG pattern from Word documents stored in Azure Blob Storage Account.</p>

<FluentCard Style="display: flex; flex-direction: column">
    <div class="chat-header">
        <FluentHeader Style="border-radius: 5px; width: 100%;">
            Chat
            <FluentSpacer />
            <FluentButton @onclick="this.ClearChat" Appearance="Appearance.Neutral" IconStart="@(new Icons.Regular.Size24.ChatSparkle())">New Chat</FluentButton>
        </FluentHeader>
    </div>
    <div class="chat-messages">
        @if (this.chatHistory.Any())
        {
            @foreach (ChatMessageContent message in this.chatHistory)
            {
                if ((message.Role.ToString().ToLower() == "tool" && !DisplayToolMessages) || string.IsNullOrEmpty(message.Content))
                {
                    continue;
                }

                <div class="@message.Role.ToString().ToLower() message-card">
                    <div class="@message.Role.ToString().ToLower() message-container">
                        @if (message.Role.ToString() == "user")
                        {
                            <FluentIcon Value="@(new Icons.Regular.Size24.PersonChat())"/>
                        }
                        else if (message.Role.ToString().ToLower() == "tool" && DisplayToolMessages)
                        {
                            <FluentIcon Value="@(new Icons.Regular.Size24.Wrench())" />
                        }
                        else
                        {
                            <FluentIcon Value="@(new Icons.Regular.Size24.Bot())"/>
                        }
                        <div class="message-content">
                            @if (!string.IsNullOrEmpty(message.Content))
                            {
                                @((MarkupString)Markdown.ToHtml(message.Content, this.pipeline))
                            }
                        </div>
                    </div>
                </div>
            }
        }
        else
        {
            <FluentLabel>No messages yet. Start the conversation!</FluentLabel>
        }
        <FluentProgressRing Visible="@(this.loading)" style="width: 42px; height: 82px; position: absolute; bottom: 50%; left: 50%;"></FluentProgressRing>
    </div>
    <div class="chat-input">
        <FluentStack Orientation="Orientation.Horizontal" Width="100%" HorizontalGap="8">
            <FluentTextArea @ref="this.inputTextArea" Immediate="true" ImmediateDelay="15" @bind-Value="this.MessageInput" placeholder="Type a message..." Rows="2" Appearance=FluentInputAppearance.Filled Class="chat-input-area" />
            <FluentButton @ref="this.submitButton" Loading="@(this.loading)" @onclick="this.SendMessage" IconStart="@(new Icons.Regular.Size20.Send())" Appearance="Appearance.Lightweight" Class="send-button">Send</FluentButton>
        </FluentStack>
    </div>
</FluentCard>
<FluentKeyCodeProvider />

@code {
    [Inject]
    private IKeyCodeService KeyCodeService { get; set; }

    private string? MessageInput { get; set; }
    private bool loading = false;
    private const bool DisplayToolMessages = false;
    private readonly ChatHistory chatHistory = [];
    private FluentButton submitButton = new();
    private FluentTextArea inputTextArea = new();
    private readonly MarkdownPipeline pipeline = new MarkdownPipelineBuilder()
        .UseAdvancedExtensions()
        .UseBootstrap()
        .UseEmojiAndSmiley()
        .Build();

    protected override async Task OnInitializedAsync()
    {
        // This is used by Blazor to capture the user input for shortcut keys.
        this.KeyCodeService.RegisterListener(this.OnKeyDownAsync);
        await this.LoadChatHistory();
    }

    private async Task LoadChatHistory()
    {
        this.loading = true;
        this.loading = false;
    }

    private async Task SendMessage()
    {
        if (string.IsNullOrEmpty(this.MessageInput))
        {
            return;
        }

        this.loading = true;

        this.chatHistory.AddUserMessage(this.MessageInput);
        this.chatHistory.AddAssistantMessage("TEST");

        this.MessageInput = string.Empty;
        await this.LoadChatHistory();
    }

    private async Task ClearChat()
    {
        this.loading = true;
        this.chatHistory.Clear();
        await this.LoadChatHistory();
    }

    private async Task OnKeyDownAsync(FluentKeyCodeEventArgs args)
    {
        if (args is { CtrlKey: true, Value: "Enter" })
        {
            // Ctrl + Enter Pressed
            await this.InvokeAsync(async () =>
            {
                this.StateHasChanged();
                await Task.Delay(180);
                await this.submitButton.OnClick.InvokeAsync();
            });
        }
    }
}
```

## Next Steps

[Infrastructure](Infrastructure.md)
