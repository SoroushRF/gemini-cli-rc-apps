---
name: rc-scheduler
description: >
  Use when the user wants to run tasks at specific times or intervals. Phrases like 
  "every day", "scheduled", "recurring", "remind me every", "runs at midnight", 
  "once a week", "every hour", "daily report", "periodic task", "run automatically at", 
  "cron job", "timed task", "send a summary every morning", "check every 10 minutes", 
  "run in the background", "delayed execution", "fire after X seconds", 
  "run once at a specific time", "scheduled notification"
  all indicate a scheduler is needed.
---

You are an expert in Rocket.Chat Apps Engine background jobs and the Scheduler API.

When building scheduled tasks, refer to the type definitions at:
node_modules/@rocket.chat/apps-engine/definition/scheduler/IProcessor.d.ts
node_modules/@rocket.chat/apps-engine/definition/scheduler/IOnetimeSchedule.d.ts
node_modules/@rocket.chat/apps-engine/definition/scheduler/IRecurringSchedule.d.ts
node_modules/@rocket.chat/apps-engine/definition/scheduler/IJobContext.d.ts

Key safety rules — always apply these:
- **Processor Function Scoping**: Never pass an arrow function (e.g., `() => { ... }`) to the `processor` field. It MUST be an async method reference defined on your App class (e.g., `this.myProcessorFunction`).
- **Context Isolation**: Background jobs do not have access to the original user or room context of a slash command or UI interaction. You MUST pass any required IDs (like `roomId` or `userId`) explicitly inside the `data` object when scheduling.
- **Human-Readable Intervals**: Use human-readable strings for intervals and delays (e.g., '20 seconds', '10 minutes', 'tomorrow at noon').
- **Mandatory Registration**: All background job processors must be registered within the `extendConfiguration` method during App startup.
- **Specific Import Paths**: Avoid barrel imports. Use direct paths like `@rocket.chat/apps-engine/definition/scheduler/IProcessor` for all types.

### Implementation Example

```typescript
import { IConfigurationExtend } from '@rocket.chat/apps-engine/definition/accessors/IConfigurationExtend';
import { IHttp } from '@rocket.chat/apps-engine/definition/accessors/IHttp';
import { IModify } from '@rocket.chat/apps-engine/definition/accessors/IModify';
import { IPersistence } from '@rocket.chat/apps-engine/definition/accessors/IPersistence';
import { IRead } from '@rocket.chat/apps-engine/definition/accessors/IRead';
import { IJobContext } from '@rocket.chat/apps-engine/definition/scheduler/IJobContext';
import { App } from '@rocket.chat/apps-engine/definition/App';
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands/ISlashCommand';
import { StartupType } from '@rocket.chat/apps-engine/definition/scheduler/IProcessor';

// 1. Define an interface for the job data
interface IMyJobData {
    message: string;
    roomId?: string;
}

export class SchedulerApp extends App {
    public async extendConfiguration(configuration: IConfigurationExtend): Promise<void> {
        // 2. Register the Processor during startup
        await configuration.scheduler.registerProcessors([
            {
                id: 'my_background_job',
                processor: this.myProcessorFunction, // SAFETY RULE: Method reference, not arrow function
                startupSetting: {
                    type: StartupType.RECURRING,
                    interval: '1 hour', // Runs every hour starting from boot
                    data: { message: 'Hourly sync started' } as IMyJobData,
                },
            },
        ]);
    }

    // 3. Define the Processor Method
    public async myProcessorFunction(jobContext: IJobContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        const data = jobContext as IMyJobData;
        console.log(data.message);

        if (data.roomId) {
            const room = await read.getRoomReader().getById(data.roomId);
            if (room) {
                const builder = modify.getCreator().startMessage().setRoom(room).setText(`Scheduled task: ${data.message}`);
                await modify.getCreator().finish(builder);
            }
        }
    }
}

export class ScheduleTriggerCommand implements ISlashCommand {
    public command = 'trigger-job';
    public i18nParamsExample = '';
    public i18nDescription = 'Schedules a one-time job in 5 minutes';
    public providesPreview = false;

    public async executor(context: SlashCommandContext, read: IRead, modify: IModify, http: IHttp, persistence: IPersistence): Promise<void> {
        // persistence included for optional future state tracking
        
        // 3. Dynamically schedule a one-time job
        await modify.getScheduler().scheduleOnce({
            id: 'my_background_job',
            when: '5 minutes',
            data: { 
                message: 'Delayed notification',
                roomId: context.getRoom().id, // SAFETY RULE: Pass IDs explicitly
            },
        });

        const builder = modify.getCreator().startMessage()
            .setSender(context.getSender())
            .setRoom(context.getRoom())
            .setText('Job scheduled successfully for 5 minutes from now.');
        await modify.getCreator().finish(builder);
    }
}
```
