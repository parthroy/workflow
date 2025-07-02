# Activepieces: Trigger Events System Documentation

## Overview

The Trigger Events system in Activepieces is a critical component that manages the capture, storage, and processing of incoming events from various sources that initiate workflow executions. This documentation covers the database structure, core components, and operational flow of this system.

## Database Structure

### `TriggerEventEntity` (Core Data Model)

The primary entity that stores information about webhook events and polling trigger results:

```typescript
// packages/server/api/src/app/flows/trigger-events/trigger-event.entity.ts
export const TriggerEventEntity = new EntitySchema<TriggerEventSchema>({
    name: 'trigger_event',
    columns: {
        ...BaseColumnSchemaPart,  // id, created, updated timestamps
        flowId: ApIdSchema,       // Reference to the flow this event belongs to
        projectId: ApIdSchema,    // Project context for multi-tenancy
        sourceName: {             // Name of the trigger source (webhook, polling, etc.)
            type: String,
        },
        fileId: {                 // Reference to the file containing the payload data
            type: String,
        },
    },
    // Database indexes for performance optimization
    indices: [
        {
            name: 'idx_trigger_event_project_id_flow_id',
            columns: ['projectId', 'flowId'],
            unique: false,
        },
        {
            name: 'idx_trigger_event_flow_id',
            columns: ['flowId'],
            unique: false,
        },
        {
            name: 'idx_trigger_event_file_id',
            columns: ['fileId'],
            unique: true,
        },
    ],
    // Entity relationships
    relations: {
        project: {
            type: 'many-to-one',
            target: 'project',
            cascade: true,
            onDelete: 'CASCADE',
            joinColumn: {
                name: 'projectId',
                foreignKeyConstraintName: 'fk_trigger_event_project_id',
            },
        },
        file: {
            type: 'many-to-one',
            target: 'file',
            cascade: true,
            onDelete: 'CASCADE',
            joinColumn: {
                name: 'fileId',
                foreignKeyConstraintName: 'fk_trigger_event_file_id',
            },
        },
        flow: {
            type: 'many-to-one',
            target: 'flow',
            cascade: true,
            onDelete: 'CASCADE',
            joinColumn: {
                name: 'flowId',
                foreignKeyConstraintName: 'fk_trigger_event_flow_id',
            },
        },
    },
})
```

## Trigger Event Service

The `TriggerEventService` handles operations related to trigger events:

### Key Methods

#### 1. `saveEvent` - Store New Events

```typescript
// packages/server/api/src/app/flows/trigger-events/trigger-event.service.ts
async saveEvent({
    projectId,
    flowId,
    payload,
}: SaveEventParams): Promise<TriggerEventWithPayload> {
    // Fetch the flow to verify it exists and get trigger information
    const flow = await flowService(log).getOnePopulatedOrThrow({
        id: flowId,
        projectId,
    })

    // Store the payload data in a file
    const data = Buffer.from(JSON.stringify(payload))
    const file = await fileService(log).save({
        projectId,
        fileName: `${apId()}.json`,
        data,
        size: data.length,
        type: FileType.TRIGGER_EVENT_FILE,
        compression: FileCompression.NONE,
    })
    
    // Determine the source name (trigger type)
    const sourceName = getSourceName(flow.version.trigger)

    // Create the trigger event record in the database
    const trigger = await triggerEventRepo().save({
        id: apId(),
        fileId: file.id,
        projectId,
        flowId: flow.id,
        sourceName,
    })
    
    // Return the trigger event with its payload
    return {
        ...trigger,
        payload,
    }
}
```

#### 2. `test` - Test Trigger Functionality

```typescript
// packages/server/api/src/app/flows/trigger-events/trigger-event.service.ts
async test({
    projectId,
    flow,
}: TestParams): Promise<SeekPage<TriggerEventWithPayload>> {
    const trigger = flow.version.trigger
    const emptyPage = paginationHelper.createPage<TriggerEventWithPayload>([], null)
    
    switch (trigger.type) {
        case TriggerType.PIECE: {
            // Execute trigger test hook to get sample data
            const engineResponse = await userInteractionWatcher(log).submitAndWaitForResponse<EngineHelperResponse<EngineHelperTriggerResult<TriggerHookType.TEST>>>({
                hookType: TriggerHookType.TEST,
                flowVersion: flow.version,
                test: true,
                projectId,
                jobType: UserInteractionJobType.EXECUTE_TRIGGER_HOOK,
            })
            
            // Clear existing test events
            await triggerEventRepo().delete({
                projectId,
                flowId: flow.id,
            })
            
            // Handle test errors
            if (!engineResponse.result.success) {
                throw new ActivepiecesError({
                    code: ErrorCode.TEST_TRIGGER_FAILED,
                    params: {
                        message: engineResponse.result.message!,
                    },
                })
            }

            // Save sample data as trigger events
            for (const output of engineResponse.result.output) {
                await this.saveEvent({
                    projectId,
                    flowId: flow.id,
                    payload: output,
                })
            }

            // Return the list of created events
            return this.list({
                projectId,
                flow,
                cursor: null,
                limit: engineResponse.result.output.length,
            })
        }
        case TriggerType.EMPTY:
            return emptyPage
    }
}
```

