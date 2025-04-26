# Agent Runtime Architecture Guide

This guide outlines the architectural patterns and principles for creating agent runtime modules in the ATOS platform.

## Hexagonal Architecture Overview

The ATOS platform uses a hexagonal architecture (also known as ports and adapters) for agent runtime modules. This architecture separates the core business logic from external concerns, making the system more maintainable, testable, and adaptable to change.

These are general guidelines and patterns. 

```
<YOUR-AGENT-NAME>/
├── adapters/
│   ├── primary/    (input adapters - handle incoming requests)
│   │   ├── -<YOUR-AGENT-NAME>-action.adapter.ts
│   │   └── request-<YOUR-AGENT-NAME>.adapter.ts
│   └── secondary/  (output adapters - handle outgoing operations)
│       ├── openai.adapter.ts
│       └── datastore.adapter.ts
└── usecases/      (core business logic)
    └── -<YOUR-AGENT-NAME>-<action>.usecase.ts
```

## Key Architectural Concepts

### 1. Separation of Concerns

Each agent runtime module is divided into three main components:

- **Primary Adapters**: Handle incoming requests (HTTP, SQS) and transform them into a format the use cases can process
- **Use Cases**: Contain the core business logic independent of external systems
- **Secondary Adapters**: Handle communication with external systems (AI providers, databases)

### 2. Dependency Rule

Dependencies always point inward:

- Use cases don't depend on adapters
- Primary adapters depend on use cases
- Secondary adapters are called by use cases

### 3. Factory Pattern

The system uses factory functions to create standardized components:

- Lambda adapter factory for HTTP requests
- SQS adapter factory for queue processing
- Repository factory for data access

## Implementation Patterns

### 1. Primary Adapters

Primary adapters handle incoming requests and typically:

- Parse and validate input data using schemas
- Transform external data formats into domain objects
- Call the appropriate use case
- Handle errors and return appropriate responses

Common primary adapters include:

- **HTTP Request Adapters**: Handle API Gateway events
- **SQS Adapters**: Process messages from SQS queues

**Example: HTTP Request Adapter (request-<YOUR-AGENT-NAME>.adapter.ts)**

```typescript
/**
 * <YOUR-AGENT-NAME> Request Adapter
 * 
 * This module provides a Lambda adapter for handling <YOUR-AGENT-NAME> requests.
 * It uses the lambda adapter factory to create a standardized Lambda handler
 * with authentication, validation, and error handling.
 */
import { Request<YOUR-AGENT-NAME>InputSchema, Request<YOUR-AGENT-NAME>Input, <YOUR-AGENT-NAME>Order } from "@metadata/agents/<YOUR-AGENT-NAME>.schema";
import { OrchestratorHttpResponses } from '@metadata/http-responses.schema';
import { 
  createLambdaAdapter, 
  EventParser,
  LambdaAdapterOptions 
} from '@lib/lambda-adapter.factory';
import { randomUUID } from 'crypto';
import { updateCreditsAdapter } from "src/control-plane/billing/adapters/primary/update-remaining-credits.adapter";
import { Queue, Topic, AgentCost } from "@metadata/orders.schema";
import { createOrderProcessor } from "@lib/order-processor.factory";
import { orderManagerAdapter } from "@orchestrator/adapters/router/orders.adapters";
import { APIGatewayProxyEventV2 } from 'aws-lambda';
import { ValidUser } from '@utils/metadata/saas-identity.schema';

/**
 * Parser function that transforms the API Gateway event into the format
 * expected by the <YOUR-AGENT-NAME> use case
 */
const <YOUR-AGENT-NAME>EventParser: EventParser<Request<YOUR-AGENT-NAME>Input> = (
  event: APIGatewayProxyEventV2,
  validUser: ValidUser
) => {
  // Parse the request body
  if (!event.body) {
    throw new Error("Missing request body");
  }
  
  const eventBody = JSON.parse(event.body);
  
  // Generate IDs manually
  const orderId = randomUUID();
  const deliverableId = randomUUID();
  
  return {
    userId: validUser.userId,
    keyId: validUser.keyId,
    orderId,
    deliverableId,
    // Agent-specific fields from request
    field1: eventBody.field1,
    field2: eventBody.field2,
    field3: eventBody.field3,
    deliverableName: eventBody.deliverableName || '',
    agentId: eventBody.agentId || ''
  };
};

// Create the order processor for <YOUR-AGENT-NAME> orders
const orderProcessor = createOrderProcessor<<YOUR-AGENT-NAME>Order>({
  queueName: Queue.<YOUR-AGENT-NAME>,
  topicName: Topic.AGENT_ORDER_CREATED,
  agentCost: AgentCost.<YOUR-AGENT-NAME>,
  orderManager: orderManagerAdapter,
  creditsManager: updateCreditsAdapter
});

// Define the adapter options
const adapterOptions: LambdaAdapterOptions = {
  requireAuth: true,
  validateRequest: true,
  verboseLogging: true
};

// Create and export the Lambda adapter
export const request<YOUR-AGENT-NAME>Adapter = createLambdaAdapter({
  schema: Request<YOUR-AGENT-NAME>InputSchema,
  eventParser: <YOUR-AGENT-NAME>EventParser,
  useCase: orderProcessor,
  options: adapterOptions,
  responses: OrchestratorHttpResponses
});
```

