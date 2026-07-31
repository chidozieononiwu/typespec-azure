Enforces standard acronym casing in C# SDK names. This initial rule covers `IP`, `DB`, and `OS`:

- `Ip` at a camelCase boundary (e.g. `PublicIpAddress`) → `IP` (e.g. `PublicIPAddress`)
- `Db` at a camelCase boundary (e.g. `CosmosDb`) → `DB` (e.g. `CosmosDB`)
- `Os` at a camelCase boundary (e.g. `OsProfile`) → `OS` (e.g. `OSProfile`)

The rule checks the C#-resolved name of models, enums, and properties, and respects `@clientName` overrides. It is careful not to mangle words where the letters are only a prefix (e.g. `Oslo`, `Ipsum`, `Osmosis` are left unchanged).

## Impact

- **Area:** SDK generation, **C# only**. Affects the generated type and property names in the C# SDK (applies to both data-plane and management-plane).
- **Not affected:** Other language SDKs, the service definition, and the wire protocol are unchanged — the serialized names are untouched, only the C# client-surface names.

## ❌ Incorrect Usage

```tsp
model IpAddress {
  value: string;
}

model CosmosDb {
  id: string;
}
```

## Diagnostic Message

For the models above, the linter reports each name that should use standard acronym casing:

```text
Name 'IpAddress' should use standard C# acronym casing (e.g. 'IPAddress'). Use @clientName("IPAddress", "csharp") to rename it for C#.
Name 'CosmosDb' should use standard C# acronym casing (e.g. 'CosmosDB'). Use @clientName("CosmosDB", "csharp") to rename it for C#.
```

## ✅ How to Fix

Rename the type or property to use standard acronym casing:

```tsp
model IPAddress {
  value: string;
}

model CosmosDB {
  id: string;
}
```

Or use `@@clientName` in `client.tsp` to override just the C# name without changing the TypeSpec source:

```tsp
// client.tsp
@@clientName(IpAddress, "IPAddress", "csharp");
@@clientName(CosmosDb, "CosmosDB", "csharp");
```

## Suppression

This rule is a `warning` and can be suppressed. Suppress it only when the non-standard casing is intentional and accepted for the C# SDK:

```tsp
#suppress "@azure-tools/typespec-client-generator-core/csharp-use-standard-acronyms" "intentional non-standard acronym casing"
model IpAddress {
  value: string;
}
```

## LintDiff Equivalent

There is no direct LintDiff (azure-openapi-validator) equivalent for this rule. It is specific to C# SDK naming conventions enforced at the TypeSpec level.