#### 3. `list` - Retrieve Trigger Events

```typescript
// packages/server/api/src/app/flows/trigger-events/trigger-event.service.ts
async list({
    projectId,
    flow,
    cursor,
    limit,
}: ListParams): Promise<SeekPage<TriggerEventWithPayload>> {
    const decodedCursor = paginationHelper.decodeCursor(cursor)
    const sourceName = getSourceName(flow.version.trigger)
    const flowId = flow.id
    
    // Set up pagination
    const paginator = buildPaginator({
        entity: TriggerEventEntity,
        query: {
            limit,
            order: Order.DESC,
            afterCursor: decodedCursor.nextCursor,
            beforeCursor: decodedCursor.previousCursor,
        },
    })
    
    // Build and execute query
    const query = triggerEventRepo().createQueryBuilder('trigger_event').where({
        projectId,
        flowId,
        sourceName,
    })
    
    const { data, cursor: newCursor } = await paginator.paginate(query)
    
    // Load payload data for each event
    const dataWithPayload = await Promise.all(data.map(async (triggerEvent) => {
        const fileData = await fileService(log).getDataOrThrow({
            fileId: triggerEvent.fileId,
        })
        const decodedPayload = JSON.parse(fileData.data.toString())
        return {
            ...triggerEvent,
            payload: decodedPayload,
        }
    }))
    
    // Return paginated results
    return paginationHelper.createPage<TriggerEventWithPayload>(dataWithPayload, newCursor)
}
```

## Trigger Type Implementation

### 1. Webhook Triggers

Webhook triggers allow external systems to initiate flows by sending HTTP requests to Activepieces:

```typescript
// Example webhook trigger implementation
export const newWebhookTrigger = createTrigger({
    name: 'webhook_trigger',
    displayName: 'New Webhook',
    type: TriggerStrategy.WEBHOOK,
    
    // Setup webhook when trigger is enabled
    async onEnable(context) {
        // Setup code for the webhook - often registers with external service
        const webhookId = await externalService.registerWebhook(
            context.auth,
            context.webhookUrl,
            EVENT_TYPE
        );

        // Store webhook ID for cleanup during disable
        await context.store?.put('webhook_trigger_info', {
            webhookId: webhookId,
        });
    },
    
    // Cleanup when trigger is disabled
    async onDisable(context) {
        const webhookInfo = await context.store?.get('webhook_trigger_info');
        if (webhookInfo) {
            await externalService.unregisterWebhook(
                context.auth, 
                webhookInfo.webhookId
            );
        }
    },
    
    // Process incoming webhook data
    async run(context) {
        // Extract and transform data from the webhook payload
        return [context.payload.body];
    }
});
```

### 2. Polling Triggers

Polling triggers periodically check for new data:

```typescript
// Example polling trigger implementation
export const pollingTrigger = createTrigger({
    name: 'polling_trigger',
    displayName: 'Poll for New Data',
    type: TriggerStrategy.POLLING,
    
    // Set up initial state when trigger is enabled
    async onEnable(context) {
        await pollingHelper.onEnable(polling, {
            auth: context.auth,
            store: context.store,
            propsValue: context.propsValue,
        });
    },
    
    // Clean up when trigger is disabled
    async onDisable(context) {
        await pollingHelper.onDisable(polling, {
            auth: context.auth,
            store: context.store,
            propsValue: context.propsValue,
        });
    },
    
    // Poll for new data
    async run(context) {
        return await pollingHelper.poll(polling, context);
    }
});

// Polling configuration
const polling = {
    strategy: DedupeStrategy.TIMEBASED,
    async items({ auth }) {
        // Fetch new items from external service
        const items = await externalService.getItems(auth);
        
        // Format for deduplication
        return items.map((item) => ({
            epochMilliSeconds: new Date(item.createdAt).valueOf(),
            data: item,
        }));
    },
};
```

## Webhook Event Handling Flow

1. **Event Reception**: External system sends webhook to Activepieces endpoint
2. **Request Processing**: The `app-event-routing.module.ts` handles the incoming webhook
3. **Flow Version Determination**: System determines which flow version to run
4. **Job Creation**: A webhook job is added to the queue
5. **Worker Processing**: The worker picks up the job and executes the flow
6. **Event Storage**: The trigger event is stored for historical reference