**Example: SQS Adapter (create-<YOUR-AGENT-NAME>.adapter.ts)**

```typescript
/**
 * <YOUR-AGENT-NAME> SQS Adapter
 * 
 * This module provides an adapter for processing SQS events related to <YOUR-AGENT-NAME> creation.
 * It leverages the SQS adapter factory for standardized handling of SQS events, including:
 * - Message parsing and validation
 * - Error handling
 * - Parallel processing of messages
 */
import { createSqsAdapter } from '@lib/sqs-adapter.factory';
import { Request<YOUR-AGENT-NAME>InputSchema } from '@metadata/agents/<YOUR-AGENT-NAME>.schema';
import { create<YOUR-AGENT-NAME>Usecase } from '@agent-runtime/<YOUR-AGENT-NAME>/usecases/create-<YOUR-AGENT-NAME>.usecase';

export const create<YOUR-AGENT-NAME>Adapter = createSqsAdapter({
  schema: Request<YOUR-AGENT-NAME>InputSchema,
  useCase: create<YOUR-AGENT-NAME>Usecase,
  adapterName: '<YOUR-AGENT-NAME>',
  options: {
    verboseLogging: true,
    processInParallel: true,
    continueOnError: false
  }
});
```

### 2. Use Cases

Use cases implement the core business logic and typically:

- Are independent of external systems
- Orchestrate the flow of data between adapters
- Implement domain-specific rules and transformations
- Return standardized response objects

**Example: Use Case (create-<YOUR-AGENT-NAME>.usecase.ts)**

```typescript
import { Request<YOUR-AGENT-NAME>Input } from '@metadata/agents/<YOUR-AGENT-NAME>.schema';
import { run<YOUR-AGENT-NAME> } from '@agent-runtime/<YOUR-AGENT-NAME>/adapters/secondary/openai.adapter';
import { DeliverableDTO } from '@metadata/agents/<YOUR-AGENT-NAME>.schema';
import { randomUUID } from 'crypto';
import { deliverableRepository } from '@agent-runtime/<YOUR-AGENT-NAME>/adapters/secondary/datastore.adapter';
import { Message } from '@utils/metadata/message.schema';

export const create<YOUR-AGENT-NAME>Usecase = async (input: Request<YOUR-AGENT-NAME>Input): Promise<Message> => {
  console.info("Creating <YOUR-AGENT-NAME> deliverable for User");

  try {
    // Call the AI provider adapter to generate content
    const content = await run<YOUR-AGENT-NAME>(input);
    
    // Transform the generated content into a deliverable
    const deliverable: DeliverableDTO = {
      userId: input.userId,
      orderId: input.orderId,
      deliverableId: input.deliverableId,
      deliverableName: input.deliverableName,
      agentId: input.agentId,
      ...content
    };
    
    // Save the deliverable to the database
    await deliverableRepository.saveDeliverable(deliverable);
    
    // Return success message
    return {
      message: '<YOUR-AGENT-NAME> created successfully',
    };

  } catch (error) {
    // Log the error
    console.error('Error generating <YOUR-AGENT-NAME>:', error);
    
    // Throw a domain-specific error
    throw new Error('Failed to generate <YOUR-AGENT-NAME>');
  }
};
```

