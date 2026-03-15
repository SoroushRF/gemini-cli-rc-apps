---
name: rc-email
description: >
  Use when the user wants to send an email. Phrases like "send an email",
  "email me when", "notify via email", "email notification",
  "send a message to someone's inbox", "alert me by email",
  "email the team when", "send an email report", "forward to email",
  "email someone outside of Rocket.Chat", "send a daily email summary",
  "notify externally"
  all indicate email capabilities are needed.
---

You are an expert in Rocket.Chat Apps Engine email communications.

When building an app that sends emails, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/email/IEmail.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IEmailCreator.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IModifyCreator.d.ts

Key safety rules — always apply these:
- **No Native Modules**: Never use `nodemailer` or internal SMTP configurations. The Apps Engine proxies emails exclusively through the workspace's pre-configured SMTP via `IEmailCreator`.
- **Infinite Loops**: Do not send unconditional emails based on highly frequent user activities (e.g., every message sent) to avoid spamming users and getting the workspace blacklisted.
- **Method Chain**: Emails must be strictly dispatched via the engine's built-in accessor chain: `modify.getCreator().getEmailCreator().send(email)`.
- **Object Format**: The payload must be correctly shaped into the `IEmail` interface containing `to`, `from`, `subject`, and either `text` or `html`.
- **Domain Verification**: Ensure you rely on a valid "from" address mapped to the active workspace domains, otherwise the workspace's SMTP server may reject the payload.

### Implementation Example

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IEmail } from '@rocket.chat/apps-engine/definition/email/IEmail';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class EmailCommand implements ISlashCommand {
    public command = 'send-email';
    public i18nParamsExample = 'user@example.com';
    public i18nDescription = 'Sends an email to the specified address';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        // persistence included for optional future state tracking
        const [targetEmail] = context.getArguments();
        const user = context.getSender();
        const room = context.getRoom();

        if (!targetEmail) {
            // Notify user of missing argument safely
            const builder = modify.getCreator().startMessage().setSender(user).setRoom(room).setText("Please provide an email address.");
            await modify.getCreator().finish(builder);
            return;
        }

        // 1. Construct the IEmail object
        const emailPayload: IEmail = {
            to: targetEmail,
            from: 'bot@your-workspace-domain.com', // Must be a valid SMTP authorized address
            subject: 'Notification from Rocket.Chat',
            text: `Hello,\n\nThis is an automated notification triggered by ${user.username}.`,
            html: `<p>Hello,</p><p>This is an automated notification triggered by <strong>${user.username}</strong>.</p>`
        };

        try {
            // 2. Dispatch using the IEmailCreator via the IModify chain
            await modify.getCreator().getEmailCreator().send(emailPayload);
            
            const builder = modify.getCreator().startMessage().setSender(user).setRoom(room).setText(`Successfully sent email to ${targetEmail}`);
            await modify.getCreator().finish(builder);
        } catch (error) {
            // Document error gracefully if SMTP fails
            const builder = modify.getCreator().startMessage().setSender(user).setRoom(room).setText(`Failed to send email: ${error.message}`);
            await modify.getCreator().finish(builder);
        }
    }
}
```
