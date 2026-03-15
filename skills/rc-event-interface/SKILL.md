---
name: rc-event-interface
description: >
  Use when the user wants to trigger actions automatically. Phrases like
  "when someone joins", "when someone leaves", "automatically reply when",
  "trigger when a message is sent", "react to messages", "do something when a user joins the room",
  "fire when a room is created", "listen for messages", "intercept messages before they're sent",
  "run something automatically", "when a new user registers", "detect when someone types",
  "respond automatically to", "monitor messages", "watch for activity"
  all indicate an event interface is needed.
---

You are an expert in Rocket.Chat Apps Engine event interfaces.

When building an app that reacts to events, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/messages/IPreMessageSentPrevent.d.ts
node_modules/@rocket.chat/apps-engine/definition/messages/IPostMessageSent.d.ts
node_modules/@rocket.chat/apps-engine/definition/rooms/IPreRoomUserJoined.d.ts
node_modules/@rocket.chat/apps-engine/definition/users/IPostUserCreated.d.ts

Key safety rules — always apply these:
- **Mandatory Sender Check**: When implementing `IPostMessageSent`, you MUST check if the message sender is the bot itself. If you do not do this `if (message.sender.id === botUser.id) return;`, your bot will trigger its own events resulting in catastrophic infinite loops.
- **IPre vs IPost**: Understand that `IPre` (Prevent/Modify/Extend) events fire *before* database insertion allowing you to block the payload. `IPost` events happen *after* the fact and are entirely read-only.
- **Boolean Return (IPre)**: When executing `IPreMessageSentPrevent`, explicitly returning `true` PREVENTS the message. Returning `false` allows it to pass through. Do not blindly `return true` unless you genuinely intend to silence the message.
- Always implement the event interface directly on the main App class.
- Dynamically fetch the App's bot user using `read.getUserReader().getAppUser(this.getID())` for your verification checks.
- When implementing room or user event handlers (`IPostRoomUserJoined`, `IPostUserCreated`), always verify the event's room or user object exists before accessing its properties — these can be undefined in edge cases like system-generated events.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IMessage } from '@rocket.chat/apps-engine/definition/messages/IMessage';
import { IPreMessageSentPrevent } from '@rocket.chat/apps-engine/definition/messages/IPreMessageSentPrevent';
import { IPostMessageSent } from '@rocket.chat/apps-engine/definition/messages/IPostMessageSent';

export class EventApp extends App implements IPreMessageSentPrevent, IPostMessageSent {
    
    // IPre Handler: Decides whether to intercept
    public async checkPreMessageSentPrevent(message: IMessage, read: IRead, http: IHttp): Promise<boolean> {
        // Run intercept logic only if message text strictly matches "blockme"
        return message.text === 'blockme'; 
    }

    // IPre Handler: Executes the prevention
    public async executePreMessageSentPrevent(message: IMessage, read: IRead, http: IHttp, persistence: IPersistence): Promise<boolean> {
        // Returning true prevents the message from being sent
        // Returning false allows the message through
        return true; 
    }

    // IPost Handler: Decides whether to act AFTER message was sent natively
    public async checkPostMessageSent(message: IMessage, read: IRead, http: IHttp): Promise<boolean> {
        return !!message.text; // Ensure text exists
    }

    // IPost Handler: Executes actions (e.g. Reply)
    public async executePostMessageSent(message: IMessage, read: IRead, http: IHttp, persistence: IPersistence, modify: IModify): Promise<void> {
        const botUser = await read.getUserReader().getAppUser(this.getID());
        
        if (!botUser) {
            return;
        }

        // SAFETY RULE: Prevent infinite bot loops
        if (message.sender.id === botUser.id) {
            return;
        }

        const room = message.room;

        // Automatically reply to messages
        if (message.text === 'ping') {
            const builder = modify.getCreator().startMessage()
                .setSender(botUser)
                .setRoom(room)
                .setText('pong');
                
            await modify.getCreator().finish(builder);
        }
    }
}
```
