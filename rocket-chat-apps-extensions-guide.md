# Rocket.Chat Apps-Engine Developer Guide

This document acts as a comprehensive resource for building various extension skills for Rocket.Chat apps using the Apps-Engine framework. The information compiled here mitigates the need for continuous web scraping.

---

## 1. Event Interfaces (`rc-event-interface`) - (FOR FUTURE)

```yaml
---
name: rc-event-interface
description: >
  Use when the user wants the app to react to activities happening in the 
  workspace. Phrases like "when someone joins", "when someone leaves",
  "automatically reply when", "trigger when a message is sent",
  "react to messages", "do something when a user joins the room",
  "fire when a room is created", "listen for messages",
  "intercept messages before they're sent", "run something automatically",
  "when a new user registers", "detect when someone types",
  "respond automatically to", "monitor messages",
  "watch for activity" all indicate an event interface is needed.
---
```

You are an expert in Rocket.Chat Apps Engine event interfaces.

When building an event interface app, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/messages/IPreMessageSentPrevent.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/messages/IPostMessageSent.d.ts`
(and analogous definitions in the `rooms/` and `users/` directories).

Key safety rules — always apply these:
- **Never** trigger from `IPostMessageSent` (or any `IPost` handler) without a sender check: `if (message.sender.id === botUser.id) return;` otherwise you will create an infinite loop where the bot replies to itself endlessly!
- **Understand `IPre` vs `IPost`**:
  - `IPre` handlers (like `IPreMessageSentPrevent`, `IPreMessageSentModify`) fire *before* the action hits the database. They can prevent the action (return boolean) or mutate the payload.
  - `IPost` handlers (like `IPostMessageSent`) fire *after* the action has completed. They are read-only regarding the original event and are typically used for side-effects (like logging, sending a follow-up message, or calling an external API).

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IMessage } from '@rocket.chat/apps-engine/definition/messages/IMessage';
import { IPreMessageSentPrevent } from '@rocket.chat/apps-engine/definition/messages/IPreMessageSentPrevent';
import { IPostMessageSent } from '@rocket.chat/apps-engine/definition/messages/IPostMessageSent';

export class EventExampleApp extends App implements IPreMessageSentPrevent, IPostMessageSent {
    
    // IPre Handler: Decides if the execution method should run
    public async checkPreMessageSentPrevent(message: IMessage, read: IRead, http: IHttp): Promise<boolean> {
        return message.text === 'badword';
    }

    // IPre Handler: Executes the prevention
    public async executePreMessageSentPrevent(message: IMessage, read: IRead, http: IHttp, persistence: IPersistence): Promise<boolean> {
        // Returning true prevents the message from being sent
        // Returning false allows the message through
        return true; 
    }

    // IPost Handler: Decides if it should run
    public async checkPostMessageSent(message: IMessage, read: IRead, http: IHttp): Promise<boolean> {
        return message.text === 'hello';
    }

    // IPost Handler: Executes AFTER the message is sent
    public async executePostMessageSent(message: IMessage, read: IRead, http: IHttp, persistence: IPersistence, modify: IModify): Promise<void> {
        // SAFETY RULE: Always prevent infinite loops from the bot responding to itself
        const botUser = await read.getUserReader().getAppUser(this.getID());
        if (botUser && message.sender.id === botUser.id) {
            return;
        }

        const room = message.room;
        const msgBuilder = modify.getCreator().startMessage()
            .setRoom(room)
            .setSender(botUser)
            .setText('Hello to you too!');

        await modify.getCreator().finish(msgBuilder);
    }
}
```

## 2. Email Communications (`rc-email`) - (FOR FUTURE)

```yaml
---
name: rc-email
description: >
  Use when the user wants to send emails or notify users externally.
  Phrases like "send an email", "email me when", "notify via email",
  "email notification", "send a message to someone's inbox",
  "alert me by email", "email the team when", "send an email report",
  "forward to email", "email someone outside of Rocket.Chat",
  "send a daily email summary", "notify externally"
  all indicate an email communication skill is needed.
---
```

You are an expert in Rocket.Chat Apps Engine email communications.

