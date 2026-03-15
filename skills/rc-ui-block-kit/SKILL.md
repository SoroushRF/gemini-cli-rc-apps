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

You are an expert in Rocket.Chat Apps Engine UI integrations and Block Kit.

When building interactive UI capabilities, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/uikit/IUIKitActionHandler.d.ts
node_modules/@rocket.chat/apps-engine/definition/uikit/UIKitInteractionContext.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IUIController.d.ts
node_modules/@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand.d.ts

Key safety rules — always apply these:
- **Mandatory Handler Requirement**: If you add interactive elements (like buttons or menus) to messages or modals, you MUST implement `IUIKitInteractionHandler` in your App. Without this, clicking buttons will freeze the UI with no feedback.
- **Use openSurfaceView**: Always use the modern `openSurfaceView` method on `IUIController` rather than the deprecated `openModalView` method. Failing to do this will result in broken UI on newer Rocket.Chat versions.
- **Valid Interaction Response**: You must always return a valid `IUIKitResponse` from `IUIKitInteractionHandler` execution methods, typically using `context.getInteractionResponder().successResponse()`.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/uikit/IUIKitInteractionHandler` to ensure bulletproof code generation.
- **Trigger ID**: Opening a modal or surface requires a valid `triggerId` from the interaction context or slash command context. Ensure it is passed correctly.

### Implementation Example

```typescript
import { IHttp } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { IModify } from '@rocket.chat/apps-engine/definition/accessors/IModify';
import { IPersistence } from '@rocket.chat/apps-engine/definition/accessors/IPersistence';
import { IRead } from '@rocket.chat/apps-engine/definition/accessors/IRead';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { UIKitViewSubmitInteractionContext, UIKitBlockInteractionContext } from '@rocket.chat/apps-engine/definition/uikit/UIKitInteractionContext';
import { IUIKitResponse, IUIKitInteractionHandler } from '@rocket.chat/apps-engine/definition/uikit/IUIKitActionHandler';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand';
import { IUser } from '@rocket.chat/apps-engine/definition/users/IUser';

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

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        // persistence included for optional future state tracking
        const triggerId = context.getTriggerId();
        const user: IUser = context.getSender();
        
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

        // Open the Modal using the UI Controller (openSurfaceView)
        await modify.getUiController().openSurfaceView(
            view, 
            { triggerId }, 
            user
        );
    }
}
```
