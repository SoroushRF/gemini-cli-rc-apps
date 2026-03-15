---
name: rc-api-endpoint
description: >
  Use when the user wants to receive data from external services via webhooks or create a custom REST API. Phrases like 
  "incoming webhook", "receive data from", "external service calls our app", 
  "create a REST endpoint", "expose an API", "listener for external events", 
  "handle a POST request from", "get a notification from GitHub/Stripe", 
  "callback URL", "public endpoint", "accept external data", "custom route", 
  "webhook integration" all indicate an API endpoint is needed.
---

You are an expert in Rocket.Chat Apps Engine API Endpoints and Webhooks.

When building API endpoints, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/api/IApi.d.ts
node_modules/@rocket.chat/apps-engine/definition/api/IApiEndpoint.d.ts
node_modules/@rocket.chat/apps-engine/definition/api/IApiRequest.d.ts
node_modules/@rocket.chat/apps-engine/definition/api/IApiResponse.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IRead.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IModify.d.ts

Key safety rules — always apply these:
- **Mandatory Registration**: All API endpoints must be registered within the `extendConfiguration` method during App startup using `configuration.api.provideApi({ visibility, security, endpoints })`.
- **Visibility awareness**: Use `ApiVisibility.PUBLIC` for webhooks that need to be reached from the internet. Use `ApiSecurity.UNSECURE` only if you implement your own authentication (like checking a secret token in the headers).
- **Endpoint Prefixing**: Remember that all custom endpoints are prefixed by Rocket.Chat as: `/api/apps/public/{app-id}/{path}`.
- **Input Validation**: Never trust incoming data. Always validate the payload (e.g., check for required fields in `request.content`) and the headers (e.g., verify a signature or API key).
- **Return Correct Status Codes**: Always return a valid `IApiResponse`. Use `return this.success()` for 200/OK, `return this.json({ status: 201 })` for creations, or `return this.badRequest()` for invalid inputs.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/api/IApiEndpoint` for all types.

### Implementation Example

```typescript
import { IConfigurationExtend } from '@rocket.chat/apps-engine/definition/accessors/IConfigurationExtend';
import { IHttp } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { IModify } from '@rocket.chat/apps-engine/definition/accessors/IModify';
import { IPersistence } from '@rocket.chat/apps-engine/definition/accessors/IPersistence';
import { IRead } from '@rocket.chat/apps-engine/definition/accessors/IRead';
import { ApiEndpoint } from '@rocket.chat/apps-engine/definition/api/IApiEndpoint';
import { IApiEndpointInfo } from '@rocket.chat/apps-engine/definition/api/IApiEndpointInfo';
import { IApiRequest } from '@rocket.chat/apps-engine/definition/api/IApiRequest';
import { IApiResponse } from '@rocket.chat/apps-engine/definition/api/IApiResponse';
import { ApiSecurity, ApiVisibility } from '@rocket.chat/apps-engine/definition/api/IApi';
import { App } from '@rocket.chat/apps-engine/definition/App';

export class WebhookApp extends App {
    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        // 1. Provide the API definition
        await configuration.api.provideApi({
            visibility: ApiVisibility.PUBLIC,
            security: ApiSecurity.UNSECURE,
            endpoints: [
                new IncomingWebhookEndpoint(this),
            ],
        });
    }
}

// 2. Define the Endpoint class
export class IncomingWebhookEndpoint extends ApiEndpoint {
    public path = 'incoming-data';

    // Handle POST requests (e.g., from an external service)
    public async post(request: IApiRequest, endpoint: IApiEndpointInfo, read: IRead, modify: IModify, http: IHttp, persis: IPersistence): Promise<IApiResponse> {
        // persistence included for optional future state tracking
        
        const payload = request.content;

        // SAFETY RULE: Validate incoming data
        if (!payload || !payload.text) {
            return this.json({
                status: 400,
                content: { error: 'Payload must contain a "text" field' },
            });
        }

        // Action: Post a message to a specific room when webhook is hit
        const roomId = 'GENERAL'; // In production, read this from a setting
        const room = await read.getRoomReader().getById(roomId);

        if (room) {
            const builder = modify.getCreator().startMessage()
                .setRoom(room)
                .setText(`Incoming Webhook Data: ${payload.text}`);
            await modify.getCreator().finish(builder);
        }

        return this.success();
    }

    public async get(request: IApiRequest, endpoint: IApiEndpointInfo, read: IRead, modify: IModify, http: IHttp, persis: IPersistence): Promise<IApiResponse> {
        return this.success();
    }
}
```