### 3. Secondary Adapters

Secondary adapters handle communication with external systems and typically:

- Transform domain objects into formats required by external systems
- Handle communication details (API calls, database queries)
- Abstract away the specifics of external systems
- Provide error handling and retries

Common secondary adapters include:

- **AI Provider Adapters**: Communicate with AI services (OpenAI, etc.)
- **Datastore Adapters**: Handle persistence operations

**Example: OpenAI Adapter (openai.adapter.ts)**

```typescript
import OpenAI from "openai";
import { 
  Deliverable, 
  DeliverableSchema, 
  Request<YOUR-AGENT-NAME>Input, 
  <YOUR-AGENT-NAME>SystemPrompt 
} from "@metadata/agents/<YOUR-AGENT-NAME>.schema";
import { zodResponseFormat } from "openai/helpers/zod";
import { Resource } from "sst";
import { withRetry } from "@utils/tools/retry";
import { zodToJsonSchema } from "zod-to-json-schema";

const client = new OpenAI({
  apiKey: Resource.OpenAiApiKey.name
});

export const run<YOUR-AGENT-NAME> = async (input: Request<YOUR-AGENT-NAME>Input): Promise<Deliverable> => {
  try {
    // Create an Assistant
    const assistant = await client.beta.assistants.create({
      name: "<YOUR-AGENT-NAME>",
      instructions: <YOUR-AGENT-NAME>SystemPrompt(input),
      model: "o3-mini",
      response_format: zodResponseFormat(DeliverableSchema, "deliverable"),
      tools: [{ type: "function", function: {
        name: "generateDeliverable",
        parameters: zodToJsonSchema(DeliverableSchema)
      }}]
    });

    // Create a Thread
    const thread = await client.beta.threads.create();

    // Add the user's message to the thread
    const prompt = `
    Field1: ${input.field1}
    
    Field2: ${input.field2}
    
    Field3: ${input.field3}
    `;
    
    await client.beta.threads.messages.create(thread.id, {
      role: "user",
      content: prompt
    });
    
    console.info("Added user message to thread");
    
    // Run the Assistant and wait for completion
    const run = await client.beta.threads.runs.create(thread.id, {
      assistant_id: assistant.id
    });
    
    console.info("Created run");
    
    // Wait for completion with proper status handling
    const completedRun = await waitForRunCompletion(client, thread.id, run.id);
    
    console.info("Completed run");
    
    // Get the messages
    const messages = await client.beta.threads.messages.list(thread.id);
    
    // Find the assistant's response
    const assistantMessages = messages.data.filter(msg => msg.role === 'assistant');
    if (assistantMessages.length === 0) {
      throw new Error("No assistant response found");
    }
    
    // Parse the response
    const latestMessage = assistantMessages[0];
    const content = latestMessage.content[0];
    
    if (content.type !== 'text') {
      throw new Error("Expected text response from assistant");
    }
    
    // Parse the JSON response
    const jsonMatch = content.text.value.match(/```json\n([\s\S]*?)\n```/);
    if (!jsonMatch) {
      throw new Error("No JSON found in response");
    }
    
    const jsonResponse = JSON.parse(jsonMatch[1]);
    return jsonResponse.deliverable;
    
  } catch (error) {
    console.error("Error in <YOUR-AGENT-NAME>:", error);
    throw new Error("Failed to generate <YOUR-AGENT-NAME>");
  }
};

async function waitForRunCompletion(client: OpenAI, threadId: string, runId: string) {
  let run = await client.beta.threads.runs.retrieve(threadId, runId);
  
  while (run.status === "queued" || run.status === "in_progress") {
    // Wait for 1 second before checking again
    await new Promise(resolve => setTimeout(resolve, 1000));
    run = await client.beta.threads.runs.retrieve(threadId, runId);
  }
  
  if (run.status !== "completed") {
    throw new Error(`Run failed with status: ${run.status}`);
  }
  
  return run;
}
```