```typescript
// packages/server/api/src/app/app-event-routing/app-event-routing.module.ts
const eventsQueue = listeners.map((listener) => {
    return flowService(request.log).getOnePopulated({
        id: listener.flowId,
        projectId: listener.projectId,
        versionId: undefined,
    }).then(async (flow) => {
        const flowVersionIdToRun = await webhookHandler.getFlowVersionIdToRun(
            WebhookFlowVersionToRun.LOCKED_FALL_BACK_TO_LATEST, 
            flow
        )

        return jobQueue(request.log).add({
            id: requestId,
            type: JobType.WEBHOOK,
            data: {
                projectId: listener.projectId,
                schemaVersion: LATEST_JOB_DATA_SCHEMA_VERSION,
                requestId,
                payload,
                flowId: listener.flowId,
                runEnvironment: RunEnvironment.PRODUCTION,
                saveSampleData: await webhookSimulationService(request.log).exists(listener.flowId),
                flowVersionIdToRun,
                execute: flow.status === FlowStatus.ENABLED,
            },
            priority: DEFAULT_PRIORITY,
        })
    })
})
```

## Frontend Integration

The React UI components interact with trigger events through the API client:

```typescript
// packages/react-ui/src/features/flows/lib/trigger-events-api.ts
export const triggerEventsApi = {
    // Test polling triggers
    pollTrigger(request: TestPollingTriggerRequest) {
        return api.get<SeekPage<TriggerEventWithPayload>>(
            '/v1/trigger-events/poll',
            request,
        );
    },
    
    // List trigger events
    list(request: ListTriggerEventsRequest): Promise<SeekPage<TriggerEventWithPayload>> {
        return api.get<SeekPage<TriggerEventWithPayload>>(
            '/v1/trigger-events',
            request,
        );
    },
    
    // Start webhook simulation for testing
    startWebhookSimulation(flowId: string) {
        return api.post<void>('/v1/webhook-simulation', {
            flowId,
        });
    },
    
    // Delete webhook simulation
    deleteWebhookSimulation(flowId: string) {
        return api.delete<void>('/v1/webhook-simulation', {
            flowId,
        });
    },
    
    // Get webhook simulation
    async getWebhookSimulation(flowId: string) {
        try {
            return await api.get<WebhookSimulation>('/v1/webhook-simulation/', {
                flowId,
            });
        } catch (e) {
            console.error(e);
            return null;
        }
    },
    
    // Save mock data for trigger testing
    saveTriggerMockdata(flowId: string, mockData: unknown) {
        return api.post<TriggerEventWithPayload>(
            `/v1/trigger-events?flowId=${flowId}`,
            mockData,
        );
    },
};
```

## Best Practices for Trigger Event Development

1. **Always clean up resources**: Implement `onDisable` to properly unregister webhooks.
2. **Handle authentication changes**: Update or recreate webhooks when authentication details change.
3. **Store configuration**: Use the `context.store` to save webhook IDs and other configuration.
4. **Validate payload**: Always verify webhook signatures or tokens when available.
5. **Use request sampling**: Implement sample data for testing without executing real API calls.
6. **Error handling**: Provide clear error messages when webhooks fail to register or triggers fail.
7. **Deduplication**: Implement proper deduplication in polling triggers to avoid duplicate executions.

## Security Considerations

1. **Webhook Verification**: Implement signature verification for webhook payloads.
2. **Data Isolation**: Enforce proper project-level isolation for trigger events.
3. **Cleanup**: Always clean up webhooks when flows are disabled.
4. **Rate Limiting**: Implement rate limiting for webhook endpoints.
5. **Data Sanitization**: Sanitize and validate all incoming data before processing.

This document provides a comprehensive overview of the Trigger Events system in Activepieces, covering the database structure, service methods, trigger implementations, and event flow through the system.# Activepieces Workflow Engine Documentation

## Architecture Overview

Activepieces is an extensible automation platform built on a robust architecture designed to handle complex workflows with various providers. This document provides a comprehensive technical overview of how the workflow engine functions.

## Core Components

### 1. Workflow Structure

Workflows in Activepieces are organized into several key entities:

#### Flow Entity
```typescript
// Flow represents a complete automation workflow
interface Flow {
  id: string;           // Unique identifier
  projectId: string;    // Project container
  name: string;         // User-friendly name
  version: FlowVersion; // Current active version
  trigger: Trigger;     // Starting event
  steps: Step[];        // Actions to perform
  folder?: Folder;      // Organization structure
  status: FlowStatus;   // Active, disabled, etc.
}
```

#### Folder Entity
The `FolderEntity` organizes flows into logical groups:

```typescript
// From folder.entity.ts
export const FolderEntity = new EntitySchema<FolderSchema>({
    name: 'folder',
    columns: {
        ...BaseColumnSchemaPart,
        displayName: { type: String },
        projectId: ApIdSchema,
        displayOrder: { type: Number, default: 0 }
    },
    // Relations to flows and projects
    relations: {
        flows: { type: 'one-to-many', target: 'flow', inverseSide: 'folder' },
        project: { type: 'many-to-one', target: 'project', cascade: true, onDelete: 'CASCADE' }
    }
})
```