When building an email capabilities app, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/email/IEmail.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/IEmailCreator.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/IModifyCreator.d.ts`

Key safety rules — always apply these:
- **Never** send emails unconditionally in a loop (e.g., inside an `IPostMessageSent` handler without strict rate-limiting or conditional checks).
- Understand that email capabilities rely entirely on the host Rocket.Chat instance having its SMTP settings correctly configured by the workspace administrator.
- Always use `modify.getCreator().getEmailCreator().send()` instead of attempting to bundle external SMTP libraries.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class SendEmailCommand implements ISlashCommand {
    public command = 'send-email';
    public i18nParamsExample = 'user@example.com';
    public i18nDescription = 'Sends a test email to the specified address';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp): Promise<void> {
        const [recipient] = context.getArguments();

        if (!recipient) {
            throw new Error('You must provide a recipient email address.');
        }

        // Use the native Apps-Engine EmailCreator to dispatch an email
        await modify.getCreator().getEmailCreator().send({
            to: recipient,
            subject: 'Notification from Rocket.Chat App',
            text: 'This is an automated email sent natively from a Rocket.Chat App using the IEmailCreator accessor.',
            html: '<p>This is an automated email sent natively from a Rocket.Chat App using the <b>IEmailCreator</b> accessor.</p>'
        });

        // Notify the user in the room that the email was dispatched
        const msgBuilder = modify.getCreator().startMessage()
            .setRoom(context.getRoom())
            .setSender(context.getSender())
            .setText(`Email dispatch requested for: ${recipient}`);

        await modify.getCreator().finish(msgBuilder);
    }
}
```

---

## 3. HTTP Outbound Requests (`rc-http-outbound`)

```yaml
---
name: rc-http-outbound
description: >
  Use when the user wants to fetch or send data to an external service.
  Phrases like "fetch from", "get data from", "connect to API",
  "pull from website", "call an external service",
  "get info from the internet", "look something up online",
  "retrieve data from", "integrate with", "pull live data",
  "get the current price of", "check the weather",
  "query an external database", "hit an API",
  "send data to an external service", "post to a webhook",
  "sync with" all indicate an HTTP outbound skill is needed.
---
```

You are an expert in Rocket.Chat Apps Engine HTTP outbounds.

When making outbound API requests, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/accessors/IHttp.d.ts`

Key safety rules — always apply these:
- **Never** use browser-native `fetch()` or `XMLHttpRequest`. You must use the Apps-Engine `IHttp` accessor passed to executors and handlers.
- Always handle HTTP timeouts or unsuccessful status codes (e.g., throwing a friendly error to the user if an API is down).
- Pay attention to `IHttpRequest` options. If you need to send JSON, set the `headers: { 'Content-Type': 'application/json' }` appropriately.
- If you expect non-UTF8 binary data, correctly utilize the `encoding` option.

### Implementation Example

```typescript
import { IHttp, IModify, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class HTTPRequestCommand implements ISlashCommand {
    public command = 'external-api';
    public i18nParamsExample = '';
    public i18nDescription = 'Sends an HTTP GET or POST Request to a mock API';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp): Promise<void> {
        
        // Example 1: Executing a GET Request
        const getUrl = 'https://jsonplaceholder.typicode.com/todos/1';
        const getResponse = await http.get(getUrl);
        
        if (getResponse.statusCode !== 200) {
            throw new Error(`Failed to fetch GET data. Status code: ${getResponse.statusCode}`);
        }

        // Example 2: Executing a POST Request
        const postUrl = 'https://jsonplaceholder.typicode.com/posts';
        const postResponse = await http.post(postUrl, {
            headers: {
                'Content-Type': 'application/json',
                'Authorization': 'Bearer YOUR_TOKEN_HERE' 
            },
            data: {
                title: 'foo',
                body: 'bar',
                userId: 1,
            }
            // `timeout`, `encoding`, and `strictSSL` are other available properties
        });

        if (postResponse.statusCode !== 201) {
             throw new Error(`Failed to POST data. Status code: ${postResponse.statusCode}`);
        }

        // Format and print back to the room
        const messageText = `GET Result:\n${JSON.stringify(getResponse.data, null, 2)}\n\nPOST Result:\n${JSON.stringify(postResponse.data, null, 2)}`;
        
        const messageStructure = modify.getCreator().startMessage()
            .setSender(context.getSender())
            .setRoom(context.getRoom())
            .setText(messageText);

        await modify.getCreator().finish(messageStructure);
    }
}
```

---

## 4. UI Block Kit (`rc-ui-block-kit`)

```yaml
---
name: rc-ui-block-kit
description: >
  Use when the user wants to trigger interactive UI flows. Phrases like 
  "show a form", "popup", "button that does", "collect input from user", 
  "show a dialog", "ask the user for information", "interactive message", 
  "clickable button", "dropdown menu", "let the user pick from a list", 
  "show a modal", "open a window", "display a card", "create a survey", 
  "show options to choose from", "let users fill out", "interactive UI", 
  "confirmation dialog", "form with fields" all indicate UI block kit is needed.
