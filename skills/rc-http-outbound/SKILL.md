---
name: rc-http-outbound
description: >
  Use when the user wants to connect to an external service or website.
  Phrases like "fetch from", "get data from", "connect to API", "pull from website",
  "call an external service", "get info from the internet", "look something up online",
  "retrieve data from", "integrate with", "pull live data", "get the current price of",
  "check the weather", "query an external database", "hit an API",
  "send data to an external service", "post to a webhook", "sync with"
  all indicate outbound HTTP requests are needed.
---

You are an expert in Rocket.Chat Apps Engine outbound HTTP communication.

When building an app that makes external requests, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/accessors/IHttp.d.ts

Key safety rules — always apply these:
- **No Native fetch() or axios**: Never use native `fetch()`, `axios`, or `XMLHttpRequest`. You MUST exclusively use the `IHttp` accessor provided by the engine.
- **Data Property for POST**: When sending data in a POST request, use the `data` property inside the `IHttpRequest` options. Do NOT use `body`.
- **Strict JSON Headers**: When sending JSON payloads, you must explicitly set the `Content-Type` header to `application/json`.
- **Explicit Error Handling**: Always check `response.statusCode`. Outbound HTTP requests in the Apps Engine do NOT throw runtime errors for 4xx or 5xx status codes; you must handle them manually.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/accessors/IHttp` for all types.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors/index';
import { IHttpResponse } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand';
import { IUser } from '@rocket.chat/apps-engine/definition/users/IUser';
import { IRoom } from '@rocket.chat/apps-engine/definition/rooms/IRoom';

export class HttpCommand implements ISlashCommand {
    public command = 'fetch-data';
    public i18nParamsExample = '';
    public i18nDescription = 'Fetches data from an external API';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        // persistence included for optional future state tracking
        const user = context.getSender();
        const room = context.getRoom();

        const url = 'https://api.example.com/data';

        // 1. Performing a GET request
        const getResponse: IHttpResponse = await http.get(url);

        if (getResponse.statusCode !== 200) {
            await this.notify(user, room, modify, `Failed to fetch data. Status: ${getResponse.statusCode}`);
            return;
        }

        // 2. Performing a POST request with JSON data
        const postUrl = 'https://api.example.com/submit';
        const postResponse: IHttpResponse = await http.post(postUrl, {
            headers: {
                'Content-Type': 'application/json',
            },
            data: {
                user: user.username,
                timestamp: new Date().toISOString(),
                content: getResponse.content, // Forwarding fetched data
            },
        });

        if (postResponse.statusCode === 201 || postResponse.statusCode === 200) {
            await this.notify(user, room, modify, 'Successfully synchronized data with external service.');
        } else {
            await this.notify(user, room, modify, `Sync failed with status: ${postResponse.statusCode}`);
        }
    }

    private async notify(user: IUser, room: IRoom, modify: IModify, text: string): Promise<void> {
        const builder = modify.getCreator().startMessage()
            .setSender(user)
            .setRoom(room)
            .setText(text);
        await modify.getCreator().finish(builder);
    }
}
```