#### Trigger Event Entity
The `TriggerEventEntity` stores incoming webhook data:

```typescript
// From trigger-event.entity.ts
export const TriggerEventEntity = new EntitySchema<TriggerEventSchema>({
    name: 'trigger_event',
    columns: {
        ...BaseColumnSchemaPart,
        flowId: ApIdSchema,
        projectId: ApIdSchema,
        sourceName: { type: String },
        fileId: { type: String }
    },
    // Relations
    relations: {
        project: { type: 'many-to-one', target: 'project' },
        file: { type: 'many-to-one', target: 'file' },
        flow: { type: 'many-to-one', target: 'flow' }
    }
})
```

### 2. Piece Framework

The "piece" framework provides integration with external services:

#### Piece Metadata Entity
```typescript
// From piece-metadata-entity.ts
export const PieceMetadataEntity = new EntitySchema<PieceMetadataSchemaWithRelations>({
    name: 'piece_metadata',
    columns: {
        ...BaseColumnSchemaPart,
        name: { type: String, nullable: false },
        displayName: { type: String, nullable: false },
        logoUrl: { type: String, nullable: false },
        version: { type: String, nullable: false },
        actions: { type: JSON_COLUMN_TYPE, nullable: false },
        triggers: { type: JSON_COLUMN_TYPE, nullable: false },
        pieceType: { type: String, nullable: false },
        packageType: { type: String, nullable: false }
    }
})
```

## Workflow Creation Process

### 1. Creating a Workflow

The workflow creation process follows these steps:

1. **Project Selection**: A workflow belongs to a project container
2. **Flow Creation**: Initial flow skeleton is created
3. **Trigger Configuration**: Set up the event that starts the flow
4. **Action Definition**: Add processing steps to manipulate data
5. **Testing & Publishing**: Validate and activate the workflow

### 2. API Endpoints

Core API endpoints for flow management:

- `POST /v1/flows`: Create a new flow
- `GET /v1/flows`: List flows
- `PATCH /v1/flows/{id}`: Update flow properties
- `DELETE /v1/flows/{id}`: Delete a flow
- `POST /v1/flows/{id}/publish`: Publish a flow version
- `POST /v1/flows/{id}/test`: Test a flow with sample data

## Provider Integration System

### 1. Trigger Types

Activepieces supports multiple trigger strategies:

#### Webhook Triggers
```typescript
// From enable-trigger-hook.ts
case TriggerStrategy.WEBHOOK: {
    switch (renewConfiguration?.strategy) {
        case WebhookRenewStrategy.CRON: {
            await jobQueue(log).add({
                id: flowVersion.id,
                type: JobType.REPEATING,
                data: {
                    projectId,
                    flowVersionId: flowVersion.id,
                    flowId: flowVersion.flowId,
                    jobType: RepeatableJobType.RENEW_WEBHOOK,
                },
                scheduleOptions: {
                    cronExpression: renewConfiguration.cronExpression,
                    timezone: 'UTC',
                    failureCount: 0,
                },
            })
            break
        }
    }
    break
}
```

#### Polling Triggers
```typescript
// From enable-trigger-hook.ts
case TriggerStrategy.POLLING: {
    if (isNil(engineHelperResponse.result.scheduleOptions)) {
        engineHelperResponse.result.scheduleOptions = {
            cronExpression: POLLING_FREQUENCY_CRON_EXPRESSION,
            timezone: 'UTC',
            failureCount: 0,
        }
    }
    await jobQueue(log).add({
        id: flowVersion.id,
        type: JobType.REPEATING,
        data: {
            projectId,
            flowVersionId: flowVersion.id,
            flowId: flowVersion.flowId,
            triggerType: TriggerType.PIECE,
            jobType: RepeatableJobType.EXECUTE_TRIGGER,
        },
        scheduleOptions: engineHelperResponse.result.scheduleOptions,
    })
}
```

### 2. Piece Management

Pieces are the building blocks for provider integrations:

#### Registry Piece Manager
```typescript
// From registry-piece-manager.ts
export class RegistryPieceManager extends PieceManager {
    protected override async installDependencies({
        projectPath,
        pieces,
        log,
    }: InstallParams): Promise<void> {
        await this.savePackageArchivesToDiskIfNotCached(pieces)
        const dependenciesToInstall = await this.filterExistingPieces(projectPath, pieces)
        if (dependenciesToInstall.length === 0) {
            return
        }
        // Install pieces as NPM dependencies
        await packageManager(log).add({ path: projectPath, dependencies })
    }
}
```