**Example: Datastore Adapter (datastore.adapter.ts)**

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';
import { DeliverableDTO, <YOUR-AGENT-NAME>Schema } from "@metadata/agents/<YOUR-AGENT-NAME>.schema";
import { 
  createDeliverableRepository as createGenericDeliverableRepository, 
  IDeliverableRepository 
} from "@lib/deliverable-repository.factory";
import { z } from "zod";

// Define the output type for <YOUR-AGENT-NAME> deliverables
export type <YOUR-AGENT-NAME>Output = z.infer<typeof <YOUR-AGENT-NAME>Schema>;

// Set up DynamoDB client
const dynamoDbClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

/**
 * Create a repository for <YOUR-AGENT-NAME> deliverables
 * 
 * @param dbClient The DynamoDB document client
 * @returns A deliverable repository instance for <YOUR-AGENT-NAME> deliverables
 */
const createDeliverableRepository = (dbClient: DynamoDBDocumentClient): IDeliverableRepository<DeliverableDTO, <YOUR-AGENT-NAME>Output> => {
  return createGenericDeliverableRepository<DeliverableDTO, <YOUR-AGENT-NAME>Output>(dbClient, {
    agentName: '<YOUR-AGENT-NAME>',
    verbose: true,
    errorMessages: {
      notFound: "<YOUR-AGENT-NAME> deliverable not found",
      saveFailed: "Failed to save <YOUR-AGENT-NAME> deliverable",
      getFailed: "Failed to get <YOUR-AGENT-NAME> deliverable"
    }
  });
};

export const deliverableRepository = createDeliverableRepository(dynamoDbClient);
```

## Integration Patterns

### 1. Schema Validation

All input and output data is validated using Zod schemas:

- Input schemas define the expected request format
- Output schemas define the expected response format
- Schemas are shared between metadata and runtime components

**Example: Schema Definition (<YOUR-AGENT-NAME>.schema.ts)**

```typescript
import { z } from 'zod';
import { BasePayloadSchema, PlaceOrderSchema } from '@metadata/orders.schema';

/**
 * System prompt for the <YOUR-AGENT-NAME> agent
 * @returns The system prompt string
 */
export const <YOUR-AGENT-NAME>SystemPrompt = (input: Request<YOUR-AGENT-NAME>Input): string => `
  You are an expert [describe expertise].
  Your order is to [describe primary task].

  Field1: ${input.field1}
  Field2: ${input.field2}
  Field3: ${input.field3}

  [Include specific instructions for the agent]
`;

export const Request<YOUR-AGENT-NAME>InputSchema = BasePayloadSchema.extend({
  field1: z.string().nonempty("Field1 is required"),
  field2: z.string().nonempty("Field2 is required"),
  field3: z.string().nonempty("Field3 is required"),
});

export const <YOUR-AGENT-NAME>Schema = z.object({
  sections: z.object({
    section1: z.object({
      id: z.string(),
      label: z.string(),
      type: z.literal('text'),
      description: z.string().optional(),
      data: z.string(),
    }),
    section2: z.object({
      id: z.string(),
      label: z.string(),
      type: z.literal('list'),
      description: z.string().optional(),
      data: z.array(z.string()),
    }),
    section3: z.object({
      id: z.string(),
      label: z.string(),
      type: z.literal('markdown'),
      description: z.string().optional(),
      data: z.string(),
    }),
  })
});

export const DeliverableSchema = z.object({
  deliverableContent: <YOUR-AGENT-NAME>Schema,
});

export const DeliverableDTOSchema = BasePayloadSchema.extend({
  deliverableContent: <YOUR-AGENT-NAME>Schema,
});

export const <YOUR-AGENT-NAME>OrderSchema = PlaceOrderSchema.extend({
  payload: Request<YOUR-AGENT-NAME>InputSchema,
});