---
```

You are an expert in Rocket.Chat Apps Engine UI integrations and Block Kit.

When building interactive UI capabilities, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/uikit/IUIKitActionHandler.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/IUIController.d.ts`

Key safety rules — always apply these:
- **Mandatory Handler Requirement**: If you add interactive elements (like buttons or menus) to messages or modals, you MUST implement `IUIKitInteractionHandler` in your App. Without this, clicking buttons will freeze the UI with no feedback.
- Always use the new `openSurfaceView` method on `IUIController` rather than the deprecated `openModalView` method.
- You must always return a valid `IUIKitResponse` from `IUIKitInteractionHandler` execution methods, typically using `context.getInteractionResponder().successResponse()`.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { UIKitViewSubmitInteractionContext, UIKitBlockInteractionContext } from '@rocket.chat/apps-engine/definition/uikit';
import { IUIKitResponse, IUIKitInteractionHandler } from '@rocket.chat/apps-engine/definition/uikit';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class UIKitApp extends App implements IUIKitInteractionHandler {
    
    // Safety Rule Implementation: Handle the interactions from UI
    public async executeViewSubmitHandler(context: UIKitViewSubmitInteractionContext, read: IRead, http: IHttp, persistence: IPersistence, modify: IModify): Promise<IUIKitResponse> {
        const data = context.getInteractionData();
        const state = data.view.state; // Process form inputs here
        
        // Always return a responder status
        return context.getInteractionResponder().successResponse();
    }

    public async executeBlockActionHandler(context: UIKitBlockInteractionContext, read: IRead, http: IHttp, persistence: IPersistence, modify: IModify): Promise<IUIKitResponse> {
        return context.getInteractionResponder().successResponse();
    }
}

export class OpenModalCommand implements ISlashCommand {
    public command = 'open-modal';
    public i18nParamsExample = '';
    public i18nDescription = 'Opens an interactive modal window';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp): Promise<void> {
        const triggerId = context.getTriggerId();
        
        if (!triggerId) {
            throw new Error('Trigger ID missing');
        }

        // Define the Modal Surface
        const view = {
            id: 'my-modal-id',
            title: {
                type: 'plain_text' as const,
                text: 'My Interactive Modal'
            },
            submit: {
                type: 'plain_text' as const,
                text: 'Submit'
            },
            blocks: [
                {
                    type: 'input',
                    blockId: 'input_block',
                    label: { type: 'plain_text' as const, text: 'Enter something here:' },
                    element: {
                        type: 'plain_text_input',
                        actionId: 'input_action',
                        appId: 'my-app-id',
                    }
                }
            ]
        };

        // Open the Modal using the UI Controller
        await modify.getUiController().openSurfaceView(
            view, 
            { triggerId }, 
            context.getSender()
        );
    }
}
```

---

## 5. App Configuration (`rc-app-configuration`)

```yaml
---
name: rc-app-configuration
description: >
  Use when the user wants to add settings that workspace admins can configure.
  Phrases like "add a setting", "configurable value", "admin setting",
  "let the workspace admin change", "workspace configuration",
  "add an option to turn this on or off", "dynamic values",
  "make this editable in settings", "app preferences"
  all indicate app configuration skills are needed.
---
```

You are an expert in Rocket.Chat Apps Engine configuration and settings.

When building an app with configuration settings, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/settings/ISetting.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/IEnvironmentRead.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/ISettingRead.d.ts`

Key safety rules — always apply these:
- **Unique IDs**: Setting IDs must be unique across the entire App. Use an `enum` to maintain setting IDs centrally.
- Every setting must specify `public: true` if you want it to be exposed to the client or `public: false` if it should strictly stay server-side.
- To read a setting dynamically at execution time, you MUST use `read.getEnvironmentReader().getSettings().getValueById(AppSetting.YourSettingId)`. Avoid caching setting values globally, as the admin might change them mid-execution.

### Implementation Example