#### Development Piece Builder
```typescript
// From pieces-builder.ts
export async function piecesBuilder(app: FastifyInstance, io: Server, packages: string[], piecesSource: PiecesSource): Promise<void> {
    // Only for file-based pieces during development
    if (piecesSource !== PiecesSource.FILE) return

    // Setup watchers for local piece development
    for (const packageName of packages) {
        const pieceDirectory = await filePiecesUtils(packages, app.log).findPieceDirectoryByFolderName(packageName)
        // Watch for changes and rebuild pieces
        const watcher = chokidar.watch(resolve(pieceDirectory))
        watcher.on('all', (event, path) => {
            if (path.endsWith('.ts') || path.endsWith('package.json')) {
                debouncedHandleFileChange()
            }
        })
    }
}
```

## Workflow Execution Lifecycle

### 1. Trigger Processing

When a trigger event occurs:

```typescript
// From trigger-event.service.ts
// Event capture and processing
export const triggerEventRepo = repoFactory(TriggerEventEntity)

// Events are stored in TriggerEventEntity
// - flowId: Links to the flow to execute
// - projectId: For multi-tenant isolation
// - fileId: Reference to the payload data
// - sourceName: Trigger source identification
```

### 2. Job Queue System

Jobs are scheduled and executed:

```typescript
// From enable-trigger-hook.ts
await jobQueue(log).add({
    id: flowVersion.id,
    type: JobType.REPEATING,
    data: {
        schemaVersion: LATEST_JOB_DATA_SCHEMA_VERSION,
        projectId,
        environment: RunEnvironment.PRODUCTION,
        flowVersionId: flowVersion.id,
        flowId: flowVersion.flowId,
        triggerType: TriggerType.PIECE,
        jobType: RepeatableJobType.EXECUTE_TRIGGER,
    },
    scheduleOptions: engineHelperResponse.result.scheduleOptions,
})
```

### 3. Execution Engine

The worker processes flow execution:

```typescript
// From extract-trigger-payload-hooks.ts
function handleFailureFlow(flowVersion: FlowVersion, projectId: ProjectId, engineToken: string, success: boolean, log: FastifyBaseLogger): void {
    const engineController = engineApiService(engineToken, log)

    rejectedPromiseHandler(engineController.updateFailureCount({
        flowId: flowVersion.flowId,
        projectId,
        success,
    }), log)
}
```

## Piece Installation System

### 1. Community Piece Installation

Users can install pieces from the community:

```typescript
// From community-piece-module.ts
app.post(
    '/',
    {
        config: {
            allowedPrincipals: [PrincipalType.USER],
        },
        schema: {
            body: AddPieceRequestBody,
        },
    },
    async (req, res): Promise<PieceMetadataModel> => {
        const platformId = req.principal.platform.id
        const projectId = req.principal.projectId
        const pieceMetadata = await pieceService(req.log).installPiece(
            platformId,
            projectId,
            req.body,
        )
        return res.code(StatusCodes.CREATED).send(pieceMetadata)
    },
)
```

### 2. NPM Package Management

Pieces are installed as NPM packages:

```typescript
// From registry-piece-manager.ts
private async savePackageArchivesToDiskIfNotCached(pieces: PiecePackage[]): Promise<void> {
    const packages = await this.getUncachedArchivePackages(pieces)
    const saveToDiskJobs = packages.map((piece) =>
        this.getArchiveAndSaveToDisk(piece),
    )
    await Promise.all(saveToDiskJobs)
}
```

## Multi-Tenant Architecture

### 1. Project Isolation

Data is isolated by project:

```typescript
// From folder.entity.ts
export const FolderEntity = new EntitySchema<FolderSchema>({
    columns: {
        projectId: ApIdSchema,
    },
    relations: {
        project: {
            type: 'many-to-one',
            target: 'project',
            cascade: true,
            onDelete: 'CASCADE',
        },
    },
})
```

### 2. Platform-Level Separation

Enterprise features include platform-level isolation:

```typescript
// From piece-metadata-entity.ts
export const PieceMetadataEntity = new EntitySchema<PieceMetadataSchemaWithRelations>({
    columns: {
        projectId: { type: String, nullable: true },
        platformId: { type: String, nullable: true },
    },
})
```

## Customization and White-Labeling

Activepieces supports extensive customization:

```typescript
// Platform branding options
interface Platform {
    id: string;
    name: string;
    primaryColor: string;
    logoIconUrl: string;
    fullLogoUrl: string;
    favIconUrl: string;
    emailFrom: string;
}
```

## Security Model

### 1. Authentication

Multiple authentication methods:

- API Key authentication
- JWT token-based authentication
- OAuth for third-party providers

### 2. Authorization

Role-based access control:

```typescript
// Example of authorization check
app.post(
    '/',
    {
        config: {
            allowedPrincipals: [PrincipalType.USER],
        },
    },
    async (req, res) => { /* ... */ }
)
```

## Conclusion

Activepieces provides a comprehensive workflow automation platform with:

1. **Extensible Architecture**: Add new integrations through the pieces framework
2. **Multi-Trigger Support**: Webhook, polling, and scheduled triggers
3. **Version Control**: Flow versioning and history
4. **Development Experience**: Hot-reloading for local piece development
5. **Security**: Multi-tenant isolation and role-based access
6. **Customization**: White-labeling and branding options

