Enforces three C# SDK model naming conventions:

- Use `Config` instead of `Options`, except for client options (types ending with `ClientOptions` are exempt).
- Use `Content` instead of `Request`.
- Use `Result` instead of `Response`.

The rule checks the C#-resolved model name and respects `@clientName` overrides.

## Impact

- **Area:** SDK generation, **C# only**. Affects the generated model type name in the C# SDK (applies to both data-plane and management-plane).
- **Not affected:** Other language SDKs, the service definition, and the wire protocol are unchanged — the serialized shape is untouched, only the C# client-surface name.

## ❌ Incorrect Usage

```tsp
model SearchOptions {
  filter: string;
}

model CreateWidgetRequest {
  name: string;
}

model CreateWidgetResponse {
  id: string;
}
```

## Diagnostic Message

For the models above, the linter reports each model whose C# name ends with a disallowed suffix:

```text
Model 'SearchOptions' ends with 'Options'. Use 'Config' suffix instead (e.g. 'SearchConfig'). Use @clientName("SearchConfig", "csharp") to rename it for C#.
Model 'CreateWidgetRequest' ends with 'Request'. Use 'Content' suffix instead (e.g. 'CreateWidgetContent'). Use @clientName("CreateWidgetContent", "csharp") to rename it for C#.
Model 'CreateWidgetResponse' ends with 'Response'. Use 'Result' suffix instead (e.g. 'CreateWidgetResult'). Use @clientName("CreateWidgetResult", "csharp") to rename it for C#.
```

## ✅ How to Fix

Rename the model to use the recommended suffix:

```tsp
model SearchConfig {
  filter: string;
}

model CreateWidgetContent {
  name: string;
}

model CreateWidgetResult {
  id: string;
}
```

Or use `@@clientName` in `client.tsp` to override just the C# name without changing the TypeSpec source:

```tsp
// client.tsp
@@clientName(SearchOptions, "SearchConfig", "csharp");
@@clientName(CreateWidgetRequest, "CreateWidgetContent", "csharp");
@@clientName(CreateWidgetResponse, "CreateWidgetResult", "csharp");
```

## Suppression

This rule is a `warning` and can be suppressed. Suppress it only when the non-standard suffix is intentional and accepted for the C# SDK:

```tsp
#suppress "@azure-tools/typespec-client-generator-core/csharp-model-suffix" "intentional Options suffix for this model"
model SearchOptions {
  filter: string;
}
```

## LintDiff Equivalent

There is no direct LintDiff (azure-openapi-validator) equivalent for this rule. It is specific to C# SDK model naming conventions enforced at the TypeSpec level.