**1. Define Settings (e.g., in a `Settings.ts` file)**
```typescript
import { ISetting, SettingType } from '@rocket.chat/apps-engine/definition/settings';

export enum AppSetting {
    AppDemoOutputChannel = 'appdemo_outputchannel',
    AppDemoBoolean = 'appdemo_boolean'
}

export const settings: Array<ISetting> = [
    {
        id: AppSetting.AppDemoOutputChannel,
        section: "AppDemo_FeatureSettings",
        public: true, // Will be exposed to client
        type: SettingType.STRING,
        value: "#General",
        packageValue: "",
        hidden: false,
        i18nLabel: 'Output Channel',
        required: true,
    },
    {
        id: AppSetting.AppDemoBoolean,
        section: "AppDemo_FeatureSettings",
        public: false, // Server-side only
        type: SettingType.BOOLEAN,
        value: true,
        packageValue: "",
        hidden: false,
        i18nLabel: 'Enable Feature',
        required: false,
    }
];
```

**2. Feed Settings into Configuration (Main App class)**
```typescript
import { IConfigurationExtend, IEnvironmentRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { settings } from './Settings';

export class ConfigurationExampleApp extends App {
    public async extendConfiguration(configuration: IConfigurationExtend, environmentRead: IEnvironmentRead): Promise<void> {
        await Promise.all(
            settings.map((setting) => configuration.settings.provideSetting(setting))
        );
    }
}
```

**3. Read Built-in Settings at Runtime (e.g., inside an Executor)**
```typescript
import { IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { AppSetting } from './Settings';

async function checkFeatureEnabled(read: IRead): Promise<boolean> {
    // Correct way to read the exact, current value from the server Environment Reader
    const isEnabled = await read.getEnvironmentReader().getSettings().getValueById(AppSetting.AppDemoBoolean);
    
    return isEnabled as boolean;
}
```

---

## 6. Persistence (`rc-persistence`)

```yaml
---
name: rc-persistence
description: >
  Use when the user wants to store, save, or retrieve data across sessions.
  Phrases like "save data", "remember a user's choice", "store settings",
  "write to database", "save preferences", "keep a record of",
  "retrieve saved data", "persist information", "remember state",
  "save for later", "store history", "query past data"
  all indicate persistence/storage skills are needed.
---
```

You are an expert in Rocket.Chat Apps Engine state persistence.

When saving data or retrieving it from the database, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/accessors/IPersistence.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/accessors/IPersistenceRead.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/metadata/RocketChatAssociations.d.ts`

Key safety rules — always apply these:
- **Never store data in memory or global variables** as the Apps Engine aggressively isolates processes and restricts state management across instances.
- Always use `RocketChatAssociationRecord` arrays to link data to entities (like a specific user or room). Storing data *without* associations creates orphaned records that cannot be queried reliably.
- Use `IPersistence.updateByAssociations` with `upsert: true` instead of creating redundant records when modifying existing data. Use `IPersistenceRead.readByAssociations` to query it.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { RocketChatAssociationModel, RocketChatAssociationRecord } from '@rocket.chat/apps-engine/definition/metadata';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class PersistenceCommand implements ISlashCommand {
    public command = 'save-data';
    public i18nParamsExample = 'value';
    public i18nDescription = 'Saves and reads a parameter using Persistence Associations';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        const [value] = context.getArguments();
        const user = context.getSender();
        const room = context.getRoom();

        // SAFETY RULE: Create Association Records to link the data
        const association = new RocketChatAssociationRecord(RocketChatAssociationModel.USER, user.id);
        const roomAssociation = new RocketChatAssociationRecord(RocketChatAssociationModel.ROOM, room.id);

        if (value) {
            // Write to Database using Upsert (create or update)
            await persistence.updateByAssociations(
                [association, roomAssociation], // Array of associations acting as AND modifiers
                { storedValue: value, timestamp: new Date() },
                true // Upsert boolean
            );
        }

        // Read from Database
        const records = await read.getPersistenceReader().readByAssociations([association, roomAssociation]);
        
        let outputText = "No records found.";
        if (records.length > 0) {
            // Records are returned as an array of objects
            outputText = `Found records for you in this room: \n${JSON.stringify(records, null, 2)}`;
        }

        const msgBuilder = modify.getCreator().startMessage()
            .setRoom(room)
            .setSender(user)
            .setText(outputText);

        await modify.getCreator().finish(msgBuilder);
    }
}
```

---

## 7. OAuth2 Client (`rc-oauth2`) - (FOR FUTURE)