This architecture enables both technical and non-technical users to create powerful automation workflows that connect with hundreds of different services.

# Creating Custom Components & Trigger Points

Activepieces is designed to be highly extensible, allowing developers to create custom components and trigger points to meet specific integration needs. This section provides a comprehensive guide on creating your own custom pieces and triggers.

### Custom Piece Development

#### 1. Setting Up Your Custom Piece

A custom piece is an integration with external services or systems. To create a custom piece:

```bash
# Install the Activepieces CLI
npm install -g @activepieces/cli

# Create a new piece (in your project directory)
ap piece create my-custom-piece

# Navigate to your piece directory
cd packages/pieces/community/my-custom-piece
```

#### 2. Piece Structure

Each piece follows this structure:

```
my-custom-piece/
├── src/
│   ├── lib/
│   │   ├── actions/           # Custom actions
│   │   │   ├── action-one.ts  # Individual action
│   │   │   └── index.ts       # Action exports
│   │   ├── triggers/          # Custom triggers
│   │   │   ├── trigger-one.ts # Individual trigger
│   │   │   └── index.ts       # Trigger exports
│   │   ├── common/            # Shared utilities
│   │   │   ├── api-client.ts  # API client
│   │   │   ├── auth.ts        # Authentication
│   │   │   └── index.ts       # Common exports
│   │   └── index.ts           # Main piece definition
│   └── index.ts               # Package entry point
├── package.json               # Dependencies
└── tsconfig.json              # TypeScript config
```

#### 3. Creating a Piece Definition

The main piece definition file (`src/lib/index.ts`):

```typescript
import { createPiece } from '@activepieces/pieces-framework';
import { myActions } from './actions';
import { myTriggers } from './triggers';
import { myPieceAuth } from './common/auth';

export const myCustomPiece = createPiece({
  name: 'my-custom-piece',
  displayName: 'My Custom Integration',
  logoUrl: 'https://example.com/my-logo.png',
  authors: ['your-name'],
  version: '0.1.0',
  minimumSupportedRelease: '0.20.0',
  actions: myActions,
  triggers: myTriggers,
  auth: myPieceAuth,
});
```

### Custom Action Development

#### 1. Creating a Custom Action

Custom actions define operations your piece can perform:

```typescript
// src/lib/actions/send-data.ts
import { createAction, Property } from '@activepieces/pieces-framework';
import { httpClient } from '@activepieces/pieces-common';
import { myPieceAuth } from '../common/auth';

export const sendData = createAction({
  name: 'send_data',
  displayName: 'Send Data',
  description: 'Send data to my custom service',
  
  // Define the authentication required
  auth: myPieceAuth,
  
  // Define input properties
  props: {
    message: Property.LongText({
      displayName: 'Message',
      description: 'The message to send',
      required: true,
    }),
    priority: Property.Number({
      displayName: 'Priority',
      description: 'Message priority (1-5)',
      required: false,
      defaultValue: 3,
    }),
    tags: Property.Array({
      displayName: 'Tags',
      description: 'Message tags',
      required: false,
    }),
  },
  
  // Implement the action logic
  async run(context) {
    const { auth, propsValue } = context;
    
    // Access the authentication values
    const { apiKey } = auth;
    
    // Access the input properties
    const { message, priority, tags } = propsValue;
    
    // Make API requests using the HTTP client
    const response = await httpClient.post({
      url: 'https://api.myservice.com/messages',
      headers: {
        'Authorization': `Bearer ${apiKey}`,
        'Content-Type': 'application/json'
      },
      body: {
        message,
        priority,
        tags,
        timestamp: new Date().toISOString()
      }
    });
    
    // Return the response data
    return response.body;
  },
});

// Add your action to the exports
// src/lib/actions/index.ts
import { sendData } from './send-data';

export const myActions = [
  sendData,
  // Add more actions here
];
```

### Custom Trigger Development

#### 1. Creating a Webhook Trigger

Webhook triggers allow external systems to initiate your flows:

```typescript
// src/lib/triggers/webhook-trigger.ts
import { createTrigger, TriggerStrategy } from '@activepieces/pieces-framework';
import { httpClient } from '@activepieces/pieces-common';
import { myPieceAuth } from '../common/auth';

export const newWebhookEvent = createTrigger({
  name: 'new_webhook_event',
  displayName: 'New Webhook Event',
  description: 'Triggered when a new event is received',
  type: TriggerStrategy.WEBHOOK,
  
  // Define the authentication required
  auth: myPieceAuth,
  
  // Define configuration properties
  props: {
    eventType: Property.StaticDropdown({
      displayName: 'Event Type',
      description: 'Type of event to listen for',
      required: true,
      options: {
        options: [
          { label: 'New Message', value: 'message' },
          { label: 'Status Changed', value: 'status' },
          { label: 'User Activity', value: 'user' },
        ],
      },
    }),
  },
  
  // Setup webhook when trigger is enabled
  async onEnable(context) {
    const { auth, webhookUrl, propsValue } = context;
    const { apiKey } = auth;
    const { eventType } = propsValue;
    
    // Register webhook with external service
    const response = await httpClient.post({
      url: 'https://api.myservice.com/webhooks',
      headers: {
        'Authorization': `Bearer ${apiKey}`,
        'Content-Type': 'application/json'
      },
      body: {
        url: webhookUrl,
        events: [eventType],
        name: 'Activepieces Integration'
      }
    });
    
    // Store webhook ID for cleanup during disable
    await context.store.put('webhook_id', response.body.id);
    
    return {
      webhookId: response.body.id
    };
  },
  
  // Cleanup when trigger is disabled
  async onDisable(context) {
    const { auth } = context;
    const { apiKey } = auth;
    
    // Get stored webhook ID
    const webhookId = await context.store.get('webhook_id');
    
    if (webhookId) {
      // Delete webhook from external service
      await httpClient.delete({
        url: `https://api.myservice.com/webhooks/${webhookId}`,
        headers: {
          'Authorization': `Bearer ${apiKey}`
        }
      });
    }
  },
  
  // Process incoming webhook data
  async run(context) {
    // Extract and return webhook payload
    const payload = context.payload.body;
    
    // You can transform the payload here if needed
    return [payload];
  },
  
  // Sample data for testing
  async sampleData() {
    return {
      id: '12345',
      eventType: 'message',
      content: 'This is a sample message',
      timestamp: new Date().toISOString()
    };
  },
});
```

#### 2. Creating a Polling Trigger

Polling triggers periodically check for new data:

```typescript
// src/lib/triggers/polling-trigger.ts
import { createTrigger, TriggerStrategy, Property, DedupeStrategy } from '@activepieces/pieces-framework';
import { pollingHelper } from '@activepieces/pieces-common';
import { myApiClient } from '../common/api-client';
import { myPieceAuth } from '../common/auth';

export const pollForNewData = createTrigger({
  name: 'poll_for_new_data',
  displayName: 'Poll For New Data',
  description: 'Trigger when new data is available',
  type: TriggerStrategy.POLLING,
  
  // Define the authentication required
  auth: myPieceAuth,
  
  // Define configuration properties
  props: {
    dataType: Property.StaticDropdown({
      displayName: 'Data Type',
      description: 'Type of data to poll for',
      required: true,
      options: {
        options: [
          { label: 'New Orders', value: 'orders' },
          { label: 'New Products', value: 'products' },
          { label: 'Inventory Updates', value: 'inventory' },
        ],
      },
    }),
    maxItems: Property.Number({
      displayName: 'Maximum Items',
      description: 'Maximum number of items to fetch per poll',
      required: false,
      defaultValue: 10,
    }),
  },
  
  // Polling configuration
  async run(context) {
    const { auth, propsValue, store } = context;
    const { dataType, maxItems } = propsValue;
    
    // Use polling helper to handle deduplication and storage
    return await pollingHelper.poll(
      {
        // Deduplication strategy (time-based or unique ID based)
        strategy: DedupeStrategy.TIMEBASED,
        
        // Function to fetch items
        async items({ auth, lastFetchEpochMS }) {
          const client = myApiClient(auth);
          
          // Convert timestamp to date string format if needed
          const since = lastFetchEpochMS ? new Date(lastFetchEpochMS).toISOString() : undefined;
          
          // Fetch data from external service
          const items = await client.fetchItems({
            type: dataType,
            limit: maxItems,
            since: since,
          });
          
          // Format for deduplication (must include epochMilliSeconds)
          return items.map((item) => ({
            epochMilliSeconds: new Date(item.createdAt).valueOf(),
            data: item,
          }));
        },
      }, 
      context
    );
  },
  
  // Sample data for testing
  async sampleData() {
    return {
      id: '12345',
      type: 'order',
      amount: 99.99,
      createdAt: new Date().toISOString()
    };
  },
});
```

### Authentication Setup

Authentication is a critical component for any piece:

```typescript
// src/lib/common/auth.ts
import { createPieceAuth, Property } from '@activepieces/pieces-framework';

// API Key Authentication
export const myPieceAuth = createPieceAuth({
  type: 'custom',
  required: true,
  props: {
    apiKey: Property.SecretText({
      displayName: 'API Key',
      description: 'API Key from your account settings',
      required: true,
    }),
    apiUrl: Property.ShortText({
      displayName: 'API URL',
      description: 'Your API URL (e.g., https://api.example.com)',
      required: false,
      defaultValue: 'https://api.example.com',
    }),
  },
});

