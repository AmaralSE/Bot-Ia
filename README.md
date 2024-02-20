using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Builder.Integration;
using Microsoft.Bot.Builder.Dialogs.Choices;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Bot.Schema;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

namespace MeuBot
{
    public class EchoBot : IBot
    {
        private readonly BotState _userState;
        private readonly BotState _conversationState;
        private readonly ILogger _logger;

        public EchoBot(UserState userState, ConversationState conversationState, ILoggerFactory loggerFactory)
        {
            _userState = userState ?? throw new ArgumentNullException(nameof(userState));
            _conversationState = conversationState ?? throw new ArgumentNullException(nameof(conversationState));
            _logger = loggerFactory?.CreateLogger<EchoBot>() ?? throw new ArgumentNullException(nameof(loggerFactory));
        }

        public async Task OnTurnAsync(ITurnContext turnContext, CancellationToken cancellationToken = default(CancellationToken))
        {
            if (turnContext == null)
            {
                throw new ArgumentNullException(nameof(turnContext));
            }

            var dc = await _dialogs.CreateContextAsync(turnContext, cancellationToken);

            if (turnContext.Activity.Type == ActivityTypes.Message)
            {
                await dc.ContinueDialogAsync(cancellationToken);

                if (!turnContext.Responded)
                {
                    await dc.BeginDialogAsync(nameof(MainDialog), cancellationToken);
                }
            }
            else if (turnContext.Activity.Type == ActivityTypes.ConversationUpdate)
            {
                if (turnContext.Activity.MembersAdded != null)
                {
                    foreach (var member in turnContext.Activity.MembersAdded)
                    {
                        if (member.Id != turnContext.Activity.Recipient.Id)
                        {
                            await turnContext.SendActivityAsync($"Olá e bem-vindo!");
                        }
                    }
                }
            }

            await _conversationState.SaveChangesAsync(turnContext, false, cancellationToken);
            await _userState.SaveChangesAsync(turnContext, false, cancellationToken);
        }
    }

    public class MainDialog : ComponentDialog
    {
        public MainDialog() : base(nameof(MainDialog))
        {
            AddDialog(new TextPrompt(nameof(TextPrompt)));
            AddDialog(new WaterfallDialog(nameof(WaterfallDialog), new WaterfallStep[]
            {
                PromptStepAsync,
                FinalStepAsync
            }));

            InitialDialogId = nameof(WaterfallDialog);
        }

        private async Task<DialogTurnResult> PromptStepAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
        {
            return await stepContext.PromptAsync(nameof(TextPrompt), new PromptOptions { Prompt = MessageFactory.Text("Olá! Como posso ajudar?") }, cancellationToken);
        }

        private async Task<DialogTurnResult> FinalStepAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
        {
            var response = (string)stepContext.Result;
            await stepContext.Context.SendActivityAsync($"Você disse '{response}'.", cancellationToken: cancellationToken);
            return await stepContext.EndDialogAsync();
        }
    }

    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddBot<EchoBot>(options =>
            {
                options.CredentialProvider = new ConfigurationCredentialProvider();
                options.Middleware.Add(new AutoSaveStateMiddleware(options.State));
            });

            // Adicione o UserState e o ConversationState
            services.AddSingleton<UserState>();
            services.AddSingleton<ConversationState>();

            services.AddSingleton<MainDialog>();
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseDefaultFiles()
                .UseStaticFiles()
                .UseWebSockets()
                .UseBotFramework();
        }
    }
}