```yaml
---
name: rc-oauth2
description: >
  Use when the user wants to authenticate with a 3rd-party service via OAuth2.
  Phrases like "login with Google", "connect my Salesforce account",
  "authenticate via OAuth", "get a user's GitHub token", "sign in with",
  "link account", "authorize external service", "OAuth flow"
  all indicate the OAuth2 client is needed.
---
```

You are an expert in Rocket.Chat Apps Engine OAuth2 authentication.

When integrating OAuth2, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/oauth2/IOAuth2.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/oauth2/OAuth2.d.ts`

Key safety rules — always apply these:
- **Never hardcode Client IDs or Secrets**: The OAuth2 Client automatically parses these values from App Settings based on the `alias` you designate.
- After deploying, the admin must set `{alias}-oauth-client-id` and `{alias}-oauth-client-secret` in the App settings panel before OAuth will work.
- Store token logic safely inside the engine; the native client handles the entire callback and redirection pipeline automatically.
- Ensure that the callback endpoint logic explicitly captures the final token and associates it with the Rocket.Chat `IUser` via persistence.

### Implementation Example

```typescript
import { IConfigurationExtend, IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IAuthData, IOAuth2ClientOptions } from '@rocket.chat/apps-engine/definition/oauth2/IOAuth2';
import { createOAuth2Client } from '@rocket.chat/apps-engine/definition/oauth2/OAuth2';
import { IUser } from '@rocket.chat/apps-engine/definition/users';

export class OAuth2ExampleApp extends App {
    private oauth2Client;

    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        
        const oauth2Options: IOAuth2ClientOptions = {
            alias: 'github', // This automatically creates settings like `github-oauth-client-id`
            accessTokenUri: 'https://github.com/login/oauth/access_token',
            authUri: 'https://github.com/login/oauth/authorize',
            refreshTokenUri: 'https://github.com/login/oauth/access_token',
            revokeTokenUri: 'https://github.com/login/oauth/revoke',
            defaultScopes: ['read:user', 'repo'],
            
            // Callback executed by the engine when the user successfully authorizes
            authorizationCallback: async (token: IAuthData | undefined, user: IUser, read: IRead, modify: IModify, http: IHttp, persis: IPersistence) => {
                if (token) {
                    // Logic to store token specifically for this user goes here
                    const text = `Successfully authenticated! Token: ${token.token.substring(0, 5)}...`;
                    
                    const room = await read.getRoomReader().getByName('general');
                    if (room) {
                        const msg = modify.getCreator().startMessage().setRoom(room).setText(text).setSender(user);
                        await modify.getCreator().finish(msg);
                    }
                }
                
                return { responseContent: '<html><body>Auth successful! You can close this window.</body></html>' };
            }
        };

        // Create the client
        this.oauth2Client = createOAuth2Client(this, oauth2Options);
        
        // Setup internal App settings and webhook endpoints required for the callback
        await this.oauth2Client.setup(configuration);
    }
}
```

---

## 8. Scheduler API (`rc-scheduler`)

```yaml
---
name: rc-scheduler
description: >
  Use when the user wants to run tasks in the background or at specific times.
  Phrases like "run this every day", "schedule a message",
  "delay this action", "run something in 10 minutes",
  "recurring task", "background job", "execute later",
  "cron job", "run repeatedly", "wait before doing",
  "set a timer", "schedule an event"
  all indicate the scheduler API is needed.
---
```

You are an expert in Rocket.Chat Apps Engine background schedulers.

When configuring background or delayed jobs, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/scheduler/IProcessor.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/scheduler/IOnetimeSchedule.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/scheduler/IRecurringSchedule.d.ts`

Key safety rules — always apply these:
- **Processor Function Binding**: Never pass an arrow function or a bound context to the `processor` parameter in `registerProcessors`. It must be a direct reference to an `async` function (e.g., `this.myProcessor`).
- Processor functions are executed transparently in the background, which means they do NOT carry over context such as the current Room or current User. You must explicitly pass necessary identifiers using the `data` object to persist state across to the processor execution.

### Implementation Example

