---
name: jetlinks-crud
description: 在 JetLinks 脚手架中实现标准或高级 CRUD 开发。适用于需要新增或修改实体、服务、控制器、查询流程、批量更新，或处理与 CRUD 相关副作用，同时遵循当前模块现有风格的场景。
---

# JetLinks CRUD

Read [`references/common-crud-rules.md`](references/common-crud-rules.md) first.

## Workflow

1. Confirm the target module's execution model and CRUD base abstractions.
2. If this is a new backend feature, large CRUD change, or CRUD change spanning multiple layers, first follow [`../jetlinks-router/references/backend-design-test-driven-rules.md`](../jetlinks-router/references/backend-design-test-driven-rules.md): write the design draft and test goals to the appropriate docs directory and wait for explicit user confirmation.
3. Follow the smallest existing Entity, Service, and Controller pattern that matches the task.
4. If the task includes `createQuery()`, `QueryParamEntity`, sorting, nested conditions, pagination, or AssetsHolder query injection, read [`references/query-dsl-rules.md`](references/query-dsl-rules.md).
5. If the task includes complex query, batch processing, or CRUD side effects, read [`references/advanced-crud-rules.md`](references/advanced-crud-rules.md).
6. If the task includes custom `termType`, QueryParam condition mapping, related-table filters, or `SubTableTermFragmentBuilder`-style exists queries, also read [`references/dynamic-term-rules.md`](references/dynamic-term-rules.md).
7. Pair with `$jetlinks-assets-permission` whenever CRUD query, detail, update, delete, batch operation, export, or custom endpoint needs data permission control through AssetsHolder.
8. Pair with `$jetlinks-conventions` or `$jetlinks-reactive` when imports, i18n, or reactive style need extra care.

## Required Constraints

- Do not generate generic CRUD boilerplate that duplicates existing base classes.
- Do not add custom endpoints when the existing query abstraction already covers the use case.
- Do not hand-roll SQL or private filter DTOs when `createQuery()` / `QueryParamEntity` can express the condition, sorting, pagination, or nested logic.
- Prefer moving heavy side effects out of the main CRUD flow.
- When Apache Commons utilities are already available in the target module or adjacent CRUD code, prefer them for common null or empty checks instead of handwritten repetitive validation branches.
- Do not implement a large CRUD feature before the design draft, task breakdown, and realistic test goals have been documented and confirmed.
- For any CRUD query, detail, update, delete, batch operation, export, or custom endpoint, analyze whether AssetsHolder data permission control is required. Route implementation details to `$jetlinks-assets-permission`. If asset type, related asset field, permission action, binding relation, or admin / tenant / platform exception semantics are unclear, ask the user before implementation.
- Do not weaken tests to satisfy the CRUD gate; tests must assert realistic business results, persistence effects, permissions, validation, and regression paths that matter to the change.
- When CRUD code changes are made, run relevant validation when possible; otherwise state the exact pending commands and remaining CRUD risks.
- For validation, not-found, and conflict errors visible to users, prefer the module's i18n-aware exception pattern over hardcoded exception messages.

## Response Shape

1. CRUD scope
2. Existing abstractions to reuse
3. Design doc path and test goals when the backend design gate applies
4. Whether advanced CRUD rules are needed
5. Whether Query DSL rules are needed
6. Verification evidence or exact pending commands
7. Remaining CRUD risks
