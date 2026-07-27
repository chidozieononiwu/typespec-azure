---
title: Modeling Agents with ARM Base Types
description: How to model an Agent resource and its Conversation and Response child resources using the experimental ARM base types
llmstxt: true
---

:::caution
ARM base types are **experimental** and may change in breaking ways. Using the
`@azureBaseType` decorator (directly or through the `Agent` template) emits the
`@azure-tools/typespec-azure-resource-manager/basetypes-experimental` warning. Suppress it
explicitly when you have opted in to the feature.
:::

ARM _base types_ describe a structured contract — a set of required and optional properties — that a
resource conforms to. The `@azureBaseType` decorator marks an ARM resource as implementing one of
these contracts, and the `Agent` base type provides ready-made templates for modeling an AI agent
resource together with its `Conversation` and `Response` child resources.

All Agent base type symbols live in the `Azure.ResourceManager.BaseTypes` and
`Azure.ResourceManager.BaseTypes.Agents` namespaces.

## The `@azureBaseType` decorator

`@azureBaseType(baseType: valueof BaseTypeInfo)` marks a resource properties model as implementing a
base type. It may be applied more than once to declare conformance to multiple base types; duplicate
entries are ignored. The `BaseTypeInfo` value carries the base type identifier and its schema
version:

```typespec
using Azure.ResourceManager.BaseTypes;

@azureBaseType(#{ baseType: BaseType.Agent, version: "2024-06-01" })
model ContosoAgent is TrackedResource<ContosoAgentProperties> {
  ...ResourceNameParameter<ContosoAgent>;
}
```

`BaseType` is an extensible (open) enum, so the currently defined members (`BaseType.Agent`,
`BaseType.Relationship`) can grow without a breaking change.

You rarely need to apply `@azureBaseType` by hand for agents — the `Agent` template applies it for
you (see below).

## Appliance vs. Platform deployment models

The Agent base type offers two deployment models that differ only in property visibility:

- **Appliance** — the service owns and reports agent configuration, so definition and properties are
  **read-only** (`AgentDefinitionAppliance`, `AgentPropertiesAppliance`, `AgentToolTypeAppliance`).
- **Platform** — the client owns and manages agent configuration, so definition and properties are
  **writable** (`AgentDefinitionPlatform`, `AgentPropertiesPlatform`, `AgentToolTypePlatform`). The
  ARM-managed `baseTypes` property remains read-only.

### Defining the agent definition

`AgentDefinitionAppliance<HasModelDeploymentRef, HasInstructions>` and
`AgentDefinitionPlatform<HasModelDeploymentRef, HasInstructions>` describe the agent's model and
behavior. Both template parameters default to `false`; set them to `true` to include the optional
`modelDeploymentRef` and `instructions` properties:

```typespec
using Azure.ResourceManager.BaseTypes.Agents;

// Service-managed (read-only) definition with both optional properties present
model ContosoApplianceDefinition is AgentDefinitionAppliance<true, true>;

// Client-managed (writable) definition with both optional properties present
model ContosoPlatformDefinition is AgentDefinitionPlatform<true, true>;
```

### Defining the agent properties

`AgentPropertiesAppliance<AgentDefinitionType>` and `AgentPropertiesPlatform<AgentDefinitionType>`
provide the resource property bag. Pass your definition model as the template argument and spread the
standard provisioning-state property:

```typespec
model ContosoApplianceAgentProperties is AgentPropertiesAppliance<ContosoApplianceDefinition> {
  ...DefaultProvisioningStateProperty;
}

model ContosoPlatformAgentProperties is AgentPropertiesPlatform<ContosoPlatformDefinition> {
  ...DefaultProvisioningStateProperty;
}
```

## Defining the Agent resource

The `Agent<Properties>` template creates an ARM `TrackedResource` and applies the
`@azureBaseType(#{ baseType: BaseType.Agent, version: "2024-06-01" })` decorator automatically. Add
the `ResourceNameParameter` spread as with any other resource:

```typespec
#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
model ContosoApplianceAgent is Agent<ContosoApplianceAgentProperties> {
  ...ResourceNameParameter<ContosoApplianceAgent>;
}
```

## Conversation and Response child resources

Every Agent resource **must** have both a `Conversation` and a `Response` child resource. If either
is missing, the [`arm-agent-base-type-child-resources`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-child-resources)
rule fires.

Use the `AgentConversation<Properties, AgentResource>` and `AgentResponse<Properties, AgentResource>`
templates. `Properties` must extend `ConversationProperties` / `ResponseProperties`, and
`AgentResource` is the parent Agent resource type:

```typespec
model ContosoConversationProperties is ConversationProperties {
  ...DefaultProvisioningStateProperty;
}

model ContosoResponseProperties is ResponseProperties {
  ...PreviousResponseProperty;
  ...DefaultProvisioningStateProperty;
}

model ApplianceConversation is AgentConversation<
  ContosoConversationProperties,
  ContosoApplianceAgent
> {
  ...ResourceNameParameter<ApplianceConversation>;
}

model ApplianceResponse is AgentResponse<ContosoResponseProperties, ContosoApplianceAgent> {
  ...ResourceNameParameter<ApplianceResponse>;
}
```

`ResponseProperties` carries an `input: ConversationItem` (required on create), a read-only
`status: ResponseStatus`, and a read-only `createdAt`. Optional mix-ins are available:

- `...PreviousResponseProperty` — adds `previousResponseId` for multi-turn chaining.
- `...ResponseInstructionsProperty` — adds a per-response `instructions` override.
- `...ResponseOutputProperty` — adds the read-only `output: ResponseItem[]` list.

Conversation and response items are modeled with the discriminated `ConversationItem` /
`ResponseItem` models, keyed off the `type: ItemType` discriminator (`Message`, `FunctionCall`,
`FunctionCallOutput`, `Compaction`) and, for messages, the `role: MessageRole` enum (`Developer`,
`User`, `Assistant`, `Tool`).

## Required lifecycle operations

The `Conversation` and `Response` child resources must each define create, read, update, and delete
operations. Omitting any of them triggers the
[`arm-agent-base-type-lifecycle-operations`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-lifecycle-operations)
rule:

```typespec
@armResourceOperations
interface ApplianceConversations {
  get is ArmResourceRead<ApplianceConversation>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ApplianceConversation>;
  update is ArmCustomPatchSync<
    ApplianceConversation,
    Azure.ResourceManager.Foundations.ResourceUpdateModel<
      ApplianceConversation,
      ContosoConversationProperties
    >
  >;
  delete is ArmResourceDeleteWithoutOkAsync<ApplianceConversation>;
  listByAgent is ArmResourceListByParent<ApplianceConversation>;
}
```

The Agent resource itself uses the standard tracked-resource operation set (see
[ARM Resource Operations](/docs/howtos/arm/resource-operations)):

```typespec
@armResourceOperations
interface ApplianceAgents {
  get is ArmResourceRead<ContosoApplianceAgent>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ContosoApplianceAgent>;
  update is ArmTagsPatchSync<ContosoApplianceAgent>;
  delete is ArmResourceDeleteWithoutOkAsync<ContosoApplianceAgent>;
  listBySubscription is ArmListBySubscription<ContosoApplianceAgent>;
  listByResourceGroup is ArmResourceListByParent<ContosoApplianceAgent>;
}
```