// OAuth Authentication Example
export const myOAuthPieceAuth = createPieceAuth({
  type: 'oauth2',
  
  // OAuth2 configuration
  authUrl: 'https://auth.myservice.com/oauth2/authorize',
  tokenUrl: 'https://auth.myservice.com/oauth2/token',
  required: true,
  
  // Additional OAuth properties
  props: {
    scope: Property.StaticDropdown({
      displayName: 'Scope',
      description: 'The scope of access',
      required: true,
      options: {
        options: [
          { label: 'Read Only', value: 'read' },
          { label: 'Read and Write', value: 'read write' },
          { label: 'Full Access', value: 'read write admin' },
        ],
      },
      defaultValue: 'read',
    }),
  },
});
```

### Testing Your Custom Piece

Test your piece before publishing:

```bash
# Build your piece
npm run build

# Test locally
ap piece test my-custom-piece
```

### Publishing Your Custom Piece

#### 1. Private Usage

For private usage within your organization:

```bash
# Build your piece
npm run build

# Copy to your Activepieces instance
cp -r packages/pieces/community/my-custom-piece /path/to/activepieces/custom-pieces/
```

#### 2. Contributing to the Community

To contribute your piece to the Activepieces community:

1. Fork the Activepieces repository
2. Add your piece to the `packages/pieces/community` directory
3. Create a pull request with your changes

### Registering Custom Pieces in Your Activepieces Instance

To register custom pieces in your self-hosted instance:

1. Configure the pieces source in your environment:

```bash
# In your Activepieces .env file
AP_PIECES_SOURCE=FILE
AP_DEV_PIECES=my-custom-piece
```

2. Mount your pieces directory:

```yaml
# In your docker-compose.yml
volumes:
  - ./custom-pieces:/app/custom-pieces
```

## Advanced Customization Techniques

### 1. Dynamic Properties

Create properties that depend on previous selections:

```typescript
import { createAction, Property } from '@activepieces/pieces-framework';

export const dynamicPropsAction = createAction({
  name: 'dynamic_props',
  displayName: 'Dynamic Properties Example',
  
  props: {
    resourceType: Property.StaticDropdown({
      displayName: 'Resource Type',
      required: true,
      options: {
        options: [
          { label: 'Users', value: 'users' },
          { label: 'Products', value: 'products' },
        ],
      },
    }),
    
    // Dynamic property that depends on resourceType
    resourceId: Property.Dropdown({
      displayName: 'Resource',
      required: true,
      refreshers: ['resourceType', 'auth'],
      options: async ({ resourceType, auth }) => {
        if (!resourceType) {
          return {
            disabled: true,
            options: [],
            placeholder: 'Select a resource type first',
          };
        }
        
        // Fetch options based on resourceType
        const client = createApiClient(auth);
        const resources = await client.listResources(resourceType);
        
        return {
          options: resources.map(resource => ({
            label: resource.name,
            value: resource.id,
          })),
        };
      },
    }),
  },
  
  async run(context) {
    // Implementation
  },
});
```

### 2. Custom Data Storage

Use the storage API for persistent data:

```typescript
// In your trigger or action
async run(context) {
  // Get stored data
  const storedData = await context.store.get('my_data_key');
  
  // Process data
  const newData = processData(storedData);
  
  // Store updated data
  await context.store.put('my_data_key', newData);
  
  return { result: 'Data processed successfully' };
}
```

### 3. File Handling

Process files in your custom pieces:

```typescript
import { createAction, Property } from '@activepieces/pieces-framework';
import { fileStore } from '@activepieces/pieces-common';

export const processFile = createAction({
  name: 'process_file',
  displayName: 'Process File',
  
  props: {
    fileUrl: Property.ShortText({
      displayName: 'File URL',
      required: true,
    }),
  },
  
  async run(context) {
    const { fileUrl } = context.propsValue;
    
    // Download file
    const fileData = await fileStore.downloadFile(fileUrl);
    
    // Process file content
    // ...
    
    // Upload processed file
    const uploadedFile = await fileStore.uploadFile({
      fileName: 'processed-file.txt',
      data: processedContent,
    });
    
    return {
      fileId: uploadedFile.id,
      downloadUrl: uploadedFile.url,
    };
  },
});
```

## Best Practices for Custom Component Development

1. **Error Handling**: Implement robust error handling with clear error messages
2. **Rate Limiting**: Respect API rate limits of external services
3. **Authentication Security**: Never expose authentication credentials
4. **Input Validation**: Validate all inputs before processing
5. **Documentation**: Provide clear documentation for your custom components
6. **Testing**: Create comprehensive tests for various scenarios
7. **Versioning**: Follow semantic versioning for your custom pieces
8. **Modular Design**: Create reusable functions and utilities
9. **Logging**: Implement appropriate logging for troubleshooting
10. **Performance**: Optimize for performance, especially for polling triggers

This comprehensive guide should help you get started with creating custom components and trigger points in Activepieces, enabling you to extend the platform to meet your specific integration needs.