export type Request<YOUR-AGENT-NAME>Input = z.infer<typeof Request<YOUR-AGENT-NAME>InputSchema>;
export type Deliverable = z.infer<typeof DeliverableSchema>;
export type DeliverableDTO = z.infer<typeof DeliverableDTOSchema>;
export type <YOUR-AGENT-NAME>Order = z.infer<typeof <YOUR-AGENT-NAME>OrderSchema>;
```

### 2. Error Handling

Standardized error handling patterns:

- Primary adapters catch and transform errors into appropriate responses
- Use cases propagate domain-specific errors
- Secondary adapters handle and retry transient failures

**Example: Error Handling with Retries**

```typescript
// Error handling with retries in a secondary adapter
try {
  // Call external service
  return await withRetry(
    async () => {
      const result = await externalService.operation();
      return result;
    },
    {
      maxRetries: 3,
      retryDelay: 1000,
      shouldRetry: (error) => {
        // Only retry on specific error types
        return error.code === 'RATE_LIMIT' || error.code === 'TIMEOUT';
      },
      onRetry: (error, attempt) => {
        console.warn(`Retry attempt ${attempt} after error: ${error.message}`);
      }
    }
  );
} catch (error) {
  // Log with context
  console.error('External service error', { 
    error: error.message,
    service: 'external-service-name'
  });
  
  // Rethrow with domain-specific error
  throw new Error('Failed to perform operation');
}
```

### 3. Dependency Injection

Dependencies are injected rather than imported directly:

- Factories accept dependencies as parameters
- Adapters are composed at the application boundary
- This facilitates testing and flexibility

**Example: Factory with Dependency Injection**

```typescript
// Factory function with dependency injection
export const create<YOUR-AGENT-NAME>Usecase = (
  aiProvider: {
    generateContent: (input: Request<YOUR-AGENT-NAME>Input) => Promise<Deliverable>
  },
  repository: {
    saveDeliverable: (deliverable: DeliverableDTO) => Promise<void>
  }
) => {
  return async (input: Request<YOUR-AGENT-NAME>Input): Promise<Message> => {
    console.info("Creating <YOUR-AGENT-NAME> deliverable");
    
    try {
      // Use injected dependencies
      const content = await aiProvider.generateContent(input);
      
      const deliverable: DeliverableDTO = {
        userId: input.userId,
        orderId: input.orderId,
        deliverableId: input.deliverableId,
        deliverableName: input.deliverableName,
        agentId: input.agentId,
        ...content
      };
      
      await repository.saveDeliverable(deliverable);
      
      return {
        message: '<YOUR-AGENT-NAME> created successfully'
      };
    } catch (error) {
      console.error('Error in <YOUR-AGENT-NAME> usecase:', error);
      throw new Error('Failed to create <YOUR-AGENT-NAME>');
    }
  };
};

// Usage in composition root
const aiProvider = {
  generateContent: run<YOUR-AGENT-NAME>
};

const repository = deliverableRepository;

// Create the use case with dependencies
const usecase = create<YOUR-AGENT-NAME>Usecase(aiProvider, repository);
```

## Implementation Workflow

When creating a new agent runtime module:

1. **Define the domain model** in the metadata layer
2. **Implement secondary adapters** for external systems
3. **Implement use cases** for core business logic
4. **Implement primary adapters** for handling requests
5. **Register the agent** in the infrastructure

## Best Practices

1. **Single Responsibility**: Each component should have a single responsibility
2. **Immutability**: Prefer immutable data structures
3. **Pure Functions**: Minimize side effects in business logic
4. **Consistent Naming**: Follow established naming conventions
5. **Comprehensive Logging**: Include appropriate logging for debugging
6. **Error Handling**: Implement proper error handling at each layer
7. **Testing**: Write unit tests for each component

## Common Anti-patterns to Avoid

1. **Leaky Abstractions**: Don't let external concerns leak into use cases
2. **Direct Dependencies**: Don't import external services directly in use cases
3. **Inconsistent Error Handling**: Don't swallow errors without proper handling
4. **Tight Coupling**: Don't create tight coupling between components
5. **Mixed Responsibilities**: Don't mix business logic with I/O operations 