```typescript
import { IConfigurationExtend, IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IJobContext, StartupType } from '@rocket.chat/apps-engine/definition/scheduler';

export class SchedulerExampleApp extends App {
    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        
        // 1. Register the Processor mapping it to a method
        await configuration.scheduler.registerProcessors([
            {
                id: 'my_background_job',
                processor: this.myProcessorFunction, // SAFETY RULE: Pass method reference, not arrow function
                startupSetting: {
                    type: StartupType.RECURRING, // Alternatively StartupType.ONETIME
                    interval: '20 seconds', // Uses Agenda.js human-interval syntax
                    data: { initializedBy: 'system' }
                }
            }
        ]);
    }

    // 2. Define the Processor Execution Method
    private async myProcessorFunction(jobContext: IJobContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {     
        // Any parameters passed into the `data` object when scheduling the job reside here
        const data = jobContext; 

        // Safely retrieve bot user and log the action
        const bot = await read.getUserReader().getAppUser(this.getID());
        if (bot) {
            this.getLogger().info(`Task Executed with context: ${JSON.stringify(data)}`);
        }
    }

    // 3. (Optional) Example of scheduling Jobs dynamically instead of at startup
    // Generally triggered inside a slash command or event handler
    public async runDynamicSchedules(modify: IModify, customData: object) {
        
        // Recurring loop schedule
        await modify.getScheduler().scheduleRecurring({
            id: 'my_background_job',
            interval: '10 seconds',
            data: customData // Pass room IDs or user IDs through here
        });

        // One-time delayed schedule
        await modify.getScheduler().scheduleOnce({
            id: 'my_background_job',
            when: '10 minutes',
            data: customData
        });
        
        // Canceling a specific job
        await modify.getScheduler().cancelJob('my_background_job');
    }
}
```

---

## 9. API Endpoints (`rc-api-endpoint`)

```yaml
---
name: rc-api-endpoint
description: >
  Use when the user wants to create a webhook or an API that external services can call.
  Phrases like "create a webhook", "expose an endpoint", "listen for webhooks",
  "receive data from", "create a REST API", "add a custom route",
  "handle incoming requests", "let external services push data",
  "build an integration webhook" all indicate an API endpoint is needed.
---
```

You are an expert in Rocket.Chat Apps Engine incoming webhooks and custom API endpoints.

When building API endpoints, refer to the type definitions at:
`node_modules/@rocket.chat/apps-engine/definition/api/IApiEndpoint.d.ts`
`node_modules/@rocket.chat/apps-engine/definition/api/IApi.d.ts`

Key safety rules — always apply these:
- **Security Posture**: By default, `ApiSecurity.UNSECURE` is acceptable for local development or public payloads (like simple GitHub webhooks), but you should implement some form of token verification or instruct the user to consider `ApiSecurity.SECURED` for production applications preventing unauthorized data ingress.
- Do not build long-running synchronous code in the `post` or `get` methods otherwise the external webhook will timeout. Return a success response quickly, and consider handing off payloads to the scheduler if extensive processing is required.

### Implementation Example

```typescript
import { IConfigurationExtend, IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { ApiEndpoint, IApiEndpointInfo, IApiRequest, IApiResponse, ApiSecurity, ApiVisibility } from '@rocket.chat/apps-engine/definition/api';
import { App } from '@rocket.chat/apps-engine/definition/App';

// 1. Define the Endpoint
export class WebhookEndpoint extends ApiEndpoint {
    public path = 'webhook'; // Will result in /api/apps/public/<app-id>/webhook

    // Implement methods matching HTTP Verbs (get, post, put, delete)
    public async get(request: IApiRequest, endpoint: IApiEndpointInfo, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<IApiResponse> {
        return this.success(JSON.stringify({ status: "ok" }));
    }

    public async post(request: IApiRequest, endpoint: IApiEndpointInfo, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<IApiResponse> {
        
        // Ensure request content exists
        if (!request.content) {
            return this.success(JSON.stringify({ error: "No payload provided" }));
        }

        const bodyContent = JSON.stringify(request.content);
        
        // Example: Notify a specific room upon receiving the webhook
        const room = await read.getRoomReader().getByName('general');
        if (room) {
            const builder = modify.getCreator().startMessage()
                .setText(`Incoming Webhook payload: \n\`\`\`json\n${bodyContent}\n\`\`\``)
                .setRoom(room);
            await modify.getCreator().finish(builder);
        }

        // Must return a valid IApiResponse
        return this.success(JSON.stringify({ status: "ok", received: true }));
    }
}

// 2. Register the API Endpoint to the Application
export class WebhookApp extends App {
    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        await configuration.api.provideApi({
            visibility: ApiVisibility.PUBLIC, 
            security: ApiSecurity.UNSECURE, 
            endpoints: [
                new WebhookEndpoint(this)
            ],
        });
    }
}
```
