---
name: rc-persistence
description: >
  Use when the user wants to save, remember, or keep track of data. Phrases like 
  "remember", "save", "store", "keep track of", "don't forget", "persist between sessions", 
  "keep a record of", "maintain state", "store user preferences", "save progress", 
  "remember what the user said", "track how many times", "keep a count of", 
  "log activity", "save data per user", "store per room", "retain information after restart", 
  "database", "keep history of" all indicate persistence is needed.
---

You are an expert in Rocket.Chat Apps Engine Persistence.

When building features that store data, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/accessors/IPersistence.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IPersistenceRead.d.ts
node_modules/@rocket.chat/apps-engine/definition/metadata/RocketChatAssociations.d.ts

Key safety rules — always apply these:
- **No Local Memory variables**: Never use global variables (e.g., `let cache = {}`) to store state. Rocket.Chat sandboxes can reset or move between threads; your data will be lost. You MUST use the `persistence` accessor.
- **Always Use Associations**: Never store data "blindly". Use `RocketChatAssociationRecord` to link data to a `USER`, `ROOM`, or `MISC` category. This prevents "orphaned" data that cannot be queried later.
- **Upsert by Association**: Use `persistence.updateByAssociations(associations, data, true)` for an "upsert" pattern. The third parameter `true` ensures the record is created if it doesn't exist, preventing duplicates.
- **Async Operations**: All persistence methods (both reading and writing) are asynchronous. Always use `await` when interacting with the database.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/accessors/IPersistence` for all types.

### Implementation Example

```typescript
import { IHttp } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { IModify } from '@rocket.chat/apps-engine/definition/accessors/IModify';
import { IPersistence } from '@rocket.chat/apps-engine/definition/accessors/IPersistence';
import { IRead } from '@rocket.chat/apps-engine/definition/accessors/IRead';
import { RocketChatAssociationModel, RocketChatAssociationRecord } from '@rocket.chat/apps-engine/definition/metadata/RocketChatAssociations';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand';

export class PersistenceCommand implements ISlashCommand {
    public command = 'save-data';
    public i18nParamsExample = 'my_value';
    public i18nDescription = 'Saves a piece of data associated with your user';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        const [dataToSave] = context.getArguments();
        const user = context.getSender();

        if (!dataToSave) {
            return;
        }

        // 1. Define the Association (e.g., link this record to the current user)
        const association = new RocketChatAssociationRecord(RocketChatAssociationModel.USER, user.id);
        
        // TIP: To scope data to both a user AND a room, use multiple associations:
        // const roomAssociation = new RocketChatAssociationRecord(RocketChatAssociationModel.ROOM, context.getRoom().id);
        // await persistence.updateByAssociations([association, roomAssociation], { ... }, true);

        // 2. Save the data (using Upsert pattern)
        interface IStoredData {
            savedAt: Date;
            content: string;
        }
        
        await persistence.updateByAssociations([association], { savedAt: new Date(), content: dataToSave }, true);

        // 3. Read the data back immediately for verification
        const results = await read.getPersistenceReader().readByAssociations([association]);
        const savedData = results.length > 0 ? (results[0] as IStoredData).content : 'none';

        const builder = modify.getCreator().startMessage()
            .setSender(user)
            .setRoom(context.getRoom())
            .setText(`Successfully stored. Current value for you is: ${savedData}`);
            
        await modify.getCreator().finish(builder);
    }
}
```
