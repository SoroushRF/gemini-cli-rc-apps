---
name: rc-app-configuration
description: >
  Use when the user wants to let admins configure the app. Phrases like
  "let admin configure", "settings page", "customizable", "admin panel",
  "configurable option", "let the server admin set", "admin should be able to change",
  "workspace setting", "toggle a feature on or off", "allow configuration of",
  "set a default value for", "admin-controlled", "configurable by the server owner",
  "let admins input their API key", "environment-specific setting"
  all indicate app configuration settings are needed.
---

You are an expert in Rocket.Chat Apps Engine configuration and settings.

When building app settings, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/settings/ISetting.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IConfigurationExtend.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/ISettingRead.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IEnvironmentRead.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IRead.d.ts
node_modules/@rocket.chat/apps-engine/definition/accessors/IModify.d.ts
node_modules/@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand.d.ts

Key safety rules — always apply these:
- **Unique IDs**: Use a strictly unique ID for each setting, preferably defined in an Enum to avoid string-key collisions across the app.
- **Privacy Awareness**: Use `public: true` only for settings that need to be accessible by the client (web/mobile). Keep API keys and secrets as `public: false` (the default) to ensure they are only accessible on the server.
- **Dynamic Retrieval**: Never cache setting values in global variables. Always read them at runtime using `read.getEnvironmentReader().getSettings().getValueById('SETTING_ID')` to ensure the app reacts to administrator changes immediately.
- **Configuration Lifecycle**: Settings must be registered during the `extendConfiguration` boot hook using `configuration.settings.provideSetting()`.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/settings/ISetting` for all types.

### Implementation Example

```typescript
import { IConfigurationExtend } from '@rocket.chat/apps-engine/definition/accessors/IConfigurationExtend';
import { IHttp } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { IModify } from '@rocket.chat/apps-engine/definition/accessors/IModify';
import { IPersistence } from '@rocket.chat/apps-engine/definition/accessors/IPersistence';
import { IRead } from '@rocket.chat/apps-engine/definition/accessors/IRead';
import { ISetting, SettingType } from '@rocket.chat/apps-engine/definition/settings/ISetting';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand';

// 1. Define Setting IDs in an Enum for safety
export enum AppSetting {
    MySetting = 'my_setting_id',
    ApiKey = 'api_key_id'
}

export class ConfigApp extends App {
    // 2. Register Settings during the extendConfiguration phase
    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        await configuration.settings.provideSetting({
            id: AppSetting.MySetting,
            type: SettingType.STRING,
            packageValue: 'default_value', // Default value at install time
            // value field is managed by RC at runtime — do not set manually
            required: true,
            public: true, // Visible to clients
            i18nLabel: 'My Setting Label',
            i18nDescription: 'Description for my setting',
        });

        await configuration.settings.provideSetting({
            id: AppSetting.ApiKey,
            type: SettingType.STRING,
            packageValue: '',
            required: false,
            public: false, // Secret (Server-only)
            i18nLabel: 'External API Key',
        });
    }
}

export class CheckConfigCommand implements ISlashCommand {
    public command = 'check-config';
    public i18nParamsExample = '';
    public i18nDescription = 'Reads the current app configuration';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        // persistence included for optional future state tracking
        
        // 3. Read settings at runtime using the environment reader
        const mySettingValue = await read.getEnvironmentReader().getSettings().getValueById(AppSetting.MySetting);
        const apiKey = await read.getEnvironmentReader().getSettings().getValueById(AppSetting.ApiKey);

        const text = `Current Config:\n - Setting: ${mySettingValue}\n - API Key Set: ${apiKey ? 'Yes' : 'No'}`;
        
        const builder = modify.getCreator().startMessage()
            .setSender(context.getSender())
            .setRoom(context.getRoom())
            .setText(text);
        await modify.getCreator().finish(builder);
    }
}
```
