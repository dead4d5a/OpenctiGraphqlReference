# Octi GraphQL Documentation (Snapshot-based, plan only)

## 0) Front matter

### 0.1 Snapshot scope, provenance, and caveats

This documentation is based on a **point-in-time archive** of the Octi/OpenCTI GraphQL layer captured from a running container filesystem.

#### Provenance (what this snapshot is)

From `MANIFEST.txt`, this archive was generated from:

* **Snapshot timestamp:** `2026-01-27_141500` (the capture label used for the archive)
* **Container name:** `identity-core-opencti-1`
* **Image:** `opencti/platform:6.9.10`
* **Container created:** `2026-01-27T21:12:58.975722779Z`

These values are the primary “source of truth” for *which version* of the schema/runtime this documentation describes (`MANIFEST.txt`).

#### What is in scope (what we can document accurately from this snapshot)

Based on the archive layout plus module guidance, this snapshot gives us enough to document:

1. **The canonical GraphQL SDL (core schema):**

   * `opt/opencti/config/schema/opencti.graphql`
     This file defines:
   * Global directives (e.g., `@auth`, `@public`, and licensing/OTP-related directives)
   * Core scalars/enums (including the `Capabilities` enum used by `@auth`)
   * Root operation types: `Query`, `Subscription`, and `Mutation`
   * Large portions of the platform’s type system, unions, inputs, etc.

2. **The module system + how modules are expected to be built (structure & responsibilities):**

   * `src/modules/README.md`
     This describes the intended module pattern: a module typically has a `.graphql` schema file, resolver file(s), domain logic wrapper, types, converter, and a module definition file.

3. **What modules and GraphQL registrations are actually included in this build:**

   * `src/modules/index.ts`
     This file is effectively the “module manifest” for the runtime: it imports (in a specific order) attribute registrations, relationship reference registrations, modules, and the module GraphQL registrations (side-effect imports).

#### What is *not* in scope (caveats you should assume unless we add more sources)

This snapshot is **not a full repository checkout**; it is a focused slice of the GraphQL system. Concretely:

* The module README references project-wide build tooling like `graphql-codegen.yml`, but that file is **not present in this archive** (`src/modules/README.md`), so we cannot document codegen configuration details from this snapshot alone.
* Many resolvers call into deeper “engine”/data-access layers; if those layers live outside the captured `opt/opencti/src/{graphql,modules,schema}` surface, then we **should not claim** details about storage, caching, loaders, database queries, or worker pipelines unless they’re visible in the captured code.
* Treat this documentation as describing **“GraphQL surface + composition + enforcement patterns”**, not necessarily end-to-end business logic execution paths.

#### Important structural caveats (things that affect correctness of future sections)

1. **Import ordering is intentional and must be documented as such**
   In `src/modules/index.ts`, the file is explicitly organized into regions:

   * Attribute registrations: “need to be imported before any other modules”
   * RelationsRef registrations: also must precede modules
   * Module imports: “need to be imported before graphql code registration”
   * GraphQL registrations: module `*-graphql` imports

   This means our docs should treat `src/modules/index.ts` as defining the **required initialization ordering** for correct schema + metadata registration.

2. **Some modules are explicitly labeled “incomplete” in this build**
   `src/modules/index.ts` contains a clearly marked “incomplete modules” block (e.g., report/note/externalReference/incident/observedData/stixCyberObservable/etc.). That’s an explicit signal that:

   * schema coverage may exist, but implementation may be partial, legacy, or pending
   * we should be cautious making strong claims about completeness for those modules until we cross-check their SDL + resolver presence/behavior

3. **Schema unions require manual maintenance when adding module types**
   The module README states that when adding a new module/type, you must update unions in `opencti.graphql` (notably unions like `StixCoreObjectOrStixCoreRelationship`, `StixObjectOrStixRelationship`, and `StixObjectOrStixRelationshipOrCreator`).
   This is a key “provenance + governance” caveat for schema evolution and will matter later when we document contribution workflows.

#### Path conventions used in this documentation

* When we refer to files like `src/modules/index.ts`, we mean the archive-relative path under:

  * `opt/opencti/src/modules/index.ts`
* Similarly, the canonical core schema is at:

  * `opt/opencti/config/schema/opencti.graphql`

---

### 0.2 How this documentation is organized (generated vs curated)

This documentation is deliberately split into two “lanes” to minimize drift and maximize correctness:

* **Generated docs** are produced directly from the *same merged schema* the server runs (the “executable schema”).
* **Curated docs** explain *how* that schema is assembled and *how* the runtime behaves (protections, plugins, error behavior, config knobs). These are written by humans and reviewed against the server wiring code.

#### A) Generated documentation (API reference)

**What is generated**

* **Type/field/argument reference** (including descriptions, inputs, enums, scalars, interfaces, unions)
* **Root operation reference** (`Query`, `Mutation`, `Subscription`)
* **Directive catalog** (what directives exist and where they apply)
* **Derived indices** we can safely compute from the schema (e.g., “all mutations”, “all subscriptions”, “types by domain area”)

**Why generation is required (source of truth)**

* The server schema is constructed programmatically in `src/graphql/schema.js` via `makeExecutableSchema({ typeDefs, resolvers, inheritResolversFromInterfaces: true })`.
* The schema is not “just one SDL file”: it starts from the core SDL and is then extended through a **registration mechanism**. Specifically, `schema.js` exports `registerGraphqlSchema({ schema, resolver })`, which appends additional module schemas/resolvers into the arrays used to build the final executable schema.
* Because modules register themselves by executing side-effect imports that call `registerGraphqlSchema`, the only reliable way to document the full API surface is to generate from the final merged result.

**Key schema-build behaviors to reflect in generated docs**

* **Core SDL baseline:** `schemaTypeDefs` starts with the imported core SDL (`opencti.graphql`) as the initial typeDefs source.
* **Global scalars are implemented in code:** the schema layer defines and provides resolvers for custom scalars like `StixId`, `StixRef`, `Any`, along with `DateTime` and `Upload` in `schema.js`—these must appear in the API reference and should be described as code-backed scalars (not merely SDL placeholders).
* **Directive transforms are applied after schema creation:** the returned schema is transformed by an auth directive transformer and a rate-limit transformer, and additionally wrapped with constraint directive documentation support. Generated docs should therefore treat directives as first-class documented behavior, not cosmetic annotations.

> Practical implication for the doc pipeline: the “reference docs” should be generated from **the built executable schema** (or from an introspection result produced by building the schema with the same initialization/import order the server uses), not from `opencti.graphql` alone.

---

#### B) Curated documentation (implementation guide)

**What is curated**

* **How the server is created and wired** (Apollo Server configuration and request lifecycle)
* **Protections and operational hardening**
* **Behavioral semantics** that are not obvious from SDL alone (error formatting rules, introspection gating behavior, validation rules, plugin behavior)
* **Config knobs** (what toggles exist, what defaults are used, what happens when enabled/disabled)

**Why it must be curated**
Even with a perfect schema dump, the runtime behavior is shaped by server wiring in `src/graphql/graphql.js`, including:

1. **Validation + hardening layers**

* A constraint-validation plugin is enabled (via Apollo’s plugin interface) and includes a custom `not-blank` format rule.
* Alias/batching protection is enabled via `graphql-no-alias`, with per-root-type defaults (notably separate defaults for Query vs Mutation and special casing for certain operations like `subTypes` and `token`).

2. **Optional “armor” protections**

* When not disabled by config, Apollo Armor protections are applied (depth, cost, token count, directive count, etc.), with specific defaults pulled from `conf`.

3. **Introspection behavior is more nuanced than “on/off”**

* Apollo is constructed with `introspection: true`, but `graphql.js` adds a plugin that **blocks introspection queries** unless conditions are met:

  * If playground is disabled (or introspection is disabled by playground config), introspection is rejected.
  * Even when allowed, the plugin requires an authenticated user in context for introspection access.

4. **Error formatting is environment-sensitive**

* `formatError` normalizes the error shape for client compatibility, and in non-dev mode removes exception stack details.

5. **Observability is controlled by feature flags**

* Tracing and metrics plugins are conditionally added based on config flags, so their presence/behavior must be documented as “conditional runtime features”.

---

#### How the two lanes are presented in the final docs

* **“API Reference” section = generated**
  Always produced from the merged executable schema so it stays correct as modules evolve.

* **“Implementation Guide” sections = curated**
  Written from `schema.js` + `graphql.js` behavior, with explicit callouts for:

  * schema assembly model
  * directive enforcement and transformers
  * validation/hardening layers
  * introspection rules
  * error behavior
  * observability toggles

* **Cross-links between lanes**
  Curated pages (e.g., “Auth Directive”) will link into generated pages (“Directive: @auth”, “Capabilities enum”, “Operations using @auth”), so readers can go from “how it works” → “where it applies” without duplicating content.

---

## 1) Orientation

### 1.1 Repository / snapshot structure overview

This snapshot is organized around three cooperating layers:

1. **GraphQL runtime + schema assembly** (`src/graphql/*`)
2. **Feature “modules”** (`src/modules/*`)
3. **Schema model / registration framework** (`src/schema/*`)

Together, these are what turn the core SDL + module extensions into the executable GraphQL schema the server runs.

---

#### A) `src/graphql/schema.js` — “Schema factory” and extension point

`src/graphql/schema.js` is the **composition hub** for GraphQL:

* It imports the **core SDL** (`../../config/schema/opencti.graphql`) as the baseline `typeDefs`.
* It defines a **global schema registry** inside the module:

  * `schemaTypeDefs`: starts with the core schema and is appended to over time.
  * `schemaResolvers`: starts with “global” resolvers (and other resolver sets) and is appended to over time.
* It exposes a single extension API used by modules:

  * `registerGraphqlSchema({ schema, resolver })`
    Any module “GraphQL registration” file can call this to extend the schema.

Then it builds the executable schema via `makeExecutableSchema({ typeDefs, resolvers, inheritResolversFromInterfaces: true })`, applies directive-related transformations, and returns the finished schema.

**Important structural note from this snapshot:**
`schema.js` references many resolver modules under `../resolvers/*`. Those resolver files are **not present in this archive**, which strongly suggests this snapshot is a *targeted GraphQL-layer dump* rather than a full source tree. This doesn’t prevent documenting *how schema composition works*, but it does mean deeper resolver behaviors may need additional source snapshots later.

---

#### B) `src/modules/index.ts` — “Initialization order and registration manifest”

`src/modules/index.ts` is the **orchestrator** that makes module registration happen in the correct order. It is primarily a sequence of **side-effect imports** grouped into explicit regions:

1. **Attribute registrations** (must happen first)
   A block of imports like `./attributes/*-registrationAttributes` runs before modules so that type attribute metadata exists when modules are registered.

2. **Relations-ref registrations** (must also happen early)
   A block of imports like `./relationsRef/*-registrationRef` ensures relationship reference metadata is registered before module definitions depend on it.

3. **Module registrations** (domain/model registration)
   Imports like `./channel/channel`, `./case/case`, etc. load module definitions and register them into the schema model layer.

4. **GraphQL registrations** (SDL + resolver registration into GraphQL schema)
   Imports like `./channel/channel-graphql`, `./case/case-graphql`, etc. register module SDL + resolvers via the `registerGraphqlSchema(...)` hook provided by `src/graphql/schema.js`.

There is also an explicitly labeled section of **“incomplete modules”** in this file. That’s a strong signal that some modules may be only partially implemented in this build; we should reflect that cautiously in later documentation (e.g., “present in schema but implementation may be partial”).

**In short:** `src/modules/index.ts` is both the **module inventory** *and* the **correct boot sequence** for getting a fully registered schema.

---

#### C) `src/schema/module.ts` — “ModuleDefinition contract” and registry wiring

`src/schema/module.ts` defines the core **module definition contract** and the core registration function:

* `ModuleDefinition<T, Z>` describes what a module must provide:

  * type identity/category metadata
  * identifier model
  * attribute definitions
  * relationship definitions + relationship-ref definitions
  * converters (STIX 2.1 conversion), representative field logic
  * optional validators and layout customization
* `registerDefinition(definition)` is the canonical “install this module into the platform schema model” function.

When called, `registerDefinition(...)` performs the systemic registration work, including:

* registering the module type into internal type registries (`schemaTypesDefinition`)
* registering converters and “representative” logic
* registering validators (create/update) when provided
* registering model identifiers
* registering the full attribute set (including alias-related fields when applicable)
* registering relationship mappings and relationship-ref types
* optionally registering overview layout customization
* recording the module definition in a global `modules` map

This is how module “domain registration” (the `./<module>/<module>` imports in `src/modules/index.ts`) plugs into the rest of the platform’s schema metadata model.

---

### Mental model for the snapshot

* **`src/schema/*`** defines the *data model and registries* (“what types/attributes/relationships exist, and how they behave”).
* **`src/modules/*`** provides *feature packages* that register themselves into those registries and into GraphQL.
* **`src/graphql/schema.js`** turns the collected SDL + resolvers into the **executable schema**, with directive enforcement layers applied.

That separation is the reason the documentation can (and should) be split into:

* “how schema is assembled” (this section + next sections), and
* “what the API is” (generated reference from the merged schema).

---

### 1.2 “What is the GraphQL surface area?”

The “GraphQL surface area” of Octi in this snapshot is the **complete set of types + directives + root operations** that the server exposes after **merging**:

1. the **core schema SDL** (`opt/opencti/config/schema/opencti.graphql`), and
2. **all module SDL files** (`*.graphql` under `src/modules/**`)

In other words: if it’s definable in SDL and appears in either the core schema or any module schema, it is part of the **documentable surface area**.

---

#### A) What counts as “surface area” (in this snapshot)

From the SDL set (core + all module `*.graphql`), the surface includes:

* **Directives (5)**
  `@auth`, `@public`, `@allowUnprotectedOTP`, `@allowUnlicensedLTS`, `@constraint`
  (Directives define access semantics and input validation semantics that matter to clients.)

* **Scalars (8)**
  `DateTime`, `ConstraintString`, `ConstraintNumber`, `Upload`, `StixId`, `StixRef`, `Any`, `JSON`

* **Schema constructs (snapshot totals, core+modules)**

  * **Types:** 624 total (`Query`, `Mutation`, `Subscription` included) → **621 non-root types**
  * **Interfaces:** 19
  * **Inputs:** 220
  * **Enums:** 156
  * **Unions:** 6

These numbers are a useful “size-of-surface” indicator for planning docs and reviews.

---

#### B) Root operations: how big is the API entry surface?

Across **all SDL files in this snapshot**, the root operation fields count is:

* **Query fields:** **383**
* **Mutation fields:** **438**
* **Subscription fields:** **23**

Those fields are **distributed across files**:

* From core schema (`opencti.graphql`):

  * **227 / 383** queries
  * **157 / 438** mutations
  * **17 / 23** subscriptions

* From modules (all module `*.graphql` combined):

  * **156** additional queries
  * **281** additional mutations
  * **6** additional subscriptions

**Important implementation detail (relevant to docs):**
Module SDL files commonly declare additional root fields using their own `type Query { ... }`, `type Mutation { ... }`, and sometimes `type Subscription { ... }` blocks (instead of `extend type ...`). The schema assembly layer is responsible for merging these into the final executable schema. That means **the “true” root surface area is the merged result**, not any single file.

---

#### C) Core schema “top-level navigation” (built-in groupings)

Inside `opencti.graphql`, the root types are organized with clear section headings (comment blocks) that act as a **natural documentation taxonomy**.

**Core `Query` headings (14 groups):**

* INTERNAL
* INDEXED FILES
* ENTITIES
* INTERNAL OBJECT ENTITIES
* STIX OBJECT ENTITIES
* STIX META OBJECT ENTITIES
* STIX CORE OBJECT ENTITIES
* STIX DOMAIN OBJECT ENTITIES
* STIX CYBER OBSERVABLE ENTITIES
* STIX RELATIONSHIPS
* STIX CORE RELATIONSHIPS
* STIX SIGHTING RELATIONSHIPS
* STIX REF RELATIONSHIPS
* ALL

**Core `Mutation` headings (15 groups):**

* INTERNAL
* INTERNAL OBJECT ENTITIES
* STIX OBJECT
* STIX OBJECT ENTITIES
* STIX META OBJECT ENTITIES
* STIX CORE OBJECT ENTITIES
* STIX DOMAIN OBJECT ENTITIES
* Containers
* Identities
* Locations
* STIX CYBER OBSERVABLE ENTITIES
* STIX RELATIONSHIPS
* STIX CORE RELATIONSHIPS
* STIX REF RELATIONSHIPS
* STIX SIGHTING RELATIONSHIPS

For documentation, these headings are extremely useful: they are the closest thing to an “official” grouping of the core surface.

---

#### D) Subscriptions: where real-time surface is defined

Only **six SDL files** define `type Subscription` in this snapshot:

* `opt/opencti/config/schema/opencti.graphql` (core subscriptions)
* `src/modules/managerConfiguration/managerConfiguration.graphql`
* `src/modules/workspace/workspace.graphql`
* `src/modules/ai/ai.graphql`
* `src/modules/entitySetting/entitySetting.graphql`
* `src/modules/notification/notification.graphql`

So the real-time surface is **highly concentrated**, which is helpful for targeted documentation and review.

---

#### E) Module SDL coverage: what modules expand the surface?

There are **59 module SDL files** (`*.graphql`) across **46 module families** under `src/modules/**`.

At a planning level, the best way to understand the module-added surface is to track **(a) how many root fields** each module adds, and **(b) whether it adds subscriptions**.

Here is the **module-by-module root field contribution** from SDL alone:

| Module                | SDL files | Query fields | Mutation fields | Subscription fields |
| :-------------------- | --------: | -----------: | --------------: | ------------------: |
| administrativeArea    |         1 |            2 |               7 |                   0 |
| ai                    |         1 |            0 |              14 |                   1 |
| auth                  |         1 |            0 |               4 |                   0 |
| case                  |         6 |           16 |              18 |                   0 |
| catalog               |         1 |            3 |               0 |                   0 |
| channel               |         1 |            2 |               7 |                   0 |
| dataComponent         |         1 |            2 |               7 |                   0 |
| dataSource            |         1 |            2 |               9 |                   0 |
| decayRule             |         2 |            4 |               6 |                   0 |
| deleteOperation       |         1 |            4 |               0 |                   0 |
| disseminationList     |         1 |            3 |               6 |                   0 |
| draftWorkspace        |         1 |            6 |               4 |                   0 |
| emailTemplate         |         1 |            3 |               7 |                   0 |
| entitySetting         |         1 |            3 |               1 |                   1 |
| event                 |         1 |            4 |               4 |                   0 |
| exclusionList         |         1 |            2 |               6 |                   0 |
| fintelDesign          |         1 |            3 |               6 |                   0 |
| fintelTemplate        |         1 |            1 |               8 |                   0 |
| form                  |         1 |            3 |               6 |                   0 |
| grouping              |         1 |            6 |               7 |                   0 |
| indicator             |         1 |            5 |               7 |                   0 |
| ingestion             |         5 |           14 |              23 |                   0 |
| internal              |         3 |            7 |               9 |                   0 |
| language              |         1 |            1 |               6 |                   0 |
| malwareAnalysis       |         1 |            2 |               9 |                   0 |
| managerConfiguration  |         1 |            2 |               1 |                   1 |
| metrics               |         1 |            6 |               0 |                   0 |
| narrative             |         1 |            1 |               7 |                   0 |
| notification          |         1 |           10 |              10 |                   2 |
| notifier              |         1 |            2 |               7 |                   0 |
| organization          |         1 |            3 |              10 |                   0 |
| pir                   |         1 |            6 |               6 |                   0 |
| playbook              |         1 |            4 |              14 |                   0 |
| publicDashboard       |         1 |           12 |               3 |                   0 |
| requestAccess         |         1 |            0 |               3 |                   0 |
| savedFilter           |         1 |            4 |               8 |                   0 |
| securityCoverage      |         1 |            2 |               8 |                   0 |
| securityPlatform      |         1 |            3 |               6 |                   0 |
| stixCyberObservable   |         1 |            0 |               0 |                   0 |
| support               |         1 |            2 |               7 |                   0 |
| task                  |         2 |            5 |               8 |                   0 |
| theme                 |         1 |            2 |               4 |                   0 |
| threatActorIndividual |         1 |            2 |               9 |                   0 |
| vocabulary            |         1 |            2 |               7 |                   0 |
| workspace             |         1 |            2 |               9 |                   1 |
| xtm                   |         1 |            5 |               4 |                   0 |

**Notable takeaways for documentation planning**

* The largest “root surface” expansions come from:

  * `ingestion` (37 root fields),
  * `case` (34),
  * `notification` (22),
  * `playbook` (18),
  * `internal` (16).
* `stixCyberObservable` (deprecated SDL) is **type-only** here (no root fields), so it should be documented as schema/type support rather than an API entry surface.

---

#### F) How we will document the surface area (without duplicating everything)

To keep docs usable at this scale:

1. **Generated reference will enumerate all fields/types** (the exhaustive list lives there).
2. **Curated docs will describe the shape of the surface**:

   * core groupings (the headings above),
   * module families and what they add,
   * where subscriptions live,
   * how to “find” the correct SDL source for a given operation/type.

That gives reviewers and developers a quick mental map, while still preserving a fully accurate, exhaustive reference elsewhere.

---

### 1.3 Core concepts (types, relationships, “STIX-ish” object families)

This platform treats **every stored item** as having an `entity_type` (a string) and then classifies that type into **object families** (things) and **relationship families** (edges). The “STIX-ish” model shows up as a set of *abstract* parent types (e.g., `Stix-Core-Object`, `Stix-Domain-Object`, `stix-core-relationship`, etc.) plus concrete child types (e.g., `Report`, `IPv4-Addr`, `uses`, `object-marking`, …).

Grounded in the following schema-model files:

* `src/schema/schema-types.ts`
* `src/schema/stixCoreObject.ts`
* `src/schema/stixDomainObject.ts`
* `src/schema/stixCyberObservable.ts`
* `src/schema/stixRelationship.ts`
* `src/schema/stixCoreRelationship.ts`
* `src/schema/stixRefRelationship.ts`
* `src/schema/stixEmbeddedRelationship.ts`
* `src/schema/stixSightingRelationship.ts`
* `src/schema/stixMetaObject.ts`

---

## A) The type registry that makes “families” work (`schema-types.ts`)

At the heart of classification is a lightweight **type hierarchy registry**:

* `schemaTypesDefinition.register(parent, children[])` creates a parent→children set.
* `schemaTypesDefinition.isTypeIncludedIn(type, parent)` answers “is this concrete type in that family?”
* `schemaTypesDefinition.add(parent, child|children[])` extends a family.

All of the “isStixXxx(…)” predicates below are ultimately wrappers around this registry (sometimes plus special cases).

---

## B) Object families (nodes)

### B.1 “Basic object” → “STIX object” → “STIX core object”

`stixCoreObject.ts` defines the top-level identity checks used across the platform:

* **STIX Core Object**: a type is considered a *core object* if it is:

  * a STIX Domain Object, **or**
  * a STIX Cyber Observable, **or**
  * the abstract type `Stix-Core-Object` itself.
* **STIX Object** expands that set further by including:

  * STIX Core Objects, **plus**
  * STIX Meta Objects, **plus**
  * the abstract type `Stix-Object`.
* **Basic Object** expands again by including:

  * Internal Objects, **plus**
  * STIX Objects, **plus**
  * the abstract type `Basic-Object`.

**Why this matters in practice**

* Many behaviors (UI routing, stream export eligibility, cross-entity linking rules) are implemented by checking these “family” predicates rather than listing every concrete type everywhere.

### B.2 “Exportable in stream data”

Also in `stixCoreObject.ts`, `isStixExportableInStreamData(instance)` treats something as exportable if it is:

* a STIX Object, **or**
* a STIX relationship *excluding* ref relationships (core + sighting), **or**
* one of two explicitly-allowed internal relationship types: `participate-to` and `in-pir`.

This is a key conceptual boundary: **ref relationships** are treated differently from “true” relationships for some export flows.

---

## C) STIX Domain Objects (SDOs) (`stixDomainObject.ts`)

Domain objects are the “classic” STIX-ish entities: reports, campaigns, malware, etc., plus OpenCTI’s container/case system.

### C.1 Concrete SDO types exposed by this layer

This file defines the canonical string names for the “built-in” domain objects such as:

* `Attack-Pattern`, `Campaign`, `Course-Of-Action`, `Infrastructure`, `Intrusion-Set`, `Malware`, `Tool`, `Vulnerability`, `Incident`
* Container-like SDOs: `Note`, `Observed-Data`, `Opinion`, `Report`
* Location subtypes: `City`, `Country`, `Region`, `Position`
* Identity subtypes: `Individual`, `Sector`, `System`

It also defines a couple of additional domain-level types used by the platform:

* `Data-Component`, `Data-Source`
* `Resolved-Filters`

### C.2 “Grouped” SDO families: Container / Identity / Location / Threat-Actor

A major design pattern here is: **a parent abstract type (e.g., `Container`) is registered with concrete children**, and then helpers check membership.

#### Containers

The `Container` family is registered with a container list that includes:

* the core container SDOs (`Note`, `Observed-Data`, `Opinion`, `Report`),
* module-provided container types (`Grouping`, `Feedback`, `Task`),
* and the case subtypes (imported from the case modules).

You then get:

* `isStixDomainObjectContainer(type)` → whether the type is part of the `Container` family (or is the abstract `Container` type itself).

#### Shareable containers

There is a smaller “shareable” subset:

* `Observed-Data`, `Grouping`, `Report`, and all case subtypes.

You then get:

* `isStixDomainObjectShareableContainer(type)` → true only for that curated list.

#### Identities

The `Identity` family is registered with:

* `Individual`, `Sector`, `System`

You then get:

* `isStixDomainObjectIdentity(type)` → membership (or the abstract `Identity` itself).

#### Locations

The `Location` family is registered with:

* `City`, `Country`, `Region`, `Position`

You then get:

* `isStixDomainObjectLocation(type)` → membership (or the abstract `Location` itself).

#### Threat actors

The `Threat-Actor` family is registered with:

* `Threat-Actor-Group`
* `Threat-Actor-Individual` (imported from the `threatActorIndividual` module)

You then get:

* `isStixDomainObjectThreatActor(type)` → membership (or the abstract `Threat-Actor` itself).

### C.3 What “isStixDomainObject” means here

`isStixDomainObject(type)` returns true if:

* the type is registered under the abstract `Stix-Domain-Object`, **or**
* it’s part of Identity/Location/Container/Threat-Actor families, **or**
* it is the abstract `Stix-Domain-Object` itself.

This is why you’ll see identity/location/container checks show up in many places—those families are core to the platform’s “domain object” boundary.

### C.4 Alias behavior (“aliased objects”) and which alias field to use

`stixDomainObject.ts` defines a “this type can have aliases” concept:

* A baseline list of **aliased SDOs** (e.g., `Course-Of-Action`, `Attack-Pattern`, `Campaign`, `Infrastructure`, `Intrusion-Set`, `Malware`, `Threat-Actor-Group`, `Tool`, `Incident`, `Vulnerability`).
* A hook to extend that list: `registerStixDomainAliased(type)`.

And then:

* `isStixObjectAliased(type)` is true if:

  * the type is in the aliased SDO list, **or**
  * it’s an Identity (with one notable exception), **or**
  * it’s a Location.

Finally, alias *storage field* choice is itself type-dependent:

* `resolveAliasesField(type)` chooses between the standard `aliases` attribute and the `x_opencti_aliases` variant, depending on type (identities/locations and a few other types always use the OpenCTI-flavored alias field).

**Doc implication:** “aliases” are not a uniform concept; the platform deliberately routes some families through `x_opencti_aliases`.

### C.5 Organization scoping sets

This file also defines two important type lists:

* `STIX_ORGANIZATIONS_UNRESTRICTED`: types that are considered “unrestricted” with respect to organization scoping (includes several abstract families plus a few special objects like “work”, “taxii collection”, internal file, etc.).
* `STIX_ORGANIZATIONS_RESTRICTED`: a small list of types (notably including `DeleteOperation`) that need restrictions applied even though they’re “internal-ish”.

This is a policy boundary you’ll want to reflect later in the authorization/scoping docs.

---

## D) STIX Cyber Observables (SCOs) (`stixCyberObservable.ts`)

Cyber observables are registered under the abstract family `Stix-Cyber-Observable`.

### D.1 Observable types (standard + custom)

The file defines the canonical set of observable `entity_type` strings, including:

* Standard-ish: `IPv4-Addr`, `IPv6-Addr`, `Domain-Name`, `Url`, `Email-Addr`, `Email-Message`, `Network-Traffic`, `Process`, `User-Account`, `Windows-Registry-Key`, etc.
* “Hashed observable” subfamily: `Artifact`, `StixFile` (named that way because “File” is already used), `X509-Certificate`.
* Custom extensions explicitly called out as custom (e.g., `Cryptographic-Key`, `Cryptocurrency-Wallet`, `Hostname`, `Text`, `Credential`, `User-Agent`, `Bank-Account`, `Phone-Number`, etc.).

### D.2 Hashed Observable subfamily

A subfamily `Hashed-Observable` is registered for:

* `Artifact`, `StixFile`, `X509-Certificate`

And helpers exist:

* `isStixCyberObservableHashedObservable(type)`
* `isStixCyberObservable(type)`

---

## E) STIX Meta Objects (SMOs) (`stixMetaObject.ts`)

Meta objects are registered under `Stix-Meta-Object` and include:

* `Label`
* `External-Reference`
* `Kill-Chain-Phase`
* `Marking-Definition`

A subset of these are treated as **“embedded objects”**:

* `Label`, `External-Reference`, `Kill-Chain-Phase`

`isStixMetaObject(type)` checks membership (or the abstract type itself).

**Why this matters:** meta objects are frequently connected via “ref relationships” (see below), not core relationships.

---

## F) Relationship families (edges)

### F.1 Relationship classification (`stixRelationship.ts`)

This layer defines the relationship taxonomy:

* **STIX Relationship (overall):**

  * STIX Core Relationships (SRO-like),
  * STIX Sighting Relationship,
  * STIX Ref Relationships (embedded/meta refs),
  * or the abstract `stix-relationship`.

* **STIX Relationship *except ref*:**

  * core relationships + sighting relationships only.
  * This distinction is used by stream/export code paths to treat ref relationships differently.

* **Basic Relationship:**

  * internal relationships + STIX relationships + `basic-relationship`.

It also includes a type guard:

* `isStixRelation(sco)` checks whether a given STIX core object looks like a relationship by verifying it has a `relationship_type`.

---

## G) STIX Core Relationships (SROs) (`stixCoreRelationship.ts`)

STIX core relationship types are registered under `stix-core-relationship` and include:

* **Standard STIX-like relationship types** (e.g., `uses`, `targets`, `indicates`, `mitigates`, etc.).
* **Extended relationships** explicitly annotated as extensions (OpenCTI + MITRE), including:

  * OpenCTI extensions like `part-of`, `cooperates-with`, `participates-in`, `publishes`, …
  * MITRE extensions like `subtechnique-of`, `revoked-by`, `detects`.

Helper:

* `isStixCoreRelationship(type)` → membership (or abstract).

**Doc implication:** “relationship_type” is a *controlled vocabulary* in this system, and the platform intentionally expands beyond vanilla STIX where needed.

---

## H) STIX Sighting Relationship (`stixSightingRelationship.ts`)

There is a single dedicated relationship type:

* `stix-sighting-relationship`

Helper:

* `isStixSightingRelationship(type)`.

This is treated as a first-class STIX relationship in the higher-level predicates.

---

## I) STIX Ref Relationships (embedded refs + meta refs) (`stixRefRelationship.ts`)

This is one of the most important conceptual layers: it models “relationships” that are actually **reference fields** on objects rather than standalone relationship entities.

### I.1 Two buckets: “ref relationships” and “meta relationships”

* `STIX_REF_RELATIONSHIPS`: operational “ref” edges used heavily by cyber observables (e.g., operating systems, samples, contains, resolves-to, network src/dst, email fields, process parent/child/image, etc.).
* `META_RELATIONS`: the cross-cutting “meta” refs that apply broadly across objects:

  * labeling, markings, created-by, external references, kill chain phases, object_refs, and several OpenCTI-specific meta refs (organization/assignee/participant).

Both buckets are registered under the abstract `stix-ref-relationship` family by registering each RefAttribute’s `databaseName`.

Helper:

* `isStixRefRelationship(type)`.

### I.2 What a `RefAttribute` represents

Each ref relationship is described as a `RefAttribute` object with properties like:

* `name` (GraphQL/input-facing name),
* `databaseName` (the relationship type string used internally),
* `stixName` (field name used in STIX-ish representations),
* `multiple`, `upsert`, `datable`, `isFilterable`,
* and most importantly: `toTypes` + a type-check function (`isRefExistingForTypes`) that governs which type pairs are valid.

**Doc implication:** ref relationships are not “free-form edges”; they’re *schema-governed reference attributes* with explicit constraints.

### I.3 Unidirectional meta relationships

`isStixRefUnidirectionalRelationship(type)` identifies meta-ref edges that behave unidirectionally, with a notable exception: the generic `object` (“object_refs”) relationship is treated differently.

---

## J) Embedded relationship extraction (`stixEmbeddedRelationship.ts`)

This file is about **how ref relationships are stored/extracted** from data objects, especially when refs exist in multiple “channels” (e.g., internal vs inferred):

### J.1 “Single” ref relationships

`isSingleRelationsRef(entityType, databaseName)` returns true when:

* the relationship is a STIX ref relationship, **and**
* the schema says that relationship is not “multiple” for that entity type.

### J.2 Extracting meta refs from stored objects

`instanceMetaRefsExtractor(relationshipType, isInferred, data)`:

* chooses whether to read the internal ref field or inferred ref field
* builds the storage key using `buildRefRelationKey(relationshipType, internal_id|inferred_id)`
* returns the stored array (or empty array)

### J.3 Extracting all refs from a STIX-ish object representation

`stixRefsExtractor(data)`:

* only runs if the object has an OpenCTI extension type (`extensions.x_opencti.type`)
* looks up ref field names for that type
* excludes a small set of “embedded” attributes (labels, external references, kill chain phases, multipart body)
* additionally treats the core STIX relationship attributes (`source_ref`, `target_ref`, etc.) as ref-like for extraction
* flattens all the referenced values into a single list

**Doc implication:** “refs” are treated as a *first-class export and indexing concept*, and there are rules for what is considered embedded vs extracted.

---

## Summary mental model

If you’re new to the codebase, the quickest “core concepts” map is:

* **Objects**

  * `Basic-Object`

    * `Internal-Object` (platform internals)
    * `Stix-Object`

      * `Stix-Core-Object`

        * `Stix-Domain-Object` (SDOs like Report, Malware, Incident, etc.)
        * `Stix-Cyber-Observable` (SCOs like IPv4-Addr, Domain-Name, etc.)
      * `Stix-Meta-Object` (Label, Marking-Definition, etc.)

* **Relationships**

  * `basic-relationship`

    * `internal-relationship` (platform internal edges)
    * `stix-relationship`

      * `stix-core-relationship` (relationship_type vocabulary)
      * `stix-sighting-relationship`
      * `stix-ref-relationship` (reference attributes + meta refs)

That taxonomy is what powers validation, filtering, exportability, and many UI/runtime decisions throughout the GraphQL and module layers.

---

## 2) Runtime architecture (GraphQL server)

### 2.1 Server entrypoint and lifecycle

In this snapshot, the GraphQL server “entrypoint” is expressed as a **factory function** that builds:

1. the **executable GraphQL schema**, and
2. an **Apollo Server instance** configured with the platform’s validation, security, and observability plugins.

That factory is `createApolloServer()` in `src/graphql/graphql.js`, and it depends directly on `createSchema()` from `src/graphql/schema.js`.

---

#### A) Lifecycle at a glance (what happens, in order)

When the runtime wants to bring GraphQL online, it calls:

1. **`createSchema()`** (`src/graphql/schema.js`)
   Builds a merged, directive-transformed GraphQL schema.

2. **`createApolloServer()`** (`src/graphql/graphql.js`)
   Wraps the schema in an `ApolloServer` with:

   * validation rules (batch protection, optional armor)
   * request plugins (logging, response shaping, constraint validation, introspection guard, optional tracing/metrics)
   * error formatting and logging hooks

`createApolloServer()` returns an object `{ schema, apolloServer }`, which implies that **another layer** (not present in this snapshot) is responsible for:

* attaching Apollo Server to HTTP middleware (Express/Fastify/etc.)
* providing `contextValue` (notably `contextValue.user` used for introspection authorization)
* actually calling `apolloServer.start()` and binding routes

---

#### B) Schema creation lifecycle (`src/graphql/schema.js`)

`createSchema()` is responsible for producing the **final executable schema** that Apollo runs.

**1) Inputs to the schema**

* **Core SDL baseline:** `globalTypeDefs` is imported from `../../config/schema/opencti.graphql` and becomes the first entry in `schemaTypeDefs`.

* **Schema extensions (modules):** the file provides a global registration hook:

* `registerGraphqlSchema({ schema, resolver })`

  * appends `schema` into `schemaTypeDefs`
  * appends `resolver` into `schemaResolvers`

This is the mechanism modules use to “extend” the schema from outside this file.

**2) Global scalar implementations**
The schema layer defines implementations for important scalars (these are runtime behaviors, not just SDL):

* `DateTime` (via `graphql-scalars`)
* `Upload` (via `graphql-upload`)
* `StixId` (validated STIX ID scalar)
* `StixRef` (nullable STIX reference scalar, validated like STIX ID when applicable)
* `Any` (arbitrary structured input parsed from AST)

**3) Resolver composition**
`schemaResolvers` is an array of resolver maps imported from many places (core resolvers plus module resolvers).
When building the schema, it merges them into a single resolver map via:

* `const resolvers = mergeResolvers(schemaResolvers);`

**4) Executable schema build**
The executable schema is created via:

* `makeExecutableSchema({ typeDefs: schemaTypeDefs, resolvers, inheritResolversFromInterfaces: true })`

**5) Directive and transformer lifecycle**
After building the schema, it applies directive-related behavior:

* **Constraint directive docs support:** `schema = constraintDirectiveDocumentation()(schema)`
* **Auth directive transformer:** from `authDirectiveBuilder('auth')`
* **Rate limit directive transformer:** from `rateLimitDirective({ onLimit: () => throw FunctionalError('Too many requests') })`

Then the schema is transformed in this order:

* `schema = rateLimitDirectiveTransformer(authDirectiveTransformer(schema))`

Meaning:

* auth enforcement is applied first,
* then rate-limit directive behavior is applied on top.

---

#### C) Apollo Server creation lifecycle (`src/graphql/graphql.js`)

`createApolloServer()` instantiates Apollo Server with a specific set of plugins + validation rules. The construction sequence is deliberate:

**1) Build schema**

* `const schema = createSchema();`

**2) Configure query/input validation (constraint directive plugin)**
A `graphql-constraint-directive` Apollo 4 validation plugin is created with a custom format:

* `not-blank`: throws if the string is non-empty but trims to empty/whitespace

This becomes an Apollo plugin in the plugin chain.

**3) Create the base plugin chain**
Apollo plugins begin as:

* `loggerPlugin`
* `httpResponsePlugin`
* `constraintPlugin`

(Then additional plugins are appended conditionally.)

**4) Add batching protection (alias-based anti-batch rules)**
The server uses `graphql-no-alias` to restrict “batching through aliases.”

It defines permissions per root type:

* `Query['*']` default is `conf.get('app:graphql:batching_protection:query_default') ?? 2`
* `Query.subTypes` is allowed more (`?? 4`) because clients fetch schema-related subtype info frequently
* `Mutation['*']` default is `conf.get('app:graphql:batching_protection:mutation_default') ?? 1`
* `Mutation.token` is forced to `1` (login mutation)

This produces a GraphQL validation rule (`batchValidationRule`) appended to `apolloValidationRules`.

**5) Optionally enable GraphQL Armor protections**
If `GRAPHQL_ARMOR_DISABLED` is false, it constructs an `ApolloArmor` instance and enables protections including:

* blocking field suggestions
* cost limit (`maxCost` default 3,000,000)
* max depth (default 20)
* max directives (default 20)
* max tokens (default 100,000)
* max aliases is explicitly disabled here (handled by graphql-no-alias)

Armor contributes both:

* additional Apollo plugins (`protection.plugins`)
* additional validation rules (`protection.validationRules`)

**6) Enforce “secure introspection” via a plugin**
Even though Apollo is created with `introspection: true`, a custom plugin restricts introspection at runtime:

* Detects introspection by checking `request.query?.includes('__schema')`
* Blocks introspection when:

  * playground is not enabled, or introspection is disabled via playground config
* Also requires authentication:

  * if `contextValue.user` is missing, it throws `AuthRequired()`

This is a critical behavior: introspection access is **policy-driven** and **context-driven**, not purely Apollo config-driven.

**7) Conditionally add tracing and metrics**

* If `ENABLED_TRACING`: add `tracingPlugin`
* If `ENABLED_METRICS`: add `telemetryPlugin`

**8) Add Apollo “lifecycle failure” logging hooks**
A final plugin is appended to log:

* startup failures
* context creation failures
* unexpected request processing errors

**9) Instantiate ApolloServer**
ApolloServer is created with:

* `schema`
* `introspection: true` (then restricted by plugin if needed)
* `persistedQueries: false`
* `validationRules: apolloValidationRules`
* `csrfPrevention: false` (comment: handled by helmet)
* `tracing: DEV_MODE`
* `plugins: apolloPlugins`
* a custom `logger` that routes to `logApp`
* `formatError` that:

  * rewrites `name` to `extensions.code` (compat with older client expectations)
  * strips `extensions.exception` in non-dev mode

Finally:

* `return { schema, apolloServer };`

---

#### D) What this section intentionally does *not* claim (because it isn’t in these files)

These two files do **not** show:

* the HTTP framework integration (route mounting, CORS, websocket/sse binding, etc.)
* the **context builder** (how `contextValue.user` is populated)
* how `src/modules/index.ts` is invoked during startup to ensure modules actually register their schema/resolvers

Those will be documented in later runtime sections once we cover the boot sequence wiring around module registration and context creation (if/when that code is in scope).

---

### 2.2 Request flow (context, auth, execution, response)

This section describes what happens to a GraphQL request **from the moment Apollo receives it** until the response is sent, based on:

* `src/graphql/graphql.js` (Apollo Server wiring)
* `src/graphql/httpResponsePlugin.js` (HTTP response shaping)
* `src/graphql/loggerPlugin.js` (request logging + error/audit handling)

---

#### A) Request entry and context

1. **Apollo receives the GraphQL request** (query + variables + operationName).
2. A **context object is created by the HTTP integration layer** (not shown in these files).

   * These files *assume* the context is available as `contextValue`.
   * The server expects an authenticated user to be present at `contextValue.user` for certain operations (notably introspection).

**Where this is used**

* `graphql.js` uses `contextValue.user` to allow/deny introspection.
* `loggerPlugin.js` reads `context.contextValue.user` to attach `user_id` and to emit audit events.

---

#### B) Validation & pre-execution controls

Before execution (and in early request lifecycle hooks), the server applies several protections:

1. **Batching protection (anti-alias batching)** — enabled by default
   Implemented via `graphql-no-alias` and installed into Apollo `validationRules`:

   * Query defaults:

     * `Query['*']`: config `app:graphql:batching_protection:query_default` (default `2`)
     * `Query.subTypes`: config `app:graphql:batching_protection:query_subtypes` (default `4`)
   * Mutation defaults:

     * `Mutation['*']`: config `app:graphql:batching_protection:mutation_default` (default `1`)
     * `Mutation.token`: forced to `1`

2. **Constraint validation plugin** — enabled
   Uses `graphql-constraint-directive/apollo4` and defines a custom string format:

   * `not-blank`: rejects strings that are non-empty but only whitespace.

3. **Optional GraphQL Armor protections** — conditional
   If `GRAPHQL_ARMOR_DISABLED` is false, the server enables:

   * field suggestion blocking
   * cost limit (default `3,000,000`)
   * max depth (default `20`)
   * max directives (default `20`)
   * max tokens (default `100,000`)
   * max aliases is disabled here (handled by `graphql-no-alias`)

4. **Secure introspection gate (policy + auth)** — always installed
   A custom Apollo plugin checks for introspection by testing:

   * `request.query?.includes('__schema')`

   If it’s an introspection request:

   * It is **blocked** when playground is disabled or introspection is disabled:

     * `!PLAYGROUND_ENABLED || PLAYGROUND_INTROSPECTION_DISABLED`
     * throws a *muted* `ResourceNotFoundError('GraphQL introspection not authorized!')`
   * It also requires authentication:

     * if `!contextValue?.user`, throws `AuthRequired()`

> Note: This introspection detection checks for `__schema` specifically (not `__type`). That’s important for accuracy when documenting “what counts as introspection” in this runtime.

---

#### C) Execution (resolver evaluation)

Once validation passes and the operation is allowed, Apollo executes the operation using the server schema (built earlier). From this section’s file set, the key runtime behaviors tied to execution are:

* Whether the operation is **READ vs WRITE** is inferred from:

  * `context.operation.operation === 'mutation'` (in `loggerPlugin.js`)

This affects how request metadata is computed and logged (next section).

---

#### D) Logging, auditing, and diagnostics (`loggerPlugin.js`)

The logger plugin is attached first in the plugin chain and logs **either**:

* calls that produced a non-muted error, **or**
* all calls when performance logging is enabled

Key behaviors:

1. **When it logs**

* `isCallError = context.errors && context.errors.length > 0 && !isMutedError(context.errors[0])`
* If not an error and performance logging is off (`app:performance_logger`), it returns without logging.

2. **Mutation (“WRITE”) introspection of inputs**
   If the operation is a mutation, it inspects `variables.input` to compute an “inner relation creation” count by checking presence/length for:

* `createdBy`
* `markingDefinitions`
* `labels`
* `killChainPhases`
* `objectRefs`
* `observableRefs`
* `relationRefs`

3. **Redaction of sensitive input fields**
   If configured, fields in `variables.input` are replaced with `** Redacted **` based on:

* `conf.get('app:app_logs:logs_redacted_inputs') ?? []`

4. **Extended error logging option**
   If `appLogExtendedErrors` is enabled:

* attaches full `variables` (with promise values resolved when possible)
* attaches `operation_query` with ignored characters stripped

5. **Error normalization for Apollo framework errors**
   If the top error looks like an Apollo-generated error (`extensions.code` is one of `ApolloServerErrorCode`), it wraps it as a platform `ValidationError('[OPENCTI] API GraphQL framework error', …)`.

6. **Audit event for forbidden access**
   If the error code indicates forbidden access (`FORBIDDEN_ACCESS`), it publishes a user action event via `publishUserAction(...)` with:

* scope: `unauthorized`
* access: `administration`
* includes operation name and original input variables

7. **Log severity rules**

* Auth-related errors are generally not logged as “errors” (and forbidden is audited separately).
* Functional errors are logged at **warn**.
* All other errors are logged at **error**.
* Perf logging (when enabled) logs at **info** with additional memory stats.

---

#### E) Response shaping (`httpResponsePlugin.js` + Apollo config)

1. **HTTP status behavior**
   The `httpResponsePlugin` forces HTTP status **200** when GraphQL errors exist:

* If `errors` is present, set `response.http.status = 200`.

This makes error signaling **payload-driven** (GraphQL `errors[]`) rather than HTTP-status-driven.

2. **Error formatting**
   Apollo `formatError` is configured to:

* rewrite `name` to match `extensions.code` (for client compatibility)
* strip `extensions.exception` in non-dev mode (so production responses don’t leak stack/exception details)

3. **Operational failure hooks**
   A final Apollo plugin logs lifecycle failures:

* startup failures
* context creation failures
* unexpected request processing errors

---

#### Summary: request flow in one pass

1. Request enters Apollo → context created (external) → context available as `contextValue`
2. Validation rules run (anti-alias batching; optional armor rules)
3. Operation gate runs (introspection policy + requires `contextValue.user`)
4. Execution runs
5. `willSendResponse` hooks run:

   * logging/audit/redaction logic
   * HTTP status forced to 200 on GraphQL errors
6. Error payload is formatted (name remap; exception stripping in prod) and returned

---

### 2.3 Observability (telemetry + tracing)

This GraphQL runtime has **three distinct observability layers** that work together:

1. **Structured request logging** (`loggerPlugin.js`)
2. **Metrics / telemetry emission** (`telemetryPlugin.js`)
3. **Distributed tracing spans** (`tracingPlugin.js`)

All three are implemented as **Apollo Server plugins** using the same lifecycle hook pattern (`requestDidStart` → `willSendResponse`, plus `didResolveOperation` for tracing).

---

## A) Telemetry (metrics) — `src/graphql/telemetryPlugin.js`

This plugin emits **three metric signals** per request:

* `meterManager.request(operationAttributes)` for successful calls
* `meterManager.error(operationAttributes)` for failed calls
* `meterManager.latency(elapsedMs, operationAttributes)` for all calls

### A.1 What gets tagged onto metrics

For each request, it builds an `operationAttributes` object with:

* `operation`: GraphQL operation type (defaults to `'query'` if unavailable)
* `name`: `operationName` (defaults to `'Unspecified'`)
* `status`: `'SUCCESS'` or `'ERROR'`
* `type`: only set on error; prefers the GraphQL error “name” from `singleResult.errors.at(0)?.name`, otherwise falls back to the underlying error’s `.name`
* `user_agent`: taken from `sendContext.contextValue.req.header('user-agent')`, defaulting to `'Unspecified'`

### A.2 Error classification rule (important)

Telemetry **intentionally ignores authentication-related failures** when deciding whether a call is an “error”:

* If the first error’s `name` matches any of:

  * `AUTH_REQUIRED`
  * `AUTH_FAILURE`
  * `FORBIDDEN_ACCESS`

…then `getRequestError(...)` returns `undefined`, meaning the request will be treated as **SUCCESS** for error-rate purposes (and no “error type” will be attached).

This reduces noise so auth failures don’t dominate error metrics.

### A.3 Unnamed operation warning

If `operationName` is missing, telemetry logs:

* `logApp.info('[TELEMETRY] GraphQL operation is unnamed', { query: stripIgnoredCharacters(...) })`

This is a doc-worthy runtime behavior because it’s a strong nudge toward naming operations in clients.

---

## B) Tracing — `src/graphql/tracingPlugin.js`

This plugin creates one OpenTelemetry span per GraphQL operation (when the operation successfully resolves far enough to reach `didResolveOperation`).

### B.1 When the span starts

* In `didResolveOperation(resolveContext)`:

  * Determines if the operation is a mutation:

    * mutation → treated as write
  * Maps operation type to a “DB operation”-like label:

    * mutation → `INSERT`
    * query (and others) → `SELECT`

It then starts a span:

* Span name: `"<SELECT|INSERT> <operationName>"`
* Span kind: `1` (server-style span kind)
* Attributes set at span start:

  * `'enduser.type'`: from `context.source`
  * `SEMATTRS_DB_OPERATION`: `SELECT` or `INSERT`
  * `SEMATTRS_ENDUSER_ID`: from `context.user?.origin?.user_id ?? 'anonymous'`

The plugin also calls `context.tracing.setCurrentCtx(tracingSpan)` to bind the active context.

> Practical implication: tracing relies on the presence of a `context.tracing` object that provides `getTracer()` and `setCurrentCtx(...)`, and a `context.source` string.

### B.2 What gets attached before span end

In `willSendResponse(sendContext)`:

* If `tracingSpan` exists (note the comment: “can be null for invalid operations”):

  * Computes payload size as the byte length of JSON-serialized variables:

    * `Buffer.byteLength(JSON.stringify(sendContext.request.variables || {}))`
  * Records that size in:

    * `SEMATTRS_MESSAGING_MESSAGE_PAYLOAD_COMPRESSED_SIZE_BYTES`
      (the name says “compressed”; the code uses serialized size)

### B.3 Span status semantics (and auth-noise suppression)

Just like telemetry, tracing suppresses “auth errors” as span errors:

* If the top error is `AUTH_REQUIRED`, `AUTH_FAILURE`, or `FORBIDDEN_ACCESS`, the request is treated as non-error.

Otherwise:

* On error → `tracingSpan.setStatus({ code: 2, message: requestError.name })`
* On success → `tracingSpan.setStatus({ code: 1 })`

Finally:

* `tracingSpan.end()`

---

## C) Logging (request logs + performance logs) — `src/graphql/loggerPlugin.js`

The logger plugin is the **structured “transaction log”** for GraphQL calls. It logs either:

* **Errors** (except muted errors), or
* **All calls**, when performance logging is enabled.

### C.1 When it logs (gating)

* It detects errors as:

  * `context.errors.length > 0` **and**
  * the first error is **not muted** (`!isMutedError(context.errors[0])`)
* If it’s not an error and `app:performance_logger` is false → it does nothing.

### C.2 What it records for every logged call

It computes:

* elapsed time (`time`)
* payload size (`size`) = byte length of JSON variables
* operation type:

  * `WRITE` if mutation
  * `READ` otherwise
* operation name (`operation`) default `'Unspecified'`
* user id (`user_id`) from `context.contextValue.user?.id`
* plus an “inner relation creation” count for writes (see next)

### C.3 “Inner relation creation” counting for mutations

If the request is a mutation and uses `variables.input`, it counts “inner relation creation” by checking and counting values in:

* `createdBy` (+1 if present and non-empty)
* `markingDefinitions` (count non-empty entries)
* `labels` (count non-empty entries)
* `killChainPhases` (count non-empty entries)
* `objectRefs` (count non-empty entries)
* `observableRefs` (count non-empty entries)
* `relationRefs` (count non-empty entries)

This gives a quick signal of how “relationship-heavy” a write mutation is.

### C.4 Sensitive input redaction

If configured via `app:app_logs:logs_redacted_inputs`, then for each configured field name:

* if `input[field]` is not empty → replace it with `** Redacted **` before logging.

### C.5 Extended error logging mode

If `appLogExtendedErrors` is enabled, the plugin adds:

* `variables` (after attempting to resolve any promise-valued keys)
* `operation_query` (query text with ignored characters stripped)

### C.6 Error normalization + severity + audit

On error, it:

1. Normalizes Apollo framework errors
   If `callError.extensions.code` is one of the ApolloServer error codes, it wraps it as a platform `ValidationError` with a standardized message.

2. Sets logging severity by error class:

   * Auth errors → generally not logged as error (and forbidden is audited)
   * Functional errors → `logApp.warn(...)`
   * Everything else → `logApp.error(...)`

3. Emits an audit-style user action for forbidden access
   If the error is `FORBIDDEN_ACCESS`, it publishes a user action event (scope `unauthorized`, access `administration`) including:

   * operation name
   * original input variables

### C.7 Performance log mode (non-error)

If performance logging is enabled and the call was not an error, it logs:

* message: `GRAPHQL_API`
* metadata: same call meta + `memory: getMemoryStatistics()`

(There’s also a comment indicating downstream tooling depends on that log message constant.)

---

## D) How these three layers fit together

* **Telemetry** gives you coarse operational health:

  * request rate, error rate (minus auth noise), and latency with consistent tags.
* **Tracing** gives you per-operation spans with user identity context and variable payload size.
* **Logging** gives you the most detail:

  * error normalization, severity rules, audit events for forbidden access,
  * and optionally deep request context for debugging (with redaction support).

---

### 2.4 Real-time: subscriptions & SSE

This snapshot exposes **two real-time delivery mechanisms**:

1. **GraphQL subscriptions** built on a Redis-backed PubSub async iterator (`subscriptionWrapper.ts`)
2. **Server-Sent Events (SSE)** endpoints for “streaming STIX” and live collections (`sseMiddleware.js`)

They solve different problems:

* **GraphQL subscriptions** are *application/event oriented* (notifications, workspace events, user-scoped events, settings changes).
* **SSE streaming** is *data/stream oriented* (continuous STIX-ish change feed with filters, recovery/backfill, dependency resolution).

---

## A) GraphQL subscriptions plumbing (`src/graphql/subscriptionWrapper.ts`)

This file provides a small set of reusable subscription helpers that resolvers call from their `Subscription` fields.

### A.1 Shared model: Redis PubSub iterator + server-side filtering

All subscription helpers follow the same pattern:

* Create an async iterator over one or more topics:
  `pubSubAsyncIterator(topics)`
* Wrap with `withFilter(...)` to enforce *per-event visibility*.
* Return an object that implements `Symbol.asyncIterator`.

A key behavior to preserve in docs: **payloads can be empty on disconnect**; each filter explicitly returns `false` when `payload` is falsy.

### A.2 `subscribeToUserEvents(context, topics)`

Purpose: deliver events scoped to “this user”.

Filter logic:

* Accept if `payload.instance.user_id` **or** `payload.instance.id` matches `context.user.id`.

### A.3 `subscribeToAiEvents(context, id, topics)`

Purpose: deliver AI events for a specific “bus” id to the requesting user.

Filter logic:

* Accept if `payload.user.id === context.user.id` **and** `payload.instance.bus_id === id`.

### A.4 `subscribeToInstanceEvents(parent, context, id, topics, opts)`

Purpose: deliver events for a specific entity instance, with an explicit pre-check that the user can “see” the instance.

Lifecycle & enforcement:

* Optional `preFn()` is called immediately (setup hook).
* Then it performs an access check by loading the instance via:
  `internalLoadById(context, context.user, id, { baseData: true, type })`

  * If not found/visible → throws `ForbiddenAccess('You are not allowed to listen this.')`

Filtering:

* If `notifySelf` is false (default), it suppresses echo:

  * only passes events where `payload.user.id !== context.user.id` and `payload.instance.id === id`
* If `notifySelf` is true, it allows self-notifications:

  * passes all events where `payload.instance.id === id`

Cancellation hook:

* If `cleanFn` is provided, the iterator is wrapped with `withCancel(...)` so `cleanFn()` runs when the subscription is canceled/unsubscribed (important for cleaning up per-subscription resources).

### A.5 `subscribeToPlatformSettingsEvents(context)`

Purpose: deliver *settings message* changes that matter to a specific user.

How it works:

* Subscribes to the platform settings “edit” topic:
  `BUS_TOPICS[ENTITY_TYPE_SETTINGS].EDIT_TOPIC`
* Loads the *current* settings once from cache using `SYSTEM_USER`:
  `getEntityFromCache(context, SYSTEM_USER, ENTITY_TYPE_SETTINGS)`
* On each update payload, it compares:

  * `oldMessages` = `getMessagesFilteredByRecipients(context.user, settings)`
  * `newMessages` = `getMessagesFilteredByRecipients(context.user, payload.instance)`

It emits an event **only when**:

* an activated message was removed (removed message exists and `activated`), **or**
* an existing message changes activation status, **or**
* an activated message’s text changes, **or**
* a newly-added message is activated.

**Doc implication:** This subscription is not “any settings edit”; it’s “edits that materially change what this user should see.”

---

## B) SSE live stream plumbing (`src/graphql/sseMiddleware.js`)

This file implements an HTTP middleware bundle that mounts **three endpoints** under `basePath`:

* `GET  {basePath}/stream` — “generic stream” (bypass-only)
* `GET  {basePath}/stream/:id` — “live stream collection” (public or restricted)
* `POST {basePath}/stream/connection/:id` — legacy/compat connection management

It maintains an in-process registry of active SSE clients:

* `broadcastClients[connectionId] = client`

### B.1 Authentication modes

#### Generic stream (`GET /stream`) — bypass-only

Middleware: `authenticate`

* Uses `createAuthenticatedContext(req, res, 'stream')`
* Requires `context.user` and populates on `req`:

  * `req.context`, `req.user`, `req.userId`, `req.capabilities`, `req.allowed_marking`
* Sets `req.expirationTime` to “now + 1 day”
* Hard requirement: user must have `BYPASS` capability, otherwise 401.

#### Live stream collections (`GET /stream/:id`) — public-or-restricted

Middleware: `authenticateForPublic`

* Builds context using `createAuthenticatedContext(req, res, 'stream_authenticate')`
* If no user, it assigns `SYSTEM_USER` (enables true public streams)
* Then calls `computeUserAndCollection(...)` which enforces:

Collection rules:

* `id === 'live'` is a special “global” stream **only for BYPASS** users.
* Otherwise the id must match a cached `ENTITY_TYPE_STREAM_COLLECTION` item.
* If `collection.stream_live` is false → 410 (“stopped”).
* `collection.filters` is parsed as JSON into `streamFilters`.

Access rules:

* If `collection.stream_public` → allowed (even if not authenticated).
* If not public:

  * user must be authenticated **and** have `TAXIIAPI` capability.
  * if restricted members are configured, user must be in the allowed member access ids (unless BYPASS).
  * it also pre-checks marking filters: if the stream filters require markings the user does not have, it blocks the connection (401) rather than streaming “nothing”.

### B.2 SSE connection & message format

When a connection is accepted, it creates an SSE channel and writes headers:

* `Connection: keep-alive`
* `Content-Type: text/event-stream; charset=utf-8`
* `Access-Control-Allow-Origin: *`
* `Cache-Control: no-cache, no-transform`

It also immediately sends a `connected` event with stream processor info plus `connectionId`.

SSE message lines are built like:

* optional `id: <eventId>`
* optional `event: <topic>`
* optional `data: <JSON>`
* blank line terminator

A privacy/entitlement nuance:

* For “data topics” (anything except `heartbeat`), if the user exists and does **not** have `KNOWLEDGE_ORGANIZATION_RESTRICT`, it removes:

  * `event.data.extensions[STIX_EXT_OCTI].granted_refs`
    before serializing.

### B.3 Heartbeats and liveness

Each SSE channel sets a timer:

* every `HEARTBEAT_PERIOD` ms (default 5000)

If there is a `lastEventId` and the socket buffer is empty, it emits a `heartbeat` event whose data is the ISO time derived from the `lastEventId` timestamp portion.

This is explicitly intended to keep connections alive.

### B.4 Stream processor integration

Both stream endpoints create a `processor = createStreamProcessor(userEmail, handler, opts)` with:

* `opts = { autoReconnect: true, bufferTime: 0 }`

They call:

* `await initBroadcasting(req, res, client, processor)`
* `await processor.start(startStreamId)` (generic)
* or `processor.start(recoverStreamId or startStreamId)` (live collections)

A close handler is registered on both `req` and `res`:

* removes client from `broadcastClients`
* closes channel
* shuts down the processor

### B.5 Cursoring: `from`, `last-event-id`, and recovery

Both handlers accept a “start from” cursor via:

* query param `from`, or headers `from` / `last-event-id`

Accepted formats:

* `0` or `0-0` (start-of-stream sentinel)
* an event id formatted like `<timestamp>-<seq>`
* an ISO-ish date string (converted to `<timestamp>-0`)

Live collections also support a recovery cursor:

* query param `recover`, or headers `recover` / `recover-date`

Constraint:

* recovery is only allowed if you specify a start date (`recover` without `from` throws an UnsupportedError).

### B.6 Filtering, visibility, and “dependency publishing”

Live collections are more than “dump events”; they enforce filter visibility and can publish dependencies so clients can reconstruct graphs.

Key request toggles:

* `no-dependencies=true` or `no-relationships=true` → disables dependency publishing
* `listen-delete=false` → suppresses delete events
* `with-inferences=true` → includes inferred entity/relationship indices in query & matching

Visibility handling for each event (high level):

* It checks whether the STIX object matches the stream filter group (`isStixMatchFilterGroup`).
* For UPDATE events, it reconstructs the “previous” STIX document using a reverse JSON patch:

  * `jsonpatch.applyPatch(structuredClone(stix), evenContext.reverse_patch)`
    and compares *previous visibility* vs *current visibility* to decide whether to emit:
  * a DELETE (no longer visible),
  * a CREATE (newly visible),
  * or a normal UPDATE (still visible).
* For non-visible relationship events, it may still publish them as **dependencies** if either endpoint is visible under filters.
* For non-visible entity updates, it performs a container linkage check: if the entity is referenced by a visible container (via `object` ref relationship), it may publish the event anyway.

Dependency publishing itself consists of:

* **Resolving refs** from STIX data (`stixRefsExtractor`) and loading missing referenced objects via `storeLoadByIdsWithRefs`.
* Converting store objects to STIX once and caching them to avoid duplicate conversions.
* Emitting missing items as dependency CREATE events with `origin.referer = EVENT_TYPE_DEPENDENCIES`.
* Optionally resolving and publishing **core relationships** around an object via `fullRelationsList(...)` and emitting those as dependency CREATEs too.

There is also a safety check for relationship events: it avoids publishing a relationship unless it can confirm the “from” and “to” endpoints are available (either already cached or resolvable) so clients don’t see dangling edges.

### B.7 Backfill (“recovery mode”) behavior

If `recover` is specified and is after `from`:

* it queries the database between `startIsoDate` and `recoverIsoDate` using `elList(...)` with filters converted by `convertFiltersToQueryOptions(...)`
* for each returned element:

  * publishes dependencies if enabled
  * publishes an INIT-origin CREATE event (`origin.referer = EVENT_TYPE_INIT`)
* after backfill, it starts the live processor from the recovery cursor.

---

## How this maps to documentation

When we write the final “real-time” documentation, we should present it as:

* **GraphQL Subscriptions**

  * “event topics + per-user/per-entity filtering” model
  * cancellation cleanup semantics (`cleanFn` via `withCancel`)
  * special case: settings-message diffing for user relevance

* **SSE Streams**

  * endpoints and auth modes (bypass vs public vs restricted)
  * cursoring (`from`, `last-event-id`) + recovery (`recover`)
  * filter/visibility semantics and why UPDATE can yield DELETE/CREATE
  * dependency publishing (refs + core relations) and toggles
  * operational characteristics: heartbeat, cache sizes/ttl, connection shutdown

---

### 2.4.1 SSE endpoint contract

This section documents the **Server-Sent Events (SSE)** contract implemented by `src/graphql/sseMiddleware.js`: the endpoints, authentication rules, cursoring/resume semantics, event formats, and error behavior.

> All routes are mounted under `${basePath}` (imported from `../config/conf`). In the running service, `${basePath}` is whatever your HTTP stack config sets (commonly something like `/graphql` or `/api`).

---

## A) Endpoints

### A.1 `GET ${basePath}/stream` — Generic stream (BYPASS-only)

**Purpose:** a “raw” change stream for privileged users (intended for internal/admin use).

**Auth:** required (`authenticate`) and **must** have `BYPASS` capability.

**Cursor support:** yes (`from` / `Last-Event-ID`).

---

### A.2 `GET ${basePath}/stream/:id` — Live stream collection (public or restricted)

**Purpose:** stream events for a configured **Stream Collection** (`ENTITY_TYPE_STREAM_COLLECTION`) or the special global collection `live`.

**Auth modes:**

* **Public collection** (`collection.stream_public === true`): may be consumed without authentication.
* **Restricted collection**: requires authentication *and* `TAXIIAPI` capability, and may also require membership in `collection.restricted_members`.
* Special ID: `:id === "live"` is allowed **only** for `BYPASS` users.

**Cursor support:** yes (`from` / `Last-Event-ID`), plus optional recovery backfill (`recover`).

---

### A.3 `POST ${basePath}/stream/connection/:id` — Legacy connection management (compat)

**Purpose:** retro compatibility endpoint; confirms the caller “owns” the SSE connection id.

**Auth:** required (`authenticate`).

**Behavior:**

* If the connection exists and the same authenticated user owns it → returns JSON `{ "message": "ok" }`.
* Otherwise → `401`.

---

## B) Authentication and authorization rules

### B.1 Generic stream (`GET /stream`)

* If request is unauthenticated → `401`.
* If authenticated but missing `BYPASS` → `401` with statusMessage “Consume generic stream is only authorized for bypass user”.

### B.2 Live collections (`GET /stream/:id`)

Collection resolution happens server-side via cached `ENTITY_TYPE_STREAM_COLLECTION` entities:

* If `:id === "live"`:

  * allowed only for `BYPASS`, else `401`.
* If `:id` is not found in cached collections:

  * treated as unauthorized → `401`.
* If collection exists but `collection.stream_live === false`:

  * stream is stopped → `410`.
* If collection is public (`collection.stream_public === true`):

  * allowed even without auth.
* If collection is not public:

  * must be authenticated **and** have `TAXIIAPI`, else `401`.
  * if `restricted_members` is set, user must match at least one membership id unless they have `BYPASS`, else `401`.
  * if the collection’s filters include a marking filter (`object-marking`), the user must have at least one of those markings (`allowed_marking`), else `401` with statusMessage “You need to have access to specific markings for this live stream”.

---

## C) Cursoring and recovery (resume/backfill)

### C.1 “Start from” cursor (`from` / `Last-Event-ID`)

Both `GET /stream` and `GET /stream/:id` accept a start cursor from:

* Query param: `?from=...`
* Header: `from: ...`
* Header: `last-event-id: ...` (standard SSE resume header)

**Accepted formats:**

1. **Start-of-stream sentinel**

   * `0` or `0-0` → normalized to `0-0`
2. **Event id format**

   * any string of the form `<timestamp>-<seq>` (exactly one `-`)
3. **Date string**

   * any date parseable by `utcDate(param)` (Moment-based)
   * converted to `<timestamp>-0`

If parsing fails, start cursor becomes `undefined` (meaning “start now” from the processor’s perspective).

---

### C.2 Recovery backfill cursor (`recover`)

`GET /stream/:id` additionally supports recovery/backfill via:

* Query: `?recover=...`
* Header: `recover: ...`
* Header: `recover-date: ...`

**Accepted formats:** same as `from`.

**Constraint (important):**

* Recovery requires a valid **start date** (`from`).
  If `recover` is provided but `from` is missing/unparseable, the handler throws an `UnsupportedError('Recovery mode is only possible with a start date.')` and the request ends as a server error response.

**When recovery actually runs:**

* Only if `recoverIsoDate` is present and is **after** `startIsoDate`.

During recovery, the server:

* queries historical elements between `from` and `recover`
* emits “init/backfill” create events (see **Event payload → origin.referer** below)
* then starts live streaming from the recovery cursor

---

## D) Feature toggles (query params or headers)

`GET /stream/:id` supports these optional toggles (query param OR header):

### D.1 Dependency publishing

* `no-dependencies=true` **or** `no-relationships=true`
  → disables publishing referenced objects/relationships as dependencies.

Default: dependencies **are published**.

### D.2 Deletion events

* `listen-delete=false`
  → suppresses delete events.

Default: delete events **are published**.

### D.3 Inference inclusion

* `with-inferences=true`
  → includes inferred entities + inferred relationships in the stream’s query indices.

Default: inferred data **is excluded**.

---

## E) SSE response format

### E.1 Response headers

On successful connection, the server responds `200` and sets:

* `Connection: keep-alive`
* `Content-Type: text/event-stream; charset=utf-8`
* `Access-Control-Allow-Origin: *`
* `Cache-Control: no-cache, no-transform`

### E.2 Message framing

Each SSE message is constructed as:

* optional `id: <eventId>`
* optional `event: <topic>`
* optional `data: <json>`
* blank line terminator

**`data:` is always JSON-encoded** via `JSON.stringify(...)`.

* For normal events: JSON object payloads
* For heartbeat: JSON string payload (quoted ISO timestamp)

### E.3 Event topics (`event:` values)

You should expect at least:

* `connected` (sent immediately upon connection; no `id:`)
* `heartbeat` (periodic keepalive; uses last known event id)
* data events whose `event:` matches the stream element event type:

  * commonly `create`, `update`, `delete` (internally driven by `EVENT_TYPE_CREATE/UPDATE/DELETE`)
  * live streams may also emit create events where the **origin** is dependency/init, even if triggered by update logic (see below)

### E.4 Event ids (`id:`)

* For streamed data events, `id` comes from the stream processor element `id`.
* For heartbeats, the server reuses the current `lastEventId`.
* The live collection path also ensures `eventData.event_id` is populated:

  * `event_id` from the event itself if present, otherwise computed from an “updated_at” timestamp plus element index.

### E.5 Payload shape (`data:` JSON)

The stream processor provides an event envelope. In the live stream handler, it’s treated as:

* `eventData = { type, data: stix, version, context, event_id? }`

And the SSE layer may enrich/alter it:

* sets `eventData.event_id` if missing
* for dependency/backfill publishes, it emits a `create` event with:

  * `origin.referer = "dependencies"` (dependency events)
  * `origin.referer = "init"` (recovery/backfill events)
  * `version = EVENT_CURRENT_VERSION`
  * `message = generateCreateMessage(...)`
  * `data = <STIX 2.1 object or relationship>`

### E.6 Data redaction behavior

For “data topics” (anything except `heartbeat`) **when a user exists** and the user **does not** have `KNOWLEDGE_ORGANIZATION_RESTRICT`, the server removes:

* `event.data.extensions[STIX_EXT_OCTI].granted_refs`

before serializing. This is an intentional entitlement/privacy safeguard.

---

## F) Error behavior (HTTP status)

On error, the middleware generally ends the HTTP response with a status and (optionally) `res.statusMessage`:

* `401 Unauthorized`

  * unauthenticated
  * missing required capability (`BYPASS` or `TAXIIAPI`)
  * not in restricted membership set (when configured)
  * cannot satisfy marking access required by stream filters
  * connection management attempted for someone else’s connection
* `410 Gone`

  * live stream collection exists but `stream_live === false` (“stopped”)
* `500 Internal Server Error`

  * unexpected failures, including “recovery mode without a start date”

---

## G) Minimal client expectations

A compliant client should:

1. Open an SSE connection and immediately handle a `connected` event.
2. Treat `data:` as JSON always (including heartbeat, which is a JSON string).
3. Persist the last received SSE `id:` and reconnect using `Last-Event-ID` (or `from`) for resume.
4. For live collections, optionally use `recover` for controlled backfill, but **only** when also providing `from`.

---

### 2.4.2 Subscription contracts (topics, filters, cancellation)

This section documents the **GraphQL subscription plumbing contract** implemented in:

* `src/graphql/subscriptionWrapper.ts`

At a high level, subscription resolvers in this codebase are expected to call one of these helper functions and return the resulting **AsyncIterable**. Under the hood, each helper:

* subscribes to one or more **topics** via `pubSubAsyncIterator(topics)` (Redis-backed),
* applies a **server-side filter** via `withFilter(...)` (so unauthorized/unwanted events never reach the client),
* and returns an object that implements `[Symbol.asyncIterator]()`.

---

## A) Shared expectations and payload shape

### A.1 Topics

All helpers accept `topics: string | string[]` (except the platform settings helper, which subscribes to a fixed topic). Topics are passed directly to `pubSubAsyncIterator(...)`.

> The specific topic naming conventions are defined elsewhere (bus topic config + publishing code), but the contract here is: **topic strings map to Redis PubSub channels**.

### A.2 Payload shape (what filters assume exists)

The filter functions in this file assume the published payload has (at minimum):

* `payload.instance` (the thing that changed / the event target)
* `payload.user` (the actor)
* Commonly used fields:

  * `payload.instance.id`
  * `payload.instance.user_id` (for user-scoped events)
  * `payload.user.id`
  * `payload.instance.bus_id` (for AI events)
  * `payload.instance` as a settings entity for platform settings events

### A.3 Disconnect behavior: “empty payload”

Every filter begins with the same guard:

* If `!payload`, return `false`

There is an explicit note in the code: **when disconnected, an empty payload is dispatched**. The contract is that subscription consumers should treat “no payload” as a non-event.

---

## B) `subscribeToUserEvents(context, topics)`

**Purpose:** Subscribe a user to events that belong to them.

**Contract:**

* Inputs:

  * `context.user.id` must exist
  * `topics` is one or more event topics
* Filter rule:

  * Allow if either of these equals `context.user.id`:

    * `payload.instance.user_id`
    * `payload.instance.id`

**Why two identifiers?**
This supports both patterns:

* events that encode the target user as `instance.user_id`, and
* events that publish the user record as the instance (`instance.id`).

---

## C) `subscribeToAiEvents(context, id, topics)`

**Purpose:** Subscribe a user to AI events for a specific “bus” id.

**Contract:**

* Inputs:

  * `context.user.id` must exist
  * `id` is the AI bus id the client is listening for
* Filter rule:

  * Allow only if:

    * `payload.user.id === context.user.id` **and**
    * `payload.instance.bus_id === id`

So AI events are both **user-scoped** and **bus-scoped**.

---

## D) `subscribeToInstanceEvents(parent, context, id, topics, opts)`

**Purpose:** Subscribe to events for a specific entity instance, with an upfront **access check** and optional cleanup.

### D.1 Inputs and options

* Inputs:

  * `parent` (Apollo subscription resolver parent; passed through to `withFilter`)
  * `context.user` must exist
  * `id` is the instance id to listen on
  * `topics` is one or more event topics
* Options (`opts`):

  * `preFn?: () => void` — runs immediately before checks/subscription setup
  * `cleanFn?: () => void` — runs when the subscription is canceled/unsubscribed
  * `notifySelf?: boolean` — whether to deliver events authored by the subscribing user (default `false`)
  * `type?: string | string[]` — passed into the access check to constrain allowed types

### D.2 Access check (hard requirement)

Before any subscription iterator is returned, this function loads the target instance:

* `internalLoadById(context, context.user, id, { baseData: true, type })`

If the load fails (not found OR not visible to the user), it throws:

* `ForbiddenAccess('You are not allowed to listen this.')`

**Doc implication:** subscribing to an instance is treated as a privileged action that requires the same visibility as reading the instance.

### D.3 Filter rules

After the access check, it subscribes and filters:

* Always require: `payload.instance.id === id`
* If `notifySelf` is **false** (default):

  * suppress “echo” events:

    * require `payload.user.id !== context.user.id`
* If `notifySelf` is **true**:

  * allow self-authored events as long as the instance id matches

### D.4 Cancellation cleanup (`cleanFn`)

If `cleanFn` is provided, the iterator is wrapped with a small helper (`withCancel`) that intercepts the iterator’s `.return()` call.

Effect:

* when the subscription is canceled/unsubscribed, `cleanFn()` runs.

This is the intended hook for releasing per-subscription resources (timers, in-memory registrations, etc.) created in `preFn` or elsewhere.

---

## E) `subscribeToPlatformSettingsEvents(context)`

**Purpose:** Notify a user only when **settings messages relevant to them** change in a meaningful way.

**Topic:** fixed topic:

* `BUS_TOPICS[ENTITY_TYPE_SETTINGS].EDIT_TOPIC`

### E.1 Baseline state (captured at subscribe time)

At subscription setup time, the function loads current settings from cache:

* `getEntityFromCache(context, SYSTEM_USER, ENTITY_TYPE_SETTINGS)`

This yields a baseline `settings` object used for comparison against future edits.

### E.2 Recipient-aware message filtering

For both old and new settings states, it computes:

* `oldMessages = getMessagesFilteredByRecipients(context.user, settings)`
* `newMessages = getMessagesFilteredByRecipients(context.user, payload.instance)`

So the subscription reacts only to message changes **after applying recipient targeting rules**.

### E.3 “Meaningful change” rules (when the subscription fires)

The filter returns `true` (emits an event) if any of the following is true:

1. **A message was removed and it was activated**

* It computes `removedMessage = R.difference(oldMessages, newMessages)`
* If exactly one message was removed and `removedMessage[0].activated` → emit

2. **An existing message changed activation state**

* For some `nm` in `newMessages`, if a matching `om` exists:

  * `nm.activated !== om.activated` → emit

3. **An activated message changed its text**

* For some `nm` in `newMessages`, if a matching `om` exists:

  * `nm.activated && nm.message !== om.message` → emit

4. **A new message was added and is activated**

* If no matching `om` exists for `nm`:

  * `nm.activated` → emit

**Doc implication:** this subscription is explicitly designed to reduce noise by ignoring changes that don’t affect what the user should see.

---

## F) What subscription resolvers should do (implementation contract)

A typical subscription field resolver should:

* call the appropriate helper (user-scoped, AI-scoped, instance-scoped, settings-scoped),
* return the AsyncIterable it produces,
* and keep any payload-to-output mapping in the subscription field’s `resolve` function (if used).

The helper functions here own:

* topic subscription wiring,
* filtering/authorization gates,
* and unsubscribe cleanup behavior.

If you want, the next natural follow-on is a short **“Subscription payload conventions”** section (what publishers must include in payloads: `user`, `instance`, ids) so teams adding new subscription topics don’t accidentally break these filters.

---

## 3) Schema assembly (how SDL + resolvers become the executable schema)

### 3.1 Base SDL: “core schema”

The file `opt/opencti/config/schema/opencti.graphql` is the **canonical baseline SDL** for this snapshot. It provides:

* the **global directive vocabulary** (auth/public/licensing/validation),
* the **shared primitives** (pagination, filtering, edit mutation patterns, stats/info types),
* the **root operation shells** (`Query`, `Subscription`, `Mutation`) and a very large portion of their fields,
* and many of the platform’s **core interfaces/types/unions** that modules extend and/or must remain compatible with.

It is the starting `typeDefs` input to schema assembly before module SDL is registered and merged.

---

#### A) File layout: major top-level blocks

This core SDL is intentionally structured into labeled blocks (verbatim headings in the file):

* `### DIRECTIVES`
* `### SCALAR`
* `### RELAY`
* `### EDIT`
* `### INFO`
* `### STATS`
* `### INTERFACES & TYPES`
* `### QUERIES`  → `type Query { ... }`
* `### SUBSCRIPTIONS` → `type Subscription { ... }`
* `### MUTATIONS` → mutation helper types like `*EditMutations`
* `### MUTATIONS DECLARATION` → `type Mutation { ... }`

That “MUTATIONS vs MUTATIONS DECLARATION” split is important: the core schema defines a **large set of mutation-return helper types** (the `*EditMutations` pattern) before it finally declares the root `Mutation` type.

---

#### B) Global directives: access control + validation

The directive vocabulary defined at the top of the file is:

```graphql
directive @auth(for: [Capabilities] = [], and: Boolean = false) on OBJECT | FIELD_DEFINITION
directive @public on OBJECT | FIELD_DEFINITION
directive @allowUnprotectedOTP on OBJECT | FIELD_DEFINITION
directive @allowUnlicensedLTS on OBJECT | FIELD_DEFINITION
directive @constraint(...) on INPUT_FIELD_DEFINITION
```

Key points for later docs:

* `@auth` is the central policy mechanism. It takes a list of `Capabilities` and an `and` flag that controls whether capability requirements behave like **OR** (default) or **AND**.
* `@public` marks a small set of schema fields as unauthenticated, but several `@public` fields are annotated in-schema as **“Code protected”** (meaning you should treat “public in SDL” as “public *in principle*; verify runtime enforcement”).
* `@allowUnprotectedOTP` and `@allowUnlicensedLTS` are field-level escape hatches used to gate behavior around OTP and licensing modes.
* `@constraint` defines input-field constraints which are enforced at runtime by the constraint directive validation plugin.

**Notable nuance:** the SDL *uses* `@rateLimit(...)` at least once (on an OTP-related mutation), but **does not define** `directive @rateLimit` in this file. In this snapshot, that implies the `@rateLimit` directive is **injected/handled by runtime schema transformation** rather than being declared in the core SDL.

---

#### C) Custom scalars and foundational enums

The core file declares the platform’s scalar surface:

* `DateTime`, `ConstraintString`, `ConstraintNumber`, `Upload`, `StixId`, `StixRef`, `Any`, `JSON`

It also declares key enums that drive cross-cutting behavior; most importantly:

* `Capabilities` — a large enum listing the permission “atoms” used throughout `@auth(for: [...])`.

(These become central reference points in docs because they’re referenced across *hundreds* of fields.)

---

#### D) Shared primitives that shape almost every operation

Even before you get to `Query`, the core schema defines reusable building blocks that become “house style” across the API:

1. **Relay-like pagination**

* `PageInfo` plus `*Connection` / `*Edge` patterns are used broadly.
* Many list queries follow a consistent signature pattern with arguments like:

  * `first`, `after`, `orderBy`, `orderMode`, `filters`, `search`

2. **Filtering model**

* A recursive `FilterGroup` input (`mode`, `filters`, nested `filterGroups`) plus a `Filter` input (`key`, `values`, `operator`, `mode`).
* This is the fundamental “query language” used across most list endpoints.

3. **Edit mutation model**

* Generic edit inputs (`EditInput`, `EditOperation`, etc.) appear early and are then reused by dozens of mutation helper types.
* The schema strongly favors “edit mutation objects” (`XxxEditMutations`) that bundle operations like patching fields, managing relationships, deleting, etc., behind a single `...Edit(id: ...)` entrypoint.

4. **Info / stats support**

* There are dedicated “INFO” and “STATS” type families (application info, memory stats, queue metrics, distributions/time series, etc.) that support admin/ops-facing queries.

---

#### E) Root operation entry surface in the core schema

The root operation blocks are defined in-file as:

* `### QUERIES` → `type Query { ... }`
* `### SUBSCRIPTIONS` → `type Subscription { ... }`
* `### MUTATIONS DECLARATION` → `type Mutation { ... }`

In this snapshot’s core SDL specifically, the root field counts (from the `type ... {}` blocks in this file alone) are:

* `Query`: **225** fields
* `Subscription`: **17** fields
* `Mutation`: **157** fields

Modules add additional root fields beyond this baseline (covered in later schema assembly sections), so the **final surface area** must always be treated as “core + modules merged”.

**Access pattern:** the vast majority of root fields in the core schema are guarded with `@auth(...)`. A small number are `@public` (e.g., login/token, public settings, some feed/stream/taxii list endpoints), and several of those are explicitly commented as “Code protected”.

---

#### F) Curated unions that must stay in sync with modules

The core schema defines large “umbrella” unions used throughout the API for polymorphic returns (e.g., “any STIX object or relationship”). These unions enumerate many concrete types explicitly and therefore represent a **manual maintenance surface**.

Examples present in this file include:

* `StixCoreObjectOrStixCoreRelationship`
* `StixObjectOrStixRelationship`
* `StixObjectOrStixRelationshipOrCreator`

These unions are one of the main reasons the core SDL remains central even in a “moduleized” system: modules may add types, but the unions often still require updating in `opencti.graphql` to keep polymorphic queries complete.

---

#### G) How we will use this file in the documentation set

* Treat `opencti.graphql` as the **authoritative baseline** for:

  * directive semantics and global scalars,
  * shared primitives (filters/pagination/edit patterns),
  * core root operation taxonomy and headings.
* Treat the merged schema (core + module SDL) as the **authoritative API reference**, generated later.
* Call out explicitly where runtime behavior extends the SDL (e.g., directives handled by transformers like `@rateLimit`).

---

### 3.2 Module SDL + module registration (what gets included)

In this snapshot, **module inclusion is explicit and ordered**: nothing is “auto-discovered.” The runtime gets its module set from **side-effect imports** in `src/modules/index.ts`, which is effectively the **manifest** for:

1. what **schema-model modules** are registered (types/attributes/relationships), and
2. what **GraphQL SDL + resolvers** are registered into the executable schema.

---

## A) The inclusion mechanism

### A.1 Two different “registration lanes”

`src/modules/index.ts` drives two distinct registration flows:

1. **Schema-model registration (domain/type metadata)**

   * Imported via `import './<module>/<module>';` (and some nested module entrypoints).
   * These entrypoints typically call `registerDefinition(...)` (defined in the schema layer) to register:

     * entity type identifiers,
     * attributes,
     * relationship mappings,
     * converters/representative logic, etc.
   * In this snapshot, there are **66** module imports in the “registration modules” region, including **14 marked as incomplete**.

2. **GraphQL registration (SDL + resolvers)**

   * Imported via `import './<module>/<module>-graphql';`
   * Each `*-graphql.ts` consistently calls `registerGraphqlSchema({ schema, resolver })` from `src/graphql/schema.js`.
   * Those files import their SDL as a `.graphql` module (string) and bind it to resolvers.

### A.2 Order matters (and is encoded in the manifest)

`src/modules/index.ts` is structured into regions and the comments are operational requirements:

* **registration attributes** (must be imported before modules)
* **registration ref** (must be imported before modules)
* **registration modules** (must be imported before GraphQL registration)
* **graphql registration**

This ordering is how the runtime ensures “type metadata exists before modules” and “module model exists before GraphQL hooks register.”

---

## B) What GraphQL module SDL is actually included

A key accuracy point: **not every `*.graphql` file under `src/modules/**` is automatically included**. Module SDL becomes part of the executable schema **only if** it is referenced by a `*-graphql.ts` file that is **imported by `src/modules/index.ts`.**

In this snapshot:

* There are **59** `*.graphql` files under `src/modules/**`.
* **57** of them are *actually wired in* via `src/modules/index.ts` → `*-graphql.ts` → `registerGraphqlSchema(...)`.
* **2** exist but are not wired in (deprecated/unreferenced; listed below).

### B.1 Wired module SDL list (57) and where it’s registered

Below is the *authoritative “included module SDL”* set for this snapshot (each SDL file is registered by the corresponding `*-graphql.ts`):

* **administrativeArea**

  * `src/modules/administrativeArea/administrativeArea.graphql` (registered by `src/modules/administrativeArea/administrativeArea-graphql.ts`)
* **ai**

  * `src/modules/ai/ai.graphql` (registered by `src/modules/ai/ai-graphql.ts`)
* **auth**

  * `src/modules/auth/auth.graphql` (registered by `src/modules/auth/auth-graphql.ts`)
* **case**

  * `src/modules/case/case.graphql` (registered by `src/modules/case/case-graphql.ts`)
  * `src/modules/case/case-template/case-template.graphql` (registered by `src/modules/case/case-template/case-template-graphql.ts`)
  * `src/modules/case/case-incident/case-incident.graphql` (registered by `src/modules/case/case-incident/case-incident-graphql.ts`)
  * `src/modules/case/case-rfi/case-rfi.graphql` (registered by `src/modules/case/case-rfi/case-rfi-graphql.ts`)
  * `src/modules/case/case-rft/case-rft.graphql` (registered by `src/modules/case/case-rft/case-rft-graphql.ts`)
  * `src/modules/case/feedback/feedback.graphql` (registered by `src/modules/case/feedback/feedback-graphql.ts`)
* **catalog**

  * `src/modules/catalog/catalog.graphql` (registered by `src/modules/catalog/catalog-graphql.ts`)
* **channel**

  * `src/modules/channel/channel.graphql` (registered by `src/modules/channel/channel-graphql.ts`)
* **dataComponent**

  * `src/modules/dataComponent/dataComponent.graphql` (registered by `src/modules/dataComponent/dataComponent-graphql.ts`)
* **dataSource**

  * `src/modules/dataSource/dataSource.graphql` (registered by `src/modules/dataSource/dataSource-graphql.ts`)
* **decayRule**

  * `src/modules/decayRule/decayRule.graphql` (registered by `src/modules/decayRule/decayRule-graphql.ts`)
  * `src/modules/decayRule/exclusions/decayExclusionRule.graphql` (registered by `src/modules/decayRule/exclusions/decayExclusionRule-graphql.ts`)
* **deleteOperation**

  * `src/modules/deleteOperation/deleteOperation.graphql` (registered by `src/modules/deleteOperation/deleteOperation-graphql.ts`)
* **disseminationList**

  * `src/modules/disseminationList/disseminationList.graphql` (registered by `src/modules/disseminationList/disseminationList-graphql.ts`)
* **draftWorkspace**

  * `src/modules/draftWorkspace/draftWorkspace.graphql` (registered by `src/modules/draftWorkspace/draftWorkspace-graphql.ts`)
* **emailTemplate**

  * `src/modules/emailTemplate/emailTemplate.graphql` (registered by `src/modules/emailTemplate/emailTemplate-graphql.ts`)
* **entitySetting**

  * `src/modules/entitySetting/entitySetting.graphql` (registered by `src/modules/entitySetting/entitySetting-graphql.ts`)
* **event**

  * `src/modules/event/event.graphql` (registered by `src/modules/event/event-graphql.ts`)
* **exclusionList**

  * `src/modules/exclusionList/exclusionList.graphql` (registered by `src/modules/exclusionList/exclusionList-graphql.ts`)
* **fintelDesign**

  * `src/modules/fintelDesign/fintelDesign.graphql` (registered by `src/modules/fintelDesign/fintelDesign-graphql.ts`)
* **fintelTemplate**

  * `src/modules/fintelTemplate/fintelTemplate.graphql` (registered by `src/modules/fintelTemplate/fintelTemplate-graphql.ts`)
* **form**

  * `src/modules/form/form.graphql` (registered by `src/modules/form/form-graphql.ts`)
* **grouping**

  * `src/modules/grouping/grouping.graphql` (registered by `src/modules/grouping/grouping-graphql.ts`)
* **indicator**

  * `src/modules/indicator/indicator.graphql` (registered by `src/modules/indicator/indicator-graphql.ts`)
* **ingestion**

  * `src/modules/ingestion/ingestion-rss.graphql` (registered by `src/modules/ingestion/ingestion-rss-graphql.ts`)
  * `src/modules/ingestion/ingestion-taxii.graphql` (registered by `src/modules/ingestion/ingestion-taxii-graphql.ts`)
  * `src/modules/ingestion/ingestion-taxii-collection.graphql` (registered by `src/modules/ingestion/ingestion-taxii-collection-graphql.ts`)
  * `src/modules/ingestion/ingestion-csv.graphql` (registered by `src/modules/ingestion/ingestion-csv-graphql.ts`)
  * `src/modules/ingestion/ingestion-json.graphql` (registered by `src/modules/ingestion/ingestion-json-graphql.ts`)
* **internal**

  * `src/modules/internal/csvMapper/csvMapper.graphql` (registered by `src/modules/internal/csvMapper/csvMapper-graphql.ts`)
  * `src/modules/internal/jsonMapper/jsonMapper.graphql` (registered by `src/modules/internal/jsonMapper/jsonMapper-graphql.ts`)
* **language**

  * `src/modules/language/language.graphql` (registered by `src/modules/language/language-graphql.ts`)
* **malwareAnalysis**

  * `src/modules/malwareAnalysis/malwareAnalysis.graphql` (registered by `src/modules/malwareAnalysis/malwareAnalysis-graphql.ts`)
* **managerConfiguration**

  * `src/modules/managerConfiguration/managerConfiguration.graphql` (registered by `src/modules/managerConfiguration/managerConfiguration-graphql.ts`)
* **metrics**

  * `src/modules/metrics/metrics.graphql` (registered by `src/modules/metrics/metrics-graphql.ts`)
* **narrative**

  * `src/modules/narrative/narrative.graphql` (registered by `src/modules/narrative/narrative-graphql.ts`)
* **notification**

  * `src/modules/notification/notification.graphql` (registered by `src/modules/notification/notification-graphql.ts`)
* **notifier**

  * `src/modules/notifier/notifier.graphql` (registered by `src/modules/notifier/notifier-graphql.ts`)
* **organization**

  * `src/modules/organization/organization.graphql` (registered by `src/modules/organization/organization-graphql.ts`)
* **pir**

  * `src/modules/pir/pir.graphql` (registered by `src/modules/pir/pir-graphql.ts`)
* **playbook**

  * `src/modules/playbook/playbook.graphql` (registered by `src/modules/playbook/playbook-graphql.ts`)
* **publicDashboard**

  * `src/modules/publicDashboard/publicDashboard.graphql` (registered by `src/modules/publicDashboard/publicDashboard-graphql.ts`)
* **requestAccess**

  * `src/modules/requestAccess/requestAccess.graphql` (registered by `src/modules/requestAccess/requestAccess-graphql.ts`)
* **savedFilter**

  * `src/modules/savedFilter/savedFilter.graphql` (registered by `src/modules/savedFilter/savedFilter-graphql.ts`)
* **securityCoverage**

  * `src/modules/securityCoverage/securityCoverage.graphql` (registered by `src/modules/securityCoverage/securityCoverage-graphql.ts`)
* **securityPlatform**

  * `src/modules/securityPlatform/securityPlatform.graphql` (registered by `src/modules/securityPlatform/securityPlatform-graphql.ts`)
* **support**

  * `src/modules/support/support.graphql` (registered by `src/modules/support/support-graphql.ts`)
* **task**

  * `src/modules/task/task.graphql` (registered by `src/modules/task/task-graphql.ts`)
  * `src/modules/task/task-template/task-template.graphql` (registered by `src/modules/task/task-template/task-template-graphql.ts`)
* **theme**

  * `src/modules/theme/theme.graphql` (registered by `src/modules/theme/theme-graphql.ts`)
* **threatActorIndividual**

  * `src/modules/threatActorIndividual/threatActorIndividual.graphql` (registered by `src/modules/threatActorIndividual/threatActorIndividual-graphql.ts`)
* **vocabulary**

  * `src/modules/vocabulary/vocabulary.graphql` (registered by `src/modules/vocabulary/vocabulary-graphql.ts`)
* **workspace**

  * `src/modules/workspace/workspace.graphql` (registered by `src/modules/workspace/workspace-graphql.ts`)
* **xtm**

  * `src/modules/xtm/hub/xtm-hub.graphql` (registered by `src/modules/xtm/hub/xtm-hub-graphql.ts`)

### B.2 Present-but-not-wired (2 deprecated SDL files)

These `*.graphql` files exist in the archive but are **not included** in the executable schema because nothing in `src/modules/index.ts` imports the corresponding registration module:

* `src/modules/internal/csvMapper/deprecated/csvMapper.graphql` (has a deprecated registration file, but it is not imported)
* `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql` (same situation)

---

## C) “GraphQL-only” vs “full module” inclusion

From `src/modules/index.ts`, there are **six GraphQL registrations that do not have a matching module-definition import** (i.e., they appear to be GraphQL feature packages rather than schema-model modules):

* `catalog`, `ai`, `requestAccess`, `auth`, `metrics`, `xtm/hub`

Everything else in the GraphQL registration list has both:

* a **module registration import** (domain/model lane), and
* a **GraphQL registration import** (SDL+resolver lane).

---

## D) Modules registered without module SDL (and the “incomplete modules” marker)

Also from `src/modules/index.ts`, the following are imported in the module-registration region but **do not have a `*-graphql` registration import** in this snapshot:

* `internal/document/document` (registered module code, no module SDL wired in here)
* plus the explicitly labeled **“incomplete modules”** block (14 imports):

  * report, note, externalReference, incident, observedData, stixCyberObservable, threatActorGroup, intrusionSet, campaign, malware, tool, vulnerability, attackPattern, courseOfAction

**Interpretation for documentation:** these types/operations likely live primarily in the **core schema** (`opencti.graphql`) and/or core resolver layer rather than being expressed as module SDL in this snapshot. The “incomplete modules” label is a warning flag for us: treat their behavior as higher-risk for drift and verify against runtime/resolvers when we document them later.

---

### 3.3 Resolver integration + schema build/merge points

This snapshot builds the **executable schema** by **accumulating** (a) SDL strings and (b) resolver maps into two global arrays, then performing a single **merge + build** step.

**Primary merge/build file:** `src/graphql/schema.js`
**Primary module inputs:** module resolver maps under `src/modules/**` (typically `*-resolver.ts` / `*-resolvers.ts`, plus 2 legacy `*.js` in deprecated folders)

---

## A) The two accumulation arrays (the real “integration points”)

Inside `src/graphql/schema.js` there are two global arrays:

* `schemaTypeDefs` — starts with **core SDL** only:

  * `const schemaTypeDefs = [globalTypeDefs];`
* `schemaResolvers` — starts with a large list of **core resolver maps** imported from `../resolvers/*` (note: those core resolver source files are *not present* in this snapshot archive, but they are referenced here).

### A.1 Module registration hook: `registerGraphqlSchema(...)`

Modules add to the schema via a single exported function:

* `registerGraphqlSchema({ schema, resolver })`

  * `schemaTypeDefs.push(schema)`
  * `schemaResolvers.push(resolver)`

This is the central “merge point” for **all module SDL + resolvers**.

---

## B) SDL merge: what actually becomes “the schema text”

When `createSchema()` runs, `makeExecutableSchema(...)` receives `typeDefs: schemaTypeDefs`.

In this snapshot, the SDL inputs are:

1. `globalTypeDefs` (from `opt/opencti/config/schema/opencti.graphql`)
2. `rateLimitDirectiveTypeDefs` (injected in `schema.js` via `graphql-rate-limit-directive`)
3. Every module SDL string registered via `registerGraphqlSchema(...)` (wired by `src/modules/index.ts` importing each `*-graphql.ts`)

**Important nuance:** `schema.js` imports `constraintDirectiveTypeDefs` but does not append it to `schemaTypeDefs`. That’s consistent with the core SDL already declaring `@constraint` itself.

---

## C) Resolver merge: how module resolvers get combined

### C.1 Merge step

`createSchema()` performs exactly one resolver merge:

* `const resolvers = mergeResolvers(schemaResolvers);`

…and then builds the schema:

* `makeExecutableSchema({ typeDefs: schemaTypeDefs, resolvers, inheritResolversFromInterfaces: true })`

### C.2 Why order matters

Because modules are integrated by **pushing** onto arrays, **import order becomes precedence order**.

* The **core resolvers** are loaded into `schemaResolvers` first (hard-coded list in `schema.js`).
* Module resolvers are appended later when `src/modules/index.ts` imports each `*-graphql.ts`, which in turn calls `registerGraphqlSchema(...)`.

So if two resolver maps define the same resolver key path (e.g., `Query.someField`), the effective behavior depends on merge resolution order. Practically: **avoid collisions** unless you are intentionally overriding.

---

## D) Post-build schema transforms (more “merge points,” but at schema level)

After `makeExecutableSchema(...)`, `createSchema()` applies schema-level transforms:

* `constraintDirectiveDocumentation()(schema)` (documentation/introspection enhancements around constraints)
* `authDirectiveTransformer` (from `authDirectiveBuilder('auth')`)
* `rateLimitDirectiveTransformer` (from `rateLimitDirective(...)`)

Applied as:

* `schema = rateLimitDirectiveTransformer(authDirectiveTransformer(schema))`

Meaning: **auth wrapping occurs first**, then **rate-limit wrapping** is applied on top.

This matters for docs because some behaviors (auth checks, rate limits) are **not visible in SDL alone**—they are enforced by these transformers.

---

## E) What module resolver files look like (the dominant conventions)

Across modules, resolver files are almost always TypeScript and export a single resolver map:

* `const <module>Resolvers: Resolvers = { ... }`
* `export default <module>Resolvers;`
* They import `Resolvers` from `../../generated/graphql` (the generated types are referenced but not included in this snapshot).

### E.1 Common shapes you’ll see

1. **Root fields**

   * `Query: { ... }`
   * `Mutation: { ... }`
   * `Subscription: { ... }` (when present, typically `{ subscribe, resolve }` pairs)

2. **Type-level resolvers**

   * e.g., `Grouping: { securityCoverage: (...) => ... }`
   * used for computed fields, cross-module joins, or relationship conveniences.

3. **Thin-controller pattern**

   * Most resolver functions delegate immediately to a “domain” function:

     * `findById(context, context.user, id)`
     * `addX(context, context.user, input)`
     * etc.
   * Many of those domain files import from `../../database/*` and other layers that are **not present** in this snapshot, so this archive is best for documenting *GraphQL integration and API shape*, not the full persistence implementation.

### E.2 Subscription plumbing convention

When a module defines subscriptions (e.g., notifications), it typically delegates subscription wiring to helpers in `src/graphql/subscriptionWrapper.ts` such as `subscribeToUserEvents(...)`.

This yields consistent filtering behavior across modules (user-scoped events, instance-scoped events, etc.).

---

## F) Inventory notes from the archive (useful for Appendix A cross-checking)

* Module resolver files matching `*-resolver(s).ts|js`: **59** total in the archive

  * **57** TypeScript (active modules)
  * **2** legacy JS in deprecated folders:

    * `src/modules/internal/csvMapper/deprecated/csvMapper-resolver.js`
    * `src/modules/stixCyberObservable/deprecated/stixCyberObservable-resolver.js`
* Module GraphQL wiring files `*-graphql.ts`: **57** total
  These are the ones that actually call `registerGraphqlSchema(...)` and therefore control whether a module’s resolvers participate in the executable schema.

---

## G) Practical “doc accuracy” checklist for this section

When we later write the canonical “how resolver integration works” page, we should verify (per module):

1. The module is **wired** in `src/modules/index.ts` (imports `./<module>/<module>-graphql`)
2. The module’s `*-graphql.ts` **registers** the correct SDL and resolver map
3. The resolver map includes:

   * all root fields declared in that module SDL (`Query/Mutation/Subscription`)
   * any type-level resolvers required for fields declared in module types
4. Any auth/rate-limit directives used in module SDL are consistent with the schema transformers’ expectations (notably `@auth` and `@rateLimit`)

---

## 4) Authorization & directives

### 4.1 Directive definitions (auth/public/etc.)

This snapshot’s **core directive vocabulary** is defined at the very top of `opt/opencti/config/schema/opencti.graphql` under the `### DIRECTIVES` heading. In this file, there are **exactly five** custom directives defined:

---

#### `@auth`

**SDL signature (as defined in `opencti.graphql`):**

* `@auth(for: [Capabilities] = [], and: Boolean = false)`
* **Locations:** `OBJECT | FIELD_DEFINITION`

**What the definition tells us (from SDL alone):**

* `@auth` can guard either:

  * an entire object type (all fields inherit the directive semantics via schema wiring), or
  * a specific field.
* It accepts:

  * `for`: a list of `Capabilities` (defaults to `[]`)
  * `and`: a boolean flag (defaults to `false`)

**Doc note (important for correctness):**

* The *actual enforcement behavior* (what an empty `for: []` means, OR vs AND semantics, how capabilities are checked, etc.) is implemented by the runtime directive transformer applied during schema construction (i.e., enforced in code, not by SDL alone). This section should treat the SDL as the **contract surface**; enforcement rules belong in the next “behavior” subsection.

---

#### `@public`

**SDL signature:**

* `@public`
* **Locations:** `OBJECT | FIELD_DEFINITION`

**What the definition tells us:**

* Marks a type or field as “public” at the schema layer (i.e., intended to be accessible without the usual authorization gating).
* As with `@auth`, this can be placed at the type level or the field level.

**Doc note:**

* In this schema file, some `@public` usage is accompanied by comments like “code protected” (meaning the SDL declares it public, but the implementation may still apply runtime checks). So: **treat `@public` as intent**, and document any runtime overrides separately in the implementation guide.

---

#### `@allowUnprotectedOTP`

**SDL signature:**

* `@allowUnprotectedOTP`
* **Locations:** `OBJECT | FIELD_DEFINITION`

**What it signals (at the contract level):**

* This is an explicit escape-hatch marker for OTP-related behavior.
* It can be applied to a type or field, just like `@auth` / `@public`.

**Doc note:**

* This directive’s *meaning is policy-driven* (implemented in runtime logic / directive handling). The SDL only tells us that the marker exists and where it can appear.

---

#### `@allowUnlicensedLTS`

**SDL signature:**

* `@allowUnlicensedLTS`
* **Locations:** `OBJECT | FIELD_DEFINITION`

**What it signals:**

* A licensing/LTS-related escape-hatch marker.
* Like the other auth-ish directives, it can annotate either a whole type or a specific field.

**Doc note:**

* Same as above: enforcement is runtime-driven; the SDL is the “where can this apply” contract.

---

#### `@constraint`

**SDL signature (arguments + locations):**

* `@constraint(...)`
* **Location:** `INPUT_FIELD_DEFINITION` (input fields only)

**Arguments (as defined in `opencti.graphql`):**

String-oriented constraints:

* `minLength: Int`
* `maxLength: Int`
* `startsWith: String`
* `endsWith: String`
* `notContains: String`
* `pattern: String`
* `format: String`

Number-oriented constraints:

* `min: Int`
* `max: Int`
* `exclusiveMin: Int`
* `exclusiveMax: Int`
* `multipleOf: Int`

**What it means for clients:**

* This directive is part of the API contract for validating inputs (arguments/fields inside input objects).
* The SDL expresses *which* constraints exist and their parameter types; runtime validation is handled by the server’s constraint-validation plugin (documented in the runtime section).

**Important nuance to capture later:**

* The runtime adds at least one custom `format` rule (`not-blank`) via the Apollo constraint plugin configuration—so the **set of formats may be richer than the schema file implies**.

---

### “Directive surface” note: `@rateLimit` is used but not defined here

Even though `opencti.graphql` defines only the five directives above, the schema **does contain at least one usage of** `@rateLimit(...)` (e.g., on an OTP-related mutation). That directive is **not defined** in this SDL file—so it must come from **runtime schema assembly** (typeDefs injected during schema build). This is worth calling out in the final docs so readers don’t go hunting for a missing directive definition in `opencti.graphql`.

---

### 4.2 Auth directive enforcement (how it’s applied at runtime)

**Files reviewed (snapshot):**

* `opt/opencti/src/graphql/authDirective.ts`
* `opt/opencti/src/graphql/schema.js`

---

## A) Where enforcement is applied in the schema build

In `src/graphql/schema.js`, the auth directive logic is not “automatic GraphQL behavior” — it’s applied as a **schema transform step** after the executable schema is created:

1. Build the base executable schema:

* `makeExecutableSchema({ typeDefs: schemaTypeDefs, resolvers, inheritResolversFromInterfaces: true })`

2. Apply transforms in this order:

* `schema = constraintDirectiveDocumentation()(schema);`
* `schema = rateLimitDirectiveTransformer(authDirectiveTransformer(schema));`

So **auth enforcement is wrapped first**, and then **rate limiting** is layered on top of the auth-wrapped schema.

Also in `schema.js`, the `@rateLimit` directive SDL is injected into the schema build via:

* `const { rateLimitDirectiveTypeDefs, rateLimitDirectiveTransformer } = rateLimitDirective(...)`
* `schemaTypeDefs.push(rateLimitDirectiveTypeDefs)`

That’s why `@rateLimit` can exist in the merged schema even though it isn’t defined in `opencti.graphql`.

---

## B) How the auth transformer works (schema-level rewrite)

`src/graphql/authDirective.ts` exports:

* `authDirectiveBuilder(directiveName: string)`

`schema.js` calls it as:

* `const { authDirectiveTransformer } = authDirectiveBuilder('auth');`

### B.1 Type-level vs field-level directive resolution

The transformer does two passes via `mapSchema(...)`:

1. **Collect type-level `@auth` args**:

   * For any type with `@auth(...)`, it stores the directive arguments keyed by `type.name`.

2. **Wrap field resolvers** (object fields):

   * For each field, it looks for `@auth` on the field.
   * If missing, it falls back to the **type-level** `@auth` for that object type (if present).
   * If an `@auth` directive is found (field or type), it replaces `fieldConfig.resolve` with a wrapper that enforces auth checks at runtime.

### B.2 “Secure schema” enforcement for Query/Mutation fields

There’s an explicit schema-hardening rule:

* If a field is on `Query` or `Mutation` and has **neither**:

  * an `@auth` directive (field-level or type-level), **nor**
  * a `@public` directive (field-level only),
* …then schema build **throws**:

> `UnsupportedError('Unsecure schema: missing auth or public directive', { field: _fieldName })`

Practical implication: the server will refuse to start with an “unprotected” root query/mutation field.

**Important nuance:** this check looks for `@public` only on the **field**, not on the parent type.

---

## C) What the runtime wrapper enforces for `@auth` fields

When a field is governed by `@auth`, the wrapper runs these checks in order.

### C.1 Authentication is always required

If `context.user` is missing:

* throws `AuthRequired()`

Even if `@auth(for: [])` is used (empty capability list), you still must be logged in — `@auth` with an empty list means “authenticated-only”, not “open.”

### C.2 OTP enforcement (unless the field opts out)

The wrapper reads OTP-related context flags:

* `otp_mandatory`
* `user_otp_validated`
* plus `user.otp_activated`

By default, it enforces OTP rules. A field can opt out by using:

* `@allowUnprotectedOTP`

If the field is **not** annotated with `@allowUnprotectedOTP`, then:

* If the platform mandates OTP (`otp_mandatory`):

  * If the session is not OTP-validated (`!user_otp_validated`):

    * If OTP isn’t activated on the user, throw `OtpRequiredActivation()`
    * Else throw `OtpRequired()`
* Else (platform doesn’t mandate OTP):

  * If the user has activated OTP but the session isn’t validated, throw `OtpRequired()`

**Important nuance:** the code checks `@allowUnprotectedOTP` only on the **field**, not at the object/type level.

### C.3 LTS validation enforcement (unless the field opts out)

The wrapper checks:

* `blocked_for_lts_validation`

If true, and the field is **not** annotated with:

* `@allowUnlicensedLTS`

…it throws:

* `LtsRequiredActivation()`

Again, this opt-out is checked on the **field**.

### C.4 Capability enforcement (`for` + `and`)

The directive args are interpreted as:

* `for`: list of required capability strings (from the `Capabilities` enum)
* `and`:

  * `false` (default): user must have **any** required capability
  * `true`: user must have **all** required capabilities

If `for` is empty (`length === 0`), the wrapper skips capability checks after the authentication/OTP/LTS gates.

#### Bypass rules

Capability checks are bypassed entirely if either is true:

* User has the `BYPASS` capability, **or**
* User’s id is `OPENCTI_ADMIN_UUID` (the “system/admin” user guard)

#### Draft-mode capability augmentation (feature-flagged)

If the `CAPABILITIES_IN_DRAFT` feature is enabled and the request is in a draft context (`getDraftContext(context, user)` is truthy), then:

* `user.capabilitiesInDraft` is unioned into the base capabilities list for enforcement.

#### Matching behavior is substring-based

A required capability is considered satisfied if **any** user capability string includes it as a substring:

* `userCapabilities.some((u) => requestedCapability !== BYPASS && u.includes(requestedCapability))`

This is intentionally *not* strict equality matching.

### C.5 Special-case behavior for Organization + virtual org admin

There’s a specific rule when resolving fields on the `Organization` type:

* If the field requires `VIRTUAL_ORGANIZATION_ADMIN`
* and the user does **not** have `SETTINGS_SET_ACCESSES`

Then:

* If `user.administrated_organizations` includes the organization being resolved (`source.id`), access is allowed.
* Otherwise, the resolver returns **`null`** (not `ForbiddenAccess`).

That’s a deliberate “hide the field result” behavior rather than emitting a hard authorization error for that case.

### C.6 Failure mode when capabilities don’t match

If the user is authenticated but fails the capability requirement:

* throws `ForbiddenAccess()`

---

## D) What this means for documentation

When documenting `@auth`, it’s accurate (from these files) to describe it as:

1. A **schema-transform enforced** directive (not just declarative SDL)
2. A wrapper that always enforces:

   * authenticated user
   * OTP policy (unless `@allowUnprotectedOTP`)
   * LTS validation policy (unless `@allowUnlicensedLTS`)
   * capability policy (with OR vs AND semantics via `and`)
3. A system that **refuses startup** if a Query/Mutation field is missing both `@auth` and `@public`

---

### 4.3 “Public” surface area and exceptions (snapshot-accurate)

This snapshot uses `@public` as the explicit marker for **unauthenticated access** in SDL. In the entire snapshot, `@public` appears **only** in these four files (no other `*.graphql` in the archive contains `@public`).

---

## A) Core public entrypoints (`opt/opencti/config/schema/opencti.graphql`)

### Public queries (4)

```graphql
publicSettings: PublicSettings! @public # All returned information are public
```

```graphql
taxiiCollections(
  ...
): TaxiiCollectionConnection @public # Code protected
```

```graphql
feeds(
  ...
): FeedConnection @public # Code protected
```

```graphql
streamCollections(
  ...
): StreamCollectionConnection! @public # Code protected
```

### Public mutation (1)

```graphql
token(input: UserLoginInput): String @public # Use for login
```

**Exception note (important):** three of the four public queries are explicitly commented **“Code protected”** in the SDL (`taxiiCollections`, `feeds`, `streamCollections`). Treat these as **not “guaranteed public” in practice**—the schema says “public”, but the comment is a warning that runtime logic may still enforce restrictions.

---

## B) Public dashboard surface (`src/modules/publicDashboard/publicDashboard.graphql`)

This module defines **10 public queries**, all keyed by a `uriKey` (and often a `widgetId`)—i.e., “public access by share key” shape.

Public queries:

* `publicDashboardByUriKey(uri_key: String!): PublicDashboard @public`
* `publicStixCoreObjectsNumber(uriKey, widgetId, startDate, endDate): Number @public`
* `publicStixRelationshipsNumber(uriKey, widgetId, startDate, endDate): Number @public`
* `publicStixCoreObjectsMultiTimeSeries(uriKey, widgetId, startDate, endDate): [MultiTimeSeries] @public`
* `publicStixRelationshipsMultiTimeSeries(uriKey, widgetId, startDate, endDate): [MultiTimeSeries] @public`
* `publicStixCoreObjectsDistribution(uriKey, widgetId, startDate, endDate): [PublicDistribution] @public`
* `publicStixRelationshipsDistribution(uriKey, widgetId, startDate, endDate): [PublicDistribution] @public`
* `publicBookmarks(uriKey: String!, widgetId: String!): StixDomainObjectConnection @public`
* `publicStixCoreObjects(uriKey, widgetId, startDate, endDate): StixCoreObjectConnection @public`
* `publicStixRelationships(uriKey, widgetId, startDate, endDate): StixRelationshipConnection @public`

Also notable (not public): the module’s list endpoint `publicDashboards(...)` is **authenticated** (`@auth(for: [EXPLORE])`), even though the “by key”/widget data endpoints are public.

---

## C) Auth / OTP / MFA public mutations (`src/modules/auth/auth.graphql`)

This module exposes **four public mutations**, all rate-limited in SDL:

* `askSendOtp(input: AskSendOtpInput!): String @public @rateLimit(limit: 1, duration: 1)`
* `verifyOtp(input: VerifyOtpInput!): VerifyOtp @public @rateLimit(limit: 1, duration: 1)`
* `verifyMfa(input: VerifyMfaInput!): Boolean @public @rateLimit(limit: 1, duration: 1)`
* `changePassword(input: ChangePasswordInput!): Boolean @public @rateLimit(limit: 1, duration: 1)`

**Doc implication:** these are intentionally callable *before* a normal authenticated session exists, but the SDL makes it explicit they’re meant to be throttled (`@rateLimit`).

---

## D) Public theme queries (`src/modules/theme/theme.graphql`)

Two public queries for theme retrieval:

```graphql
theme(id: ID!): Theme @public
```

```graphql
themes(
  first: Int
  after: ID
  orderBy: ThemeOrdering
  orderMode: OrderingMode
  filters: FilterGroup
  search: String
  toStix: Boolean
): ThemeConnection @public
```

---

## E) Snapshot totals (public root surface)

Across these four files, the snapshot exposes **21** `@public` root fields:

* **Queries:** 16 (4 core + 10 publicDashboard + 2 theme)
* **Mutations:** 5 (1 core token + 4 auth/otp/mfa/password)

No subscriptions are marked `@public` in this snapshot.

---

## F) Practical “exceptions” guidance for the docs

1. **“Code protected” is a red flag you should preserve verbatim.** It appears directly on the public SDL for `taxiiCollections`, `feeds`, and `streamCollections` in `opencti.graphql`, and it means “SDL says public, but don’t assume unrestricted access.”

2. **Public dashboards are *public-by-key*, not “public browse.”** The public endpoints consistently require `uriKey` / `widgetId`, which strongly suggests runtime will only return data for dashboards/widgets intended for sharing.

3. **Auth-adjacent public mutations are explicitly rate-limited.** The `auth.graphql` module makes `@rateLimit` part of the public contract for OTP/MFA/password-change flows.

---

#### 4.3.1 Snapshot note: complete `@public` operation list

Only the following **21 root operations** are annotated with `@public` in this snapshot (no other `Query`/`Mutation`/`Subscription` field in the captured SDL set uses `@public`):

**Core (`opencti.graphql`)**

* **Query**

  * `publicSettings`
  * `taxiiCollections` *(commented “Code protected” in SDL)*
  * `feeds` *(commented “Code protected” in SDL)*
  * `streamCollections` *(commented “Code protected” in SDL)*
* **Mutation**

  * `token` *(login)*

**Public Dashboard module (`publicDashboard.graphql`)**

* **Query**

  * `publicDashboardByUriKey`
  * `publicStixCoreObjectsNumber`
  * `publicStixRelationshipsNumber`
  * `publicStixCoreObjectsMultiTimeSeries`
  * `publicStixRelationshipsMultiTimeSeries`
  * `publicStixCoreObjectsDistribution`
  * `publicStixRelationshipsDistribution`
  * `publicBookmarks`
  * `publicStixCoreObjects`
  * `publicStixRelationships`

**Auth module (`auth.graphql`)**

* **Mutation**

  * `askSendOtp`
  * `verifyOtp`
  * `verifyMfa`
  * `changePassword`

**Theme module (`theme.graphql`)**

* **Query**

  * `theme`
  * `themes`

---

### 4.4 Capability mapping plan (operation → required capabilities)

This section’s goal is to produce (and keep current) a **single, canonical mapping** of each GraphQL **operation** (root `Query` / `Mutation` / `Subscription` field) to the **capabilities required to execute it**, based on the schema annotations and the runtime behavior of the auth directive transformer.

#### Source of truth

1. **Directive annotations in SDL**

   * Core schema: `opt/opencti/config/schema/opencti.graphql`
   * All module SDL: `src/modules/**/**/*.graphql`
   * In this snapshot, every operation is annotated with either `@public` or `@auth(...)` (no unannotated root ops).

2. **Runtime enforcement semantics**

   * Implemented by the schema transform in `src/graphql/authDirective.ts` (wired in `src/graphql/schema.js`).

A key correctness guard is already present in `authDirective.ts`: it performs a “secure schema” check that throws if **any `Query` or `Mutation` field** is missing **both** `@auth` and `@public`. That means the mapping can be generated exhaustively and can be validated in CI the same way.

#### What the mapping must capture

For every operation (`Query.*`, `Mutation.*`, `Subscription.*`), record:

* **Visibility**

  * `public: true` if annotated `@public`
  * otherwise it is protected by `@auth`

* **Required capabilities** (from `@auth(for: [...])`)

  * `for: []` (or omitted args) means **“authenticated session required, no capability gating.”**
  * `for: [CAP1]` means **CAP1** is required
  * `for: [CAP1, CAP2, ...]` means **any-of** those capabilities is sufficient (unless `and: true`, see below)

* **Match mode** (`and:`)

  * The directive supports `and: true|false` at runtime.
  * In this snapshot, **no `@auth` usage sets `and: true`** in SDL, so operation capability lists behave as **OR** lists (any capability satisfies).

* **Additional runtime gates that can block access even after auth/capability**

  * **OTP gate**: if `otp_mandatory` is set in context and the user isn’t OTP-activated, requests are blocked unless the field is annotated `@allowUnprotectedOTP`.

    * In this snapshot, the OTP-exempt operations are:

      * `Query.otpGeneration`
      * `Mutation.otpActivation`
      * `Mutation.otpLogin`
      * `Mutation.logout`
  * **LTS gate**: if `blocked_for_lts_validation` is set in context, requests are blocked unless annotated `@allowUnlicensedLTS`.

    * In this snapshot, the LTS-exempt operations are:

      * `Query.settings`
      * `Mutation.setupEnterpriseLicense`

Also note a subtlety of enforcement in `authDirective.ts`: capability checks are implemented using a substring match (`u.includes(requestedCapability)`), so “broader” capability names can be satisfied by “more specific” capability strings that contain them.

#### Proposed extraction + generation workflow

1. **Parse the schema**

   * Build the effective schema from the same inputs the server uses (core SDL + module SDL).
   * Prefer parsing via a GraphQL AST (e.g., `graphql` JS parser) to avoid edge cases with multiline fields and directives.

2. **Enumerate operations**

   * Walk the root type definitions and capture every field in:

     * `type Query { ... }`
     * `type Mutation { ... }`
     * `type Subscription { ... }`

3. **Extract directives**

   * If `@public`: mark as public and stop.
   * Else read `@auth`:

     * `requiredCapabilities = for` (default `[]`)
     * `matchMode = and ? "ALL" : "ANY"` (default `"ANY"`)
   * Also capture `@allowUnprotectedOTP` and `@allowUnlicensedLTS` flags on the same field, because they materially change runtime access.

4. **Validate (CI-friendly)**

   * Re-run the same invariant as runtime:

     * every `Query` + `Mutation` field must have `@public` or `@auth`
   * Optionally assert snapshot expectations (e.g., “no operation uses `and:true`”) so unexpected changes get caught early.

5. **Emit artifacts**

   * **Machine-readable**: JSON mapping `{ operationType, name, public, requiredCapabilities, matchMode, otpExempt, ltsExempt, sourceFile }`
   * **Human-readable**: markdown tables generated from JSON, grouped:

     * by capability (capability → operations)
     * by operation type (Query/Mutation/Subscription)
     * optionally by module/source file

#### Snapshot sizing (helps set expectations)

In this snapshot there are **844 operations total**:

* `Query`: 383
* `Mutation`: 438
* `Subscription`: 23

Of these:

* **21** are `@public`
* **823** are `@auth`-protected

  * **66** are auth-only (`@auth` with no `for:` capabilities)
  * **616** require exactly one capability
  * **141** list multiple capabilities (OR semantics in this snapshot)

This gives us a clean foundation to generate (and continuously validate) the operation → capability map directly from the schema, while still reflecting the real runtime gates enforced by `authDirective.ts`.

---

## 5) Module system (how features are packaged)

### 5.1 Module anatomy pattern (SDL + graphql wrapper + resolver + domain/types/etc.)

In this snapshot, a “module” is typically a **self-contained feature package** that can register:

1. **Schema-model metadata** (entity type, identifiers, attributes, ref-relations, converters, etc.), and
2. **GraphQL surface** (SDL + resolver map) into the executable schema via `registerGraphqlSchema(...)`.

These are wired together at boot by **ordered side-effect imports** in `opt/opencti/src/modules/index.ts`.

---

#### A) Boot-time wiring and ordering (`src/modules/index.ts`)

`opt/opencti/src/modules/index.ts` is the *manifest* and the *required initialization order*:

1. **registration attributes** (must run first)

   ```ts
   // region registration attributes, need to be imported before any other modules
   import './attributes/basicObject-registrationAttributes';
   ...
   // endregion
   ```

2. **registration ref** (must run early)

   ```ts
   // region registration ref, need to be imported before any other modules
   import './relationsRef/stixCoreObject-registrationRef';
   ...
   // endregion
   ```

3. **registration modules** (schema-model lane; must run before GraphQL registration)

   ```ts
   // region registration modules, need to be imported before graphql code registration
   import './case/case';
   import './theme/theme';
   import './internal/document/document';
   ...
   // endregion
   ```

   This block also contains a clearly marked **“incomplete modules”** subsection:

   ```ts
   // incomplete modules
   import './stixCyberObservable/stixCyberObservable';
   ...
   ```

4. **graphql registration** (GraphQL lane)

   ```ts
   // region graphql registration
   import './case/case-graphql';
   import './theme/theme-graphql';
   import './internal/csvMapper/csvMapper-graphql';
   ...
   // endregion
   ```

**Key point:** inclusion is *not* auto-discovery. If it isn’t imported (directly or indirectly) from `src/modules/index.ts`, it doesn’t participate in boot registration.

---

#### B) Canonical “full module” anatomy (theme as the clean exemplar)

`opt/opencti/src/modules/theme/*` shows the most straightforward, repeatable pattern:

**1) Schema-model registration (`theme.ts`)**
A `ModuleDefinition<StoreEntityX, StixX>` object is declared and installed via `registerDefinition(...)`:

```ts
const THEME_DEFINITION: ModuleDefinition<StoreEntityTheme, StixTheme> = {
  type: { id: 'theme', name: ENTITY_TYPE_THEME, category: ABSTRACT_INTERNAL_OBJECT, aliased: false },
  identifier: { definition: { [ENTITY_TYPE_THEME]: () => uuidv4() } },
  attributes: [ ... ],
  relations: [],
  representative: (stix) => stix.name,
  converter_2_1: convertThemeToStix,
};

registerDefinition(THEME_DEFINITION);
```

**2) GraphQL SDL (`theme.graphql`)**
Defines the GraphQL types and root fields for the feature.

**3) Resolver map (`theme-resolvers.ts`)**
Exports a typed resolver map (imports `Resolvers` from `../../generated/graphql`) and delegates to domain functions:

```ts
const themeResolvers: Resolvers = {
  Query: { theme: (_, { id }, context) => findById(context, context.user, id) },
  Mutation: { themeAdd: (_, { input }, context) => addTheme(context, context.user, input) },
};
export default themeResolvers;
```

**4) Domain logic (`theme-domain.ts`)**
Implements the actual operations (CRUD, imports, etc.). In this snapshot it clearly follows a “thin resolver → domain function” style (domain functions typically accept `(context, user, args...)`).

**5) GraphQL wrapper (`theme-graphql.ts`)**
This is the *only* thing the schema assembly layer needs from a module: “here is SDL + resolvers”:

```ts
import { registerGraphqlSchema } from '../../graphql/schema';
import themeTypeDefs from './theme.graphql';
import themeResolvers from './theme-resolvers';

registerGraphqlSchema({ schema: themeTypeDefs, resolver: themeResolvers });
```

**6) Types/converter/supporting files**
`theme-types.ts`, `theme-converter.ts`, `theme-constants.ts`, etc. are the typical supporting cast.

This “theme pattern” is what you can treat as the default mental model.

---

#### C) “Module families” (case, ingestion): one feature, multiple submodules

Some features are packaged as **families**: a top-level module plus several nested submodules, each with its own SDL + wrapper + resolvers + domain + types.

**Case family (`src/modules/case/*`)**

* Top-level: `case.ts`, `case.graphql`, `case-resolvers.ts`, `case-domain.ts`, `case-graphql.ts`, etc.
* Nested: `case-template/*`, `case-incident/*`, `case-rfi/*`, `case-rft/*`, `feedback/*`
* Each nested folder follows the same mini-pattern:

  * `case-template.ts` (registerDefinition)
  * `case-template.graphql`
  * `case-template-resolvers.ts`
  * `case-template-domain.ts`
  * `case-template-graphql.ts` (registerGraphqlSchema)

**Ingestion family (`src/modules/ingestion/*`)**

* Multiple “flavors” (`ingestion-rss`, `ingestion-taxii`, `ingestion-taxii-collection`, `ingestion-csv`, `ingestion-json`)
* Each flavor has its own:

  * `ingestion-<flavor>.ts` (schema-model definition)
  * `ingestion-<flavor>.graphql`
  * `ingestion-<flavor>-resolver.ts`
  * `ingestion-<flavor>-domain.ts`
  * `ingestion-<flavor>-graphql.ts`
* Shared types/converters live in shared files like `ingestion-types.ts` / `ingestion-converter.ts`.

**Doc implication:** for these families, “the module” in docs is often *the directory subtree*, not a single SDL file.

---

#### D) Internal modules: mixed patterns (full modules + schema-only + deprecated GraphQL)

`opt/opencti/src/modules/internal/*` includes multiple styles:

1. **Full modules (csvMapper/jsonMapper)**
   They have the usual pieces (`*.ts`, `*.graphql`, `*-graphql.ts`, `*-resolvers.ts`, `*-domain.ts`, `*-types.ts`, `*-converter.ts`, etc.).

2. **Schema-only registration (document)**
   `internal/document/document.ts` does **not** define a module SDL or GraphQL wrapper. Instead it directly registers attributes into the schema model:

```ts
schemaAttributesDefinition.registerAttributes(ENTITY_TYPE_INTERNAL_FILE, attributes);
```

So it participates in schema-model metadata but not as a standalone GraphQL module in this snapshot.

3. **Deprecated GraphQL registered via side effects**
   Some internal modules explicitly import deprecated registrations from within their module-definition code. Example: `internal/csvMapper/csvMapper.ts` includes:

```ts
import './deprecated/csvMapper-deprecated';
```

…and `deprecated/csvMapper-deprecated.ts` calls `registerGraphqlSchema(...)` with the deprecated SDL/resolver.

**Doc implication:** while the “graphql registration” region in `src/modules/index.ts` is the normal lane, **some deprecated GraphQL surfaces are registered indirectly** during the “registration modules” imports.

---

#### E) Summary: the anatomy contract you can document

A “standard full module” in this snapshot is:

* **Model lane**

  * `<module>.ts` → declares `ModuleDefinition<Store, Stix>` and calls `registerDefinition(...)`
  * `<module>-types.ts`, `<module>-converter.ts`, plus any schema metadata helpers

* **GraphQL lane**

  * `<module>.graphql` → SDL
  * `<module>-resolvers.ts` (or `*-resolver.ts`) → resolver map
  * `<module>-domain.ts` → implementation functions invoked by resolvers
  * `<module>-graphql.ts` → `registerGraphqlSchema({ schema, resolver })`

With common real-world variations:

* **Module families** with nested submodules (case/ingestion)
* **Schema-only packages** (internal document attributes)
* **Deprecated GraphQL** registered by side-effect imports inside module code (not only from `index.ts`’s graphql block)

---

#### 5.1.1 How to add a new module in this snapshot model

This snapshot’s module system has two distinct registration “lanes” that must both be wired correctly: the **schema-model lane** (type/attribute/relationship metadata) and the **GraphQL lane** (SDL + resolvers). The wiring is **explicit** and **order-dependent** via `src/modules/index.ts`.

---

##### A) Create the module structure (recommended baseline)

In `src/modules/<yourModule>/`, create the standard set:

1. **Schema-model definition**

   * `src/modules/<yourModule>/<yourModule>.ts`
   * Declare a `ModuleDefinition<StoreEntityX, StixX>` and install it via `registerDefinition(...)`.

2. **GraphQL SDL**

   * `src/modules/<yourModule>/<yourModule>.graphql`
   * Define types plus any `type Query { ... }` / `type Mutation { ... }` / `type Subscription { ... }` fields the module adds.

3. **Resolvers**

   * `src/modules/<yourModule>/<yourModule>-resolvers.ts`
   * Export a resolver map (typically `Resolvers` from `../../generated/graphql`) and delegate to domain functions.

4. **Domain implementation**

   * `src/modules/<yourModule>/<yourModule>-domain.ts`
   * Implement the resolver targets (usually `(context, user, ...)` signatures).

5. **GraphQL registration wrapper**

   * `src/modules/<yourModule>/<yourModule>-graphql.ts`
   * Import SDL + resolvers and call:

     * `registerGraphqlSchema({ schema: <typeDefs>, resolver: <resolvers> })`

Optional (common):

* `<yourModule>-types.ts`, `<yourModule>-converter.ts`, `<yourModule>-constants.ts`

---

##### B) Wire it into boot: update `src/modules/index.ts` (order matters)

`src/modules/index.ts` is the runtime manifest and must be updated in the correct regions:

1. **Schema-model registration import** (module lane)
   Add this to the **registration modules** section:

   * `import './<yourModule>/<yourModule>';`

2. **GraphQL registration import** (GraphQL lane)
   Add this to the **graphql registration** section:

   * `import './<yourModule>/<yourModule>-graphql';`

**Do not** place these in the attributes or relationsRef regions. Those regions are for shared cross-cutting registrations that must run before modules.

---

##### C) Annotate operations correctly (security invariants)

For every new root field you add in SDL:

* If it is meant to be protected: add `@auth(...)`
* If it is meant to be callable without auth: add `@public`

**Why this is mandatory in this snapshot model:** the auth schema transformer enforces a “secure schema” invariant at startup: **every `Query` and `Mutation` field must have `@auth` or `@public`**, otherwise the server throws during schema build.

Also consider:

* If the operation must bypass OTP gating, annotate it with `@allowUnprotectedOTP`
* If the operation must bypass LTS validation gating, annotate it with `@allowUnlicensedLTS`
* If it should be throttled, annotate with `@rateLimit(...)` (provided by runtime typeDefs injection)

---

##### D) Keep the “manual maintenance” surfaces in sync

Adding a new type often requires updating core “umbrella” unions in `opt/opencti/config/schema/opencti.graphql`, as described in the module README pattern. Common unions that frequently need updates include:

* `StixCoreObjectOrStixCoreRelationship`
* `StixObjectOrStixRelationship`
* `StixObjectOrStixRelationshipOrCreator`

If your new type is intended to participate in polymorphic queries, make sure those unions (or any other explicit unions) include it.

---

##### E) Quick self-check before shipping

1. **Boot wiring**

   * Your module definition import exists in the “registration modules” region.
   * Your GraphQL registration import exists in the “graphql registration” region.

2. **Schema coverage**

   * Every new `Query`/`Mutation` field has `@auth` or `@public`.
   * Any special OTP/LTS exceptions are explicitly annotated.

3. **Collision avoidance**

   * Your module does not redefine an existing root field name unless you are intentionally overriding behavior.

4. **Reference integrity**

   * If your module adds new STIX-ish types, confirm the relevant unions and type families include them where needed.

---

### 5.2 Module definition model: attributes, relations, identifiers

This GraphQL layer has a **schema-model registry** that sits “under” GraphQL SDL. Modules don’t just add SDL + resolvers — they also register **type metadata** (attributes, identifiers, relationship vocabularies, ref-relationship definitions, validators, layout hints, etc.) into global registries used across the platform.

The core entrypoint for that model registration is:

* `src/schema/module.ts` → `registerDefinition(definition: ModuleDefinition)`

The supporting registries and contracts live in:

* `src/schema/attribute-definition.ts` (attribute + ref-relationship definition types)
* `src/schema/schema-attributes.ts` (attribute registry + helpers)
* `src/schema/schema-relationsRef.ts` (ref-relationship registry + name mapping helpers)
* `src/schema/identifier.js` (standard_id generation + model identifier contributions)
* `src/schema/internalObject.ts` / `src/schema/internalRelationship.ts` (internal type families)

---

## A) `ModuleDefinition`: what a “registered type” must declare (`src/schema/module.ts`)

A module registers one logical “entity type” by providing a `ModuleDefinition` object with these major parts:

### A.1 Type identity and classification

`definition.type` includes:

* `id: string` — module/type identifier (used as the key in the global `modules` map)
* `name: string` — the platform `entity_type` string for the type
* `category: ModuleCategory` — how the type is classified (drives type-family registration)
* `aliased?: boolean` — indicates the type participates in the “aliased object” model (affects attribute injection; see below)

### A.2 Identifier model (standard_id contributions)

`definition.identifier` includes:

* `definition: ModuleIdentifierDefinition` — how to build identifiers for one or more types
* `resolvers?: ModuleIdentifierResolvers` — optional computed-field resolvers used by the identifier engine

This is plugged into the global identifier system via `registerModelIdentifier(...)` (see section **E**).

### A.3 STIX and UI/UX helpers

The definition may include:

* `representative: (stix: StixData) => RepresentativeFn` — how to compute the “representative” field/value used for display/summary patterns
* `converter_2_1: StixConverter` — STIX 2.1 conversion implementation for the type
* optional `bundleResolver?: BundleResolver` — bundle-related behavior hook
* optional `overviewLayoutCustomization?: OverviewLayoutCustomization` — UI/layout hints for overview pages

### A.4 Attributes + relationships

The core model surface is:

* `attributes: AttributeDefinition[]` — field metadata for the entity type
* `relations: { name: string; targets: RelationDefinition[] }[]` — “core relationship” mapping rules (must use known relationship vocabulary)
* optional `relationsRefs?: RefAttribute[]` — “ref relationship” definitions (databaseName/stixName/name mapping)
* optional `depsKeys?: string[]` — dependency extraction hints (registered globally)

### A.5 Optional validators

* `validatorCreation?: ValidatorInput[]`
* `validatorUpdate?: ValidatorInput[]`

These are attached to the entity type via `registerEntityValidator(...)`.

---

## B) What `registerDefinition(...)` actually does (`src/schema/module.ts`)

When a module calls `registerDefinition(definition)`, it performs **system-wide registration** steps in a fixed order.

### B.1 Registers the type into one or more “type families”

Based on `definition.type.category`, it registers the type into internal type-family registries (via `schemaTypesDefinition.register(...)`). The key categories and family effects in this snapshot are:

* **Threat actor** → registers under `ENTITY_TYPE_THREAT_ACTOR` and also under the general STIX domain object family (`ABSTRACT_STIX_DOMAIN_OBJECT`)
* **Location** → registers under `ENTITY_TYPE_LOCATION` and STIX domain object
* **Container** → registers under `ENTITY_TYPE_CONTAINER`
  *Special case:* if the type is a “case container” (`ENTITY_TYPE_CONTAINER_CASE`), it also registers under that family; otherwise it still gets registered under STIX domain object
* **Identity** → registers under `ENTITY_TYPE_IDENTITY` and STIX domain object
* **Stix-Domain-Object** → registers under STIX domain object
* **Stix-Meta-Object** → registers under STIX meta object (`ABSTRACT_STIX_META_OBJECT`)
* **Internal-Object** → registers under internal object (`ABSTRACT_INTERNAL_OBJECT`)

This classification is what lets the rest of the platform answer questions like “is this a domain object?”, “is this internal?”, “is this a meta object?”, etc., without hard-coding per-type lists everywhere.

### B.2 Registers converters and representative logic

* Always registers the representative logic for the type.
* Registers the STIX converter into either the **domain object converter registry** or the **meta object converter registry**, depending on category.
* If `aliased: true`, it also registers the type as “aliased” (via `registerStixDomainAliased(typeName)`).

### B.3 Registers validators (if provided)

* If creation/update validators exist, they’re registered with `registerEntityValidator(typeName, creation, update)`.

### B.4 Registers identifier contributions

* Calls `registerModelIdentifier(definition.identifier)` (see **E**).

### B.5 Registers attributes (plus injected “standard” attributes)

Attributes are registered via:

* `schemaAttributesDefinition.registerAttributes(typeName, [...])`

But the list isn’t just `definition.attributes`. The module layer **injects**:

1. `standardId` (always)
2. `definition.attributes` (module-provided)
3. If `aliased: true`:

   * an “aliases” attribute (the field name is chosen by `resolveAliasesField(typeName)`), and
   * `iAliasedIds` (a standard “alias id(s)” attribute set)

This is why “aliased types” behave differently: they get additional attribute metadata even if the module didn’t explicitly declare those attributes.

### B.6 Registers dependency keys (if provided)

* If `depsKeys` is present, each key is appended to the global `depsKeysRegister` list (from `src/schema/schema-attributes.ts`).

### B.7 Registers “core relationships” mapping rules

For each entry in `definition.relations`, `registerDefinition` enforces that the relationship name is part of the recognized **STIX core relationship vocabulary** (`STIX_CORE_RELATIONSHIPS`). If not, it fails fast.

Valid relationship mappings are then recorded into `stixCoreRelationshipsMapping` (via `coreRels.add(typeName, targets)`).

**Doc implication:** modules can’t invent arbitrary “core relationship” names here — they must use the platform’s controlled vocabulary.

### B.8 Registers “ref relationships” (relationsRefs) and their type-family membership

If `definition.relationsRefs` exists:

1. It registers those ref attributes into the ref registry:

   * `schemaRelationsRefDefinition.registerRelationsRef(typeName, relationsRefs)`
2. It also registers each ref relationship’s `databaseName` as a member of the abstract family `ABSTRACT_STIX_REF_RELATIONSHIP`.

That second step is important: it ensures the platform can treat those `databaseName` strings as first-class “relationship types” in the broader relationship taxonomy.

### B.9 Registers overview/layout customization and stores the definition

* Layout customization is registered (if present).
* The definition is stored into a global `modules` map keyed by `definition.type.id`.

---

## C) Attribute model (`src/schema/attribute-definition.ts` + `src/schema/schema-attributes.ts`)

### C.1 AttributeDefinition: common fields

All attribute definitions share a common “base” metadata set, including:

* `name`, `label?`, `description?`
* `multiple?`
* `mandatoryType?: 'external' | 'internal' | 'custom' | 'none'`
* `isFilterable?: boolean`
* `upsert?: boolean`
* `upsert_force_replace?: boolean`
* `editDefault?: boolean`
* `update?: boolean`
* `featureFlag?: string`

Think of these as “how the field behaves” in edit flows, filtering, and feature gating (the deeper semantics are implemented elsewhere, but the *shape* of the metadata is here).

### C.2 Concrete attribute kinds

The attribute contract supports multiple concrete kinds (via discriminated unions), including:

* **string** attributes with formats like:

  * `id`, `short`, `text`, `enum`, `vocabulary`, `json`
* **numeric** attributes (with precision like integer/long/float and `scalable?`)
* **boolean**
* **date** (with a precision flag like `date` vs `date-time`)
* **object** attributes with a structured mapping model (formats `standard`, `flat`, `nested`)
* **ref** attributes (`RefAttribute`) for ref relationships (see **D**)

### C.3 Attribute registry behavior (schema-attributes.ts)

`schemaAttributesDefinition.registerAttributes(entityType, attributes)` builds and maintains:

* a per-type attribute map (`name → AttributeDefinition`)
* merged “effective attribute sets” for types that have parents (via `getParentTypes(...)`)
* helper indexes used elsewhere (examples: “upsertable attributes per type”, “filterable attributes per type”, object mapping shortcuts, etc.)

A key operational rule enforced here:

* **Registration is time-sensitive.** The registry has a “usage protection” guard: once certain “read/use” paths run, late registrations are rejected. In practice, this means attribute registration is expected to happen during the early boot/initialization phase (before the schema/model is actively used).

---

## D) Ref relationships model (`src/schema/schema-relationsRef.ts`)

Ref relationships (“relationsRefs”) are defined as `RefAttribute` objects (from `attribute-definition.ts`) and registered per entity type.

### D.1 What a RefAttribute carries

A ref relationship includes:

* `name` — the input/GraphQL-facing name
* `databaseName` — the internal relationship “type” string
* `stixName` — the STIX-ish field name
* `toTypes: string[]` — the allowed target types
* `isRefExistingForTypes(fromType, toType)` — a type-pair guard
* plus shared base fields like `multiple?`, `isFilterable?`, and optional `datable?`

### D.2 Registry invariants and name-mapping helpers

`schemaRelationsRefDefinition.registerRelationsRef(entityType, relationsRefs)` enforces key invariants:

* `name` **must not** equal `databaseName` (it fails fast if they match)
* `databaseName` **must not** collide with core relationship vocabulary (`STIX_CORE_RELATIONSHIPS`)

It also builds global lookup tables that are heavily used by “edit mutation” and conversion pipelines, including:

* **input name ↔ databaseName**
* **STIX name ↔ input name**
* whether a ref relationship is **multiple**
* per-relationship **toTypes** lists
* which ref relationships are **datable**

And, like the attribute registry, it supports parent-type merging through `getParentTypes(entityType)`.

---

## E) Identifier model (`src/schema/identifier.js`)

The identifier system is responsible for generating the platform’s **standard_id** values (and related identifiers) consistently across object families.

### E.1 Module contribution hook: `registerModelIdentifier(...)`

`registerModelIdentifier(modelIdentifier)` appends the module’s contributions to a global contribution list and merges:

* `definition` entries (how to build an id for specific entity types)
* optional `resolvers` (computed components needed for deterministic ids)

So the module layer is how new types “teach” the platform how to generate stable IDs.

### E.2 How IDs get generated (high-level)

`generateStandardId(type, data)` is the core entrypoint. It chooses an id strategy based on the type’s family:

* STIX objects use STIX-style ids (`<stix-type>--<uuid>`)
* relationships use relationship-type strategies (including deterministic generation for core relationships)
* internal objects and internal relationships use internal-type strategies

The exact choice logic is implemented in `identifier.js`, and it depends on the platform’s type-family predicates (domain/meta/cyber observable/internal relationship/etc.).

**Doc implication:** adding a new type is rarely “just add SDL” — if it’s a *new entity_type*, you generally need to ensure the identifier model can generate the right standard_id pattern for it.

---

## F) Internal object and internal relationship families (`internalObject.ts`, `internalRelationship.ts`)

The platform also defines explicit “internal” families that are **not** STIX-domain/meta objects.

### F.1 Internal objects (`src/schema/internalObject.ts`)

This file defines curated lists of internal entity types and registers them under `ABSTRACT_INTERNAL_OBJECT`. In this snapshot, that internal set includes (grouped roughly):

* **config/system/admin-ish:** Settings, Status/StatusTemplate, RetentionRule, Sync, MigrationStatus/MigrationReference
* **auth/identity:** Group, User, Role, Capability
* **connectors/managers/rules:** Connector, ConnectorManager, Rule, RuleManager
* **UI/workspaces:** Workspace, PublicDashboard, DraftWorkspace, Theme, Form
* **streams/ingestion-ish:** Feed, StreamCollection, TAXIICollection
* **ops/history:** History, Activity, PIR history
* **other internal entities:** InternalFile, Work, SavedFilter, ExclusionList, FintelTemplate, FintelDesign, Pir, EmailTemplate, BackgroundTask

It also defines a smaller “dated internal objects” set and a “history objects” set, and exports predicates like `isInternalObject(...)` and `isDatedInternalObject(...)`.

### F.2 Internal relationships (`src/schema/internalRelationship.ts`)

This file declares internal relationship type strings (and registers them under `ABSTRACT_INTERNAL_RELATIONSHIP`), including:

* `migrates`
* `member-of`
* `participate-to`
* `allowed-by`
* `has-role`
* `has-capability`
* `has-capability-in-draft`
* `accesses-to`
* `in-pir`

…and provides an `isInternalRelationship(...)` predicate.

---

## Practical takeaway for module authors

When you add or evolve a module type in this snapshot model, you’re typically touching **three layers**:

1. **Schema-model registration** (`registerDefinition` + attributes/relations/identifier/ref-refs)
2. **GraphQL registration** (module SDL + resolver wiring)
3. **Type-family/union governance** (e.g., “is this type STIX-domain/meta/internal?” and any unions that enumerate concrete types elsewhere)

This section is the “schema-model” half: it’s what makes the type behave correctly across validation, filtering, id generation, relationship semantics, and (eventually) conversion/export paths.

---

### 5.3 Data shape consistency: field adapters + schema utils

This snapshot includes two small but high-leverage “shape normalization” layers under `src/schema/`:

* `fieldDataAdapter.ts` — **field-level adapters** that normalize specific value shapes (notably hashes) and provide a few shared “does this type have X timestamp?” predicates.
* `schemaUtils.js` — **core schema utilities** for type normalization and hierarchy reasoning (STIX-ish → internal type strings, parent-type expansion, and “most restrictive types” filtering).

Together, these utilities help keep data consistent across:

* resolver input handling,
* resolver output formatting,
* schema-model registries that need parent-type inheritance (attributes, ref relationships, etc.).

---

## A) Field adapters (`src/schema/fieldDataAdapter.ts`)

### A.1 Hash normalization helpers (input ↔ STIX output)

This file standardizes how “hashes” are represented when crossing the GraphQL boundary.

**Constants**

* `SENSITIVE_HASHES = ['SSDEEP', 'SDHASH']`
* `FUZZY_HASH_ALGORITHMS = ['SSDEEP', 'SDHASH', 'TLSH', 'LZJD']`

**Helper: `inputHashesToStix(data: Array<HashInput>)`**

* Accepts `HashInput` objects (`{ algorithm, hash }`) and returns a STIX-ish hash map:
  `Record<string, string>` where keys are normalized algorithm names.
* Normalization rules:

  * `algorithm` is uppercased + trimmed.
  * `hash` is trimmed.
  * For **non-sensitive** algorithms, the hash value is lowercased.
  * For **sensitive** (SSDEEP/SDHASH), it preserves case (no lowercasing).
* Commented intent: “Must be called as soon as possible in resolvers” (i.e., normalize early).

**Helper: `stixHashesToInput(instance)`**

* Converts a STIX object’s `hashes` map back into GraphQL-style `[{ algorithm, hash }]` entries.
* Uppercases and trims algorithm; trims hash.

**Helper: `extractNotFuzzyHashValues(hashes)`**

* Returns *values* from the hash map excluding fuzzy algorithms (SSDEEP/SDHASH/TLSH/LZJD).
* Useful for “exact hash only” workflows (e.g., matching, indexing, or de-dup logic).

**Why this exists**
Hash algorithms are a “same concept, many conventions” field across systems. These adapters ensure:

* algorithm names are stable (uppercase),
* “regular” hashes are stable (lowercase),
* fuzzy hashes aren’t accidentally treated like exact hashes,
* GraphQL inputs and STIX-ish outputs remain round-trippable.

---

### A.2 Shared attribute/timestamp conventions

This file also defines a few cross-cutting lists used as “shared policy constants”:

* `noReferenceAttributes = ['x_opencti_graph_data']`
  A “do not treat as reference material” attribute list (used by code paths that walk fields for refs/relationships).
* `dateForStartAttributes = ['first_seen', 'start_time', 'valid_from', 'first_observed']`
* `dateForEndAttributes   = ['last_seen', 'stop_time', 'valid_until', 'last_observed']`
* `dateForLimitsAttributes = [...start, ...end]`
  These are “canonical start/end” date attribute names used by time-bounding utilities.

### A.3 “Does this type have updated_at / modified?” predicates

Two predicates tell other layers which families get which “timestamp semantics”:

**`isUpdatedAtObject(type)` returns true for**

* dated internal objects
* STIX meta objects
* STIX core objects
* STIX core relationships
* STIX sighting relationships
* STIX ref relationships

**`isModifiedObject(type)` returns true for**

* STIX meta objects
* STIX domain objects
* STIX core relationships
* STIX sighting relationships

**Practical implication**
Different families participate in different timestamp conventions (e.g., “updated_at exists broadly”, while “modified exists for STIX-domain-ish objects/relationships”). Centralizing that decision prevents each resolver or exporter from hard-coding type lists.

---

## B) Schema utilities (`src/schema/schemaUtils.js`)

This file is the “type normalization + hierarchy reasoning” toolbox used widely by registries and model logic.

### B.1 ID shape checks

* `isStixId(id)` — matches a STIX-ish `type--uuid` shape (regex-based)
* `isInternalId(id)` — checks UUID format (via `validator.isUUID`)
* `isAnId(id)` — convenience: STIX id **or** internal UUID

### B.2 Stable short hashing (for caching / fingerprints)

* `shortHash(element)`:

  * SHA-256 of `JSON.stringify(element)`
  * returns first 8 hex chars

### B.3 Strict ISO date validation

* `isValidDate(stringDate)`:

  * parses date
  * requires `new Date(parsed).toISOString() === stringDate`
  * (i.e., only accepts strings that are already canonical ISO strings)

### B.4 STIX-ish → internal entity type normalization

There are two related helpers:

**`generateInternalType(entity)`**
Maps a STIX-like object payload to the platform’s internal `entity_type` string. It:

* special-cases:

  * sighting relationship types → `stix-sighting-relationship`
  * STIX `relationship` objects → `Stix-Core-Relationship` (abstract core relationship family marker)
  * `identity` → uses `identity_class` (with a special mapping `class → Sector`)
  * `threat-actor` → chooses **individual** vs **group** based on `resource_level`
  * `location` → uses `x_opencti_location_type`
  * `ipv4-addr`, `ipv6-addr`, `file/stixfile`, `ssh-key` → maps to internal constants
* otherwise “pascalizes”:

  * splits into alphanumeric words, capitalizes each, and joins with `-`
  * e.g., `attack-pattern` → `Attack-Pattern`

**`convertStixToInternalTypes(type)`**
A “best-effort” conversion utility (explicitly warned in-file: may yield abstract types). It converts a STIX type string into either:

* an internal concrete type + all parent types, **or**
* a pascalized string for default handling.

Notably, for several special types it returns arrays like:

* `[<internalType>, ...getParentTypes(<internalType>)]`

### B.5 Parent-type expansion (core to registry inheritance)

**`getParentTypes(type)`**
Builds the list of “parent families” for a given concrete type, based on type predicates and the abstract family constants (e.g., `Basic-Object`, `Stix-Object`, `Stix-Core-Object`, `Stix-Domain-Object`, `stix-relationship`, etc.).

The output is used throughout the schema-model layer for inheritance/merging behavior (for example: “attributes defined for a parent family apply to child types too”).

If no parent types can be determined, it throws:

* `DatabaseError(\`Type ${type} not supported`)`

### B.6 “Most restrictive type set” helper

**`keepMostRestrictiveTypes(entityTypes)`**
Given an array like `[Report, Container, Stix-Relationship]`, it removes parent types that are implied by more specific ones:

* example result: `[Report, Stix-Relationship]` (because `Report` already implies `Container`)

This is used when filters, access rules, or query builders accept multiple candidate types and you want the smallest “meaningful” set.

---

## C) How to think about these in the module system

When adding or evolving modules, these two files are the “don’t make every module reinvent normalization” layer:

* If your module introduces hash-like inputs/outputs: use the adapters so your inputs round-trip and remain searchable.
* If your module introduces new `entity_type` strings or new STIX-ish mappings:

  * ensure type-family predicates and parent-type reasoning remain correct (because registries depend on `getParentTypes`)
  * and ensure any STIX→internal mapping you rely on is consistent with `generateInternalType` / `convertStixToInternalTypes`.

In short: **modules define features**, but these utilities define **shared consistency rules** that keep the overall API/model coherent.

---

### 5.4 Deprecated module patterns and legacy code paths

This snapshot contains **two “deprecated” module subtrees** that exist primarily as **backward-compatibility shims**. They are not “full modules” in the modern pattern; instead, they add **small SDL fragments** plus **thin resolvers** that either delegate to newer implementations or preserve an old operation shape long enough for clients to migrate.

---

#### A) Common deprecated-pattern traits (what to look for)

Across both deprecated folders, the pattern is consistent:

1. **A small GraphQL registration wrapper** (`*-deprecated.ts`)

   * Imports a `.graphql` file and a (legacy) resolver module.
   * Calls `registerGraphqlSchema({ schema, resolver })`.

2. **A minimal SDL fragment** (`*.graphql`)

   * Adds **one field** or a tiny type augmentation.
   * Uses GraphQL’s `@deprecated(...)` directive with an explicit **version-window reason**, e.g. `"[>=6.2 & <6.8]. Use …"`.

3. **Legacy resolver implementation in JavaScript** (`*-resolver.js`)

   * Notably **not TypeScript**, and not using the generated `Resolvers` typing pattern used by modern modules.
   * Often “wraps” a newer resolver or delegates directly to domain functions.

4. **A small domain shim** (`*-domain.ts` or `*-domain.js`)

   * Implements the deprecated behavior, typically by calling into newer/central domain logic.

5. **“Region” comments encoding version windows**

   * e.g., `// region [>=6.2 & <6.8]` surrounding the deprecated behavior.

---

#### B) Deprecated subtree: `src/modules/internal/csvMapper/deprecated/*`

**What it provides (deprecated API surface):**

* `deprecated/csvMapper.graphql` adds a deprecated **query**:

  * `Query.csvMapperTest(configuration: String!, content: String!): CsvMapperTestResult`
  * Marked `@deprecated(reason: "[>=6.4 & <6.7]. Use `csvMapperTest mutation`.")`
  * Protected by `@auth(for: [CSVMAPPERS])`.

**How it runs:**

* `deprecated/csvMapper-resolver.js` implements the query and delegates to `csvMapperTest(...)`.
* `deprecated/csvMapper-domain.ts` parses configuration JSON, parses CSV content into lines, runs the CSV “test objects” bundler, and returns a summary (`objects` JSON + relationship/entity counts). It is explicitly tagged as deprecated in-code and points to the mutation replacement.

**Important wiring nuance (why it exists even if not obvious):**

* The wrapper `deprecated/csvMapper-deprecated.ts` registers the deprecated SDL+resolver via `registerGraphqlSchema(...)`.
* That wrapper is pulled in **via side-effect import** from the *module definition* file:

  * `src/modules/internal/csvMapper/csvMapper.ts` includes `import './deprecated/csvMapper-deprecated';`
* Meaning: this deprecated query can be present even if you’re scanning only for `*-graphql.ts` imports.

**Replacement (modern API):**

* The non-deprecated module SDL (`src/modules/internal/csvMapper/csvMapper.graphql`) defines the supported replacement:

  * `Mutation.csvMapperTest(configuration: String!, file: Upload!): CsvMapperTestResult`
  * Same capability (`CSVMAPPERS`), but uses `Upload` instead of embedding file content as a string.

---

#### C) Deprecated subtree: `src/modules/stixCyberObservable/deprecated/*`

This one is even more “shim-like”: it does **not** add a new root operation. Instead, it augments an existing mutation “edit mutations” type.

**What it provides (deprecated API surface):**

* `deprecated/stixCyberObservable.graphql` adds a deprecated field to `StixCyberObservableEditMutations`:

  * `promote: StixCyberObservable @auth(for: [KNOWLEDGE_KNUPDATE])`
  * Marked `@deprecated(reason: "[>=6.2 & <6.8]. Use `promoteToIndicator`.")`
* The intent is explicitly documented in the SDL comment:

  * “this will be merged into the base StixCyberObservableEditMutations”

**How it runs:**

* `deprecated/stixCyberObservable-resolver.js` wraps the platform’s base resolver for:

  * `Mutation.stixCyberObservableEdit(id: …)`
* It returns the base edit-mutations object and injects a legacy field:

  * `promote: () => promoteObservableToIndicator(...)`
* `deprecated/stixCyberObservable-domain.js` implements `promoteObservableToIndicator(...)` and delegates into the newer indicator-creation pathway (loading the observable with refs, enforcing confidence rules, then creating an indicator).

**Wiring nuance:**

* `src/modules/stixCyberObservable/stixCyberObservable.ts` imports the deprecated registration wrapper:

  * `import './deprecated/stixCyberObservable-deprecated';`
  * and notes: “SCO are not implemented as module yet; only the deprecated part of the module is implemented”
* `src/modules/index.ts` imports `./stixCyberObservable/stixCyberObservable` (in the “incomplete modules” region), so this deprecated augmentation becomes active as part of the boot sequence.

**Replacement (modern API):**

* The base type already includes the supported replacement field in this snapshot:

  * `StixCyberObservableEditMutations.promoteToIndicator: Indicator`

---

#### D) Practical implications for documentation and maintenance

* Treat these deprecated folders as **compatibility overlays**, not exemplars of how to build new modules.
* When documenting “what exists,” include them as **legacy surfaces** with clear “use X instead” guidance:

  * `Query.csvMapperTest(...)` → `Mutation.csvMapperTest(...)`
  * `stixCyberObservableEdit(...).promote` → `stixCyberObservableEdit(...).promoteToIndicator`
* When reasoning about “what is wired,” note the **side-effect import trap**:

  * deprecated schema can be registered via imports inside module definition files, not only via `*-graphql.ts` imports listed in `src/modules/index.ts`.

---

## 6) API reference docs (generated)

### 6.1 Canonical schema artifact (merged SDL) generation plan

This snapshot’s “canonical schema” is **not** just `opencti.graphql`. The server builds an *executable* schema by accumulating SDL fragments (core + runtime-injected + module SDL) and then running schema transforms.

This section defines the **repeatable** process for producing the canonical merged SDL artifact for docs.

---

#### A) What “canonical merged SDL” means (for this doc set)

For documentation purposes, we want a single `.graphql` artifact that represents the **same schema inputs the server runs**, meaning:

1. **Core baseline SDL**
   From: `opt/opencti/config/schema/opencti.graphql`
   This is the first entry in `schemaTypeDefs` inside `src/graphql/schema.js`.

2. **Runtime-injected SDL** (important!)
   From: `src/graphql/schema.js`
   The GraphQL layer injects `rateLimitDirectiveTypeDefs` from `graphql-rate-limit-directive` by doing:

   * `const { rateLimitDirectiveTypeDefs, rateLimitDirectiveTransformer } = rateLimitDirective(...)`
   * `schemaTypeDefs.push(rateLimitDirectiveTypeDefs)`

   This matters because the core SDL **uses** `@rateLimit(...)` (e.g., on `otpLogin`) but does **not** define `directive @rateLimit` in `opencti.graphql`. The definition comes from this injection.

3. **All wired module SDL** (and only wired module SDL)
   Inclusion is controlled by `src/modules/index.ts` (GraphQL registration region), which imports module `*-graphql` wrapper files in a deliberate order. Each wrapper (e.g., `src/modules/theme/theme-graphql.ts`) calls:

   * `registerGraphqlSchema({ schema: <module>.graphql, resolver: <module>Resolvers })`

   So the authoritative list/order of module SDL inputs is:
   `src/modules/index.ts` → `import './<module>/<module>-graphql'` → wrapper imports `./<module>.graphql`.

**Non-goal:** “every `*.graphql` file in the filesystem.” Some SDL files exist but are not wired into the executable schema and should **not** be treated as part of the API surface.

---

#### B) Ordering rules (why the merged artifact must be ordered)

Schema composition is order-sensitive in this architecture:

* In `src/graphql/schema.js`, modules extend the schema by **pushing** into arrays:

  * `schemaTypeDefs.push(schema)`
  * `schemaResolvers.push(resolver)`

* Therefore, the effective SDL ordering is:

  1. `opencti.graphql` (core baseline)
  2. `rateLimitDirectiveTypeDefs` (injected by `schema.js`)
  3. module SDL fragments **in the same order** the `*-graphql` wrappers are imported (driven by `src/modules/index.ts`)

Even if GraphQL SDL merging is tolerant, keeping the real import order makes the artifact:

* diff-friendly between snapshots
* easier to map an operation/type back to its source file
* faithful to how the runtime is assembled

---

#### C) Two practical ways to generate the canonical artifact

##### Option 1 (recommended): Build the executable schema and print it (authoritative)

This matches the server most closely because it uses the same assembly code path and includes runtime injections and post-build transforms.

**High-level steps:**

1. Import `src/modules/index.ts` **first** (side-effect imports run all `registerGraphqlSchema(...)` calls).
2. Import `createSchema` from `src/graphql/schema.js`.
3. Call `createSchema()` to get a `GraphQLSchema`.
4. Print to SDL using a schema printer (e.g., `printSchema` / a “print with directives” helper).

**Why this is authoritative:**

* Includes `rateLimitDirectiveTypeDefs` (injected in `schema.js`)
* Reflects schema transforms applied after `makeExecutableSchema(...)`, including:

  * `constraintDirectiveDocumentation()(schema)`
  * `authDirectiveTransformer(schema)`
  * `rateLimitDirectiveTransformer(schema)`

> Note: `src/graphql/schema.js` does **not** import `src/modules/index.ts` itself, so a docs generator must explicitly import the modules manifest before calling `createSchema()` or the module SDL won’t be registered.

##### Option 2: Offline “static merge” of SDL files (fast, but must be patched)

If you can’t execute the schema builder, you can still generate a deterministic merged SDL by concatenating sources **in runtime order**:

1. Start with `opt/opencti/config/schema/opencti.graphql`
2. Append **a known copy** of the `@rateLimit` directive SDL (sourced from the same `graphql-rate-limit-directive` version used by the image), because it is **not** present in core/module SDL files
3. Append wired module SDL in the exact `src/modules/index.ts` GraphQL-registration import order

**Caution:** a static merge will not reflect any schema-level changes introduced by transforms (if a transform modifies schema structure rather than only wrapping resolvers). It’s still useful for navigation and quick diffs, but “generated API reference” should prefer Option 1 or introspection output.

---

#### D) What we should treat as “not part of the canonical merged schema”

* Any `*.graphql` file that is **not** registered via a `*-graphql` wrapper imported by `src/modules/index.ts`
* “Deprecated/unwired” SDL files can be preserved as **audit appendices**, but should not be included in the canonical API artifact that drives generated reference pages.

(Practically: keep them as separate artifacts like `unwired.graphql` or an appendix section in the merged file, but do not mix them into the “this is what the server runs” schema.)

---

#### E) Snapshot check: how the current artifacts line up

The provided merged artifact (`opencti-schema-snapshot-2026-01-27_141500_merged.graphql`) is a **static** merge that:

* includes core SDL
* includes module SDL in `src/modules/index.ts` import order
* also includes the two “present but not wired” deprecated SDL files as an appendix

However, it **does not include** the runtime-injected `rateLimitDirectiveTypeDefs`, so it is **not fully canonical** for “what the server runs,” especially around `@rateLimit`.

---

#### F) Recommended outputs for the docs pipeline

For each snapshot, generate and store:

1. **`schema.executable.graphql`** (authoritative)
   Printed from `createSchema()` after importing `src/modules/index.ts`

2. **`schema.sources-merged.graphql`** (optional, diff-friendly)
   Static concatenation of `opencti.graphql` + wired module SDL (plus an explicit injected `@rateLimit` SDL block if you want it to be closer to runtime)

3. **`schema.inventory.json`**
   A machine-readable index mapping:

   * module → SDL file(s) → root field contributions
   * list of wired vs unwired SDL files
   * counts (types/inputs/enums/unions, root field counts)

These three together give you: correctness (1), traceability (2), and automation hooks (3).

---

### 6.2 Static docs generation plan (rendering strategy)

This section defines **how we turn the snapshot’s merged schema into a browsable static documentation site**, while preserving the runtime semantics that are *not obvious from SDL alone* (especially `@auth`).

---

#### A) Inputs the generator must use (authoritative)

Same as **6.1**, plus one extra requirement:

1. **Canonical merged schema input (preferred):** build the executable schema via `src/graphql/schema.js` *after importing* `src/modules/index.ts`, then print it to SDL (or emit introspection JSON).

   * `schema.js` injects `rateLimitDirectiveTypeDefs` and applies schema transforms.
   * Transform order in `schema.js` is:

     * `constraintDirectiveDocumentation()(schema)`
     * then `rateLimitDirectiveTransformer(authDirectiveTransformer(schema))`

2. **Directive runtime semantics:** `src/graphql/authDirective.ts` must be treated as the source of truth for *what `@auth` (and related escape-hatch directives) actually do at runtime*, not just what the SDL says.

---

#### B) Generation approach: parse once, then render multiple “views”

Use the canonical schema artifact as input and do:

1. **Parse SDL → AST** (for descriptions + directive applications + “where defined” mapping).
2. **Build a GraphQLSchema** (for type resolution, argument/type formatting, interface/union membership).
3. **Compute derived indexes** (root operation catalogs, directive usage maps, capability maps, “public surface” list, etc.).
4. **Render Markdown pages** (or MDX) into a static site framework (Docusaurus/MkDocs/etc.).

Key rule: **don’t rely on server introspection at runtime** for generation, because introspection is policy-gated in this platform. The generator should be able to run purely from the snapshot artifacts.

---

#### C) What the static site should contain

##### C.1 API reference (generated, exhaustive)

Generate pages for:

* **Root operations**

  * `Query` fields
  * `Mutation` fields
  * `Subscription` fields
* **Types / Interfaces / Inputs / Enums / Unions / Scalars**
* **Directive catalog**

  * definitions + argument signatures
  * plus *curated semantics blurbs* for `@auth` family

Recommended layout:

* `/api/query/<field>.md`
* `/api/mutation/<field>.md`
* `/api/subscription/<field>.md`
* `/api/types/<TypeName>.md`
* `/api/directives/<directive>.md`
* `/api/enums/<EnumName>.md`
* `/api/inputs/<InputName>.md`

##### C.2 Indices (generated, navigational)

Generate “maps” that make a 600+ type schema usable:

* **All operations** (3 lists: Query/Mutation/Subscription)
* **Operations by module SDL file** (traceability to `src/modules/**`)
* **Operations by directive**

  * `@public` operations (explicit list)
  * `@auth` operations (with required caps)
  * `@rateLimit` operations
* **Capabilities map** (see below): capability → operations requiring it
* **Search index** (lunr/algolia local) over operation/type names + descriptions

---

#### D) Auth semantics that must be reflected in docs output (from `authDirective.ts`)

The generator must not treat `@auth` as a decorative annotation. The runtime transformer implements specific rules we should mirror in documentation output:

1. **“Secure schema” invariant (Query/Mutation)**
   During schema transformation, for **Query** and **Mutation** fields only:
   if a field has **neither** `@auth` nor `@public`, schema creation throws `UnsupportedError('Unsecure schema: missing auth or public directive', { field })`.
   Doc implication: the generator can (and should) validate this invariant and fail docs generation if it’s violated (helps catch drift or partial snapshots).

2. **Type-level `@auth` inheritance**
   The transformer records `@auth` on object types and applies it as a **fallback** for fields without an explicit field-level `@auth`.
   Doc implication: when rendering “required capabilities” for a field, compute the **effective auth directive** as:

   * field `@auth` if present, else
   * parent type `@auth` if present, else
   * (for Query/Mutation) field must be `@public` or it’s invalid.

3. **`@auth(for: [])` still means “authentication required”**
   The wrapped resolver always requires `context.user`; if no user it throws `AuthRequired()`.
   Capability checks are skipped only when `requiredCapabilities.length === 0`, but **auth still applies** (and so do OTP/LTS rules).

4. **OTP and LTS gating are enforced inside the auth wrapper**

   * If OTP is mandatory (or user has OTP activated) and the field is **not** annotated `@allowUnprotectedOTP`, the resolver can throw `OtpRequiredActivation()` / `OtpRequired()`.
   * If the user is blocked for LTS validation and the field is **not** annotated `@allowUnlicensedLTS`, it throws `LtsRequiredActivation()`.

   Doc implication: on each operation page, include:

   * “OTP bypass allowed: yes/no” (presence of `@allowUnprotectedOTP`)
   * “LTS bypass allowed: yes/no” (presence of `@allowUnlicensedLTS`)

5. **Capability matching is prefix/substring-based**
   Capability checks use:

   * `userCapabilities.some(u => requestedCapability !== BYPASS && u.includes(requestedCapability))`
     and `and: true` enforces **all**, otherwise **any**.
     Doc implication: when rendering capability requirements, note that a required capability is satisfied by any user capability that *contains* it (hierarchical/prefix-style behavior), not strict equality.

6. **BYPASS/system user short-circuit**
   If the user has `BYPASS` or is the admin UUID, auth checks short-circuit and the resolver executes.

7. **Draft-mode capability augmentation (feature-flagged)**
   If `CAPABILITIES_IN_DRAFT` is enabled and request is in a draft context, `capabilitiesInDraft` are unioned into base capabilities before evaluation.
   Doc implication: include this on the `@auth` directive semantics page as a conditional behavior.

---

#### E) How to render “required capabilities” consistently

For each root field page (`Query.x`, `Mutation.y`, `Subscription.z`), compute and display:

* **Protection mode**

  * `@public` (unauthenticated) OR
  * `@auth` (authenticated)
* **Required capabilities**

  * list from `@auth(for:[...])`
  * show whether it’s **ANY** (default) or **ALL** (`and: true`)
* **OTP/LTS escape hatches**

  * `@allowUnprotectedOTP` present?
  * `@allowUnlicensedLTS` present?
* **Rate limit**

  * if `@rateLimit(...)` present, show its arguments (best-effort from SDL AST)

Also generate the index pages:

* **Capability → operations**
* **Operation → required capabilities**

(These indices become the backbone for section **4.4** and for operator/security review.)

---

#### F) Output validation checks (so the generator catches bad snapshots)

As part of the docs build, run these checks:

1. **Query/Mutation fields all have `@auth` or `@public`** (matches runtime “unsecure schema” rule).
2. **Directive catalog completeness**

   * `@rateLimit` must be present in the canonical schema input (because `schema.js` injects it).
3. **Wiring sanity**

   * every module SDL included should correspond to a `*-graphql` wrapper imported by `src/modules/index.ts` (unless explicitly appended as “unwired/deprecated appendix” in a non-canonical artifact).

---

#### G) Recommended deliverables per snapshot

* `site/` (static HTML)
* `api/` (generated markdown source)
* `schema.executable.graphql` (canonical)
* `schema.inventory.json` (counts + file/module mapping + indices used to render nav)

This gives you: a stable published artifact + the exact schema inputs used to generate it + the machine index to regenerate/verify later.

---

### 6.3 Breaking-change gating plan (schema diffs)

This section defines how we detect and **block breaking GraphQL changes** between snapshots, using the **merged/executable schema** as the source of truth and `MANIFEST.txt` as the identity/version anchor.

---

#### A) Snapshot identity (what a diff is “between”)

Every schema diff report must embed the two snapshots’ identities, taken from each snapshot’s `MANIFEST.txt`. In *this* snapshot, `MANIFEST.txt` declares:

* `timestamp=2026-01-27_141500`
* `container=identity-core-opencti-1`
* `image=opencti/platform:6.9.10`
* `created=2026-01-27T21:12:58.975722779Z`

**Doc/ops implication:** a schema diff is not meaningful unless it states the exact `{image tag + snapshot timestamp}` for both “before” and “after”.

---

#### B) Source-of-truth artifact to diff

To avoid false positives/negatives, diffs should be run on the **canonical executable schema output** (the one we generate by running the same schema assembly path the server uses), not on a hand-merged concatenation.

Reason: `src/graphql/schema.js` injects SDL and applies transforms that aren’t guaranteed to appear in static SDL files, notably:

* injection of `rateLimitDirectiveTypeDefs`
* post-build schema transforms (constraint docs + auth transformer + rate-limit transformer)

**Therefore, the gating diff input should be one of:**

1. `schema.executable.graphql` (printed from a built `GraphQLSchema`), or
2. `schema.introspection.json` (introspection result from the built schema)

If we *also* keep a “sources-merged” artifact for traceability, it should be treated as a secondary, human-friendly diff—not the gating truth.

---

#### C) Two diff channels: API signature vs access-policy semantics

We gate changes in two distinct ways:

##### C.1 API signature diffs (structural compatibility)

This is the classic GraphQL compatibility layer: types/fields/args/nullability/enums/inputs.

Run a schema diff tool that understands GraphQL semantics (e.g., GraphQL Inspector or equivalent) against the canonical schema artifacts and classify changes as:

**Breaking (block)**

* Removed type / removed field
* Removed enum value (clients may still send/expect it)
* Removed or renamed argument
* Argument type changed incompatibly (including nullability tightening)
* Field return type changed incompatibly (including nullability tightening)
* Input field removed
* Input field type tightened (including making nullable → non-null)
* Interface/union membership changes that remove previously possible runtime types

**Potentially breaking / policy-driven (review or block depending on rules)**

* Adding a *required* argument (non-null without default) to an existing field
* Adding a required input field (non-null without default) to an existing input object

**Non-breaking (allow)**

* Adding new types
* Adding new fields
* Adding new optional arguments
* Adding new optional input fields
* Adding enum values (usually safe, but see notes below)

> Note on enums: adding values is typically non-breaking for clients, but can be breaking for strict client-side validation or switch exhaustiveness checks. In gating, we can mark “enum additions” as allowed but surfaced prominently.

##### C.2 Access-policy diffs (directive-driven behavior)

Because OpenCTI uses directives as *runtime enforcement controls*, we separately diff directive annotations and treat them as first-class changes.

At minimum, diff these per operation field:

* `@public` presence/absence
* effective `@auth(for:[...], and: ...)` requirements (including inherited type-level `@auth`)
* `@allowUnprotectedOTP`
* `@allowUnlicensedLTS`
* `@rateLimit(...)` arguments (behavioral throttling changes)

**Policy-breaking (block unless explicitly approved)**

* `@public` → `@auth` (tightening) or `@auth` → `@public` (exposing)
* Any change to required capabilities list or `and:` mode
* Adding/removing OTP/LTS escape hatch directives

**Behavioral-only (flag for review)**

* `@rateLimit` parameter changes (not a GraphQL “signature” break, but an operational behavior change clients will feel)

This policy-diff channel matters because a schema can be “structurally compatible” while still changing who can call what.

---

#### D) The gating workflow (repeatable pipeline)

For each snapshot build (or PR that changes schema/module wiring):

1. **Generate canonical schema artifacts**

   * Produce `schema.executable.graphql` (or `schema.introspection.json`) using the same assembly approach described in 6.1:

     * import `src/modules/index.ts` (side-effect registrations)
     * call `createSchema()` from `src/graphql/schema.js`
     * print/emit

2. **Normalize for stable diffs**

   * Prefer a consistent printer to reduce noise (stable ordering).
   * Treat description-only changes as non-breaking (but keep them visible).

3. **Run semantic schema diff**

   * Output:

     * machine JSON (`schema_diff.json`)
     * human markdown (`schema_diff.md`)
   * Classify into breaking / potentially breaking / non-breaking.

4. **Run directive/policy diff**

   * Parse SDL AST and compute an “operation policy table”:

     * operation → `{ public?, authCaps[], authAnd?, otpBypass?, ltsBypass?, rateLimit? }`
   * Diff old vs new table.
   * Output:

     * `policy_diff.json`
     * `policy_diff.md`

5. **Gate rules**

   * Fail the build if:

     * any **breaking** API changes exist, OR
     * any **policy-breaking** changes exist without an explicit allowlist/approval marker.

---

#### E) Allowlisting and intentional breaks

Some changes really are intentional (major releases, snapshot-to-snapshot upgrades). We handle that cleanly:

* Maintain a **human-reviewed allowlist** keyed by:

  * `{from image/tag, to image/tag}` or `{from timestamp, to timestamp}`
  * and the exact diff item identifiers (tool-provided IDs)
* Require an explicit “BREAKING_CHANGE_APPROVED” signal in the docs/build metadata for allowlisted breaking diffs.

This prevents “oops we broke clients” while still enabling deliberate evolution.

---

#### F) Reporting format (what reviewers should see)

Every diff report should include, at the top:

* “From” snapshot identity (from `MANIFEST.txt`)
* “To” snapshot identity (from its `MANIFEST.txt`)
* Summary counts:

  * breaking / potentially breaking / non-breaking
  * policy-breaking / behavioral-only
* Affected surface area slices:

  * root operation diffs first (`Query`/`Mutation`/`Subscription`)
  * then types/inputs/enums

And it should include a “blame trail” hint:

* operation/type → which SDL file likely introduced it (using our inventory mapping / module source mapping)

---

#### G) Why this plan fits the OpenCTI snapshot model

* Schema is assembled programmatically and includes runtime-injected SDL; diffing “just files” is insufficient.
* Authorization is enforced by a transformer (`authDirective.ts`), so directive changes are *real* behavior changes, not cosmetic metadata.
* The snapshot is explicitly versioned (`opencti/platform:6.9.10` here), making schema diffs a natural upgrade safety gate between platform versions.

---

## 7) Contributor guides (process + checklists)

### 7.1 Adding a new module (checklist template)

This checklist matches the **snapshot module pattern**: a module is typically (1) registered into the *schema-model registry* and (2) registered into the *GraphQL executable schema* via a `*-graphql.ts` wrapper, with both lanes wired explicitly in `src/modules/index.ts`.

---

#### A) Choose what kind of module you’re adding

* [ ] **Full module (recommended)**: registers a new entity/type into the **schema-model** (`registerDefinition(...)`) *and* registers GraphQL SDL + resolvers (`registerGraphqlSchema(...)`).
* [ ] **GraphQL-only module (feature surface only)**: registers only SDL + resolvers (no schema-model definition).
  *In this snapshot, some modules follow this pattern, but most “real entities” use the full model.*

---

#### B) Scaffold the module files (theme is a good reference)

Create `src/modules/<moduleName>/` and add:

**1) Schema-model lane (full modules)**

* [ ] `<moduleName>.ts` — module definition that calls `registerDefinition(...)`.

  * [ ] Provide a `ModuleDefinition` with:

    * [ ] `type: { id, name, category, aliased? }`

      * Category must be one of: `Case | Container | Location | Identity | Stix-Domain-Object | Stix-Meta-Object | Internal-Object | Threat-Actor`.
    * [ ] `identifier.definition` (and optional `identifier.resolvers`)
    * [ ] `attributes: AttributeDefinition[]`
    * [ ] `relations: [{ name, targets: RelationDefinition[] }]` (can be empty)
    * [ ] optional: `relationsRefs`, `validators`, `depsKeys`, `bundleResolver`, `overviewLayoutCustomization`
    * [ ] `representative` and `converter_2_1` (required)

**2) GraphQL lane (all modules that expose API surface)**

* [ ] `<moduleName>.graphql` — SDL (types, inputs, root fields)
* [ ] `<moduleName>-resolvers.ts` (or `-resolver.ts`) — resolver map exporting `Resolvers`
* [ ] `<moduleName>-graphql.ts` — the registration shim that imports SDL + resolvers and calls:

  * [ ] `registerGraphqlSchema({ schema: <typeDefs>, resolver: <resolvers> })`

**3) Typical “supporting” files (strongly recommended)**

* [ ] `<moduleName>-domain.ts` — domain functions called by resolvers (thin-controller pattern)
* [ ] `<moduleName>-types.ts` — Store entity + STIX types used by converter/representative
* [ ] `<moduleName>-converter.ts` — STIX 2.1 conversion
* [ ] Any constants/helpers as needed (`<moduleName>-constants.ts`, etc.)

---

#### C) Wire it into the runtime manifest (`src/modules/index.ts`)

> Import order is **not optional**; follow the file’s region structure.

* [ ] Add schema-model registration import (full modules only) under:

  * **“registration modules, need to be imported before graphql code registration”**
  * Example pattern: `import './<moduleName>/<moduleName>';`

* [ ] Add GraphQL registration import under:

  * **“graphql registration”**
  * Example pattern: `import './<moduleName>/<moduleName>-graphql';`

* [ ] Keep imports grouped and sorted consistently with nearby modules (don’t move the region blocks).

---

#### D) SDL rules of thumb (so it composes cleanly)

* [ ] Define your object type(s) and implement the appropriate interface(s) (commonly `BasicObject` + `InternalObject` or STIX interfaces depending on the entity).
* [ ] Add Relay-ish connection types (`<Type>Connection`, `<Type>Edge`) if you expose list queries.
* [ ] Add an ordering enum if you support ordering.
* [ ] Root operations:

  * [ ] Any `Query` / `Mutation` / `Subscription` field you add should be explicitly annotated with **`@auth(...)` or `@public`** (don’t leave access ambiguous).
* [ ] Inputs:

  * [ ] Use `@constraint(...)` where needed (e.g., `minLength`, `format: "not-blank"`).
* [ ] Ensure every SDL root field has a corresponding resolver entry (or is intentionally resolved by default field resolution).

---

#### E) Schema-model rules of thumb (full modules)

* [ ] Pick the correct `category` (it drives how `registerDefinition(...)` registers the type into the platform’s type families and converter registries).
* [ ] If `aliased: true`:

  * [ ] Expect alias fields to be automatically added during registration (so ensure the entity should truly behave as “aliased”).
* [ ] If you add `relations`:

  * [ ] Make sure the relation name and targets match the relationship vocabulary you intend to support, since registration updates internal relationship mappings.
* [ ] If you add `relationsRefs`:

  * [ ] Ensure the `databaseName` values are correct, because registration also adds these into the `stix-ref-relationship` family.

---

#### F) “Don’t forget” cross-cutting updates

* [ ] If your new type should appear in the **big umbrella unions** (e.g., “any STIX object/relationship” style unions), update the relevant unions in `opt/opencti/config/schema/opencti.graphql`.
  *(This snapshot’s schema model expects some unions to be manually maintained.)*

---

#### G) Verification (minimum bar before merge)

* [ ] Build/assemble the executable schema (the same way the server does) and ensure:

  * [ ] SDL merges without conflicts
  * [ ] No missing resolvers for declared root fields
  * [ ] Directive usage (`@auth`, `@public`, `@constraint`, `@rateLimit` if used) is valid
* [ ] Run your schema diff gate (if you’re changing an existing API surface):

  * [ ] no unintended breaking changes
  * [ ] no unintended access-policy changes (especially `@public` / `@auth` transitions)

---

## 7.2 Adding/changing fields/types safely (schema + validators)

This snapshot has **two complementary “safety rails”** you should treat as the baseline contract when evolving the API:

1. **GraphQL schema correctness** (SDL shape, nullability, deprecations, directives)
2. **Input validation at runtime** (schema-model attribute/ref registries + optional per-entity functional validators)

The goal is: *add features without breaking clients, and reject invalid writes early and consistently.*

---

### A) Schema-level safety rules (GraphQL surface)

**Prefer additive changes.** The safest changes in GraphQL are:

* Add a new field (or a new type / input / enum value).
* Add a new optional argument.
* Add a new optional input field.

**Avoid breaking changes unless you’re intentionally versioning.** In practice, the most common breaking changes are:

* Removing a field or type.
* Changing a field’s type.
* Tightening nullability (`String` → `String!`, or removing `null` from lists).
* Renaming fields/arguments.
* Changing semantics of an existing field without deprecating it first.

**Use deprecations instead of removals.** GraphQL supports `@deprecated`; in this snapshot model, deprecate first, then remove later with explicit gating reminders.

**Keep “edit mutation” conventions stable.** In the core SDL, the platform’s write model heavily uses `EditInput` / `EditOperation`:

```graphql
enum EditOperation { add replace remove }

input EditInput {
  key: String!
  object_path: String
  value: [Any]!
  operation: EditOperation # Undefined = REPLACE
}
```

If you introduce new editable fields, make sure they fit the platform’s edit patterns (and the validators described below).

---

### B) Runtime validation model: what actually blocks bad writes

The primary write-validation entrypoints are in `src/schema/schema-validator.ts`:

* `validateInputCreation(...)`
* `validateInputUpdate(...)`

These functions implement **two layers**:

1. **Generic/schema-driven validation** (format + multiplicity + mandatory checks)
2. **Functional/per-entity validation** (optional custom logic registered per type)

Both are wrapped with telemetry spans (`telemetry(...)`) labeled like **“CREATION VALIDATION”** and **“UPDATE VALIDATION”**.

---

### C) Generic validation (format/multiplicity) for attributes

The core “format + multiplicity” check is `validateAndFormatSchemaAttribute(attributeName, attributeDefinition, editInput)`.

It only runs if:

* an `attributeDefinition` exists **and**
* `editInput.value` is not empty (empty values are effectively ignored by this validator)

Key behaviors to preserve when adding/changing fields:

#### C.1 Multiplicity enforcement

If the attribute is not `multiple` and you provide more than one value, it throws a `ValidationError('Attribute cannot be multiple', ...)`.

One exception exists:

* If the attribute is an `object` and the edit input includes `object_path`, it treats it as a **patch inside an object** and skips multiplicity enforcement at this layer (comment indicates deeper validation happens elsewhere).

#### C.2 String trimming and JSON schema validation

For `type === 'string'`:

* Values must be strings (or falsy), otherwise it throws.
* It **trims** each string value and **mutates** `editInput.value` to the trimmed list (explicitly intended to reduce “useless stream events”).
* If `format === 'json'` and `schemaDef` is present on the attribute definition:

  * It compiles the JSON Schema via **Ajv**
  * Parses the (single) JSON string value
  * Rejects invalid JSON schema matches with `ValidationError('The JSON schema is not valid', ...)`

#### C.3 Booleans, dates, numerics

* `boolean`: accepts boolean **or string** values; rejects other types.
* `date`: code accepts strings and also other values **if** `utcDate(value).isValid()` is true (the comment says “ISO date string,” but the actual check is “string OR anything Moment can parse as valid”).
* `numeric`: accepts anything convertible by `Number(value)` (rejects `NaN`).

**Implication for schema evolution:** if you add a new editable field and expect strict validation, you need its **attribute definition** to be present in the schema attribute registry that this validator consults (see below), otherwise the validator won’t enforce these rules for that field.

---

### D) Mandatory attribute enforcement (entity settings + capability bypass)

Mandatory checks are done via:

* `validateMandatoryAttributesOnCreation(...)`
* `validateMandatoryAttributesOnUpdate(...)`

These checks pull per-type attribute configuration from the **entity settings** model (via `getAttributesConfiguration(entitySetting)`).

Important enforcement nuances:

* If the user has capability `KNOWLEDGE_KNUPDATE_KNBYPASSFIELDS`, mandatory fields can be bypassed (mandatory list becomes empty).
* On **creation**, if the entity setting has `enforce_reference` enabled, then **at least one external reference** becomes mandatory unless the user has `KNOWLEDGE_KNUPDATE_KNBYPASSREFERENCE`.

Creation vs update semantics differ:

* **Creation:** mandatory keys must be present **and** have at least one non-empty value.
* **Update:** mandatory keys only matter **if you touch them**—but if you include a mandatory key in the update, its new value must not be empty.

---

### E) Update safety: “is this field even updatable?”

Updates get an additional guard: `validateUpdatableAttribute(instanceType, inputRecord)`.

It checks **both** registries:

* `schemaAttributesDefinition.getAttribute(instanceType, key)` (attributes)
* `schemaRelationsRefDefinition.getRelationRef(instanceType, key)` (relation refs)

If neither exists **or** `update === false`, the key is rejected and `validateInputUpdate` throws:

* `ValidationError('You cannot update incompatible attribute', <firstInvalidKey>)`

**Implication:** if you introduce a new field that should be editable, you must ensure:

* it exists in the relevant registry **and**
* `update` is not set to `false` for that key.

This applies to both standard attributes **and** relation-ref style fields.

---

### F) Functional validators (per-entity rules) and how they’re wired

The optional “business logic” validators are a lightweight registry in `src/schema/validator-register.ts`:

* `registerEntityValidator(type, { validatorCreation?, validatorUpdate? })`
* `getEntityValidatorCreation(type)`
* `getEntityValidatorUpdate(type)`

`schema-validator.ts` will call these validators (if present):

* **Creation:** `validator(context, user, input)` → must resolve to `true`, else rejects with `UnsupportedError('The input is not valid', ...)`
* **Update:** `validator(context, user, instanceFromInputs, initial)` → same expectation

**When to use functional validators:**

* cross-field constraints (e.g., “if field A is set, field B must be set”)
* semantic validation that can’t be expressed via `@constraint`
* compatibility rules during transitions (temporary acceptance logic while deprecating fields)

---

### G) Type-shape utilities that influence “safe type additions”

`src/schema/schemaUtils.js` is relevant to safety because it encodes type mapping and type hierarchy assumptions that other parts of the system rely on:

* `generateInternalType(entity)` converts STIX-ish input (`entity.type`, plus some subtype fields) into internal type strings.
* `getParentTypes(type)` derives abstract parent types (basic/stix/internal/etc.). If a type isn’t recognized as a supported object or relationship family, it throws `DatabaseError('Type <x> not supported')`.
* `keepMostRestrictiveTypes(types)` removes redundant parent types (keeps the “most specific” set).

**Implication:** when you add a **new entity type** (especially one that comes from STIX ingestion or external inputs), make sure it:

* maps correctly via `generateInternalType` (or via the default pascalization),
* is recognized by the family predicates used by `getParentTypes`,
* and doesn’t create inconsistent parent/child hierarchies that would break `keepMostRestrictiveTypes`.

---

### H) Practical checklist for “safe changes” in this snapshot model

When adding/changing fields/types:

1. **SDL change**

   * Add fields/types in the correct SDL location (core vs module).
   * Apply `@auth(...)` / `@public` appropriately.
   * Use `@deprecated` rather than removing/renaming.

2. **Validation + edit support**

   * If the field is writable:

     * ensure it exists in the schema attribute or relation-ref registry (so format/updatable checks apply)
     * confirm multiplicity (`multiple`) and update policy (`update !== false`) are correct
   * Use `@constraint` for “shape” rules (lengths, patterns, and `not-blank`).

3. **Business rules**

   * If you need cross-field logic, add a functional validator and register it via `registerEntityValidator(...)`.

4. **Type hierarchy safety**

   * For new types: ensure `schemaUtils` mapping + parent-type derivation won’t reject or misclassify it.

---

### 7.3 Adding auth to new operations (required patterns)

This snapshot enforces authorization **at schema-build time** via the `@auth` directive transformer (`src/graphql/authDirective.ts`) applied during schema creation (`src/graphql/schema.js`). The practical rule is:

* **Every `Query` and `Mutation` field must be annotated with either `@auth` or `@public`.**

  * If you forget, schema construction throws: **`Unsecure schema: missing auth or public directive`** (and the server won’t start).

#### 7.3.1 Directive contract (SDL)

From `opt/opencti/config/schema/opencti.graphql`:

```graphql
directive @auth(for: [Capabilities] = [], and: Boolean = false) on OBJECT | FIELD_DEFINITION
directive @public on OBJECT | FIELD_DEFINITION
directive @allowUnprotectedOTP on OBJECT | FIELD_DEFINITION
directive @allowUnlicensedLTS on OBJECT | FIELD_DEFINITION
```

Key semantics:

* `@auth` with **no `for:`** (or `for: []`) means: **authenticated user required**, but **no capability check** beyond being logged in.
* `@auth(for: [...])` means: authenticated user + capability check.
* `and: true` means **ALL** listed capabilities are required; otherwise it’s **ANY** (OR).

#### 7.3.2 What runtime enforcement actually does

When a field is protected by `@auth`, the wrapper enforces (in order):

1. **Authentication required**: no `context.user` → `AuthRequired()`.
2. **OTP enforcement**, unless the field is annotated `@allowUnprotectedOTP`:

   * If OTP is mandatory and the session OTP is not validated → throws `OtpRequiredActivation()` (if OTP not activated) or `OtpRequired()`.
   * If OTP is optional but user activated OTP and session isn’t validated → `OtpRequired()`.
3. **LTS license activation enforcement**, unless the field is annotated `@allowUnlicensedLTS`:

   * If `blocked_for_lts_validation` is true → `LtsRequiredActivation()`.
4. **Capability check** (unless `for` is empty):

   * **Bypass**: if the user has `BYPASS` capability *or* is the platform admin UUID, the capability check is skipped.
   * Capability matching is **substring-based** (`userCapability.includes(requiredCapability)`), which supports “capability prefix” patterns.
   * If not granted → `ForbiddenAccess()`.

> Note: this “must have @auth or @public” enforcement is applied specifically to **`Query` and `Mutation`** fields. Subscriptions are not guarded by that startup check in this file, but you should still treat them as requiring explicit access control.

#### 7.3.3 Required patterns when adding a new operation

##### A) Most operations should be `@auth(...)`

**New query (authenticated-only, no capability gate):**

```graphql
type Query {
  myNewThing(id: ID!): Thing @auth
}
```

**New mutation (capability-gated):**

```graphql
type Mutation {
  myNewThingAdd(input: ThingAddInput!): Thing @auth(for: [KNOWLEDGE_KNUPDATE])
}
```

**Require multiple capabilities:**

```graphql
type Mutation {
  mySensitiveOp(input: X!): Y @auth(for: [SETTINGS_SETACCESSES, SETTINGS_SETMARKINGS], and: true)
}
```

##### B) Use `@public` only for truly unauthenticated entrypoints

If you mark an operation `@public`, **authDirective.ts will not wrap it**, so it will not require `context.user`. Make sure the resolver and any called helpers can safely handle unauthenticated context.

```graphql
type Query {
  publicLandingData: PublicLandingData @public
}
```

**Important nuance:** A `@public` root field can still return types that have **some fields protected by `@auth`** (field-level). That’s a valid pattern in this snapshot: unauthenticated clients must simply not request protected subfields.

##### C) OTP + LTS “escape hatch” directives (only meaningful on `@auth` fields)

These exist specifically to keep certain authenticated flows reachable even when OTP or LTS validation gates would otherwise block them.

* Use `@allowUnprotectedOTP` for operations needed to **generate/activate/validate OTP** or **logout** even when OTP isn’t yet validated.
* Use `@allowUnlicensedLTS` for operations needed to **complete LTS license activation** (and anything else explicitly required during that blocked state).

Example pattern:

```graphql
type Mutation {
  otpLogin(input: UserOTPLoginInput): Boolean @auth @allowUnprotectedOTP
  setupEnterpriseLicense(input: LicenseActivationInput!): Settings
    @auth(for: [SETTINGS_SETPARAMETERS, SETTINGS_SETMANAGEXTMHUB])
    @allowUnlicensedLTS
}
```

#### 7.3.4 Implementation checklist

When you add a new operation:

1. **Annotate it**: `@auth(...)` or `@public` (required for Query/Mutation).
2. If `@auth`:

   * Choose `for: [...]` capabilities (or leave empty for “logged-in only”).
   * Decide whether `and: true` is truly needed.
   * Consider whether the operation must work during OTP/LTS gating → add `@allowUnprotectedOTP` / `@allowUnlicensedLTS` if appropriate.
3. If `@public`:

   * Ensure resolver path does **not** assume `context.user`.
   * Consider protecting sensitive subfields with `@auth` so public clients can still call the root field safely.
4. Avoid using `BYPASS` as a required capability in `for: [...]`:

   * Bypass users already bypass checks, and the matcher explicitly avoids treating `BYPASS` as a normal requested capability.

---

### 7.4 Subscription / SSE additions (checklist)

This checklist is grounded in **`src/graphql/subscriptionWrapper.ts`** (GraphQL subscription helpers), **`src/graphql/sseMiddleware.js`** (SSE endpoints), and the **module SDL files that declare `type Subscription`** in this snapshot:

* Core: `opt/opencti/config/schema/opencti.graphql`
* Modules: `managerConfiguration`, `workspace`, `ai`, `entitySetting`, `notification`

---

## A) Decide which “real-time” mechanism you’re adding

### Use **GraphQL Subscriptions** when:

* You’re emitting **application events** (user notifications, per-entity updates, settings/UX messages, AI bus events).
* You want a GraphQL-native contract (`type Subscription { ... }`) with per-field auth and typed payloads.

### Use **SSE Streams** when:

* You’re emitting **streaming STIX-ish data** (create/update/delete feeds, filtered live collections, resume/backfill).
* You need cursoring (`from` / `last-event-id`) and optional recovery/backfill (`recover`), plus dependency publishing.

---

## B) GraphQL Subscriptions checklist

### B.1 SDL and resolver wiring

* [ ] Add the subscription field in the appropriate SDL (`opencti.graphql` or a module `*.graphql` that’s actually wired into the executable schema).
* [ ] Implement the resolver under `Subscription: { <field>: { subscribe, resolve? } }`.

### B.2 Use the shared helper patterns (preferred)

`src/graphql/subscriptionWrapper.ts` defines the “house style” for subscription setup and filtering:

* **User-scoped events**: `subscribeToUserEvents(context, topics)`

  * Filter accepts when `payload.instance.user_id` **or** `payload.instance.id` matches `context.user.id`.
* **AI bus events**: `subscribeToAiEvents(context, id, topics)`

  * Filter accepts when `payload.user.id === context.user.id` **and** `payload.instance.bus_id === id`.
* **Instance-scoped events**: `subscribeToInstanceEvents(parent, context, id, topics, opts)`

  * Performs an **upfront access check** via `internalLoadById(..., { baseData: true, type })` and throws:

    * `ForbiddenAccess('You are not allowed to listen this.')` if not visible.
  * Default behavior suppresses “echo” events unless `notifySelf: true`.
  * Supports:

    * `preFn()` (runs immediately before subscribing)
    * `cleanFn()` (runs on unsubscribe via `withCancel()` iterator wrapper)
* **Platform settings “messages” events**: `subscribeToPlatformSettingsEvents(context)`

  * Subscribes to `BUS_TOPICS[ENTITY_TYPE_SETTINGS].EDIT_TOPIC`.
  * Emits only when *recipient-filtered* message sets change in meaningful ways (activated removal, activation toggle, activated message text change, new activated message).

Checklist:

* [ ] Pick the helper that matches the subscription semantic (user / AI / instance / settings).
* [ ] Prefer `subscribeToInstanceEvents(...)` for entity-specific topics so you get the built-in visibility pre-check.
* [ ] If you allocate per-subscription resources (timers, in-memory registries), pass a `cleanFn` and rely on the built-in unsubscribe hook.

### B.3 Payload conventions you must satisfy (or filtering will drop everything)

Every filter in `subscriptionWrapper.ts` assumes a payload shaped like:

* `payload.instance` (the target)
* `payload.user` (the actor)

And commonly:

* `payload.instance.id`
* `payload.user.id`
* plus `payload.instance.user_id` (user-scoped) or `payload.instance.bus_id` (AI).

Checklist:

* [ ] Ensure publishers include `{ user: { id }, instance: { id, ... } }` at minimum.
* [ ] If you’re adding a new topic family, verify that the published payload matches these expectations *before* documenting the contract.

### B.4 Disconnection behavior (“empty payload”)

All subscription filters explicitly return `false` when `payload` is falsy, with an inline comment: *“When disconnected, an empty payload is dispatched.”*

Checklist:

* [ ] Don’t treat `null` payloads as data; they’re part of disconnect semantics.
* [ ] If you implement custom filters, include the same `if (!payload) return false` guard.

---

## C) SSE additions checklist (`src/graphql/sseMiddleware.js`)

### C.1 Pick the correct endpoint model (don’t invent a new one lightly)

This snapshot exposes three SSE routes:

1. **`GET {basePath}/stream`** — generic stream (privileged)

   * Auth required (`authenticate`)
   * Requires `BYPASS` capability (enforced inside handler)
   * Cursor: `from` / `headers.from` / `headers['last-event-id']`

2. **`GET {basePath}/stream/:id`** — live stream collections (public or restricted)

   * Middleware: `authenticateForPublic`
   * Supports public streams (SYSTEM_USER allowed) and restricted streams (requires auth + `TAXIIAPI`, plus optional membership and marking checks)
   * Cursor: `from` / `last-event-id`
   * Recovery/backfill: `recover` / `recover-date` (only valid if a start date exists)

3. **`POST {basePath}/stream/connection/:id`** — legacy connection check

   * Confirms the caller owns the connection id; otherwise 401

Checklist:

* [ ] If your change is “streaming data,” extend the **live stream collection** path (`/stream/:id`) unless you have a strong reason not to.
* [ ] Do not weaken generic stream: it is explicitly BYPASS-only.

### C.2 AuthZ rules you must preserve

Live collection authorization is centralized in `computeUserAndCollection(...)`:

* Special id `"live"`: **BYPASS-only**
* Nonexistent collection: 401
* `collection.stream_live === false`: 410 (“stopped”)
* `collection.stream_public === true`: allowed without auth
* Restricted streams:

  * must have an authenticated user and `TAXIIAPI`
  * optional membership check against `collection.restricted_members` (BYPASS bypasses membership)
  * marking filter pre-check: if filters require markings the user lacks → **401** (fail fast rather than streaming nothing)

Checklist:

* [ ] Keep the “stop” semantics (410) and “fail fast” marking behavior (401) intact.
* [ ] If you add new filter types that imply access constraints, implement the same “block instead of silent empty stream” behavior where appropriate.

### C.3 Cursoring and recovery (must remain compatible)

Cursor parsing is intentionally permissive:

* `from` / `last-event-id` accepts:

  * `'0'` or `'0-0'` → normalized to `'0-0'`
  * an event id string with exactly one `-` (treated as already valid)
  * a parseable date string → converted to `"<timestamp>-0"`
  * otherwise undefined (“start now” behavior depends on processor)

Recovery constraints:

* If `recover` is set but `from` is missing/unparseable → throws `UnsupportedError('Recovery mode is only possible with a start date.')`

Checklist:

* [ ] If you introduce any new cursor formats, update both parsing and docs together.
* [ ] Don’t allow recovery without an explicit start date.

### C.4 Event framing and “connected/heartbeat” semantics

SSE messages are written with:

* headers:

  * `Content-Type: text/event-stream; charset=utf-8`
  * `Connection: keep-alive`
  * `Cache-Control: no-cache, no-transform`
* message format:

  * optional `id: ...`
  * optional `event: ...`
  * optional `data: <JSON>`
  * blank line terminator

Heartbeats:

* Sent every `HEARTBEAT_PERIOD` (default 5000ms via config)
* Only sent when:

  * `lastEventId` exists **and**
  * `res.writableLength === 0` (socket buffer empty)
* Heartbeat `data` is an ISO timestamp derived from the `lastEventId`’s timestamp prefix.

Checklist:

* [ ] Preserve heartbeat behavior (it’s explicitly for connection liveness).
* [ ] If you change event ids, ensure the heartbeat logic still produces valid ISO timestamps.

### C.5 Data redaction behavior (don’t regress this)

For “data topics” (anything except heartbeat) where a user exists and the user **does not** have `KNOWLEDGE_ORGANIZATION_RESTRICT`, the middleware removes:

* `event.data.extensions[STIX_EXT_OCTI].granted_refs`

Checklist:

* [ ] Any new event payload shapes must keep this entitlement behavior correct.
* [ ] If you add additional sensitive fields, apply redaction in the same build-message stage (not downstream).

### C.6 Dependency publishing toggles

Live collections support toggles via **query param OR header**:

* `no-dependencies=true` or `no-relationships=true` → disables dependency publishing
* `listen-delete=false` → suppresses delete events
* `with-inferences=true` → includes inferred indices in stream query

Checklist:

* [ ] If you add new dependency kinds, ensure they obey the existing “disable dependencies” toggles.
* [ ] Keep delete suppression as an explicit opt-out (`listen-delete=false`), not opt-in.

### C.7 Shutdown and resource lifecycle

The SSE channel registers close handlers on both `req` and `res` and shuts down:

* removes client from `broadcastClients`
* closes the channel
* shuts down the stream processor

Checklist:

* [ ] Any new per-connection state must be cleaned up in the same close path.
* [ ] Avoid leaks: treat client registry + processor lifecycle as mandatory cleanup responsibilities.

---

## D) Documentation updates required when you add real-time surface

### For a GraphQL subscription:

* [ ] Update the generated API reference (subscription field appears under `type Subscription`).
* [ ] Document:

  * topic naming and payload shape expectations (`user`, `instance`, ids)
  * whether it’s user-scoped / instance-scoped / AI-scoped
  * whether it can echo self-events (`notifySelf` semantics if applicable)

### For SSE:

* [ ] Document:

  * endpoint and auth mode (BYPASS-only vs public vs restricted)
  * cursoring (`from` / `last-event-id`) and recovery (`recover`) rules
  * supported toggles (`no-dependencies`, `listen-delete`, `with-inferences`)
  * redaction behavior (`granted_refs` removal rules)
  * heartbeat behavior and expected client reconnection strategy

---

# Appendix A) Module file inventory (review these per-module for accuracy)

> This is the concrete list of module files present in the zip. When documenting any specific module, use the module’s file set below as the “must review” list.

* **modules-root** (2 files)

  * `src/modules/README.md`
  * `src/modules/index.ts`

* **administrativeArea** (7 files)

  * `src/modules/administrativeArea/administrativeArea-converter.ts`
  * `src/modules/administrativeArea/administrativeArea-domain.ts`
  * `src/modules/administrativeArea/administrativeArea-graphql.ts`
  * `src/modules/administrativeArea/administrativeArea-resolver.ts`
  * `src/modules/administrativeArea/administrativeArea-types.ts`
  * `src/modules/administrativeArea/administrativeArea.graphql`
  * `src/modules/administrativeArea/administrativeArea.ts`

* **ai** (9 files)

  * `src/modules/ai/ai-domain.ts`
  * `src/modules/ai/ai-graphql.ts`
  * `src/modules/ai/ai-nlq-few-shot-examples.ts`
  * `src/modules/ai/ai-nlq-schema.ts`
  * `src/modules/ai/ai-nlq-utils.ts`
  * `src/modules/ai/ai-nlq-values.ts`
  * `src/modules/ai/ai-resolver.ts`
  * `src/modules/ai/ai-types.ts`
  * `src/modules/ai/ai.graphql`

* **attackPattern** (1 file)

  * `src/modules/attackPattern/attackPattern.ts`

* **attributes** (13 files)

  * `src/modules/attributes/attributes-constants.ts`
  * `src/modules/attributes/attributes-converter.ts`
  * `src/modules/attributes/attributes-domain.ts`
  * `src/modules/attributes/attributes-graphql.ts`
  * `src/modules/attributes/attributes-resolvers.ts`
  * `src/modules/attributes/attributes-types.ts`
  * `src/modules/attributes/attributes.graphql`
  * `src/modules/attributes/attributes.ts`
  * `src/modules/attributes/basicObject-registration.ts`
  * `src/modules/attributes/entities-attributes-registration.ts`
  * `src/modules/attributes/relationship-attributes-registration.ts`
  * `src/modules/attributes/stix-attributes-registration.ts`
  * `src/modules/attributes/utils.ts`

* **auth** (5 files)

  * `src/modules/auth/auth-domain.ts`
  * `src/modules/auth/auth-graphql.ts`
  * `src/modules/auth/auth-resolver.ts`
  * `src/modules/auth/auth-types.ts`
  * `src/modules/auth/auth.graphql`

* **campaign** (1 file)

  * `src/modules/campaign/campaign.ts`

* **case** (42 files)

  * `src/modules/case/case-constants.ts`
  * `src/modules/case/case-converter.ts`
  * `src/modules/case/case-domain.ts`
  * `src/modules/case/case-graphql.ts`
  * `src/modules/case/case-resolvers.ts`
  * `src/modules/case/case-types.ts`
  * `src/modules/case/case.graphql`
  * `src/modules/case/case.ts`
  * `src/modules/case/case-incident/case-incident-constants.ts`
  * `src/modules/case/case-incident/case-incident-converter.ts`
  * `src/modules/case/case-incident/case-incident-domain.ts`
  * `src/modules/case/case-incident/case-incident-graphql.ts`
  * `src/modules/case/case-incident/case-incident-resolvers.ts`
  * `src/modules/case/case-incident/case-incident-types.ts`
  * `src/modules/case/case-incident/case-incident.graphql`
  * `src/modules/case/case-incident/case-incident.ts`
  * `src/modules/case/case-rfi/case-rfi-constants.ts`
  * `src/modules/case/case-rfi/case-rfi-converter.ts`
  * `src/modules/case/case-rfi/case-rfi-domain.ts`
  * `src/modules/case/case-rfi/case-rfi-graphql.ts`
  * `src/modules/case/case-rfi/case-rfi-resolvers.ts`
  * `src/modules/case/case-rfi/case-rfi-types.ts`
  * `src/modules/case/case-rfi/case-rfi.graphql`
  * `src/modules/case/case-rfi/case-rfi.ts`
  * `src/modules/case/case-rft/case-rft-constants.ts`
  * `src/modules/case/case-rft/case-rft-converter.ts`
  * `src/modules/case/case-rft/case-rft-domain.ts`
  * `src/modules/case/case-rft/case-rft-graphql.ts`
  * `src/modules/case/case-rft/case-rft-resolvers.ts`
  * `src/modules/case/case-rft/case-rft-types.ts`
  * `src/modules/case/case-rft/case-rft.graphql`
  * `src/modules/case/case-rft/case-rft.ts`
  * `src/modules/case/case-template/case-template-constants.ts`
  * `src/modules/case/case-template/case-template-converter.ts`
  * `src/modules/case/case-template/case-template-domain.ts`
  * `src/modules/case/case-template/case-template-graphql.ts`
  * `src/modules/case/case-template/case-template-resolvers.ts`
  * `src/modules/case/case-template/case-template-types.ts`
  * `src/modules/case/case-template/case-template.graphql`
  * `src/modules/case/case-template/case-template.ts`
  * `src/modules/case/feedback/feedback-converter.ts`
  * `src/modules/case/feedback/feedback-domain.ts`
  * `src/modules/case/feedback/feedback-graphql.ts`
  * `src/modules/case/feedback/feedback-resolvers.ts`
  * `src/modules/case/feedback/feedback-types.ts`
  * `src/modules/case/feedback/feedback.graphql`
  * `src/modules/case/feedback/feedback.ts`

* **catalog** (5 files)

  * `src/modules/catalog/catalog-domain.ts`
  * `src/modules/catalog/catalog-graphql.ts`
  * `src/modules/catalog/catalog-resolver.ts`
  * `src/modules/catalog/catalog-types.ts`
  * `src/modules/catalog/catalog.graphql`

* **channel** (8 files)

  * `src/modules/channel/channel-constants.ts`
  * `src/modules/channel/channel-converter.ts`
  * `src/modules/channel/channel-domain.ts`
  * `src/modules/channel/channel-graphql.ts`
  * `src/modules/channel/channel-resolver.ts`
  * `src/modules/channel/channel-types.ts`
  * `src/modules/channel/channel.graphql`
  * `src/modules/channel/channel.ts`

* **courseOfAction** (1 file)

  * `src/modules/courseOfAction/courseOfAction.ts`

* **dataComponent** (6 files)

  * `src/modules/dataComponent/dataComponent-domain.ts`
  * `src/modules/dataComponent/dataComponent-graphql.ts`
  * `src/modules/dataComponent/dataComponent-resolver.ts`
  * `src/modules/dataComponent/dataComponent-types.ts`
  * `src/modules/dataComponent/dataComponent.graphql`
  * `src/modules/dataComponent/dataComponent.ts`

* **dataSource** (6 files)

  * `src/modules/dataSource/dataSource-domain.ts`
  * `src/modules/dataSource/dataSource-graphql.ts`
  * `src/modules/dataSource/dataSource-resolver.ts`
  * `src/modules/dataSource/dataSource-types.ts`
  * `src/modules/dataSource/dataSource.graphql`
  * `src/modules/dataSource/dataSource.ts`

* **decayRule** (14 files)

  * `src/modules/decayRule/decayRule-constants.ts`
  * `src/modules/decayRule/decayRule-converter.ts`
  * `src/modules/decayRule/decayRule-domain.ts`
  * `src/modules/decayRule/decayRule-graphql.ts`
  * `src/modules/decayRule/decayRule-resolvers.ts`
  * `src/modules/decayRule/decayRule-types.ts`
  * `src/modules/decayRule/decayRule-utils.ts`
  * `src/modules/decayRule/decayRule.graphql`
  * `src/modules/decayRule/decayRule.ts`
  * `src/modules/decayRule/decayRuleValidation.ts`
  * `src/modules/decayRule/scope/decayRuleScope-converter.ts`
  * `src/modules/decayRule/scope/decayRuleScope-domain.ts`
  * `src/modules/decayRule/scope/decayRuleScope-types.ts`
  * `src/modules/decayRule/scope/decayRuleScope.ts`

* **deleteOperation** (5 files)

  * `src/modules/deleteOperation/deleteOperation-domain.ts`
  * `src/modules/deleteOperation/deleteOperation-graphql.ts`
  * `src/modules/deleteOperation/deleteOperation-resolver.ts`
  * `src/modules/deleteOperation/deleteOperation-types.ts`
  * `src/modules/deleteOperation/deleteOperation.graphql`

* **disseminationList** (8 files)

  * `src/modules/disseminationList/disseminationList-converter.ts`
  * `src/modules/disseminationList/disseminationList-domain.ts`
  * `src/modules/disseminationList/disseminationList-graphql.ts`
  * `src/modules/disseminationList/disseminationList-resolver.ts`
  * `src/modules/disseminationList/disseminationList-types.ts`
  * `src/modules/disseminationList/disseminationList.graphql`
  * `src/modules/disseminationList/disseminationList.ts`
  * `src/modules/disseminationList/disseminationListValidation.ts`

* **draftWorkspace** (9 files)

  * `src/modules/draftWorkspace/draftWorkspace-converter.ts`
  * `src/modules/draftWorkspace/draftWorkspace-domain.ts`
  * `src/modules/draftWorkspace/draftWorkspace-graphql.ts`
  * `src/modules/draftWorkspace/draftWorkspace-resolver.ts`
  * `src/modules/draftWorkspace/draftWorkspace-types.ts`
  * `src/modules/draftWorkspace/draftWorkspace.graphql`
  * `src/modules/draftWorkspace/draftWorkspace.ts`
  * `src/modules/draftWorkspace/draftWorkspaceUtils.ts`
  * `src/modules/draftWorkspace/draftWorkspaceValidation.ts`

* **emailTemplate** (8 files)

  * `src/modules/emailTemplate/emailTemplate-converter.ts`
  * `src/modules/emailTemplate/emailTemplate-domain.ts`
  * `src/modules/emailTemplate/emailTemplate-graphql.ts`
  * `src/modules/emailTemplate/emailTemplate-resolver.ts`
  * `src/modules/emailTemplate/emailTemplate-types.ts`
  * `src/modules/emailTemplate/emailTemplate.graphql`
  * `src/modules/emailTemplate/emailTemplate.ts`
  * `src/modules/emailTemplate/emailTemplateValidation.ts`

* **entitySetting** (8 files)

  * `src/modules/entitySetting/entitySetting-converter.ts`
  * `src/modules/entitySetting/entitySetting-domain.ts`
  * `src/modules/entitySetting/entitySetting-graphql.ts`
  * `src/modules/entitySetting/entitySetting-resolver.ts`
  * `src/modules/entitySetting/entitySetting-types.ts`
  * `src/modules/entitySetting/entitySetting.graphql`
  * `src/modules/entitySetting/entitySetting.ts`
  * `src/modules/entitySetting/entitySettingValidation.ts`

* **event** (8 files)

  * `src/modules/event/event-converter.ts`
  * `src/modules/event/event-domain.ts`
  * `src/modules/event/event-graphql.ts`
  * `src/modules/event/event-resolver.ts`
  * `src/modules/event/event-types.ts`
  * `src/modules/event/event.graphql`
  * `src/modules/event/event.ts`
  * `src/modules/event/eventValidation.ts`

* **exclusionList** (8 files)

  * `src/modules/exclusionList/exclusionList-converter.ts`
  * `src/modules/exclusionList/exclusionList-domain.ts`
  * `src/modules/exclusionList/exclusionList-graphql.ts`
  * `src/modules/exclusionList/exclusionList-resolver.ts`
  * `src/modules/exclusionList/exclusionList-types.ts`
  * `src/modules/exclusionList/exclusionList.graphql`
  * `src/modules/exclusionList/exclusionList.ts`
  * `src/modules/exclusionList/exclusionListValidation.ts`

* **externalReference** (1 file)

  * `src/modules/externalReference/externalReference.ts`

* **fintelDesign** (8 files)

  * `src/modules/fintelDesign/fintelDesign-converter.ts`
  * `src/modules/fintelDesign/fintelDesign-domain.ts`
  * `src/modules/fintelDesign/fintelDesign-graphql.ts`
  * `src/modules/fintelDesign/fintelDesign-resolver.ts`
  * `src/modules/fintelDesign/fintelDesign-types.ts`
  * `src/modules/fintelDesign/fintelDesign.graphql`
  * `src/modules/fintelDesign/fintelDesign.ts`
  * `src/modules/fintelDesign/fintelDesignValidation.ts`

* **fintelTemplate** (8 files)

  * `src/modules/fintelTemplate/fintelTemplate-converter.ts`
  * `src/modules/fintelTemplate/fintelTemplate-domain.ts`
  * `src/modules/fintelTemplate/fintelTemplate-graphql.ts`
  * `src/modules/fintelTemplate/fintelTemplate-resolver.ts`
  * `src/modules/fintelTemplate/fintelTemplate-types.ts`
  * `src/modules/fintelTemplate/fintelTemplate.graphql`
  * `src/modules/fintelTemplate/fintelTemplate.ts`
  * `src/modules/fintelTemplate/fintelTemplateValidation.ts`

* **form** (8 files)

  * `src/modules/form/form-converter.ts`
  * `src/modules/form/form-domain.ts`
  * `src/modules/form/form-graphql.ts`
  * `src/modules/form/form-resolver.ts`
  * `src/modules/form/form-types.ts`
  * `src/modules/form/form.graphql`
  * `src/modules/form/form.ts`
  * `src/modules/form/formValidation.ts`

* **grouping** (8 files)

  * `src/modules/grouping/grouping-converter.ts`
  * `src/modules/grouping/grouping-domain.ts`
  * `src/modules/grouping/grouping-graphql.ts`
  * `src/modules/grouping/grouping-resolver.ts`
  * `src/modules/grouping/grouping-types.ts`
  * `src/modules/grouping/grouping.graphql`
  * `src/modules/grouping/grouping.ts`
  * `src/modules/grouping/groupingValidation.ts`

* **incident** (1 file)

  * `src/modules/incident/incident.ts`

* **indicator** (8 files)

  * `src/modules/indicator/indicator-converter.ts`
  * `src/modules/indicator/indicator-domain.ts`
  * `src/modules/indicator/indicator-graphql.ts`
  * `src/modules/indicator/indicator-resolver.ts`
  * `src/modules/indicator/indicator-types.ts`
  * `src/modules/indicator/indicator.graphql`
  * `src/modules/indicator/indicator.ts`
  * `src/modules/indicator/indicatorValidation.ts`

* **ingestion** (34 files)

  * `src/modules/ingestion/ingestion-csv-domain.ts`
  * `src/modules/ingestion/ingestion-csv-graphql.ts`
  * `src/modules/ingestion/ingestion-csv-resolver.ts`
  * `src/modules/ingestion/ingestion-csv-types.ts`
  * `src/modules/ingestion/ingestion-csv.graphql`
  * `src/modules/ingestion/ingestion-json-domain.ts`
  * `src/modules/ingestion/ingestion-json-graphql.ts`
  * `src/modules/ingestion/ingestion-json-resolver.ts`
  * `src/modules/ingestion/ingestion-json-types.ts`
  * `src/modules/ingestion/ingestion-json.graphql`
  * `src/modules/ingestion/ingestion-rss-domain.ts`
  * `src/modules/ingestion/ingestion-rss-graphql.ts`
  * `src/modules/ingestion/ingestion-rss-resolver.ts`
  * `src/modules/ingestion/ingestion-rss-types.ts`
  * `src/modules/ingestion/ingestion-rss.graphql`
  * `src/modules/ingestion/ingestion-taxii-collection-domain.ts`
  * `src/modules/ingestion/ingestion-taxii-collection-graphql.ts`
  * `src/modules/ingestion/ingestion-taxii-collection-resolver.ts`
  * `src/modules/ingestion/ingestion-taxii-collection-types.ts`
  * `src/modules/ingestion/ingestion-taxii-collection.graphql`
  * `src/modules/ingestion/ingestion-taxii-domain.ts`
  * `src/modules/ingestion/ingestion-taxii-graphql.ts`
  * `src/modules/ingestion/ingestion-taxii-resolver.ts`
  * `src/modules/ingestion/ingestion-taxii-types.ts`
  * `src/modules/ingestion/ingestion-taxii.graphql`
  * `src/modules/ingestion/ingestion-utils.ts`
  * `src/modules/ingestion/ingestionValidation.ts`
  * `src/modules/ingestion/ingestion.ts`
  * `src/modules/ingestion/uriClean.ts`
  * `src/modules/ingestion/uriCleanValidation.ts`
  * `src/modules/ingestion/uri.ts`
  * `src/modules/ingestion/uriValidation.ts`
  * `src/modules/ingestion/utils.ts`
  * `src/modules/ingestion/validator.ts`

* **internal** (24 files)

  * `src/modules/internal/document/document.ts`
  * `src/modules/internal/csvMapper/csvMapper-domain.ts`
  * `src/modules/internal/csvMapper/csvMapper-graphql.ts`
  * `src/modules/internal/csvMapper/csvMapper-resolver.ts`
  * `src/modules/internal/csvMapper/csvMapper-types.ts`
  * `src/modules/internal/csvMapper/csvMapper.graphql`
  * `src/modules/internal/csvMapper/csvMapper.ts`
  * `src/modules/internal/csvMapper/csvMapperValidation.ts`
  * `src/modules/internal/csvMapper/deprecated/csvMapper-graphql.js`
  * `src/modules/internal/csvMapper/deprecated/csvMapper-resolver.js`
  * `src/modules/internal/csvMapper/deprecated/csvMapper.graphql`
  * `src/modules/internal/jsonMapper/jsonMapper-domain.ts`
  * `src/modules/internal/jsonMapper/jsonMapper-graphql.ts`
  * `src/modules/internal/jsonMapper/jsonMapper-resolver.ts`
  * `src/modules/internal/jsonMapper/jsonMapper-types.ts`
  * `src/modules/internal/jsonMapper/jsonMapper.graphql`
  * `src/modules/internal/jsonMapper/jsonMapper.ts`
  * `src/modules/internal/jsonMapper/jsonMapperValidation.ts`
  * `src/modules/internal/internal-constants.ts`
  * `src/modules/internal/internal-converter.ts`
  * `src/modules/internal/internal-domain.ts`
  * `src/modules/internal/internal-resolvers.ts`
  * `src/modules/internal/internal-types.ts`
  * `src/modules/internal/internal.ts`

* **intrusionSet** (1 file)

  * `src/modules/intrusionSet/intrusionSet.ts`

* **language** (8 files)

  * `src/modules/language/language-converter.ts`
  * `src/modules/language/language-domain.ts`
  * `src/modules/language/language-graphql.ts`
  * `src/modules/language/language-resolver.ts`
  * `src/modules/language/language-types.ts`
  * `src/modules/language/language.graphql`
  * `src/modules/language/language.ts`
  * `src/modules/language/languageValidation.ts`

* **malware** (1 file)

  * `src/modules/malware/malware.ts`

* **malwareAnalysis** (8 files)

  * `src/modules/malwareAnalysis/malwareAnalysis-converter.ts`
  * `src/modules/malwareAnalysis/malwareAnalysis-domain.ts`
  * `src/modules/malwareAnalysis/malwareAnalysis-graphql.ts`
  * `src/modules/malwareAnalysis/malwareAnalysis-resolver.ts`
  * `src/modules/malwareAnalysis/malwareAnalysis-types.ts`
  * `src/modules/malwareAnalysis/malwareAnalysis.graphql`
  * `src/modules/malwareAnalysis/malwareAnalysis.ts`
  * `src/modules/malwareAnalysis/malwareAnalysisValidation.ts`

* **managerConfiguration** (8 files)

  * `src/modules/managerConfiguration/managerConfiguration-converter.ts`
  * `src/modules/managerConfiguration/managerConfiguration-domain.ts`
  * `src/modules/managerConfiguration/managerConfiguration-graphql.ts`
  * `src/modules/managerConfiguration/managerConfiguration-resolver.ts`
  * `src/modules/managerConfiguration/managerConfiguration-types.ts`
  * `src/modules/managerConfiguration/managerConfiguration.graphql`
  * `src/modules/managerConfiguration/managerConfiguration.ts`
  * `src/modules/managerConfiguration/managerConfigurationValidation.ts`

* **metrics** (5 files)

  * `src/modules/metrics/metrics-domain.ts`
  * `src/modules/metrics/metrics-graphql.ts`
  * `src/modules/metrics/metrics-resolver.ts`
  * `src/modules/metrics/metrics-types.ts`
  * `src/modules/metrics/metrics.graphql`

* **narrative** (8 files)

  * `src/modules/narrative/narrative-converter.ts`
  * `src/modules/narrative/narrative-domain.ts`
  * `src/modules/narrative/narrative-graphql.ts`
  * `src/modules/narrative/narrative-resolver.ts`
  * `src/modules/narrative/narrative-types.ts`
  * `src/modules/narrative/narrative.graphql`
  * `src/modules/narrative/narrative.ts`
  * `src/modules/narrative/narrativeValidation.ts`

* **note** (1 file)

  * `src/modules/note/note.ts`

* **notification** (9 files)

  * `src/modules/notification/notification-converter.ts`
  * `src/modules/notification/notification-domain.ts`
  * `src/modules/notification/notification-graphql.ts`
  * `src/modules/notification/notification-resolver.ts`
  * `src/modules/notification/notification-types.ts`
  * `src/modules/notification/notification-utils.ts`
  * `src/modules/notification/notification.graphql`
  * `src/modules/notification/notification.ts`
  * `src/modules/notification/notificationValidation.ts`

* **notifier** (8 files)

  * `src/modules/notifier/notifier-converter.ts`
  * `src/modules/notifier/notifier-domain.ts`
  * `src/modules/notifier/notifier-graphql.ts`
  * `src/modules/notifier/notifier-resolver.ts`
  * `src/modules/notifier/notifier-types.ts`
  * `src/modules/notifier/notifier.graphql`
  * `src/modules/notifier/notifier.ts`
  * `src/modules/notifier/notifierValidation.ts`

* **observedData** (1 file)

  * `src/modules/observedData/observedData.ts`

* **organization** (8 files)

  * `src/modules/organization/organization-converter.ts`
  * `src/modules/organization/organization-domain.ts`
  * `src/modules/organization/organization-graphql.ts`
  * `src/modules/organization/organization-resolver.ts`
  * `src/modules/organization/organization-types.ts`
  * `src/modules/organization/organization.graphql`
  * `src/modules/organization/organization.ts`
  * `src/modules/organization/organizationValidation.ts`

* **pir** (8 files)

  * `src/modules/pir/pir-converter.ts`
  * `src/modules/pir/pir-domain.ts`
  * `src/modules/pir/pir-graphql.ts`
  * `src/modules/pir/pir-resolver.ts`
  * `src/modules/pir/pir-types.ts`
  * `src/modules/pir/pir.graphql`
  * `src/modules/pir/pir.ts`
  * `src/modules/pir/pirValidation.ts`

* **playbook** (9 files)

  * `src/modules/playbook/playbook-converter.ts`
  * `src/modules/playbook/playbook-domain.ts`
  * `src/modules/playbook/playbook-graphql.ts`
  * `src/modules/playbook/playbook-resolver.ts`
  * `src/modules/playbook/playbook-types.ts`
  * `src/modules/playbook/playbook-utils.ts`
  * `src/modules/playbook/playbook.graphql`
  * `src/modules/playbook/playbook.ts`
  * `src/modules/playbook/playbookValidation.ts`

* **publicDashboard** (8 files)

  * `src/modules/publicDashboard/publicDashboard-domain.ts`
  * `src/modules/publicDashboard/publicDashboard-graphql.ts`
  * `src/modules/publicDashboard/publicDashboard-resolver.ts`
  * `src/modules/publicDashboard/publicDashboard-types.ts`
  * `src/modules/publicDashboard/publicDashboard.graphql`
  * `src/modules/publicDashboard/publicDashboard.ts`
  * `src/modules/publicDashboard/publicDashboardValidation.ts`
  * `src/modules/publicDashboard/utils.ts`

* **relationsRef** (7 files)

  * `src/modules/relationsRef/entities-relationsRef-registration.ts`
  * `src/modules/relationsRef/internalObject-registrationRef.ts`
  * `src/modules/relationsRef/relationsRef-constants.ts`
  * `src/modules/relationsRef/relationsRef-converter.ts`
  * `src/modules/relationsRef/relationsRef-domain.ts`
  * `src/modules/relationsRef/relationsRef-types.ts`
  * `src/modules/relationsRef/utils.ts`

* **report** (1 file)

  * `src/modules/report/report.ts`

* **requestAccess** (5 files)

  * `src/modules/requestAccess/requestAccess-domain.ts`
  * `src/modules/requestAccess/requestAccess-graphql.ts`
  * `src/modules/requestAccess/requestAccess-resolver.ts`
  * `src/modules/requestAccess/requestAccess-types.ts`
  * `src/modules/requestAccess/requestAccess.graphql`

* **savedFilter** (8 files)

  * `src/modules/savedFilter/savedFilter-converter.ts`
  * `src/modules/savedFilter/savedFilter-domain.ts`
  * `src/modules/savedFilter/savedFilter-graphql.ts`
  * `src/modules/savedFilter/savedFilter-resolver.ts`
  * `src/modules/savedFilter/savedFilter-types.ts`
  * `src/modules/savedFilter/savedFilter.graphql`
  * `src/modules/savedFilter/savedFilter.ts`
  * `src/modules/savedFilter/savedFilterValidation.ts`

* **securityCoverage** (8 files)

  * `src/modules/securityCoverage/securityCoverage-converter.ts`
  * `src/modules/securityCoverage/securityCoverage-domain.ts`
  * `src/modules/securityCoverage/securityCoverage-graphql.ts`
  * `src/modules/securityCoverage/securityCoverage-resolver.ts`
  * `src/modules/securityCoverage/securityCoverage-types.ts`
  * `src/modules/securityCoverage/securityCoverage.graphql`
  * `src/modules/securityCoverage/securityCoverage.ts`
  * `src/modules/securityCoverage/securityCoverageValidation.ts`

* **securityPlatform** (8 files)

  * `src/modules/securityPlatform/securityPlatform-converter.ts`
  * `src/modules/securityPlatform/securityPlatform-domain.ts`
  * `src/modules/securityPlatform/securityPlatform-graphql.ts`
  * `src/modules/securityPlatform/securityPlatform-resolver.ts`
  * `src/modules/securityPlatform/securityPlatform-types.ts`
  * `src/modules/securityPlatform/securityPlatform.graphql`
  * `src/modules/securityPlatform/securityPlatform.ts`
  * `src/modules/securityPlatform/securityPlatformValidation.ts`

* **stixCyberObservable** (5 files)

  * `src/modules/stixCyberObservable/deprecated/stixCyberObservable-graphql.js`
  * `src/modules/stixCyberObservable/deprecated/stixCyberObservable-resolver.js`
  * `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql`
  * `src/modules/stixCyberObservable/stixCyberObservable.ts`
  * `src/modules/stixCyberObservable/stixCyberObservableValidation.ts`

* **support** (8 files)

  * `src/modules/support/support-converter.ts`
  * `src/modules/support/support-domain.ts`
  * `src/modules/support/support-graphql.ts`
  * `src/modules/support/support-resolver.ts`
  * `src/modules/support/support-types.ts`
  * `src/modules/support/support.graphql`
  * `src/modules/support/support.ts`
  * `src/modules/support/supportValidation.ts`

* **task** (16 files)

  * `src/modules/task/task-converter.ts`
  * `src/modules/task/task-domain.ts`
  * `src/modules/task/task-graphql.ts`
  * `src/modules/task/task-resolvers.ts`
  * `src/modules/task/task-types.ts`
  * `src/modules/task/task-utils.ts`
  * `src/modules/task/task.graphql`
  * `src/modules/task/task.ts`
  * `src/modules/task/taskValidation.ts`
  * `src/modules/task/task-template/task-template-converter.ts`
  * `src/modules/task/task-template/task-template-domain.ts`
  * `src/modules/task/task-template/task-template-graphql.ts`
  * `src/modules/task/task-template/task-template-resolvers.ts`
  * `src/modules/task/task-template/task-template-types.ts`
  * `src/modules/task/task-template/task-template.graphql`
  * `src/modules/task/task-template/task-template.ts`

* **theme** (9 files)

  * `src/modules/theme/theme-constants.ts`
  * `src/modules/theme/theme-converter.ts`
  * `src/modules/theme/theme-domain.ts`
  * `src/modules/theme/theme-graphql.ts`
  * `src/modules/theme/theme-resolvers.ts`
  * `src/modules/theme/theme-types.ts`
  * `src/modules/theme/theme.graphql`
  * `src/modules/theme/theme.ts`
  * `src/modules/theme/themeValidation.ts`

* **threatActorGroup** (1 file)

  * `src/modules/threatActorGroup/threatActorGroup.ts`

* **threatActorIndividual** (8 files)

  * `src/modules/threatActorIndividual/threatActorIndividual-converter.ts`
  * `src/modules/threatActorIndividual/threatActorIndividual-domain.ts`
  * `src/modules/threatActorIndividual/threatActorIndividual-graphql.ts`
  * `src/modules/threatActorIndividual/threatActorIndividual-resolver.ts`
  * `src/modules/threatActorIndividual/threatActorIndividual-types.ts`
  * `src/modules/threatActorIndividual/threatActorIndividual.graphql`
  * `src/modules/threatActorIndividual/threatActorIndividual.ts`
  * `src/modules/threatActorIndividual/threatActorIndividualValidation.ts`

* **tool** (1 file)

  * `src/modules/tool/tool.ts`

* **vocabulary** (9 files)

  * `src/modules/vocabulary/vocabulary-constants.ts`
  * `src/modules/vocabulary/vocabulary-converter.ts`
  * `src/modules/vocabulary/vocabulary-domain.ts`
  * `src/modules/vocabulary/vocabulary-graphql.ts`
  * `src/modules/vocabulary/vocabulary-resolver.ts`
  * `src/modules/vocabulary/vocabulary-types.ts`
  * `src/modules/vocabulary/vocabulary.graphql`
  * `src/modules/vocabulary/vocabulary.ts`
  * `src/modules/vocabulary/vocabularyValidation.ts`

* **vulnerability** (1 file)

  * `src/modules/vulnerability/vulnerability.ts`

* **workspace** (9 files)

  * `src/modules/workspace/workspace-converter.ts`
  * `src/modules/workspace/workspace-domain.ts`
  * `src/modules/workspace/workspace-graphql.ts`
  * `src/modules/workspace/workspace-resolver.ts`
  * `src/modules/workspace/workspace-types.ts`
  * `src/modules/workspace/workspace-utils.ts`
  * `src/modules/workspace/workspace.graphql`
  * `src/modules/workspace/workspace.ts`
  * `src/modules/workspace/workspaceValidation.ts`

* **xtm** (17 files)

  * `src/modules/xtm/hub/xtm-hub-domain.ts`
  * `src/modules/xtm/hub/xtm-hub-graphql.ts`
  * `src/modules/xtm/hub/xtm-hub-resolver.ts`
  * `src/modules/xtm/hub/xtm-hub-types.ts`
  * `src/modules/xtm/hub/xtm-hub.graphql`
  * `src/modules/xtm/xtm-converter.ts`
  * `src/modules/xtm/xtm-domain.ts`
  * `src/modules/xtm/xtm-graphql.ts`
  * `src/modules/xtm/xtm-resolver.ts`
  * `src/modules/xtm/xtm-types.ts`
  * `src/modules/xtm/xtm-utils.ts`
  * `src/modules/xtm/xtm.graphql`
  * `src/modules/xtm/xtm.ts`
  * `src/modules/xtm/xtmValidation.ts`
  * `src/modules/xtm/connectors/xtm-connectors.ts`
  * `src/modules/xtm/utils.ts`
  * `src/modules/xtm/validator.ts`

---

# Appendix B) Core GraphQL + schema-layer files (non-module)

I re-checked the zip contents: `opt/opencti/src/graphql/` contains **exactly 9 files**, and `opt/opencti/src/schema/` contains **exactly 26 files**. Your Appendix B list is accurate — the only change I recommend is adding the **snapshot identity + canonical core SDL** (both are non-module and referenced throughout the doc set).

---

### B.0 Snapshot identity + canonical core SDL

* `MANIFEST.txt`
* `opt/opencti/config/schema/opencti.graphql`

---

### B.1 GraphQL runtime implementation files (9)

* `src/graphql/authDirective.ts`
* `src/graphql/graphql.js`
* `src/graphql/httpResponsePlugin.js`
* `src/graphql/loggerPlugin.js`
* `src/graphql/schema.js`
* `src/graphql/sseMiddleware.js`
* `src/graphql/subscriptionWrapper.ts`
* `src/graphql/telemetryPlugin.js`
* `src/graphql/tracingPlugin.js`

---

### B.2 Schema model layer files (26)

* `src/schema/attribute-definition.ts`
* `src/schema/fieldDataAdapter.ts`
* `src/schema/general.js`
* `src/schema/identifier.js`
* `src/schema/internalObject.ts`
* `src/schema/internalRelationship.ts`
* `src/schema/module.ts`
* `src/schema/overviewLayoutCustomization-register.ts`
* `src/schema/schema-attributes.ts`
* `src/schema/schema-labels.js`
* `src/schema/schema-overviewLayoutCustomization.ts`
* `src/schema/schema-relationsRef.ts`
* `src/schema/schema-types.ts`
* `src/schema/schema-validator.ts`
* `src/schema/schemaUtils.js`
* `src/schema/stixCoreObject.ts`
* `src/schema/stixCoreRelationship.ts`
* `src/schema/stixCyberObservable.ts`
* `src/schema/stixDomainObject.ts`
* `src/schema/stixDomainObjectOptions.ts`
* `src/schema/stixEmbeddedRelationship.ts`
* `src/schema/stixMetaObject.ts`
* `src/schema/stixRefRelationship.ts`
* `src/schema/stixRelationship.ts`
* `src/schema/stixSightingRelationship.ts`
* `src/schema/validator-register.ts`

---

## Appendix C) Developer recipes (bootstrap-script based)

These recipes are written to match the **2026-01-27 schema snapshot** you provided (notably: some “id” args are `String!` in queries, while the corresponding `*Edit` mutations often take `ID!`).

### C.0 Conventions used in examples

* **GraphQL endpoint:** `POST ${OPENCTI_HTTP%/}/graphql`
* **Auth header (admin token / API token):** `Authorization: Bearer ${OPENCTI_ADMIN_TOKEN}`
* **Content-Type:** `application/json`
* **REST health check:** `GET ${OPENCTI_HTTP%/}/health?health_access_key=${OPENCTI_HEALTHCHECK_ACCESS_KEY}`
* Relationship type strings shown below align with constants in `src/schema/internalRelationship.ts` (e.g., `has-role`, `member-of`, `accesses-to`, `has-capability`).

Minimal helper:

```bash
opencti_gql () {
  curl -fsS --insecure \
    -H "Authorization: Bearer ${OPENCTI_ADMIN_TOKEN}" \
    -H "Content-Type: application/json" \
    "${OPENCTI_HTTP%/}/graphql" \
    --data "$1"
}
```

---

### C.1 Smoke tests (REST + GraphQL)

**1) Health**

```bash
curl -fsS \
  "${OPENCTI_HTTP%/}/health?health_access_key=${OPENCTI_HEALTHCHECK_ACCESS_KEY}" >/dev/null
echo "ok"
```

**2) About/version**

```bash
opencti_gql '{"query":"query { about { version } }"}' | jq
```

**3) Who am I? (me)**

> In this snapshot, `me` returns `MeUser` (not `User`).

```bash
opencti_gql '{"query":"query { me { id name user_email api_token } }"}' | jq
```

---

### C.2 RBAC bootstrap (groups + roles + allowed markings)

#### C.2.1 Ensure a group exists (“Analysts”)

**List groups**

```bash
opencti_gql '{"query":"query { groups(first: 200) { edges { node { id name } } } }"}' | jq
```

**Create group (if missing)**

> `group_confidence_level` is required in `GroupAddInput` in this snapshot.

```bash
opencti_gql "$(jq -nc '{
  query: "mutation($input: GroupAddInput!) { groupAdd(input: $input) { id name } }",
  variables: { input: {
    name: "Analysts",
    description: "Default analyst users",
    default_assignation: false,
    group_confidence_level: { max_confidence: 80, overrides: [] }
  }}
}')" | jq
```

#### C.2.2 Ensure roles exist (“Default”, “Analysts”)

**List roles**

```bash
opencti_gql '{"query":"query { roles(first: 200) { edges { node { id name } } } }"}' | jq
```

**Create “Analysts” role (if missing)**

> In this snapshot, `RoleAddInput` does **not** include `default_assignation`.

```bash
opencti_gql "$(jq -nc '{
  query: "mutation($input: RoleAddInput!) { roleAdd(input: $input) { id name } }",
  variables: { input: {
    name: "Analysts",
    description: "Standard analyst role (mirror of Default)"
  }}
}')" | jq
```

#### C.2.3 Attach roles to group (Group → Role)

> Use relationship type: **`has-role`**.

**Read group roles**

> Note: `group(id: String!)` (string) is the safe read signature in this snapshot.

```bash
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" '{
  query: "query($id:String!){ group(id:$id){ id name roles{ edges{ node{ id name } } } } }",
  variables: { id: $gid }
}')" | jq
```

**Attach “Default” role (required for baseline UI access)**

```bash
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" --arg rid "$DEFAULT_ROLE_ID" '{
  query: "mutation($id:ID!, $input:InternalRelationshipAddInput!){ groupEdit(id:$id){ relationAdd(input:$input){ id } } }",
  variables: { id: $gid, input: { relationship_type: "has-role", toId: $rid } }
}')" | jq
```

#### C.2.4 Restrict group allowed markings to “All TLP except TLP:RED”

> Use relationship type: **`accesses-to`** for Group.allowed_marking enforcement.

**List marking definitions**

```bash
opencti_gql '{"query":"query { markingDefinitions(first: 200, orderBy: definition_type, orderMode: asc) { edges { node { id definition definition_type } } } }"}' | jq
```

**Read current allowed markings**

```bash
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" '{
  query: "query($id:String!){ group(id:$id){ id name allowed_marking{ id definition definition_type } } }",
  variables: { id: $gid }
}')" | jq
```

**Remove TLP:RED (if present)**

```bash
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" --arg to "$TLP_RED_ID" '{
  query: "mutation($id:ID!, $to:StixRef){ groupEdit(id:$id){ relationDelete(toId:$to, relationship_type:\"accesses-to\"){ id } } }",
  variables: { id: $gid, to: $to }
}')" | jq
```

**Add missing non-RED TLP markings**

```bash
# for each marking id in $TLP_ALLOWED_IDS
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" --arg to "$MID" '{
  query: "mutation($id:ID!, $input:InternalRelationshipAddInput!){ groupEdit(id:$id){ relationAdd(input:$input){ id } } }",
  variables: { id: $gid, input: { relationship_type: "accesses-to", toId: $to } }
}')" | jq
```

---

### C.3 Create users and place them in groups

#### C.3.1 Create a local user and assign groups in one call

> `UserAddInput.groups` is `[ID!]` in this snapshot, so you can attach groups at creation time.

```bash
opencti_gql "$(jq -nc --arg gid "$ANALYSTS_GROUP_ID" '{
  query: "mutation($input: UserAddInput!){ userAdd(input:$input){ id name user_email groups{ edges{ node{ id name } } } } }",
  variables: { input: {
    name: "Alice Analyst",
    user_email: "alice@example.com",
    password: "ChangeMe-DevOnly",
    groups: [$gid]
  }}
}')" | jq
```

#### C.3.2 Add an existing user to a group (User → Group)

> Use relationship type: **`member-of`**.

```bash
opencti_gql "$(jq -nc --arg uid "$USER_ID" --arg gid "$ANALYSTS_GROUP_ID" '{
  query: "mutation($id:ID!, $input:InternalRelationshipAddInput!){ userEdit(id:$id){ relationAdd(input:$input){ id } } }",
  variables: { id: $uid, input: { relationship_type: "member-of", toId: $gid } }
}')" | jq
```

#### C.3.3 Renew a user API token (admin-only capability)

> `tokenRenew` exists under `userEdit` in this snapshot.

```bash
opencti_gql "$(jq -nc --arg uid "$USER_ID" '{
  query: "mutation($id:ID!){ userEdit(id:$id){ tokenRenew { id name api_token } } }",
  variables: { id: $uid }
}')" | jq
```

---

### C.4 Role capability management (useful for “Analysts mirrors Default”)

#### C.4.1 Inspect role capabilities

```bash
opencti_gql "$(jq -nc --arg rid "$DEFAULT_ROLE_ID" '{
  query: "query($id:String!){ role(id:$id){ id name capabilities{ id name } } }",
  variables: { id: $rid }
}')" | jq
```

#### C.4.2 List all capabilities (to find IDs by name)

```bash
opencti_gql '{"query":"query { capabilities(first: 500) { edges { node { id name description } } } }"}' | jq
```

#### C.4.3 Attach a capability to a role (Role → Capability)

> Use relationship type: **`has-capability`**.

```bash
opencti_gql "$(jq -nc --arg rid "$ANALYSTS_ROLE_ID" --arg cid "$CAPABILITY_ID" '{
  query: "mutation($id:ID!, $input:InternalRelationshipAddInput!){ roleEdit(id:$id){ relationAdd(input:$input){ id } } }",
  variables: { id: $rid, input: { relationship_type: "has-capability", toId: $cid } }
}')" | jq
```

---

### C.5 CSV ingestion feeds (create + validate + update)

#### C.5.1 Resolve `user_id` for ownership

```bash
opencti_gql '{"query":"query { me { id name } }"}' | jq
```

#### C.5.2 Check if a feed already exists by name

```bash
opencti_gql "$(jq -nc --arg s "My Feed" '{
  query:"query($search:String!){ ingestionCsvs(search:$search, first:50){ edges{ node{ id name uri } } } }",
  variables:{search:$s}
}')" | jq
```

#### C.5.3 Validate a mapper + config without creating the feed (tester)

> `ingestionCsvTester(input: IngestionCsvAddInput!)` exists in this snapshot, which is handy for dev iteration.

```bash
MAPPER_JSON_STR="$(jq -c . ./path/to/mapper.json)"   # one-line JSON
opencti_gql "$(jq -nc --arg uid "$ME_ID" --arg mapper "$MAPPER_JSON_STR" '{
  query: "query($input:IngestionCsvAddInput!){ ingestionCsvTester(input:$input){ ok error } }",
  variables: { input: {
    name: "DryRun Feed",
    uri: "https://example.com/feed.csv",
    user_id: $uid,
    scheduling_period: "PT1H",
    authentication_type: "none",
    csv_mapper_type: "inline",
    csv_mapper: $mapper,
    ingestion_running: false,
    markings: []
  }}
}')" | jq
```

#### C.5.4 Create a CSV feed (inline mapper)

Key snapshot-specific details:

* `authentication_type` enum values are **lowercase** (e.g., `none`, `bearer`, `basic-user-password`, …).
* `csv_mapper_type` is `inline` or `id`.
* **`csv_mapper` must be a STRING** that contains JSON (not an object).

```bash
MAPPER_JSON_STR="$(jq -c . ./path/to/mapper.json)"

opencti_gql "$(jq -nc \
  --arg uid "$ME_ID" \
  --arg mapper "$MAPPER_JSON_STR" \
  '{
    query: "mutation($input:IngestionCsvAddInput!){ ingestionCsvAdd(input:$input){ id name uri } }",
    variables: { input: {
      name: "URLHaus CSV",
      description: "Dev bootstrap feed",
      uri: "https://example.com/urlhaus.csv",
      user_id: $uid,
      scheduling_period: "PT1H",
      authentication_type: "none",
      authentication_value: null,
      csv_mapper_type: "inline",
      csv_mapper: $mapper,
      ingestion_running: true,
      confidence_level: 70,
      markings: []
    }}
  }'
)" | jq
```

#### C.5.5 Update an existing feed (field patch)

> Use `ingestionCsvFieldPatch(id: ID!, input: [EditInput!]!)`.

```bash
opencti_gql "$(jq -nc --arg id "$CSV_ID" '{
  query: "mutation($id:ID!, $input:[EditInput!]!){ ingestionCsvFieldPatch(id:$id, input:$input){ id name scheduling_period ingestion_running } }",
  variables: { id: $id, input: [
    { key: "scheduling_period", value: ["PT30M"] },
    { key: "ingestion_running", value: ["true"] }
  ]}
}')" | jq
```

#### C.5.6 Delete a feed

```bash
opencti_gql "$(jq -nc --arg id "$CSV_ID" '{
  query: "mutation($id:ID!){ ingestionCsvDelete(id:$id) }",
  variables: { id: $id }
}')" | jq
```

---

### C.6 Handy “debug your auth” queries

**Show my effective roles + groups**

```bash
opencti_gql '{"query":"query { me { id name groups { edges { node { id name } } } roles { edges { node { id name } } } } }"}' | jq
```

**Show my capabilities (fast signal when an operation is denied)**

```bash
opencti_gql '{"query":"query { me { capabilities { name } } }"}' | jq
```

---

# Appendix D) OpenCTI GraphQL Snapshot Reference

- **Snapshot:** `opencti-schema-snapshot-2026-01-27_141500`
- **Generated:** `2026-01-28T15:51:54`

## Snapshot summary
<a id="snapshot-summary"></a>

- **Core SDL:** `config/schema/opencti.graphql`
- **Modules dir:** `src/modules`
- **Modules index:** `src/modules/index.ts`

## Table of contents

- [Snapshot summary](#snapshot-summary)
- [Counts](#counts)
- [Wiring notes](#wiring-notes)
- [Wired module SDL order](#wired-module-sdl-order)
- [Deprecated / unwired SDL files](#deprecated-unwired-sdl-files)
- [Root API index](#root-api-index)
- [Global definition index (alphabetical)](#global-definition-index-alphabetical)
- [Module families](#module-families)
  - [administrativeArea](#module-family-administrativearea)
  - [ai](#module-family-ai)
  - [auth](#module-family-auth)
  - [case](#module-family-case)
  - [catalog](#module-family-catalog)
  - [channel](#module-family-channel)
  - [dataComponent](#module-family-datacomponent)
  - [dataSource](#module-family-datasource)
  - [decayRule](#module-family-decayrule)
  - [deleteOperation](#module-family-deleteoperation)
  - [disseminationList](#module-family-disseminationlist)
  - [draftWorkspace](#module-family-draftworkspace)
  - [emailTemplate](#module-family-emailtemplate)
  - [entitySetting](#module-family-entitysetting)
  - [event](#module-family-event)
  - [exclusionList](#module-family-exclusionlist)
  - [fintelDesign](#module-family-finteldesign)
  - [fintelTemplate](#module-family-finteltemplate)
  - [form](#module-family-form)
  - [grouping](#module-family-grouping)
  - [indicator](#module-family-indicator)
  - [ingestion](#module-family-ingestion)
  - [internal](#module-family-internal)
  - [language](#module-family-language)
  - [malwareAnalysis](#module-family-malwareanalysis)
  - [managerConfiguration](#module-family-managerconfiguration)
  - [metrics](#module-family-metrics)
  - [narrative](#module-family-narrative)
  - [notification](#module-family-notification)
  - [notifier](#module-family-notifier)
  - [organization](#module-family-organization)
  - [pir](#module-family-pir)
  - [playbook](#module-family-playbook)
  - [publicDashboard](#module-family-publicdashboard)
  - [requestAccess](#module-family-requestaccess)
  - [savedFilter](#module-family-savedfilter)
  - [securityCoverage](#module-family-securitycoverage)
  - [securityPlatform](#module-family-securityplatform)
  - [stixCyberObservable](#module-family-stixcyberobservable)
  - [support](#module-family-support)
  - [task](#module-family-task)
  - [theme](#module-family-theme)
  - [threatActorIndividual](#module-family-threatactorindividual)
  - [vocabulary](#module-family-vocabulary)
  - [workspace](#module-family-workspace)
  - [xtm](#module-family-xtm)


## Counts
<a id="counts"></a>

| Metric | Value |
| --- | --- |
| SDL files | core=1, module=59, total=60 |
| GraphQL TS files | 57 |
| src/modules/index.ts imports | attributes=13, relationsRef=7, modules=66, graphql=57, total=143 |
| Type system definitions | enum=156, input=220, interface=19, scalar=8, type=624, union=6 |
| Root fields | Query: core=227, modules=156, total=383 \| Mutation: core=157, modules=281, total=438 \| Subscription: core=17, modules=6, total=23 |
| Module families | 46 |
| Deprecated/unwired module SDL files | 2 |


## Manifest (raw excerpt)
<a id="manifest-raw-excerpt"></a>

```text
timestamp=2026-01-27_141500
container=identity-core-opencti-1
image=opencti/platform:6.9.10
created=2026-01-27T21:12:58.975722779Z
```


## Wiring notes
<a id="wiring-notes"></a>

- **GraphQL-only module paths:** 6
  - `./ai/ai`
  - `./auth/auth`
  - `./catalog/catalog`
  - `./metrics/metrics`
  - `./requestAccess/requestAccess`
  - `./xtm/hub/xtm-hub`
- **Module paths without GraphQL registration:** 15
  - `./attackPattern/attackPattern`
  - `./campaign/campaign`
  - `./courseOfAction/courseOfAction`
  - `./externalReference/externalReference`
  - `./incident/incident`
  - `./internal/document/document`
  - `./intrusionSet/intrusionSet`
  - `./malware/malware`
  - `./note/note`
  - `./observedData/observedData`
  - `./report/report`
  - `./stixCyberObservable/stixCyberObservable`
  - `./threatActorGroup/threatActorGroup`
  - `./tool/tool`
  - `./vulnerability/vulnerability`
- **Wired GraphQL mapping missing:** 0


## Wired module SDL order
<a id="wired-module-sdl-order"></a>

1. `src/modules/channel/channel.graphql`
2. `src/modules/catalog/catalog.graphql`
3. `src/modules/language/language.graphql`
4. `src/modules/event/event.graphql`
5. `src/modules/grouping/grouping.graphql`
6. `src/modules/narrative/narrative.graphql`
7. `src/modules/notification/notification.graphql`
8. `src/modules/dataComponent/dataComponent.graphql`
9. `src/modules/dataSource/dataSource.graphql`
10. `src/modules/vocabulary/vocabulary.graphql`
11. `src/modules/administrativeArea/administrativeArea.graphql`
12. `src/modules/task/task.graphql`
13. `src/modules/task/task-template/task-template.graphql`
14. `src/modules/case/case.graphql`
15. `src/modules/case/case-template/case-template.graphql`
16. `src/modules/case/case-incident/case-incident.graphql`
17. `src/modules/case/case-rfi/case-rfi.graphql`
18. `src/modules/case/case-rft/case-rft.graphql`
19. `src/modules/case/feedback/feedback.graphql`
20. `src/modules/entitySetting/entitySetting.graphql`
21. `src/modules/workspace/workspace.graphql`
22. `src/modules/malwareAnalysis/malwareAnalysis.graphql`
23. `src/modules/managerConfiguration/managerConfiguration.graphql`
24. `src/modules/notifier/notifier.graphql`
25. `src/modules/threatActorIndividual/threatActorIndividual.graphql`
26. `src/modules/playbook/playbook.graphql`
27. `src/modules/ingestion/ingestion-rss.graphql`
28. `src/modules/ingestion/ingestion-taxii.graphql`
29. `src/modules/ingestion/ingestion-taxii-collection.graphql`
30. `src/modules/ingestion/ingestion-csv.graphql`
31. `src/modules/ingestion/ingestion-json.graphql`
32. `src/modules/indicator/indicator.graphql`
33. `src/modules/decayRule/decayRule.graphql`
34. `src/modules/decayRule/exclusions/decayExclusionRule.graphql`
35. `src/modules/organization/organization.graphql`
36. `src/modules/internal/csvMapper/csvMapper.graphql`
37. `src/modules/internal/jsonMapper/jsonMapper.graphql`
38. `src/modules/publicDashboard/publicDashboard.graphql`
39. `src/modules/theme/theme.graphql`
40. `src/modules/ai/ai.graphql`
41. `src/modules/deleteOperation/deleteOperation.graphql`
42. `src/modules/support/support.graphql`
43. `src/modules/exclusionList/exclusionList.graphql`
44. `src/modules/draftWorkspace/draftWorkspace.graphql`
45. `src/modules/fintelTemplate/fintelTemplate.graphql`
46. `src/modules/disseminationList/disseminationList.graphql`
47. `src/modules/savedFilter/savedFilter.graphql`
48. `src/modules/requestAccess/requestAccess.graphql`
49. `src/modules/pir/pir.graphql`
50. `src/modules/fintelDesign/fintelDesign.graphql`
51. `src/modules/securityPlatform/securityPlatform.graphql`
52. `src/modules/securityCoverage/securityCoverage.graphql`
53. `src/modules/auth/auth.graphql`
54. `src/modules/emailTemplate/emailTemplate.graphql`
55. `src/modules/form/form.graphql`
56. `src/modules/xtm/hub/xtm-hub.graphql`
57. `src/modules/metrics/metrics.graphql`


## Deprecated / unwired SDL files
<a id="deprecated-unwired-sdl-files"></a>

- `src/modules/internal/csvMapper/deprecated/csvMapper.graphql`
- `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql`


## Root API index
<a id="root-api-index"></a>


### Query API index
<a id="query-api-index"></a>


#### administrativeArea
<a id="administrativearea"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| administrativeArea | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeArea(id: String!): AdministrativeArea @auth(for: [KNOWLEDGE])` |
| administrativeAreas | `AdministrativeAreaConnection` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreas( first: Int after: ID orderBy: AdministrativeAreasOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): AdministrativeAreaConnection @auth(for: [KNOWLEDGE])` |


#### case
<a id="case"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| case | `Case` | `src/modules/case/case.graphql` | `case(id: String!): Case @auth(for: [KNOWLEDGE])` |
| caseIncident | `CaseIncident` | `src/modules/case/case-incident/case-incident.graphql` | `caseIncident(id: String!): CaseIncident @auth(for: [KNOWLEDGE])` |
| caseIncidentContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/case/case-incident/case-incident.graphql` | `caseIncidentContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseIncidents | `CaseIncidentConnection` | `src/modules/case/case-incident/case-incident.graphql` | `caseIncidents( first: Int after: ID orderBy: CaseIncidentsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseIncidentConnection @auth(for: [KNOWLEDGE])` |
| caseRfi | `CaseRfi` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfi(id: String!): CaseRfi @auth(for: [KNOWLEDGE])` |
| caseRfiContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfiContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseRfis | `CaseRfiConnection` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfis( first: Int after: ID orderBy: CaseRfisOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseRfiConnection @auth(for: [KNOWLEDGE])` |
| caseRft | `CaseRft` | `src/modules/case/case-rft/case-rft.graphql` | `caseRft(id: String!): CaseRft @auth(for: [KNOWLEDGE])` |
| caseRftContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/case/case-rft/case-rft.graphql` | `caseRftContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseRfts | `CaseRftConnection` | `src/modules/case/case-rft/case-rft.graphql` | `caseRfts( first: Int after: ID orderBy: CaseRftsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseRftConnection @auth(for: [KNOWLEDGE])` |
| caseTemplate | `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` | `caseTemplate(id: String!): CaseTemplate @auth` |
| caseTemplates | `CaseTemplateConnection` | `src/modules/case/case-template/case-template.graphql` | `caseTemplates( first: Int after: ID orderBy: CaseTemplatesOrdering orderMode: OrderingMode search: String ): CaseTemplateConnection @auth` |
| cases | `CaseConnection` | `src/modules/case/case.graphql` | `cases( first: Int after: ID orderBy: CasesOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseConnection @auth(for: [KNOWLEDGE])` |
| feedback | `Feedback` | `src/modules/case/feedback/feedback.graphql` | `feedback(id: String!): Feedback @auth(for: [KNOWLEDGE])` |
| feedbackContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/case/feedback/feedback.graphql` | `feedbackContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| feedbacks | `FeedbackConnection` | `src/modules/case/feedback/feedback.graphql` | `feedbacks( first: Int after: ID orderBy: FeedbacksOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): FeedbackConnection @auth(for: [KNOWLEDGE])` |


#### catalog
<a id="catalog"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| catalog | `Catalog` | `src/modules/catalog/catalog.graphql` | `catalog(id: String!): Catalog @auth(for: [INGESTION])` |
| catalogs | `[Catalog!]!` | `src/modules/catalog/catalog.graphql` | `catalogs: [Catalog!]! @auth(for: [INGESTION])` |
| contract | `ExtendedContract` | `src/modules/catalog/catalog.graphql` | `contract(slug: String!): ExtendedContract @auth(for: [INGESTION])` |


#### channel
<a id="channel"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| channel | `Channel` | `src/modules/channel/channel.graphql` | `channel(id: String!): Channel @auth(for: [KNOWLEDGE])` |
| channels | `ChannelConnection` | `src/modules/channel/channel.graphql` | `channels( first: Int after: ID orderBy: ChannelsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): ChannelConnection @auth(for: [KNOWLEDGE])` |


#### dataComponent
<a id="datacomponent"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| dataComponent | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponent(id: String!): DataComponent @auth(for: [KNOWLEDGE])` |
| dataComponents | `DataComponentConnection` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponents( first: Int after: ID orderBy: DataComponentsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DataComponentConnection @auth(for: [KNOWLEDGE])` |


#### dataSource
<a id="datasource"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| dataSource | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSource(id: String!): DataSource @auth(for: [KNOWLEDGE])` |
| dataSources | `DataSourceConnection` | `src/modules/dataSource/dataSource.graphql` | `dataSources( first: Int after: ID orderBy: DataSourcesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DataSourceConnection @auth(for: [KNOWLEDGE])` |


#### decayRule
<a id="decayrule"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| decayExclusionRule | `DecayExclusionRule` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` | `decayExclusionRule(id: String!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRules | `DecayExclusionRuleConnection` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` | `decayExclusionRules( first: Int after: ID orderBy: DecayExclusionRuleOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DecayExclusionRuleConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRule | `DecayRule` | `src/modules/decayRule/decayRule.graphql` | `decayRule(id: String!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRules | `DecayRuleConnection` | `src/modules/decayRule/decayRule.graphql` | `decayRules( first: Int after: ID orderBy: DecayRuleOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DecayRuleConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### deleteOperation
<a id="deleteoperation"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| deleteOperation | `DeleteOperation` | `src/modules/deleteOperation/deleteOperation.graphql` | `deleteOperation(id: String!): DeleteOperation @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| deleteOperations | `DeleteOperationConnection` | `src/modules/deleteOperation/deleteOperation.graphql` | `deleteOperations( first: Int after: ID orderBy: DeleteOperationOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DeleteOperationConnection @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### disseminationList
<a id="disseminationlist"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| disseminationList | `DisseminationList` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationList(id: ID!): DisseminationList @auth(for: [KNOWLEDGE_KNDISSEMINATION, SETTINGS_SETDISSEMINATION])` |
| disseminationLists | `DisseminationListConnection` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationLists( first: Int after: ID orderBy: DisseminationListOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DisseminationListConnection @auth(for: [KNOWLEDGE_KNDISSEMINATION, SETTINGS_SETDISSEMINATION])` |


#### draftWorkspace
<a id="draftworkspace"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| draftWorkspace | `DraftWorkspace` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspace(id: String!): DraftWorkspace @auth(for: [KNOWLEDGE])` |
| draftWorkspaceEntities | `StixCoreObjectConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceEntities( draftId: String!, types: [String] first: Int after: ID orderBy: StixCoreObjectsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixCoreObjectConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaceRelationships | `StixRelationshipConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceRelationships( draftId: String!, types: [String] first: Int after: ID orderBy: StixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixRelationshipConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaceSightingRelationships | `StixSightingRelationshipConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceSightingRelationships( draftId: String!, types: [String] first: Int after: ID orderBy: StixSightingRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixSightingRelationshipConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaces | `DraftWorkspaceConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaces( first: Int after: ID orderBy: DraftWorkspacesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DraftWorkspaceConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspacesRestricted | `DraftWorkspaceConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspacesRestricted( first: Int after: ID orderBy: DraftWorkspacesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DraftWorkspaceConnection @auth(for: [KNOWLEDGE])` |


#### emailTemplate
<a id="emailtemplate"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| emailTemplate | `EmailTemplate` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplate(id: ID!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplates | `EmailTemplateConnection` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplates ( first: Int after: ID orderBy: EmailTemplateOrdering orderMode: OrderingMode filters: FilterGroup search: String ): EmailTemplateConnection @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |


#### entitySetting
<a id="entitysetting"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| entitySetting | `EntitySetting` | `src/modules/entitySetting/entitySetting.graphql` | `entitySetting(id: String!): EntitySetting @auth` |
| entitySettingByType | `EntitySetting` | `src/modules/entitySetting/entitySetting.graphql` | `entitySettingByType(targetType: String!): EntitySetting @auth` |
| entitySettings | `EntitySettingConnection` | `src/modules/entitySetting/entitySetting.graphql` | `entitySettings( first: Int after: ID orderBy: EntitySettingsOrdering orderMode: OrderingMode filters: FilterGroup search: String includeObservables: Boolean ): EntitySettingConnection @auth` |


#### event
<a id="event"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| event | `Event` | `src/modules/event/event.graphql` | `event(id: String!): Event @auth(for: [KNOWLEDGE])` |
| events | `EventConnection` | `src/modules/event/event.graphql` | `events( first: Int after: ID orderBy: EventsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): EventConnection @auth(for: [KNOWLEDGE])` |


#### exclusionList
<a id="exclusionlist"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| exclusionList | `ExclusionList` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionList(id: String!): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListCacheStatus | `ExclusionListCacheStatus` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionListCacheStatus: ExclusionListCacheStatus @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionLists | `ExclusionListConnection` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionLists( first: Int after: ID orderBy: ExclusionListOrdering orderMode: OrderingMode filters: FilterGroup search: String ): ExclusionListConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### fintelDesign
<a id="finteldesign"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| fintelDesign | `FintelDesign` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesign(id: String!): FintelDesign @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |
| fintelDesigns | `FintelDesignConnection` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesigns( first: Int after: ID orderBy: FintelDesignOrdering orderMode: OrderingMode filters: FilterGroup search: String ): FintelDesignConnection @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |


#### fintelTemplate
<a id="finteltemplate"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| fintelTemplate | `FintelTemplate` | `src/modules/fintelTemplate/fintelTemplate.graphql` | `fintelTemplate(id: ID!): FintelTemplate @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |


#### form
<a id="form"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| form | `Form` | `src/modules/form/form.graphql` | `form(id: ID!): Form @auth(for: [KNOWLEDGE_KNUPDATE])` |
| forms | `FormConnection` | `src/modules/form/form.graphql` | `forms( search: String first: Int after: ID orderBy: FormsOrdering orderMode: OrderingMode filters: FilterGroup ): FormConnection @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### grouping
<a id="grouping"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| grouping | `Grouping` | `src/modules/grouping/grouping.graphql` | `grouping(id: String!): Grouping @auth(for: [KNOWLEDGE])` |
| groupingContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/grouping/grouping.graphql` | `groupingContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| groupings | `GroupingConnection` | `src/modules/grouping/grouping.graphql` | `groupings( first: Int after: ID orderBy: GroupingsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): GroupingConnection @auth(for: [KNOWLEDGE])` |
| groupingsDistribution | `[Distribution]` | `src/modules/grouping/grouping.graphql` | `groupingsDistribution( objectId: String authorId: String field: String! operation: StatsOperation! limit: Int order: String startDate: DateTime endDate: DateTime dateAttribute: String filters: FilterGroup search: String ): [Distribution] @auth(for: [KNOWLEDGE…` |
| groupingsNumber | `Number` | `src/modules/grouping/grouping.graphql` | `groupingsNumber( groupingContext: String objectId: String authorId: String endDate: DateTime filters: FilterGroup ): Number @auth(for: [KNOWLEDGE, EXPLORE])` |
| groupingsTimeSeries | `[TimeSeries]` | `src/modules/grouping/grouping.graphql` | `groupingsTimeSeries( objectId: String authorId: String groupingType: String field: String! operation: StatsOperation! startDate: DateTime! endDate: DateTime! interval: String! filters: FilterGroup search: String ): [TimeSeries] @auth(for: [KNOWLEDGE, EXPLORE])` |


#### indicator
<a id="indicator"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| indicator | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicator(id: String!): Indicator @auth(for: [KNOWLEDGE])` |
| indicators | `IndicatorConnection` | `src/modules/indicator/indicator.graphql` | `indicators( first: Int after: ID orderBy: IndicatorsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): IndicatorConnection @auth(for: [KNOWLEDGE])` |
| indicatorsDistribution | `[Distribution]` | `src/modules/indicator/indicator.graphql` | `indicatorsDistribution( objectId: String field: String! operation: StatsOperation! limit: Int order: String startDate: DateTime endDate: DateTime dateAttribute: String ): [Distribution] @auth(for: [KNOWLEDGE])` |
| indicatorsNumber | `Number` | `src/modules/indicator/indicator.graphql` | `indicatorsNumber(pattern_type: String, objectId: String, endDate: DateTime): Number @auth(for: [KNOWLEDGE])` |
| indicatorsTimeSeries | `[TimeSeries]` | `src/modules/indicator/indicator.graphql` | `indicatorsTimeSeries( objectId: String field: String! operation: StatsOperation! startDate: DateTime! endDate: DateTime! interval: String! filters: FilterGroup ): [TimeSeries] @auth(for: [KNOWLEDGE])` |


#### ingestion
<a id="ingestion"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| csvFeedAddInputFromImport | `CSVFeedAddInputFromImport!` | `src/modules/ingestion/ingestion-csv.graphql` | `csvFeedAddInputFromImport( file: Upload! ): CSVFeedAddInputFromImport! @auth(for: [INGESTION, CSVMAPPERS])` |
| defaultIngestionGroupCount | `Int` | `src/modules/ingestion/ingestion-csv.graphql` | `defaultIngestionGroupCount: Int @auth(for: [INGESTION])` |
| ingestionCsv | `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsv(id: String!): IngestionCsv @auth(for: [INGESTION])` |
| ingestionCsvs | `IngestionCsvConnection` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvs( first: Int after: ID orderBy: IngestionCsvOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionCsvConnection @auth(for: [INGESTION])` |
| ingestionJson | `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJson(id: String!): IngestionJson @auth(for: [INGESTION])` |
| ingestionJsons | `IngestionJsonConnection` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsons( first: Int after: ID orderBy: IngestionJsonOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionJsonConnection @auth(for: [INGESTION])` |
| ingestionRss | `IngestionRss` | `src/modules/ingestion/ingestion-rss.graphql` | `ingestionRss(id: String!): IngestionRss @auth(for: [INGESTION])` |
| ingestionRsss | `IngestionRssConnection` | `src/modules/ingestion/ingestion-rss.graphql` | `ingestionRsss( first: Int after: ID orderBy: IngestionRssOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionRssConnection @auth(for: [INGESTION])` |
| ingestionTaxii | `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxii(id: String!): IngestionTaxii @auth(for: [INGESTION])` |
| ingestionTaxiiCollection | `IngestionTaxiiCollection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` | `ingestionTaxiiCollection(id: String!): IngestionTaxiiCollection @auth(for: [INGESTION])` |
| ingestionTaxiiCollections | `IngestionTaxiiCollectionConnection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` | `ingestionTaxiiCollections( first: Int after: ID orderBy: IngestionTaxiiCollectionOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionTaxiiCollectionConnection @auth(for: [INGESTION])` |
| ingestionTaxiis | `IngestionTaxiiConnection` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiis( first: Int after: ID orderBy: IngestionTaxiiOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionTaxiiConnection @auth(for: [INGESTION])` |
| taxiiFeedAddInputFromImport | `TaxiiFeedAddInputFromImport!` | `src/modules/ingestion/ingestion-taxii.graphql` | `taxiiFeedAddInputFromImport( file: Upload! ): TaxiiFeedAddInputFromImport! @auth(for: [INGESTION])` |
| userAlreadyExists | `Boolean` | `src/modules/ingestion/ingestion-csv.graphql` | `userAlreadyExists(name: String!): Boolean @auth(for: [INGESTION])` |


#### internal
<a id="internal"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| csvMapper | `CsvMapper` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapper(id: ID!): CsvMapper @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| csvMapperAddInputFromImport | `CsvMapperAddInputFromImport!` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperAddInputFromImport( file: Upload! ): CsvMapperAddInputFromImport! @auth(for: [CSVMAPPERS])` |
| csvMapperSchemaAttributes | `[CsvMapperSchemaAttributes!]!` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperSchemaAttributes: [CsvMapperSchemaAttributes!]! @auth(for: [KNOWLEDGE, CSVMAPPERS])` |
| csvMapperTest | `CsvMapperTestResult` | `src/modules/internal/csvMapper/deprecated/csvMapper.graphql` | csvMapperTest( configuration: String! content: String! ): CsvMapperTestResult @auth(for: [CSVMAPPERS]) @deprecated(reason: "[>=6.4 & <6.7]. Use `csvMapperTest mutation`.") |
| csvMappers | `CsvMapperConnection` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMappers( first: Int after: ID orderBy: CsvMapperOrdering orderMode: OrderingMode filters: FilterGroup search: String ): CsvMapperConnection @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| jsonMapper | `JsonMapper` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapper(id: ID!): JsonMapper @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| jsonMappers | `JsonMapperConnection` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMappers( first: Int after: ID orderBy: JsonMapperOrdering orderMode: OrderingMode filters: FilterGroup search: String ): JsonMapperConnection @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |


#### language
<a id="language"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| language | `Language` | `src/modules/language/language.graphql` | `language(id: String!): Language @auth(for: [KNOWLEDGE])` |
| languages | `LanguageConnection` | `src/modules/language/language.graphql` | `languages( first: Int after: ID orderBy: LanguagesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): LanguageConnection @auth(for: [KNOWLEDGE])` |


#### malwareAnalysis
<a id="malwareanalysis"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| malwareAnalyses | `MalwareAnalysisConnection` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalyses( first: Int after: ID orderBy: MalwareAnalysesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): MalwareAnalysisConnection @auth(for: [KNOWLEDGE])` |
| malwareAnalysis | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysis(id: String!): MalwareAnalysis @auth(for: [KNOWLEDGE])` |


#### managerConfiguration
<a id="managerconfiguration"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| managerConfiguration | `ManagerConfiguration` | `src/modules/managerConfiguration/managerConfiguration.graphql` | `managerConfiguration(id: String!): ManagerConfiguration @auth(for: [SETTINGS_FILEINDEXING])` |
| managerConfigurationByManagerId | `ManagerConfiguration` | `src/modules/managerConfiguration/managerConfiguration.graphql` | `managerConfigurationByManagerId(managerId: String!): ManagerConfiguration @auth` |


#### narrative
<a id="narrative"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| narrative | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrative(id: String!): Narrative @auth(for: [KNOWLEDGE])` |
| narratives | `NarrativeConnection` | `src/modules/narrative/narrative.graphql` | `narratives( first: Int after: ID orderBy: NarrativesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NarrativeConnection @auth(for: [KNOWLEDGE])` |


#### notification
<a id="notification"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| myNotifications | `NotificationConnection` | `src/modules/notification/notification.graphql` | `myNotifications( first: Int after: ID orderBy: NotificationsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotificationConnection @auth` |
| myUnreadNotificationsCount | `Int` | `src/modules/notification/notification.graphql` | `myUnreadNotificationsCount: Int @auth` |
| notification | `Notification` | `src/modules/notification/notification.graphql` | `notification(id: String!): Notification @auth` |
| notifications | `NotificationConnection` | `src/modules/notification/notification.graphql` | `notifications( first: Int after: ID orderBy: NotificationsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotificationConnection @auth(for: [SETTINGS_SETACCESSES])` |
| triggerActivity | `Trigger` | `src/modules/notification/notification.graphql` | `triggerActivity(id: String!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerKnowledge | `Trigger` | `src/modules/notification/notification.graphql` | `triggerKnowledge(id: String!): Trigger @auth` |
| triggers | `TriggerConnection` | `src/modules/notification/notification.graphql` | `triggers( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): TriggerConnection @auth` |
| triggersActivity | `TriggerConnection` | `src/modules/notification/notification.graphql` | `triggersActivity( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup search: String ): TriggerConnection @auth(for: [SETTINGS_SECURITYACTIVITY]) # Notifications` |
| triggersKnowledge | `TriggerConnection` | `src/modules/notification/notification.graphql` | `triggersKnowledge( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): TriggerConnection @auth` |
| triggersKnowledgeCount | `Int` | `src/modules/notification/notification.graphql` | `triggersKnowledgeCount(filters: FilterGroup, includeAuthorities: Boolean, search: String): Int @auth # Alerts` |


#### notifier
<a id="notifier"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| notificationNotifiers | `[Notifier!]!` | `src/modules/notifier/notifier.graphql` | `notificationNotifiers: [Notifier!]! @auth` |
| notifier | `Notifier` | `src/modules/notifier/notifier.graphql` | `notifier(id: String!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierTest | `String` | `src/modules/notifier/notifier.graphql` | `notifierTest(input: NotifierTestInput!): String @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifiers | `NotifierConnection` | `src/modules/notifier/notifier.graphql` | `notifiers( first: Int after: ID orderBy: NotifierOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotifierConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### organization
<a id="organization"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| organization | `Organization` | `src/modules/organization/organization.graphql` | `organization(id: String!): Organization @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATION_ADMIN])` |
| organizations | `OrganizationConnection` | `src/modules/organization/organization.graphql` | `organizations( first: Int after: ID orderBy: OrganizationsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): OrganizationConnection @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATI…` |
| securityOrganizations | `OrganizationConnection` | `src/modules/organization/organization.graphql` | `securityOrganizations( first: Int after: ID orderBy: OrganizationsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): OrganizationConnection @auth(for: [SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATION_…` |


#### pir
<a id="pir"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| pir | `Pir` | `src/modules/pir/pir.graphql` | `pir(id: ID!): Pir @auth(for: [PIRAPI])` |
| pirLogs | `LogConnection` | `src/modules/pir/pir.graphql` | `pirLogs( pirId: ID! first: Int after: ID orderBy: LogsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): LogConnection @auth(for: [KNOWLEDGE, PIRAPI])` |
| pirRelationships | `PirRelationshipConnection` | `src/modules/pir/pir.graphql` | `pirRelationships( pirId: ID! first: Int after: ID orderBy: PirRelationshipOrdering orderMode: OrderingMode fromId: [String] fromRole: String fromTypes: [String] startTimeStart: DateTime startTimeStop: DateTime stopTimeStart: DateTime stopTimeStop: DateTime fi…` |
| pirRelationshipsDistribution | `[Distribution]` | `src/modules/pir/pir.graphql` | `pirRelationshipsDistribution( pirId: ID! field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String fromId: [String] fromTypes: [String] relationship_type: [String] search: Stri…` |
| pirRelationshipsMultiTimeSeries | `[MultiTimeSeries]` | `src/modules/pir/pir.graphql` | `pirRelationshipsMultiTimeSeries( operation: StatsOperation! startDate: DateTime! endDate: DateTime interval: String! onlyInferred: Boolean timeSeriesParameters: [PirRelationshipsTimeSeriesParameters!]! relationship_type: [String!] ): [MultiTimeSeries] @auth(f…` |
| pirs | `PirConnection` | `src/modules/pir/pir.graphql` | `pirs( first: Int after: ID orderBy: PirOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PirConnection @auth(for: [PIRAPI])` |


#### playbook
<a id="playbook"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| playbook | `Playbook` | `src/modules/playbook/playbook.graphql` | `playbook(id: String!): Playbook @auth(for: [AUTOMATION])` |
| playbookComponents | `[PlaybookComponent]!` | `src/modules/playbook/playbook.graphql` | `playbookComponents: [PlaybookComponent]! @auth(for: [AUTOMATION])` |
| playbooks | `PlaybookConnection` | `src/modules/playbook/playbook.graphql` | `playbooks( first: Int after: ID orderBy: PlaybooksOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PlaybookConnection @auth(for: [AUTOMATION])` |
| playbooksForEntity | `[Playbook]` | `src/modules/playbook/playbook.graphql` | `playbooksForEntity(id: String!): [Playbook] @auth(for: [AUTOMATION])` |


#### publicDashboard
<a id="publicdashboard"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| publicBookmarks | `StixDomainObjectConnection` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicBookmarks( uriKey: String! widgetId : String! ): StixDomainObjectConnection @public` |
| publicDashboard | `PublicDashboard` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboard(id: String!): PublicDashboard @auth(for: [EXPLORE])` |
| publicDashboardByUriKey | `PublicDashboard` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboardByUriKey(uri_key: String!): PublicDashboard @public` |
| publicDashboards | `PublicDashboardConnection` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboards( first: Int after: ID orderBy: PublicDashboardsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PublicDashboardConnection @auth(for: [EXPLORE])` |
| publicStixCoreObjects | `StixCoreObjectConnection` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixCoreObjects( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): StixCoreObjectConnection @public` |
| publicStixCoreObjectsDistribution | `[PublicDistribution]` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixCoreObjectsDistribution( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [PublicDistribution] @public` |
| publicStixCoreObjectsMultiTimeSeries | `[MultiTimeSeries]` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixCoreObjectsMultiTimeSeries( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [MultiTimeSeries] @public` |
| publicStixCoreObjectsNumber | `Number` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixCoreObjectsNumber( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): Number @public` |
| publicStixRelationships | `StixRelationshipConnection` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixRelationships( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): StixRelationshipConnection @public` |
| publicStixRelationshipsDistribution | `[PublicDistribution]` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixRelationshipsDistribution( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [PublicDistribution] @public` |
| publicStixRelationshipsMultiTimeSeries | `[MultiTimeSeries]` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixRelationshipsMultiTimeSeries( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [MultiTimeSeries] @public` |
| publicStixRelationshipsNumber | `Number` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicStixRelationshipsNumber( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): Number @public` |


#### savedFilter
<a id="savedfilter"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| savedFilters | `SavedFilterConnection` | `src/modules/savedFilter/savedFilter.graphql` | `savedFilters( first: Int after: ID orderBy: SavedFilterOrdering orderMode: OrderingMode filters: FilterGroup search: String ): SavedFilterConnection @auth` |


#### securityCoverage
<a id="securitycoverage"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| securityCoverage | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverage(id: String!): SecurityCoverage @auth(for: [KNOWLEDGE])` |
| securityCoverages | `SecurityCoverageConnection` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverages( first: Int after: ID orderBy: SecurityCoverageOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): SecurityCoverageConnection @auth(for: [KNOWLEDGE])` |


#### securityPlatform
<a id="securityplatform"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| securityPlatform | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatform(id: String!): SecurityPlatform @auth(for: [KNOWLEDGE])` |
| securityPlatforms | `SecurityPlatformConnection` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatforms( first: Int after: ID orderBy: SecurityPlatformOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): SecurityPlatformConnection @auth(for: [KNOWLEDGE])` |


#### support
<a id="support"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| supportPackage | `SupportPackage` | `src/modules/support/support.graphql` | `supportPackage(id: String!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |
| supportPackages | `SupportPackageConnection` | `src/modules/support/support.graphql` | `supportPackages( first: Int after: ID orderBy: SupportPackageOrdering orderMode: OrderingMode filters: FilterGroup search: String ): SupportPackageConnection @auth(for: [SETTINGS_SUPPORT])` |


#### task
<a id="task"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| task | `Task` | `src/modules/task/task.graphql` | `task(id: String!): Task @auth` |
| taskContainsStixObjectOrStixRelationship | `Boolean` | `src/modules/task/task.graphql` | `taskContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| taskTemplate | `TaskTemplate` | `src/modules/task/task-template/task-template.graphql` | `taskTemplate(id: String!): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplates | `TaskTemplateConnection` | `src/modules/task/task-template/task-template.graphql` | `taskTemplates( first: Int after: ID orderBy: TaskTemplatesOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): TaskTemplateConnection @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| tasks | `TaskConnection` | `src/modules/task/task.graphql` | `tasks( first: Int after: ID orderBy: TasksOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): TaskConnection @auth` |


#### theme
<a id="theme"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| theme | `Theme` | `src/modules/theme/theme.graphql` | `theme(id: ID!): Theme @public` |
| themes | `ThemeConnection` | `src/modules/theme/theme.graphql` | `themes( first: Int after: ID orderBy: ThemeOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): ThemeConnection @public` |


#### threatActorIndividual
<a id="threatactorindividual"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| threatActorIndividual | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividual(id: String!): ThreatActorIndividual @auth(for: [KNOWLEDGE])` |
| threatActorsIndividuals | `ThreatActorIndividualConnection` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorsIndividuals( first: Int after: ID orderBy: ThreatActorsIndividualOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): ThreatActorIndividualConnection @auth(for: [KNOWLEDGE])` |


#### vocabulary
<a id="vocabulary"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| vocabularies | `VocabularyConnection` | `src/modules/vocabulary/vocabulary.graphql` | `vocabularies( category: VocabularyCategory first: Int after: ID orderBy: VocabularyOrdering orderMode: OrderingMode filters: FilterGroup search: String ): VocabularyConnection @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SETVOCABULARIES, INGESTION_SE…` |
| vocabulary | `Vocabulary` | `src/modules/vocabulary/vocabulary.graphql` | `vocabulary(id: String!): Vocabulary @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SETVOCABULARIES])` |
| vocabularyCategories | `[VocabularyDefinition!]!` | `src/modules/vocabulary/vocabulary.graphql` | `vocabularyCategories: [VocabularyDefinition!]! @auth` |


#### workspace
<a id="workspace"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| workspace | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspace(id: String!): Workspace @auth(for: [EXPLORE, SETTINGS_SETACCESSES, INVESTIGATION])` |
| workspaces | `WorkspaceConnection` | `src/modules/workspace/workspace.graphql` | `workspaces( first: Int after: ID orderBy: WorkspacesOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): WorkspaceConnection @auth(for: [EXPLORE, SETTINGS_SETACCESSES, INVESTIGATION])` |


### Mutation API index
<a id="mutation-api-index"></a>


#### administrativeArea
<a id="administrativearea"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| administrativeAreaAdd | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaAdd(input: AdministrativeAreaAddInput!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaContextClean | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaContextClean(id: ID!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaContextPatch | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaContextPatch(id: ID!, input: EditContext!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaDelete | `ID` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| administrativeAreaFieldPatch | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaRelationAdd | `StixRefRelationship` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaRelationDelete | `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` | `administrativeAreaRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### ai
<a id="ai"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| aiChangeTone | `String` | `src/modules/ai/ai.graphql` | `aiChangeTone(id: ID!, content: String!, format: Format, tone: Tone): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiContainerGenerateReport | `String` | `src/modules/ai/ai.graphql` | `aiContainerGenerateReport(id: ID!, containerId: String!, paragraphs: Int, tone: Tone, format: Format, language: String): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiConvertFilesToStix | `String` | `src/modules/ai/ai.graphql` | `aiConvertFilesToStix(id: ID!, elementId: String!, fileIds: [String]): String @auth(for: [KNOWLEDGE_KNUPDATE]) # Indicators` |
| aiConvertIndicator | `String` | `src/modules/ai/ai.graphql` | `aiConvertIndicator(id: ID!, indicatorId: String!, format: IndicatorFormat!): String @auth(for: [KNOWLEDGE_KNUPDATE]) # Generic text` |
| aiExplain | `String` | `src/modules/ai/ai.graphql` | `aiExplain(id: ID!, content: String!): String @auth(for: [KNOWLEDGE])` |
| aiFixSpelling | `String` | `src/modules/ai/ai.graphql` | `aiFixSpelling(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiImproveWriting | `String` | `src/modules/ai/ai.graphql` | `aiImproveWriting(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiMakeLonger | `String` | `src/modules/ai/ai.graphql` | `aiMakeLonger(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiMakeShorter | `String` | `src/modules/ai/ai.graphql` | `aiMakeShorter(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiNLQ | `NLQResponse` | `src/modules/ai/ai.graphql` | `aiNLQ(search: String!): NLQResponse @auth(for: [KNOWLEDGE])` |
| aiSummarize | `String` | `src/modules/ai/ai.graphql` | `aiSummarize(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiSummarizeFiles | `String` | `src/modules/ai/ai.graphql` | `aiSummarizeFiles(id: ID!, elementId: String!, paragraphs: Int, tone: Tone, format: Format, language: String, fileIds: [String]): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiThreatGenerateReport | `String` | `src/modules/ai/ai.graphql` | `aiThreatGenerateReport(id: ID!, threatId: String!, paragraphs: Int, tone: Tone, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiVictimGenerateReport | `String` | `src/modules/ai/ai.graphql` | `aiVictimGenerateReport(id: ID!, victimId: String!, paragraphs: Int, tone: Tone, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### auth
<a id="auth"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| askSendOtp | `String` | `src/modules/auth/auth.graphql` | `askSendOtp(input: AskSendOtpInput!): String @public @rateLimit(limit: 1, duration: 1)` |
| changePassword | `Boolean` | `src/modules/auth/auth.graphql` | `changePassword(input: ChangePasswordInput!): Boolean @public @rateLimit(limit: 1, duration: 1)` |
| verifyMfa | `Boolean` | `src/modules/auth/auth.graphql` | `verifyMfa(input: VerifyMfaInput!): Boolean @public @rateLimit(limit: 1, duration: 1)` |
| verifyOtp | `VerifyOtp` | `src/modules/auth/auth.graphql` | `verifyOtp(input: VerifyOtpInput!): VerifyOtp @public @rateLimit(limit: 1, duration: 1)` |


#### case
<a id="case"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| caseDelete | `ID` | `src/modules/case/case.graphql` | `caseDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseIncidentAdd | `CaseIncident` | `src/modules/case/case-incident/case-incident.graphql` | `caseIncidentAdd(input: CaseIncidentAddInput!): CaseIncident @auth` |
| caseIncidentDelete | `ID` | `src/modules/case/case-incident/case-incident.graphql` | `caseIncidentDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseRfiAdd | `CaseRfi` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfiAdd(input: CaseRfiAddInput!): CaseRfi @auth` |
| caseRfiApprove | `CaseRfi` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfiApprove(id: ID!): CaseRfi @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS, KNOWLEDGE_KNUPDATE_KNORGARESTRICT])` |
| caseRfiDecline | `CaseRfi` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfiDecline(id: ID!): CaseRfi @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS, KNOWLEDGE_KNUPDATE_KNORGARESTRICT])` |
| caseRfiDelete | `ID` | `src/modules/case/case-rfi/case-rfi.graphql` | `caseRfiDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseRftAdd | `CaseRft` | `src/modules/case/case-rft/case-rft.graphql` | `caseRftAdd(input: CaseRftAddInput!): CaseRft @auth` |
| caseRftDelete | `ID` | `src/modules/case/case-rft/case-rft.graphql` | `caseRftDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseSetTemplate | `Case` | `src/modules/case/case.graphql` | `caseSetTemplate(id: ID!, caseTemplatesId: [ID!]!): Case @auth(for: [KNOWLEDGE_KNUPDATE])` |
| caseTemplateAdd | `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` | `caseTemplateAdd(input: CaseTemplateAddInput!): CaseTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateDelete | `ID` | `src/modules/case/case-template/case-template.graphql` | `caseTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateFieldPatch | `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` | `caseTemplateFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): CaseTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateRelationAdd | `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` | `caseTemplateRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): CaseTemplate @auth(for: [KNOWLEDGE_KNUPDATE])` |
| caseTemplateRelationDelete | `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` | `caseTemplateRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): CaseTemplate @auth(for: [KNOWLEDGE_KNUPDATE])` |
| feedbackAdd | `Feedback` | `src/modules/case/feedback/feedback.graphql` | `feedbackAdd(input: FeedbackAddInput!): Feedback @auth` |
| feedbackDelete | `ID` | `src/modules/case/feedback/feedback.graphql` | `feedbackDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| feedbackEditAuthorizedMembers | `Feedback` | `src/modules/case/feedback/feedback.graphql` | `feedbackEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]): Feedback @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS])` |


#### channel
<a id="channel"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| channelAdd | `Channel` | `src/modules/channel/channel.graphql` | `channelAdd(input: ChannelAddInput!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelContextClean | `Channel` | `src/modules/channel/channel.graphql` | `channelContextClean(id: ID!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelContextPatch | `Channel` | `src/modules/channel/channel.graphql` | `channelContextPatch(id: ID!, input: EditContext!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelDelete | `ID` | `src/modules/channel/channel.graphql` | `channelDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| channelFieldPatch | `Channel` | `src/modules/channel/channel.graphql` | `channelFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelRelationAdd | `StixRefRelationship` | `src/modules/channel/channel.graphql` | `channelRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelRelationDelete | `Channel` | `src/modules/channel/channel.graphql` | `channelRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### dataComponent
<a id="datacomponent"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| dataComponentAdd | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentAdd(input: DataComponentAddInput!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentContextClean | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentContextClean(id: ID!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentContextPatch | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentContextPatch(id: ID!, input: EditContext!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentDelete | `ID` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataComponentFieldPatch | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentRelationAdd | `StixRefRelationship` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentRelationDelete | `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` | `dataComponentRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### dataSource
<a id="datasource"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| dataSourceAdd | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceAdd(input: DataSourceAddInput!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceContextClean | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceContextClean(id: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceContextPatch | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceContextPatch(id: ID!, input: EditContext!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceDataComponentAdd | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceDataComponentAdd(id: ID!, dataComponentId: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceDataComponentDelete | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceDataComponentDelete(id: ID!, dataComponentId: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceDelete | `ID` | `src/modules/dataSource/dataSource.graphql` | `dataSourceDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceFieldPatch | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceRelationAdd | `StixRefRelationship` | `src/modules/dataSource/dataSource.graphql` | `dataSourceRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceRelationDelete | `DataSource` | `src/modules/dataSource/dataSource.graphql` | `dataSourceRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### decayRule
<a id="decayrule"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| decayExclusionRuleAdd | `DecayExclusionRule` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` | `decayExclusionRuleAdd(input: DecayExclusionRuleAddInput!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRuleDelete | `ID` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` | `decayExclusionRuleDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRuleFieldPatch | `DecayExclusionRule` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` | `decayExclusionRuleFieldPatch(id: ID!, input: [EditInput!]!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleAdd | `DecayRule` | `src/modules/decayRule/decayRule.graphql` | `decayRuleAdd(input: DecayRuleAddInput!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleDelete | `ID` | `src/modules/decayRule/decayRule.graphql` | `decayRuleDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleFieldPatch | `DecayRule` | `src/modules/decayRule/decayRule.graphql` | `decayRuleFieldPatch(id: ID!, input: [EditInput!]!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### deleteOperation
<a id="deleteoperation"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| deleteOperationConfirm | `ID` | `src/modules/deleteOperation/deleteOperation.graphql` | `deleteOperationConfirm(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| deleteOperationRestore | `ID` | `src/modules/deleteOperation/deleteOperation.graphql` | `deleteOperationRestore(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### disseminationList
<a id="disseminationlist"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| disseminationListAdd | `DisseminationList` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationListAdd(input: DisseminationListAddInput!): DisseminationList @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListDelete | `ID` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationListDelete(id: ID!): ID @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListFieldPatch | `DisseminationList` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationListFieldPatch(id: ID!, input: [EditInput!]!): DisseminationList @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListSend | `Boolean` | `src/modules/disseminationList/disseminationList.graphql` | `disseminationListSend(id: ID!, input: DisseminationListSendInput!): Boolean @auth(for: [KNOWLEDGE_KNDISSEMINATION])` |


#### draftWorkspace
<a id="draftworkspace"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| draftWorkspaceAdd | `DraftWorkspace` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceAdd(input: DraftWorkspaceAddInput!): DraftWorkspace @auth(for: [KNOWLEDGE])` |
| draftWorkspaceDelete | `ID` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceDelete(id: ID!): ID @auth(for: [KNOWLEDGE])` |
| draftWorkspaceEditAuthorizedMembers | `DraftWorkspace` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceEditAuthorizedMembers(id: ID!, input: [MemberAccessInput!]): DraftWorkspace @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| draftWorkspaceValidate | `Work` | `src/modules/draftWorkspace/draftWorkspace.graphql` | `draftWorkspaceValidate(id: ID!): Work @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### emailTemplate
<a id="emailtemplate"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| emailTemplateAdd | `EmailTemplate` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplateAdd(input: EmailTemplateAddInput!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateDelete | `ID` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateFieldPatch | `EmailTemplate` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplateFieldPatch(id: ID!, input: [EditInput!]!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateTestSend | `Boolean` | `src/modules/emailTemplate/emailTemplate.graphql` | `emailTemplateTestSend(id: ID!): Boolean @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |


#### entitySetting
<a id="entitysetting"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| entitySettingsFieldPatch | `[EntitySetting]` | `src/modules/entitySetting/entitySetting.graphql` | `entitySettingsFieldPatch(ids: [ID!]!, input: [EditInput!]!, commitMessage: String, references: [String]): [EntitySetting] @auth(for: [SETTINGS_SETCUSTOMIZATION, SETTINGS_SETPARAMETERS])` |


#### event
<a id="event"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| eventAdd | `Event` | `src/modules/event/event.graphql` | `eventAdd(input: EventAddInput!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventContextClean | `Event` | `src/modules/event/event.graphql` | `eventContextClean(id: ID!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventContextPatch | `Event` | `src/modules/event/event.graphql` | `eventContextPatch(id: ID!, input: EditContext!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventDelete | `ID` | `src/modules/event/event.graphql` | `eventDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| eventFieldPatch | `Event` | `src/modules/event/event.graphql` | `eventFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventRelationAdd | `StixRefRelationship` | `src/modules/event/event.graphql` | `eventRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventRelationDelete | `Event` | `src/modules/event/event.graphql` | `eventRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### exclusionList
<a id="exclusionlist"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| exclusionListDelete | `ID` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionListDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListFieldPatch | `ExclusionList` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionListFieldPatch(id: ID!, input: [EditInput!], file: Upload): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListFileAdd | `ExclusionList` | `src/modules/exclusionList/exclusionList.graphql` | `exclusionListFileAdd(input: ExclusionListFileAddInput!): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### fintelDesign
<a id="finteldesign"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| fintelDesignAdd | `FintelDesign` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesignAdd(input: FintelDesignAddInput!): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignContextPatch | `FintelDesign` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesignContextPatch(id: ID!, input: EditContext!): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignDelete | `ID` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesignDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignFieldPatch | `FintelDesign` | `src/modules/fintelDesign/fintelDesign.graphql` | `fintelDesignFieldPatch(id: ID!, input: [EditInput!], file: Upload): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### fintelTemplate
<a id="finteltemplate"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| fintelTemplateAdd | `FintelTemplate` | `src/modules/fintelTemplate/fintelTemplate.graphql` | `fintelTemplateAdd(input: FintelTemplateAddInput!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateConfigurationImport | `FintelTemplate` | `src/modules/fintelTemplate/fintelTemplate.graphql` | `fintelTemplateConfigurationImport(file: Upload!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateDelete | `ID` | `src/modules/fintelTemplate/fintelTemplate.graphql` | `fintelTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateFieldPatch | `FintelTemplate` | `src/modules/fintelTemplate/fintelTemplate.graphql` | `fintelTemplateFieldPatch(id: ID!, input: [EditInput!]!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### form
<a id="form"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| formAdd | `Form` | `src/modules/form/form.graphql` | `formAdd(input: FormAddInput!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formDelete | `ID` | `src/modules/form/form.graphql` | `formDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| formFieldPatch | `Form` | `src/modules/form/form.graphql` | `formFieldPatch(id: ID!, input: [EditInput!]!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formImport | `Form` | `src/modules/form/form.graphql` | `formImport(file: Upload!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formSubmit | `FormSubmissionResponse` | `src/modules/form/form.graphql` | `formSubmit(input: FormSubmissionInput!, isDraft: Boolean! = false): FormSubmissionResponse @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### grouping
<a id="grouping"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| groupingAdd | `Grouping` | `src/modules/grouping/grouping.graphql` | `groupingAdd(input: GroupingAddInput!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingContextClean | `Grouping` | `src/modules/grouping/grouping.graphql` | `groupingContextClean(id: ID!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingContextPatch | `Grouping` | `src/modules/grouping/grouping.graphql` | `groupingContextPatch(id: ID!, input: EditContext): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingDelete | `ID` | `src/modules/grouping/grouping.graphql` | `groupingDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| groupingFieldPatch | `Grouping` | `src/modules/grouping/grouping.graphql` | `groupingFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingRelationAdd | `StixRefRelationship` | `src/modules/grouping/grouping.graphql` | `groupingRelationAdd(id: ID!, input: StixRefRelationshipAddInput): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingRelationDelete | `Grouping` | `src/modules/grouping/grouping.graphql` | `groupingRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### indicator
<a id="indicator"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| indicatorAdd | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicatorAdd(input: IndicatorAddInput!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorContextClean | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicatorContextClean(id: ID!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorContextPatch | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicatorContextPatch(id: ID!, input: EditContext): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorDelete | `ID` | `src/modules/indicator/indicator.graphql` | `indicatorDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| indicatorFieldPatch | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicatorFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorRelationAdd | `StixRefRelationship` | `src/modules/indicator/indicator.graphql` | `indicatorRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorRelationDelete | `Indicator` | `src/modules/indicator/indicator.graphql` | `indicatorRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### ingestion
<a id="ingestion"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| ingestionCsvAdd | `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvAdd(input: IngestionCsvAddInput!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvAddAutoUser | `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvAddAutoUser(id: ID!, input: IngestionCsvAddAutoUserInput!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvDelete | `ID` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvDelete(id: ID!): ID @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvFieldPatch | `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvFieldPatch(id: ID!, input: [EditInput!]!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvResetState | `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvResetState(id: ID!): IngestionCsv @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionCsvTester | `CsvMapperTestResult` | `src/modules/ingestion/ingestion-csv.graphql` | `ingestionCsvTester(input: IngestionCsvAddInput!): CsvMapperTestResult @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonAdd | `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonAdd(input: IngestionJsonAddInput!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonDelete | `ID` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonDelete(id: ID!): ID @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonEdit | `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonEdit(id: ID!, input: IngestionJsonAddInput!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonFieldPatch | `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonFieldPatch(id: ID!, input: [EditInput!]!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonResetState | `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonResetState(id: ID!): IngestionJson @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionJsonTester | `JsonMapperTestResult` | `src/modules/ingestion/ingestion-json.graphql` | `ingestionJsonTester(input: IngestionJsonAddInput!): JsonMapperTestResult @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionRssAdd | `IngestionRss` | `src/modules/ingestion/ingestion-rss.graphql` | `ingestionRssAdd(input: IngestionRssAddInput!): IngestionRss @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionRssDelete | `ID` | `src/modules/ingestion/ingestion-rss.graphql` | `ingestionRssDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionRssFieldPatch | `IngestionRss` | `src/modules/ingestion/ingestion-rss.graphql` | `ingestionRssFieldPatch(id: ID!, input: [EditInput!]!): IngestionRss @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiAdd | `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiiAdd(input: IngestionTaxiiAddInput!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiAddAutoUser | `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiiAddAutoUser(id: ID!, input: IngestionTaxiiAddAutoUserInput!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionAdd | `IngestionTaxiiCollection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` | `ingestionTaxiiCollectionAdd(input: IngestionTaxiiCollectionAddInput!): IngestionTaxiiCollection @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionDelete | `ID` | `src/modules/ingestion/ingestion-taxii-collection.graphql` | `ingestionTaxiiCollectionDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionFieldPatch | `IngestionTaxiiCollection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` | `ingestionTaxiiCollectionFieldPatch(id: ID!, input: [EditInput!]!): IngestionTaxiiCollection @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiDelete | `ID` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiiDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiFieldPatch | `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiiFieldPatch(id: ID!, input: [EditInput!]!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiResetState | `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` | `ingestionTaxiiResetState(id: ID!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |


#### internal
<a id="internal"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| csvMapperAdd | `CsvMapper` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperAdd(input: CsvMapperAddInput!): CsvMapper @auth(for: [CSVMAPPERS])` |
| csvMapperDelete | `ID` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperDelete(id: ID!): ID @auth(for: [CSVMAPPERS])` |
| csvMapperFieldPatch | `CsvMapper` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperFieldPatch(id: ID!, input: [EditInput!]!): CsvMapper @auth(for: [CSVMAPPERS])` |
| csvMapperTest | `CsvMapperTestResult` | `src/modules/internal/csvMapper/csvMapper.graphql` | `csvMapperTest(configuration: String!, file: Upload!): CsvMapperTestResult @auth(for: [CSVMAPPERS])` |
| jsonMapperAdd | `JsonMapper` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapperAdd(input: JsonMapperAddInput!): JsonMapper @auth(for: [CSVMAPPERS])` |
| jsonMapperDelete | `ID` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapperDelete(id: ID!): ID @auth(for: [CSVMAPPERS])` |
| jsonMapperFieldPatch | `JsonMapper` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapperFieldPatch(id: ID!, input: [EditInput!]!): JsonMapper @auth(for: [CSVMAPPERS])` |
| jsonMapperImport | `String!` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapperImport(file: Upload!): String! @auth(for: [CSVMAPPERS])` |
| jsonMapperTest | `JsonMapperTestResult` | `src/modules/internal/jsonMapper/jsonMapper.graphql` | `jsonMapperTest(configuration: String!, file: Upload!): JsonMapperTestResult @auth(for: [CSVMAPPERS])` |


#### language
<a id="language"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| languageAdd | `Language` | `src/modules/language/language.graphql` | `languageAdd(input: LanguageAddInput!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageContextClean | `Language` | `src/modules/language/language.graphql` | `languageContextClean(id: ID!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageContextPatch | `Language` | `src/modules/language/language.graphql` | `languageContextPatch(id: ID!, input: EditContext!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageDelete | `ID` | `src/modules/language/language.graphql` | `languageDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| languageFieldPatch | `Language` | `src/modules/language/language.graphql` | `languageFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageRelationAdd | `StixRefRelationship` | `src/modules/language/language.graphql` | `languageRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageRelationDelete | `Language` | `src/modules/language/language.graphql` | `languageRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### malwareAnalysis
<a id="malwareanalysis"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| malwareAnalysisAdd | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisAdd(input: MalwareAnalysisAddInput!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisContextClean | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisContextClean(id: ID!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisContextPatch | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisContextPatch(id: ID!, input: EditContext!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisDelete | `ID` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| malwareAnalysisFieldPatch | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisRelationAdd | `StixRefRelationship` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisRelationDelete | `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` | `malwareAnalysisRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### managerConfiguration
<a id="managerconfiguration"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| managerConfigurationFieldPatch | `ManagerConfiguration` | `src/modules/managerConfiguration/managerConfiguration.graphql` | `managerConfigurationFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): ManagerConfiguration @auth(for: [SETTINGS_FILEINDEXING])` |


#### metrics
<a id="metrics"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| metricPatch | `BasicObject` | `src/modules/metrics/metrics.graphql` | `metricPatch(id: ID!, input: PatchMetricInput!): BasicObject @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### narrative
<a id="narrative"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| narrativeAdd | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrativeAdd(input: NarrativeAddInput!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeContextClean | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrativeContextClean(id: ID!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeContextPatch | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrativeContextPatch(id: ID!, input: EditContext!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeDelete | `ID` | `src/modules/narrative/narrative.graphql` | `narrativeDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| narrativeFieldPatch | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrativeFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeRelationAdd | `StixRefRelationship` | `src/modules/narrative/narrative.graphql` | `narrativeRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeRelationDelete | `Narrative` | `src/modules/narrative/narrative.graphql` | `narrativeRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### notification
<a id="notification"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| notificationDelete | `ID` | `src/modules/notification/notification.graphql` | `notificationDelete(id: ID!): ID @auth` |
| notificationMarkRead | `Notification` | `src/modules/notification/notification.graphql` | `notificationMarkRead(id: ID!, read: Boolean!): Notification @auth` |
| triggerActivityDelete | `ID` | `src/modules/notification/notification.graphql` | `triggerActivityDelete(id: ID!): ID @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerActivityDigestAdd | `Trigger` | `src/modules/notification/notification.graphql` | `triggerActivityDigestAdd(input: TriggerActivityDigestAddInput!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY]) # Notifications` |
| triggerActivityFieldPatch | `Trigger` | `src/modules/notification/notification.graphql` | `triggerActivityFieldPatch(id: ID!, input: [EditInput!]!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerActivityLiveAdd | `Trigger` | `src/modules/notification/notification.graphql` | `triggerActivityLiveAdd(input: TriggerActivityLiveAddInput!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerKnowledgeDelete | `ID` | `src/modules/notification/notification.graphql` | `triggerKnowledgeDelete(id: ID!): ID @auth` |
| triggerKnowledgeDigestAdd | `Trigger` | `src/modules/notification/notification.graphql` | `triggerKnowledgeDigestAdd(input: TriggerDigestAddInput!): Trigger @auth # Alerts` |
| triggerKnowledgeFieldPatch | `Trigger` | `src/modules/notification/notification.graphql` | `triggerKnowledgeFieldPatch(id: ID!, input: [EditInput!]!): Trigger @auth` |
| triggerKnowledgeLiveAdd | `Trigger` | `src/modules/notification/notification.graphql` | `triggerKnowledgeLiveAdd(input: TriggerLiveAddInput!): Trigger @auth` |


#### notifier
<a id="notifier"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| notifierAdd | `Notifier` | `src/modules/notifier/notifier.graphql` | `notifierAdd(input: NotifierAddInput!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierDelete | `ID` | `src/modules/notifier/notifier.graphql` | `notifierDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierFieldPatch | `Notifier` | `src/modules/notifier/notifier.graphql` | `notifierFieldPatch(id: ID!, input: [EditInput!]!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### organization
<a id="organization"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| organizationAdd | `Organization` | `src/modules/organization/organization.graphql` | `organizationAdd(input: OrganizationAddInput!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationAdminAdd | `Organization` | `src/modules/organization/organization.graphql` | `organizationAdminAdd(id: ID!, memberId: String!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationAdminRemove | `Organization` | `src/modules/organization/organization.graphql` | `organizationAdminRemove(id: ID!, memberId: String!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationContextClean | `Organization` | `src/modules/organization/organization.graphql` | `organizationContextClean(id: ID!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationContextPatch | `Organization` | `src/modules/organization/organization.graphql` | `organizationContextPatch(id: ID!, input: EditContext!): Organization @auth(for: [KNOWLEDGE_KNUPDATE, SETTINGS_SETACCESSES])` |
| organizationDelete | `ID` | `src/modules/organization/organization.graphql` | `organizationDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| organizationEditAuthorizedAuthorities | `Organization` | `src/modules/organization/organization.graphql` | `organizationEditAuthorizedAuthorities(id: ID!, input: [String!]!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationFieldPatch | `Organization` | `src/modules/organization/organization.graphql` | `organizationFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Organization @auth(for: [KNOWLEDGE_KNUPDATE, SETTINGS_SETACCESSES])` |
| organizationRelationAdd | `StixRefRelationship` | `src/modules/organization/organization.graphql` | `organizationRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationRelationDelete | `Organization` | `src/modules/organization/organization.graphql` | `organizationRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### pir
<a id="pir"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| pirAdd | `Pir` | `src/modules/pir/pir.graphql` | `pirAdd(input: PirAddInput!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirDelete | `ID` | `src/modules/pir/pir.graphql` | `pirDelete(id: ID!): ID @auth(for: [PIRAPI_PIRUPDATE])` |
| pirEditAuthorizedMembers | `Pir` | `src/modules/pir/pir.graphql` | `pirEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirFieldPatch | `Pir` | `src/modules/pir/pir.graphql` | `pirFieldPatch(id: ID!, input: [EditInput!]!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirFlagElement | `ID` | `src/modules/pir/pir.graphql` | `pirFlagElement(id: ID!, input: PirFlagElementInput!): ID @auth(for: [BYPASS])` |
| pirUnflagElement | `ID` | `src/modules/pir/pir.graphql` | `pirUnflagElement(id: ID!, input: PirUnflagElementInput!): ID @auth(for: [BYPASS])` |


#### playbook
<a id="playbook"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| playbookAdd | `Playbook` | `src/modules/playbook/playbook.graphql` | `playbookAdd(input: PlaybookAddInput!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookAddLink | `String!` | `src/modules/playbook/playbook.graphql` | `playbookAddLink(id: ID!, input: PlaybookAddLinkInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookAddNode | `String!` | `src/modules/playbook/playbook.graphql` | `playbookAddNode(id: ID!, input: PlaybookAddNodeInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDelete | `ID` | `src/modules/playbook/playbook.graphql` | `playbookDelete(id: ID!): ID @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDeleteLink | `Playbook` | `src/modules/playbook/playbook.graphql` | `playbookDeleteLink(id: ID!, linkId: ID!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDeleteNode | `Playbook` | `src/modules/playbook/playbook.graphql` | `playbookDeleteNode(id: ID!, nodeId: ID!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDuplicate | `String!` | `src/modules/playbook/playbook.graphql` | `playbookDuplicate(id: ID!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookExecute | `Boolean` | `src/modules/playbook/playbook.graphql` | `playbookExecute(id: ID!, entityId: String!): Boolean @auth(for: [AUTOMATION])` |
| playbookFieldPatch | `Playbook` | `src/modules/playbook/playbook.graphql` | `playbookFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookImport | `String!` | `src/modules/playbook/playbook.graphql` | `playbookImport(file: Upload!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookInsertNode | `PlaybookInsertResult!` | `src/modules/playbook/playbook.graphql` | `playbookInsertNode(id: ID!, parentNodeId: ID!, parentPortId: ID!, childNodeId: ID!, input: PlaybookAddNodeInput!): PlaybookInsertResult! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookReplaceNode | `String!` | `src/modules/playbook/playbook.graphql` | `playbookReplaceNode(id: ID!, nodeId: ID!, input: PlaybookAddNodeInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookStepExecution | `Boolean` | `src/modules/playbook/playbook.graphql` | `playbookStepExecution(execution_id: ID!, event_id: ID!, execution_start: DateTime!, data_instance_id: ID!, playbook_id: ID!, previous_step_id: ID!, step_id: ID!, previous_bundle: String!, bundle: String!): Boolean @auth(for: [CONNECTORAPI])` |
| playbookUpdatePositions | `ID` | `src/modules/playbook/playbook.graphql` | `playbookUpdatePositions(id: ID!, positions: String!): ID @auth(for: [AUTOMATION_AUTMANAGE])` |


#### publicDashboard
<a id="publicdashboard"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| publicDashboardAdd | `PublicDashboard` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboardAdd(input: PublicDashboardAddInput!): PublicDashboard @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |
| publicDashboardDelete | `ID` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboardDelete(id: ID!): ID @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |
| publicDashboardFieldPatch | `PublicDashboard` | `src/modules/publicDashboard/publicDashboard.graphql` | `publicDashboardFieldPatch(id: ID!, input: [EditInput!]!): PublicDashboard @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |


#### requestAccess
<a id="requestaccess"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| requestAccessAdd | `ID` | `src/modules/requestAccess/requestAccess.graphql` | `requestAccessAdd(input: RequestAccessAddInput!): ID @auth(for: [KNOWLEDGE])` |
| requestAccessConfigure | `RequestAccessConfiguration` | `src/modules/requestAccess/requestAccess.graphql` | `requestAccessConfigure(input: RequestAccessConfigureInput!): RequestAccessConfiguration @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### savedFilter
<a id="savedfilter"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| savedFilterAdd | `SavedFilter` | `src/modules/savedFilter/savedFilter.graphql` | `savedFilterAdd(input: SavedFilterAddInput!): SavedFilter @auth` |
| savedFilterDelete | `ID` | `src/modules/savedFilter/savedFilter.graphql` | `savedFilterDelete(id: ID!): ID @auth` |
| savedFilterFieldPatch | `SavedFilter` | `src/modules/savedFilter/savedFilter.graphql` | `savedFilterFieldPatch(id: ID!, input: [EditInput!]): SavedFilter @auth` |


#### securityCoverage
<a id="securitycoverage"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| securityCoverageAdd | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageAdd(input: SecurityCoverageAddInput!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageContextClean | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageContextClean(id: ID!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageContextPatch | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageContextPatch(id: ID!, input: EditContext!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageDelete | `ID` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| securityCoverageFieldPatch | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageRelationAdd | `StixRefRelationship` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageRelationDelete | `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` | `securityCoverageRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### securityPlatform
<a id="securityplatform"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| securityPlatformAdd | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformAdd(input: SecurityPlatformAddInput!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformContextClean | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformContextClean(id: ID!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformContextPatch | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformContextPatch(id: ID!, input: EditContext!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformDelete | `ID` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| securityPlatformFieldPatch | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformRelationAdd | `StixRefRelationship` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformRelationDelete | `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` | `securityPlatformRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### support
<a id="support"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| supportPackageAdd | `SupportPackage` | `src/modules/support/support.graphql` | `supportPackageAdd(input: SupportPackageAddInput!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |
| supportPackageDelete | `ID` | `src/modules/support/support.graphql` | `supportPackageDelete(id: ID!): ID @auth(for: [SETTINGS_SUPPORT])` |
| supportPackageForceZip | `SupportPackage` | `src/modules/support/support.graphql` | `supportPackageForceZip(input: SupportPackageForceZipInput!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |


#### task
<a id="task"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| taskAdd | `Task` | `src/modules/task/task.graphql` | `taskAdd(input: TaskAddInput!): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskDelete | `ID` | `src/modules/task/task.graphql` | `taskDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| taskFieldPatch | `Task` | `src/modules/task/task.graphql` | `taskFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskRelationAdd | `StixRefRelationship` | `src/modules/task/task.graphql` | `taskRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskRelationDelete | `Task` | `src/modules/task/task.graphql` | `taskRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskTemplateAdd | `TaskTemplate` | `src/modules/task/task-template/task-template.graphql` | `taskTemplateAdd(input: TaskTemplateAddInput!): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplateDelete | `ID` | `src/modules/task/task-template/task-template.graphql` | `taskTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplateFieldPatch | `TaskTemplate` | `src/modules/task/task-template/task-template.graphql` | `taskTemplateFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |


#### theme
<a id="theme"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| themeAdd | `Theme` | `src/modules/theme/theme.graphql` | `themeAdd(input: ThemeAddInput!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |
| themeDelete | `ID` | `src/modules/theme/theme.graphql` | `themeDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| themeFieldPatch | `Theme` | `src/modules/theme/theme.graphql` | `themeFieldPatch(id: ID!,input: [EditInput!]!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |
| themeImport | `Theme` | `src/modules/theme/theme.graphql` | `themeImport(file: Upload!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### threatActorIndividual
<a id="threatactorindividual"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| threatActorIndividualAdd | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualAdd(input: ThreatActorIndividualAddInput!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualContextClean | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualContextClean(id: ID!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualContextPatch | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualContextPatch(id: ID!, input: EditContext): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualDelete | `ID` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| threatActorIndividualFieldPatch | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualRelationAdd | `StixRefRelationship` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualRelationDelete | `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` | `threatActorIndividualRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### vocabulary
<a id="vocabulary"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| vocabularyAdd | `Vocabulary` | `src/modules/vocabulary/vocabulary.graphql` | `vocabularyAdd(input: VocabularyAddInput!): Vocabulary @auth(for: [SETTINGS_SETVOCABULARIES])` |
| vocabularyDelete | `ID` | `src/modules/vocabulary/vocabulary.graphql` | `vocabularyDelete(id: ID!): ID @auth(for: [SETTINGS_SETVOCABULARIES])` |
| vocabularyFieldPatch | `Vocabulary` | `src/modules/vocabulary/vocabulary.graphql` | `vocabularyFieldPatch(id: ID!, input: [EditInput!]!): Vocabulary @auth(for: [SETTINGS_SETVOCABULARIES])` |


#### workspace
<a id="workspace"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| workspaceAdd | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceAdd(input: WorkspaceAddInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceConfigurationImport | `String!` | `src/modules/workspace/workspace.graphql` | `workspaceConfigurationImport(file: Upload!): String! @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceContextClean | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceContextClean(id: ID!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceContextPatch | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceContextPatch(id: ID!, input: EditContext!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceDelete | `ID` | `src/modules/workspace/workspace.graphql` | `workspaceDelete(id: ID!): ID @auth(for: [EXPLORE_EXUPDATE_EXDELETE, INVESTIGATION_INUPDATE_INDELETE])` |
| workspaceDuplicate | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceDuplicate(input: WorkspaceDuplicateInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceEditAuthorizedMembers | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceFieldPatch | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceFieldPatch(id: ID!, input: [EditInput!]!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceWidgetConfigurationImport | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspaceWidgetConfigurationImport(id: ID!, input: ImportConfigurationInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |


#### xtm
<a id="xtm"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| autoRegisterOpenCTI | `Success!` | `src/modules/xtm/hub/xtm-hub.graphql` | `autoRegisterOpenCTI(input: AutoRegisterInput!): Success! @auth(for: [SETTINGS_SETPARAMETERS])` |
| checkXTMHubConnectivity | `CheckXTMHubConnectivityResponse!` | `src/modules/xtm/hub/xtm-hub.graphql` | `checkXTMHubConnectivity: CheckXTMHubConnectivityResponse! @auth(for: [SETTINGS_SETPARAMETERS])` |
| contactUsXtmHub | `Success!` | `src/modules/xtm/hub/xtm-hub.graphql` | `contactUsXtmHub: Success! @auth` |


### Subscription API index
<a id="subscription-api-index"></a>


#### ai
<a id="ai"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| aiBus | `AIBus` | `src/modules/ai/ai.graphql` | `aiBus(id: ID!): AIBus @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### entitySetting
<a id="entitysetting"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| entitySetting | `EntitySetting` | `src/modules/entitySetting/entitySetting.graphql` | `entitySetting(id: ID!): EntitySetting @auth` |


#### managerConfiguration
<a id="managerconfiguration"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| managerConfiguration | `ManagerConfiguration` | `src/modules/managerConfiguration/managerConfiguration.graphql` | `managerConfiguration(id: ID!): ManagerConfiguration @auth` |


#### notification
<a id="notification"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| notification | `Notification` | `src/modules/notification/notification.graphql` | `notification: Notification @auth` |
| notificationsNumber | `NotificationCount` | `src/modules/notification/notification.graphql` | `notificationsNumber: NotificationCount @auth` |


#### workspace
<a id="workspace"></a>

| Field | Returns | File | Signature |
| --- | --- | --- | --- |
| workspace | `Workspace` | `src/modules/workspace/workspace.graphql` | `workspace(id: ID!): Workspace @auth(for: [EXPLORE, INVESTIGATION])` |


## Global definition index (alphabetical)
<a id="global-definition-index-alphabetical"></a>

| Name | Kind(s) | Family(s) | Definition count |
| --- | --- | --- | --- |
| `AdministrativeArea` | `type` | `administrativeArea` | 1 |
| `AdministrativeAreaAddInput` | `input` | `administrativeArea` | 1 |
| `AdministrativeAreaConnection` | `type` | `administrativeArea` | 1 |
| `AdministrativeAreaEdge` | `type` | `administrativeArea` | 1 |
| `AdministrativeAreasOrdering` | `enum` | `administrativeArea` | 1 |
| `AIBus` | `type` | `ai` | 1 |
| `AskSendOtpInput` | `input` | `auth` | 1 |
| `AttributeBasedOn` | `type` | `internal` | 1 |
| `AttributeBasedOnInput` | `input` | `internal` | 1 |
| `AttributeColumn` | `type` | `internal` | 1 |
| `AttributeColumnConfiguration` | `type` | `internal` | 1 |
| `AttributeColumnConfigurationInput` | `input` | `internal` | 1 |
| `AttributeColumnInput` | `input` | `internal` | 1 |
| `AttributePath` | `type` | `internal` | 1 |
| `AttributeRef` | `type` | `internal` | 1 |
| `AttributeRefInput` | `input` | `internal` | 1 |
| `AutoRegisterInput` | `input` | `xtm` | 1 |
| `Case` | `interface` | `case` | 1 |
| `CaseConnection` | `type` | `case` | 1 |
| `CaseEdge` | `type` | `case` | 1 |
| `CaseIncident` | `type` | `case` | 1 |
| `CaseIncidentAddInput` | `input` | `case` | 1 |
| `CaseIncidentConnection` | `type` | `case` | 1 |
| `CaseIncidentEdge` | `type` | `case` | 1 |
| `CaseIncidentsOrdering` | `enum` | `case` | 1 |
| `CaseRfi` | `type` | `case` | 1 |
| `CaseRfiAddInput` | `input` | `case` | 1 |
| `CaseRfiConnection` | `type` | `case` | 1 |
| `CaseRfiEdge` | `type` | `case` | 1 |
| `CaseRfisOrdering` | `enum` | `case` | 1 |
| `CaseRft` | `type` | `case` | 1 |
| `CaseRftAddInput` | `input` | `case` | 1 |
| `CaseRftConnection` | `type` | `case` | 1 |
| `CaseRftEdge` | `type` | `case` | 1 |
| `CaseRftsOrdering` | `enum` | `case` | 1 |
| `CasesOrdering` | `enum` | `case` | 1 |
| `CaseTemplate` | `type` | `case` | 1 |
| `CaseTemplateAddInput` | `input` | `case` | 1 |
| `CaseTemplateConnection` | `type` | `case` | 1 |
| `CaseTemplateEdge` | `type` | `case` | 1 |
| `CaseTemplatesOrdering` | `enum` | `case` | 1 |
| `Catalog` | `type` | `catalog` | 1 |
| `CatalogConnection` | `type` | `catalog` | 1 |
| `CatalogEdge` | `type` | `catalog` | 1 |
| `CatalogsOrdering` | `enum` | `catalog` | 1 |
| `ChangePasswordInput` | `input` | `auth` | 1 |
| `Channel` | `type` | `channel` | 1 |
| `ChannelAddInput` | `input` | `channel` | 1 |
| `ChannelConnection` | `type` | `channel` | 1 |
| `ChannelEdge` | `type` | `channel` | 1 |
| `ChannelsOrdering` | `enum` | `channel` | 1 |
| `CheckXTMHubConnectivityResponse` | `type` | `xtm` | 1 |
| `ComplexPath` | `type` | `internal` | 1 |
| `ComplexVariable` | `type` | `internal` | 1 |
| `CoverageResult` | `type` | `securityCoverage` | 1 |
| `CSVFeedAddInputFromImport` | `type` | `ingestion` | 1 |
| `CsvMapper` | `type` | `internal` | 1 |
| `CsvMapperAddInput` | `input` | `internal` | 1 |
| `CsvMapperAddInputFromImport` | `type` | `internal` | 1 |
| `CsvMapperConnection` | `type` | `internal` | 1 |
| `CsvMapperEdge` | `type` | `internal` | 1 |
| `CsvMapperOperator` | `enum` | `internal` | 1 |
| `CsvMapperOrdering` | `enum` | `internal` | 1 |
| `CsvMapperRepresentation` | `type` | `internal` | 1 |
| `CsvMapperRepresentationAttribute` | `type` | `internal` | 1 |
| `CsvMapperRepresentationAttributeInput` | `input` | `internal` | 1 |
| `CsvMapperRepresentationInput` | `input` | `internal` | 1 |
| `CsvMapperRepresentationTarget` | `type` | `internal` | 1 |
| `CsvMapperRepresentationTargetColumn` | `type` | `internal` | 1 |
| `CsvMapperRepresentationTargetColumnInput` | `input` | `internal` | 1 |
| `CsvMapperRepresentationTargetInput` | `input` | `internal` | 1 |
| `CsvMapperRepresentationType` | `enum` | `internal` | 1 |
| `CsvMapperSchemaAttribute` | `type` | `internal` | 1 |
| `CsvMapperSchemaAttributes` | `type` | `internal` | 1 |
| `CsvMapperTestResult` | `type` | `internal` | 1 |
| `DataComponent` | `type` | `dataComponent` | 1 |
| `DataComponentAddInput` | `input` | `dataComponent` | 1 |
| `DataComponentConnection` | `type` | `dataComponent` | 1 |
| `DataComponentEdge` | `type` | `dataComponent` | 1 |
| `DataComponentsOrdering` | `enum` | `dataComponent` | 1 |
| `DataSource` | `type` | `dataSource` | 1 |
| `DataSourceAddInput` | `input` | `dataSource` | 1 |
| `DataSourceConnection` | `type` | `dataSource` | 1 |
| `DataSourceEdge` | `type` | `dataSource` | 1 |
| `DataSourcesOrdering` | `enum` | `dataSource` | 1 |
| `DecayChartData` | `type` | `indicator` | 1 |
| `DecayData` | `type` | `decayRule` | 1 |
| `DecayExclusionRule` | `type` | `decayRule` | 1 |
| `DecayExclusionRuleAddInput` | `input` | `decayRule` | 1 |
| `DecayExclusionRuleConnection` | `type` | `decayRule` | 1 |
| `DecayExclusionRuleEdge` | `type` | `decayRule` | 1 |
| `DecayExclusionRuleOrdering` | `enum` | `decayRule` | 1 |
| `DecayHistory` | `type` | `indicator` | 1 |
| `DecayLiveDetails` | `type` | `indicator` | 1 |
| `DecayRule` | `type` | `decayRule` | 1 |
| `DecayRuleAddInput` | `input` | `decayRule` | 1 |
| `DecayRuleConnection` | `type` | `decayRule` | 1 |
| `DecayRuleEdge` | `type` | `decayRule` | 1 |
| `DecayRuleOrdering` | `enum` | `decayRule` | 1 |
| `DefaultValue` | `type` | `entitySetting` | 1 |
| `DefaultValueAttribute` | `type` | `entitySetting` | 1 |
| `DeletedElement` | `type` | `deleteOperation` | 1 |
| `DeleteOperation` | `type` | `deleteOperation` | 1 |
| `DeleteOperationConnection` | `type` | `deleteOperation` | 1 |
| `DeleteOperationEdge` | `type` | `deleteOperation` | 1 |
| `DeleteOperationOrdering` | `enum` | `deleteOperation` | 1 |
| `DigestPeriod` | `enum` | `notification` | 1 |
| `DisseminationList` | `type` | `disseminationList` | 1 |
| `DisseminationListAddInput` | `input` | `disseminationList` | 1 |
| `DisseminationListConnection` | `type` | `disseminationList` | 1 |
| `DisseminationListEdge` | `type` | `disseminationList` | 1 |
| `DisseminationListOrdering` | `enum` | `disseminationList` | 1 |
| `DisseminationListSendInput` | `input` | `disseminationList` | 1 |
| `DraftObjectsCount` | `type` | `draftWorkspace` | 1 |
| `DraftStatus` | `enum` | `draftWorkspace` | 1 |
| `DraftWorkspace` | `type` | `draftWorkspace` | 1 |
| `DraftWorkspaceAddInput` | `input` | `draftWorkspace` | 1 |
| `DraftWorkspaceConnection` | `type` | `draftWorkspace` | 1 |
| `DraftWorkspaceEdge` | `type` | `draftWorkspace` | 1 |
| `DraftWorkspacesOrdering` | `enum` | `draftWorkspace` | 1 |
| `EmailTemplate` | `type` | `emailTemplate` | 1 |
| `EmailTemplateAddInput` | `input` | `emailTemplate` | 1 |
| `EmailTemplateConnection` | `type` | `emailTemplate` | 1 |
| `EmailTemplateEdge` | `type` | `emailTemplate` | 1 |
| `EmailTemplateOrdering` | `enum` | `emailTemplate` | 1 |
| `EntitySetting` | `type` | `entitySetting` | 1 |
| `EntitySettingConnection` | `type` | `entitySetting` | 1 |
| `EntitySettingEdge` | `type` | `entitySetting` | 1 |
| `EntitySettingsOrdering` | `enum` | `entitySetting` | 1 |
| `Event` | `type` | `event` | 1 |
| `EventAddInput` | `input` | `event` | 1 |
| `EventConnection` | `type` | `event` | 1 |
| `EventEdge` | `type` | `event` | 1 |
| `EventsOrdering` | `enum` | `event` | 1 |
| `ExclusionList` | `type` | `exclusionList` | 1 |
| `ExclusionListCacheStatus` | `type` | `exclusionList` | 1 |
| `ExclusionListConnection` | `type` | `exclusionList` | 1 |
| `ExclusionListEdge` | `type` | `exclusionList` | 1 |
| `ExclusionListFileAddInput` | `input` | `exclusionList` | 1 |
| `ExclusionListOrdering` | `enum` | `exclusionList` | 1 |
| `ExtendedContract` | `type` | `catalog` | 1 |
| `Feedback` | `type` | `case` | 1 |
| `FeedbackAddInput` | `input` | `case` | 1 |
| `FeedbackConnection` | `type` | `case` | 1 |
| `FeedbackEdge` | `type` | `case` | 1 |
| `FeedbacksOrdering` | `enum` | `case` | 1 |
| `FintelDesign` | `type` | `fintelDesign` | 1 |
| `FintelDesignAddInput` | `input` | `fintelDesign` | 1 |
| `FintelDesignConnection` | `type` | `fintelDesign` | 1 |
| `FintelDesignEdge` | `type` | `fintelDesign` | 1 |
| `FintelDesignOrdering` | `enum` | `fintelDesign` | 1 |
| `FintelTemplate` | `type` | `fintelTemplate` | 1 |
| `FintelTemplateAddInput` | `input` | `fintelTemplate` | 1 |
| `FintelTemplateConnection` | `type` | `fintelTemplate` | 1 |
| `FintelTemplateEdge` | `type` | `fintelTemplate` | 1 |
| `FintelTemplateOrdering` | `enum` | `fintelTemplate` | 1 |
| `FintelTemplateWidget` | `type` | `fintelTemplate` | 1 |
| `FintelTemplateWidgetAddInput` | `input` | `fintelTemplate` | 1 |
| `Form` | `type` | `form` | 1 |
| `FormAddInput` | `input` | `form` | 1 |
| `Format` | `enum` | `ai` | 1 |
| `FormConnection` | `type` | `form` | 1 |
| `FormEdge` | `type` | `form` | 1 |
| `FormsOrdering` | `enum` | `form` | 1 |
| `FormSubmissionInput` | `input` | `form` | 1 |
| `FormSubmissionResponse` | `type` | `form` | 1 |
| `Grouping` | `type` | `grouping` | 1 |
| `GroupingAddInput` | `input` | `grouping` | 1 |
| `GroupingConnection` | `type` | `grouping` | 1 |
| `GroupingEdge` | `type` | `grouping` | 1 |
| `GroupingsOrdering` | `enum` | `grouping` | 1 |
| `HeaderInput` | `input` | `ingestion` | 1 |
| `ImportConfigurationInput` | `input` | `workspace` | 1 |
| `ImportWidgetInput` | `input` | `workspace` | 1 |
| `Indicator` | `type` | `indicator` | 1 |
| `IndicatorAddInput` | `input` | `indicator` | 1 |
| `IndicatorConnection` | `type` | `indicator` | 1 |
| `IndicatorDecayExclusionRule` | `type` | `indicator` | 1 |
| `IndicatorDecayRule` | `type` | `indicator` | 1 |
| `IndicatorEdge` | `type` | `indicator` | 1 |
| `IndicatorFormat` | `enum` | `ai` | 1 |
| `IndicatorsOrdering` | `enum` | `indicator` | 1 |
| `IngestionAuthType` | `enum` | `ingestion` | 1 |
| `IngestionCsv` | `type` | `ingestion` | 1 |
| `IngestionCsvAddAutoUserInput` | `input` | `ingestion` | 1 |
| `IngestionCsvAddInput` | `input` | `ingestion` | 1 |
| `IngestionCsvConnection` | `type` | `ingestion` | 1 |
| `IngestionCsvEdge` | `type` | `ingestion` | 1 |
| `IngestionCsvMapperType` | `enum` | `ingestion` | 1 |
| `IngestionCsvOrdering` | `enum` | `ingestion` | 1 |
| `IngestionHeader` | `type` | `ingestion` | 1 |
| `IngestionJson` | `type` | `ingestion` | 1 |
| `IngestionJsonAddInput` | `input` | `ingestion` | 1 |
| `IngestionJsonConnection` | `type` | `ingestion` | 1 |
| `IngestionJsonEdge` | `type` | `ingestion` | 1 |
| `IngestionJsonOrdering` | `enum` | `ingestion` | 1 |
| `IngestionQueryAttribute` | `type` | `ingestion` | 1 |
| `IngestionRss` | `type` | `ingestion` | 1 |
| `IngestionRssAddInput` | `input` | `ingestion` | 1 |
| `IngestionRssConnection` | `type` | `ingestion` | 1 |
| `IngestionRssEdge` | `type` | `ingestion` | 1 |
| `IngestionRssOrdering` | `enum` | `ingestion` | 1 |
| `IngestionTaxii` | `type` | `ingestion` | 1 |
| `IngestionTaxiiAddAutoUserInput` | `input` | `ingestion` | 1 |
| `IngestionTaxiiAddInput` | `input` | `ingestion` | 1 |
| `IngestionTaxiiCollection` | `type` | `ingestion` | 1 |
| `IngestionTaxiiCollectionAddInput` | `input` | `ingestion` | 1 |
| `IngestionTaxiiCollectionConnection` | `type` | `ingestion` | 1 |
| `IngestionTaxiiCollectionEdge` | `type` | `ingestion` | 1 |
| `IngestionTaxiiCollectionOrdering` | `enum` | `ingestion` | 1 |
| `IngestionTaxiiConnection` | `type` | `ingestion` | 1 |
| `IngestionTaxiiEdge` | `type` | `ingestion` | 1 |
| `IngestionTaxiiOrdering` | `enum` | `ingestion` | 1 |
| `JsonAttributeColumnConfiguration` | `interface` | `internal` | 1 |
| `JsonComplexPath` | `type` | `internal` | 1 |
| `JsonComplexPathConfiguration` | `type` | `internal` | 1 |
| `JsonComplexPathVariable` | `type` | `internal` | 1 |
| `JsonMapper` | `type` | `internal` | 1 |
| `JsonMapperAddInput` | `input` | `internal` | 1 |
| `JsonMapperAddInputFromImport` | `type` | `internal` | 1 |
| `JsonMapperConnection` | `type` | `internal` | 1 |
| `JsonMapperEdge` | `type` | `internal` | 1 |
| `JsonMapperOrdering` | `enum` | `internal` | 1 |
| `JsonMapperRepresentation` | `type` | `internal` | 1 |
| `JsonMapperRepresentationAttribute` | `type` | `internal` | 1 |
| `JsonMapperRepresentationTarget` | `type` | `internal` | 1 |
| `JsonMapperRepresentationType` | `enum` | `internal` | 1 |
| `JsonMapperTestResult` | `type` | `internal` | 1 |
| `JsonMapperVariable` | `type` | `internal` | 1 |
| `Language` | `type` | `language` | 1 |
| `LanguageAddInput` | `input` | `language` | 1 |
| `LanguageConnection` | `type` | `language` | 1 |
| `LanguageEdge` | `type` | `language` | 1 |
| `LanguagesOrdering` | `enum` | `language` | 1 |
| `MalwareAnalysesOrdering` | `enum` | `malwareAnalysis` | 1 |
| `MalwareAnalysis` | `type` | `malwareAnalysis` | 1 |
| `MalwareAnalysisAddInput` | `input` | `malwareAnalysis` | 1 |
| `MalwareAnalysisConnection` | `type` | `malwareAnalysis` | 1 |
| `MalwareAnalysisEdge` | `type` | `malwareAnalysis` | 1 |
| `ManagerConfiguration` | `type` | `managerConfiguration` | 1 |
| `Measure` | `type` | `threatActorIndividual` | 1 |
| `MeasureInput` | `input` | `threatActorIndividual` | 1 |
| `MeOrganization` | `type` | `organization` | 1 |
| `MeOrganizationConnection` | `type` | `organization` | 1 |
| `MeOrganizationEdge` | `type` | `organization` | 1 |
| `Metric` | `type` | `metrics` | 1 |
| `Narrative` | `type` | `narrative` | 1 |
| `NarrativeAddInput` | `input` | `narrative` | 1 |
| `NarrativeConnection` | `type` | `narrative` | 1 |
| `NarrativeEdge` | `type` | `narrative` | 1 |
| `NarrativesOrdering` | `enum` | `narrative` | 1 |
| `NLQResponse` | `type` | `ai` | 1 |
| `Notification` | `type` | `notification` | 1 |
| `NotificationConnection` | `type` | `notification` | 1 |
| `NotificationContent` | `type` | `notification` | 1 |
| `NotificationCount` | `type` | `notification` | 1 |
| `NotificationEdge` | `type` | `notification` | 1 |
| `NotificationEvent` | `type` | `notification` | 1 |
| `NotificationsOrdering` | `enum` | `notification` | 1 |
| `Notifier` | `type` | `notifier` | 1 |
| `NotifierAddInput` | `input` | `notifier` | 1 |
| `NotifierConnection` | `type` | `notifier` | 1 |
| `NotifierConnector` | `type` | `notifier` | 1 |
| `NotifierEdge` | `type` | `notifier` | 1 |
| `NotifierOrdering` | `enum` | `notifier` | 1 |
| `NotifierParameter` | `type` | `notifier` | 1 |
| `NotifierTestInput` | `input` | `notifier` | 1 |
| `ObservablesValues` | `type` | `indicator` | 1 |
| `Organization` | `type` | `organization` | 1 |
| `OrganizationAddInput` | `input` | `organization` | 1 |
| `OrganizationConnection` | `type` | `organization` | 1 |
| `OrganizationEdge` | `type` | `organization` | 1 |
| `OrganizationsOrdering` | `enum` | `organization` | 1 |
| `OverviewWidgetCustomization` | `type` | `entitySetting` | 1 |
| `PackageStatus` | `enum` | `support` | 1 |
| `PatchMetricInput` | `input` | `metrics` | 1 |
| `Pir` | `type` | `pir` | 1 |
| `PirAddInput` | `input` | `pir` | 1 |
| `PirConnection` | `type` | `pir` | 1 |
| `PirCriterion` | `type` | `pir` | 1 |
| `PirCriterionInput` | `input` | `pir` | 1 |
| `PirDependency` | `type` | `pir` | 1 |
| `PirDependencyInput` | `input` | `pir` | 1 |
| `PirEdge` | `type` | `pir` | 1 |
| `PirExplanation` | `type` | `pir` | 1 |
| `PirExplanationInput` | `input` | `pir` | 1 |
| `PirFlagElementInput` | `input` | `pir` | 1 |
| `PirInformation` | `type` | `pir` | 1 |
| `PirOrdering` | `enum` | `pir` | 1 |
| `PirRelationship` | `type` | `pir` | 1 |
| `PirRelationshipConnection` | `type` | `pir` | 1 |
| `PirRelationshipEdge` | `type` | `pir` | 1 |
| `PirRelationshipOrdering` | `enum` | `pir` | 1 |
| `PirRelationshipsTimeSeriesParameters` | `input` | `pir` | 1 |
| `PirScore` | `type` | `pir` | 1 |
| `PirType` | `enum` | `pir` | 1 |
| `PirUnflagElementInput` | `input` | `pir` | 1 |
| `Playbook` | `type` | `playbook` | 1 |
| `PlaybookAddInput` | `input` | `playbook` | 1 |
| `PlaybookAddLinkInput` | `input` | `playbook` | 1 |
| `PlaybookAddNodeInput` | `input` | `playbook` | 1 |
| `PlaybookComponent` | `type` | `playbook` | 1 |
| `PlaybookComponentPort` | `type` | `playbook` | 1 |
| `PlaybookConnection` | `type` | `playbook` | 1 |
| `PlaybookEdge` | `type` | `playbook` | 1 |
| `PlayBookExecution` | `type` | `playbook` | 1 |
| `PlayBookExecutionStep` | `type` | `playbook` | 1 |
| `PlaybookInsertResult` | `type` | `playbook` | 1 |
| `PlaybooksOrdering` | `enum` | `playbook` | 1 |
| `PositionInput` | `input` | `playbook` | 1 |
| `PublicDashboard` | `type` | `publicDashboard` | 1 |
| `PublicDashboardAddInput` | `input` | `publicDashboard` | 1 |
| `PublicDashboardConnection` | `type` | `publicDashboard` | 1 |
| `PublicDashboardEdge` | `type` | `publicDashboard` | 1 |
| `PublicDashboardsOrdering` | `enum` | `publicDashboard` | 1 |
| `PublicDistribution` | `type` | `publicDashboard` | 1 |
| `QueryAttribute` | `input` | `ingestion` | 1 |
| `RequestAccessAddInput` | `input` | `requestAccess` | 1 |
| `RequestAccessConfiguration` | `type` | `requestAccess` | 1 |
| `RequestAccessConfigureInput` | `input` | `requestAccess` | 1 |
| `RequestAccessMember` | `type` | `requestAccess` | 1 |
| `RequestAccessStatus` | `type` | `requestAccess` | 1 |
| `RequestAccessType` | `enum` | `requestAccess` | 1 |
| `RequestAccessWorkflow` | `type` | `requestAccess` | 1 |
| `ResolvedInstanceFilter` | `type` | `notification` | 1 |
| `RfiRequestAccessConfiguration` | `type` | `case` | 1 |
| `SavedFilter` | `type` | `savedFilter` | 1 |
| `SavedFilterAddInput` | `input` | `savedFilter` | 1 |
| `SavedFilterConnection` | `type` | `savedFilter` | 1 |
| `SavedFilterEdge` | `type` | `savedFilter` | 1 |
| `SavedFilterOrdering` | `enum` | `savedFilter` | 1 |
| `ScaleAttribute` | `type` | `entitySetting` | 1 |
| `SecurityCoverage` | `type` | `securityCoverage` | 1 |
| `SecurityCoverageAddInput` | `input` | `securityCoverage` | 1 |
| `SecurityCoverageConnection` | `type` | `securityCoverage` | 1 |
| `SecurityCoverageEdge` | `type` | `securityCoverage` | 1 |
| `SecurityCoverageOrdering` | `enum` | `securityCoverage` | 1 |
| `SecurityPlatform` | `type` | `securityPlatform` | 1 |
| `SecurityPlatformAddInput` | `input` | `securityPlatform` | 1 |
| `SecurityPlatformConnection` | `type` | `securityPlatform` | 1 |
| `SecurityPlatformEdge` | `type` | `securityPlatform` | 1 |
| `SecurityPlatformOrdering` | `enum` | `securityPlatform` | 1 |
| `StixCyberObservableEditMutations` | `type` | `stixCyberObservable` | 1 |
| `Success` | `type` | `xtm` | 1 |
| `SupportPackage` | `type` | `support` | 1 |
| `SupportPackageAddInput` | `input` | `support` | 1 |
| `SupportPackageConnection` | `type` | `support` | 1 |
| `SupportPackageEdge` | `type` | `support` | 1 |
| `SupportPackageForceZipInput` | `input` | `support` | 1 |
| `SupportPackageOrdering` | `enum` | `support` | 1 |
| `Task` | `type` | `task` | 1 |
| `TaskAddInput` | `input` | `task` | 1 |
| `TaskConnection` | `type` | `task` | 1 |
| `TaskEdge` | `type` | `task` | 1 |
| `TasksOrdering` | `enum` | `task` | 1 |
| `TaskTemplate` | `type` | `task` | 1 |
| `TaskTemplateAddInput` | `input` | `task` | 1 |
| `TaskTemplateConnection` | `type` | `task` | 1 |
| `TaskTemplateEdge` | `type` | `task` | 1 |
| `TaskTemplatesOrdering` | `enum` | `task` | 1 |
| `TaxiiFeedAddInputFromImport` | `type` | `ingestion` | 1 |
| `TaxiiVersion` | `enum` | `ingestion` | 1 |
| `Theme` | `type` | `theme` | 1 |
| `ThemeAddInput` | `input` | `theme` | 1 |
| `ThemeConnection` | `type` | `theme` | 1 |
| `ThemeEdge` | `type` | `theme` | 1 |
| `ThemeOrdering` | `enum` | `theme` | 1 |
| `ThreatActorIndividual` | `type` | `threatActorIndividual` | 1 |
| `ThreatActorIndividualAddInput` | `input` | `threatActorIndividual` | 1 |
| `ThreatActorIndividualConnection` | `type` | `threatActorIndividual` | 1 |
| `ThreatActorIndividualEdge` | `type` | `threatActorIndividual` | 1 |
| `ThreatActorsIndividualOrdering` | `enum` | `threatActorIndividual` | 1 |
| `Tone` | `enum` | `ai` | 1 |
| `Trigger` | `type` | `notification` | 1 |
| `TriggerActivityDigestAddInput` | `input` | `notification` | 1 |
| `TriggerActivityEventType` | `enum` | `notification` | 1 |
| `TriggerActivityLiveAddInput` | `input` | `notification` | 1 |
| `TriggerConnection` | `type` | `notification` | 1 |
| `TriggerDigestAddInput` | `input` | `notification` | 1 |
| `TriggerEdge` | `type` | `notification` | 1 |
| `TriggerEventType` | `enum` | `notification` | 1 |
| `TriggerLiveAddInput` | `input` | `notification` | 1 |
| `TriggersOrdering` | `enum` | `notification` | 1 |
| `TriggerType` | `enum` | `notification` | 1 |
| `TypeAttribute` | `type` | `entitySetting` | 1 |
| `VerifyMfaInput` | `input` | `auth` | 1 |
| `VerifyOtp` | `type` | `auth` | 1 |
| `VerifyOtpInput` | `input` | `auth` | 1 |
| `Vocabulary` | `type` | `vocabulary` | 1 |
| `VocabularyAddInput` | `input` | `vocabulary` | 1 |
| `VocabularyCategory` | `enum` | `vocabulary` | 1 |
| `VocabularyConnection` | `type` | `vocabulary` | 1 |
| `VocabularyDefinition` | `type` | `vocabulary` | 1 |
| `VocabularyEdge` | `type` | `vocabulary` | 1 |
| `VocabularyFieldDefinition` | `type` | `vocabulary` | 1 |
| `VocabularyOrdering` | `enum` | `vocabulary` | 1 |
| `Workspace` | `type` | `workspace` | 1 |
| `WorkspaceAddInput` | `input` | `workspace` | 1 |
| `WorkspaceConnection` | `type` | `workspace` | 1 |
| `WorkspaceDuplicateInput` | `input` | `workspace` | 1 |
| `WorkspaceEdge` | `type` | `workspace` | 1 |
| `WorkspacesOrdering` | `enum` | `workspace` | 1 |

> Tip: use your renderer’s search on this index (Ctrl/Cmd+F) to jump quickly.


## Module families
<a id="module-families"></a>


## Module family: administrativeArea
<a id="module-family-administrativearea"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/administrativeArea/administrativeArea.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| administrativeArea | `AdministrativeArea` | `administrativeArea(id: String!): AdministrativeArea @auth(for: [KNOWLEDGE])` |
| administrativeAreas | `AdministrativeAreaConnection` | `administrativeAreas( first: Int after: ID orderBy: AdministrativeAreasOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): AdministrativeAreaConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| administrativeAreaAdd | `AdministrativeArea` | `administrativeAreaAdd(input: AdministrativeAreaAddInput!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaContextClean | `AdministrativeArea` | `administrativeAreaContextClean(id: ID!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaContextPatch | `AdministrativeArea` | `administrativeAreaContextPatch(id: ID!, input: EditContext!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaDelete | `ID` | `administrativeAreaDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| administrativeAreaFieldPatch | `AdministrativeArea` | `administrativeAreaFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaRelationAdd | `StixRefRelationship` | `administrativeAreaRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| administrativeAreaRelationDelete | `AdministrativeArea` | `administrativeAreaRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): AdministrativeArea @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `AdministrativeAreasOrdering` | `src/modules/administrativeArea/administrativeArea.graphql` |

<details><summary>Show definition details</summary>


**enum AdministrativeAreasOrdering**  
Defined in `src/modules/administrativeArea/administrativeArea.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `objectMarking`
- `objectLabel`
- `x_opencti_workflow_id`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `AdministrativeAreaAddInput` | `src/modules/administrativeArea/administrativeArea.graphql` |

<details><summary>Show definition details</summary>


**input AdministrativeAreaAddInput**  
Defined in `src/modules/administrativeArea/administrativeArea.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `latitude` | `Float` |
| `longitude` | `Float` |
| `x_opencti_aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `AdministrativeArea` | `src/modules/administrativeArea/administrativeArea.graphql` |
| `AdministrativeAreaConnection` | `src/modules/administrativeArea/administrativeArea.graphql` |
| `AdministrativeAreaEdge` | `src/modules/administrativeArea/administrativeArea.graphql` |

<details><summary>Show definition details</summary>


**type AdministrativeArea**  
Defined in `src/modules/administrativeArea/administrativeArea.graphql`

_Referenced by:_
- `type` `AdministrativeAreaEdge` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `description` | `String` |
| `latitude` | `Float` |
| `longitude` | `Float` |
| `precision` | `Float` |
| `x_opencti_aliases` | `[String]` |
| `cases(first: Int)` | `CaseConnection` |
| `country` | `Country` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type AdministrativeAreaConnection**  
Defined in `src/modules/administrativeArea/administrativeArea.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[AdministrativeAreaEdge!]` |


**type AdministrativeAreaEdge**  
Defined in `src/modules/administrativeArea/administrativeArea.graphql`

_Referenced by:_
- `type` `AdministrativeAreaConnection` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `AdministrativeArea!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **150**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `AdministrativeAreasOrdering`
- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `AdministrativeAreaAddInput`
- `EditContext`
- `EditInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (122)
<a id="expanded-type-122"></a>

<details><summary>Show names</summary>

- `AdministrativeArea`
- `AdministrativeAreaConnection`
- `AdministrativeAreaEdge`
- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `Country`
- `CountryConnection`
- `CountryEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Region`
- `RegionConnection`
- `RegionEdge`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: ai
<a id="module-family-ai"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 14 |
| Subscription fields | 1 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/ai/ai.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (14)
<a id="mutation-14"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| aiChangeTone | `String` | `aiChangeTone(id: ID!, content: String!, format: Format, tone: Tone): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiContainerGenerateReport | `String` | `aiContainerGenerateReport(id: ID!, containerId: String!, paragraphs: Int, tone: Tone, format: Format, language: String): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiConvertFilesToStix | `String` | `aiConvertFilesToStix(id: ID!, elementId: String!, fileIds: [String]): String @auth(for: [KNOWLEDGE_KNUPDATE]) # Indicators` |
| aiConvertIndicator | `String` | `aiConvertIndicator(id: ID!, indicatorId: String!, format: IndicatorFormat!): String @auth(for: [KNOWLEDGE_KNUPDATE]) # Generic text` |
| aiExplain | `String` | `aiExplain(id: ID!, content: String!): String @auth(for: [KNOWLEDGE])` |
| aiFixSpelling | `String` | `aiFixSpelling(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiImproveWriting | `String` | `aiImproveWriting(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiMakeLonger | `String` | `aiMakeLonger(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiMakeShorter | `String` | `aiMakeShorter(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiNLQ | `NLQResponse` | `aiNLQ(search: String!): NLQResponse @auth(for: [KNOWLEDGE])` |
| aiSummarize | `String` | `aiSummarize(id: ID!, content: String!, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiSummarizeFiles | `String` | `aiSummarizeFiles(id: ID!, elementId: String!, paragraphs: Int, tone: Tone, format: Format, language: String, fileIds: [String]): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiThreatGenerateReport | `String` | `aiThreatGenerateReport(id: ID!, threatId: String!, paragraphs: Int, tone: Tone, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |
| aiVictimGenerateReport | `String` | `aiVictimGenerateReport(id: ID!, victimId: String!, paragraphs: Int, tone: Tone, format: Format): String @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (1)
<a id="subscription-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| aiBus | `AIBus` | `aiBus(id: ID!): AIBus @auth(for: [KNOWLEDGE_KNUPDATE])` |


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (3)
<a id="enum-3"></a>

| Name | File |
| --- | --- |
| `Format` | `src/modules/ai/ai.graphql` |
| `IndicatorFormat` | `src/modules/ai/ai.graphql` |
| `Tone` | `src/modules/ai/ai.graphql` |

<details><summary>Show definition details</summary>


**enum Format**  
Defined in `src/modules/ai/ai.graphql`

Values:
- `text`
- `html`
- `markdown`
- `json`


**enum IndicatorFormat**  
Defined in `src/modules/ai/ai.graphql`

Values:
- `stix`
- `sigma`
- `yara`


**enum Tone**  
Defined in `src/modules/ai/ai.graphql`

Values:
- `tactical`
- `operational`
- `strategic`


</details>


#### type (2)
<a id="type-2"></a>

| Name | File |
| --- | --- |
| `AIBus` | `src/modules/ai/ai.graphql` |
| `NLQResponse` | `src/modules/ai/ai.graphql` |

<details><summary>Show definition details</summary>


**type AIBus**  
Defined in `src/modules/ai/ai.graphql`

| Field | Type |
| --- | --- |
| `bus_id` | `String!` |
| `content` | `String!` |


**type NLQResponse**  
Defined in `src/modules/ai/ai.graphql`

| Field | Type |
| --- | --- |
| `filters` | `String!` |
| `notResolvedValues` | `[String!]!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **8**
- Definition count: **5**


#### Expanded: enum (3)
<a id="expanded-enum-3"></a>

<details><summary>Show names</summary>

- `Format`
- `IndicatorFormat`
- `Tone`

</details>


#### Expanded: type (2)
<a id="expanded-type-2"></a>

<details><summary>Show names</summary>

- `AIBus`
- `NLQResponse`

</details>



## Module family: auth
<a id="module-family-auth"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/auth/auth.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| askSendOtp | `String` | `askSendOtp(input: AskSendOtpInput!): String @public @rateLimit(limit: 1, duration: 1)` |
| changePassword | `Boolean` | `changePassword(input: ChangePasswordInput!): Boolean @public @rateLimit(limit: 1, duration: 1)` |
| verifyMfa | `Boolean` | `verifyMfa(input: VerifyMfaInput!): Boolean @public @rateLimit(limit: 1, duration: 1)` |
| verifyOtp | `VerifyOtp` | `verifyOtp(input: VerifyOtpInput!): VerifyOtp @public @rateLimit(limit: 1, duration: 1)` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### input (4)
<a id="input-4"></a>

| Name | File |
| --- | --- |
| `AskSendOtpInput` | `src/modules/auth/auth.graphql` |
| `ChangePasswordInput` | `src/modules/auth/auth.graphql` |
| `VerifyMfaInput` | `src/modules/auth/auth.graphql` |
| `VerifyOtpInput` | `src/modules/auth/auth.graphql` |

<details><summary>Show definition details</summary>


**input AskSendOtpInput**  
Defined in `src/modules/auth/auth.graphql`

| Field | Type |
| --- | --- |
| `email` | `String!` |


**input ChangePasswordInput**  
Defined in `src/modules/auth/auth.graphql`

| Field | Type |
| --- | --- |
| `transactionId` | `String!` |
| `otp` | `String!` |
| `newPassword` | `String!` |


**input VerifyMfaInput**  
Defined in `src/modules/auth/auth.graphql`

| Field | Type |
| --- | --- |
| `code` | `String!` |
| `transactionId` | `String!` |


**input VerifyOtpInput**  
Defined in `src/modules/auth/auth.graphql`

| Field | Type |
| --- | --- |
| `otp` | `String!` |
| `transactionId` | `String!` |


</details>


#### type (1)
<a id="type-1"></a>

| Name | File |
| --- | --- |
| `VerifyOtp` | `src/modules/auth/auth.graphql` |

<details><summary>Show definition details</summary>


**type VerifyOtp**  
Defined in `src/modules/auth/auth.graphql`

| Field | Type |
| --- | --- |
| `mfa_activated` | `Boolean!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **7**
- Definition count: **5**


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `AskSendOtpInput`
- `ChangePasswordInput`
- `VerifyMfaInput`
- `VerifyOtpInput`

</details>


#### Expanded: type (1)
<a id="expanded-type-1"></a>

<details><summary>Show names</summary>

- `VerifyOtp`

</details>



## Module family: case
<a id="module-family-case"></a>

| Metric | Value |
| --- | --- |
| SDL files | 6 |
| Query fields | 16 |
| Mutation fields | 18 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/case/case-incident/case-incident.graphql` |
| `src/modules/case/case-rfi/case-rfi.graphql` |
| `src/modules/case/case-rft/case-rft.graphql` |
| `src/modules/case/case-template/case-template.graphql` |
| `src/modules/case/case.graphql` |
| `src/modules/case/feedback/feedback.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (16)
<a id="query-16"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| case | `Case` | `case(id: String!): Case @auth(for: [KNOWLEDGE])` |
| caseIncident | `CaseIncident` | `caseIncident(id: String!): CaseIncident @auth(for: [KNOWLEDGE])` |
| caseIncidentContainsStixObjectOrStixRelationship | `Boolean` | `caseIncidentContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseIncidents | `CaseIncidentConnection` | `caseIncidents( first: Int after: ID orderBy: CaseIncidentsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseIncidentConnection @auth(for: [KNOWLEDGE])` |
| caseRfi | `CaseRfi` | `caseRfi(id: String!): CaseRfi @auth(for: [KNOWLEDGE])` |
| caseRfiContainsStixObjectOrStixRelationship | `Boolean` | `caseRfiContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseRfis | `CaseRfiConnection` | `caseRfis( first: Int after: ID orderBy: CaseRfisOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseRfiConnection @auth(for: [KNOWLEDGE])` |
| caseRft | `CaseRft` | `caseRft(id: String!): CaseRft @auth(for: [KNOWLEDGE])` |
| caseRftContainsStixObjectOrStixRelationship | `Boolean` | `caseRftContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| caseRfts | `CaseRftConnection` | `caseRfts( first: Int after: ID orderBy: CaseRftsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseRftConnection @auth(for: [KNOWLEDGE])` |
| caseTemplate | `CaseTemplate` | `caseTemplate(id: String!): CaseTemplate @auth` |
| caseTemplates | `CaseTemplateConnection` | `caseTemplates( first: Int after: ID orderBy: CaseTemplatesOrdering orderMode: OrderingMode search: String ): CaseTemplateConnection @auth` |
| cases | `CaseConnection` | `cases( first: Int after: ID orderBy: CasesOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): CaseConnection @auth(for: [KNOWLEDGE])` |
| feedback | `Feedback` | `feedback(id: String!): Feedback @auth(for: [KNOWLEDGE])` |
| feedbackContainsStixObjectOrStixRelationship | `Boolean` | `feedbackContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| feedbacks | `FeedbackConnection` | `feedbacks( first: Int after: ID orderBy: FeedbacksOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): FeedbackConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (18)
<a id="mutation-18"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| caseDelete | `ID` | `caseDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseIncidentAdd | `CaseIncident` | `caseIncidentAdd(input: CaseIncidentAddInput!): CaseIncident @auth` |
| caseIncidentDelete | `ID` | `caseIncidentDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseRfiAdd | `CaseRfi` | `caseRfiAdd(input: CaseRfiAddInput!): CaseRfi @auth` |
| caseRfiApprove | `CaseRfi` | `caseRfiApprove(id: ID!): CaseRfi @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS, KNOWLEDGE_KNUPDATE_KNORGARESTRICT])` |
| caseRfiDecline | `CaseRfi` | `caseRfiDecline(id: ID!): CaseRfi @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS, KNOWLEDGE_KNUPDATE_KNORGARESTRICT])` |
| caseRfiDelete | `ID` | `caseRfiDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseRftAdd | `CaseRft` | `caseRftAdd(input: CaseRftAddInput!): CaseRft @auth` |
| caseRftDelete | `ID` | `caseRftDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| caseSetTemplate | `Case` | `caseSetTemplate(id: ID!, caseTemplatesId: [ID!]!): Case @auth(for: [KNOWLEDGE_KNUPDATE])` |
| caseTemplateAdd | `CaseTemplate` | `caseTemplateAdd(input: CaseTemplateAddInput!): CaseTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateDelete | `ID` | `caseTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateFieldPatch | `CaseTemplate` | `caseTemplateFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): CaseTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| caseTemplateRelationAdd | `CaseTemplate` | `caseTemplateRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): CaseTemplate @auth(for: [KNOWLEDGE_KNUPDATE])` |
| caseTemplateRelationDelete | `CaseTemplate` | `caseTemplateRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): CaseTemplate @auth(for: [KNOWLEDGE_KNUPDATE])` |
| feedbackAdd | `Feedback` | `feedbackAdd(input: FeedbackAddInput!): Feedback @auth` |
| feedbackDelete | `ID` | `feedbackDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| feedbackEditAuthorizedMembers | `Feedback` | `feedbackEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]): Feedback @auth(for: [KNOWLEDGE_KNUPDATE_KNMANAGEAUTHMEMBERS])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (6)
<a id="enum-6"></a>

| Name | File |
| --- | --- |
| `CaseIncidentsOrdering` | `src/modules/case/case-incident/case-incident.graphql` |
| `CaseRfisOrdering` | `src/modules/case/case-rfi/case-rfi.graphql` |
| `CaseRftsOrdering` | `src/modules/case/case-rft/case-rft.graphql` |
| `CaseTemplatesOrdering` | `src/modules/case/case-template/case-template.graphql` |
| `CasesOrdering` | `src/modules/case/case.graphql` |
| `FeedbacksOrdering` | `src/modules/case/feedback/feedback.graphql` |

<details><summary>Show definition details</summary>


**enum CaseIncidentsOrdering**  
Defined in `src/modules/case/case-incident/case-incident.graphql`

Values:
- `name`
- `created`
- `modified`
- `context`
- `severity`
- `priority`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `objectAssignee`
- `x_opencti_workflow_id`
- `confidence`
- `objectMarking`
- `_score`


**enum CaseRfisOrdering**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `objectAssignee`
- `x_opencti_workflow_id`
- `confidence`
- `objectMarking`
- `severity`
- `priority`
- `_score`


**enum CaseRftsOrdering**  
Defined in `src/modules/case/case-rft/case-rft.graphql`

Values:
- `name`
- `created`
- `modified`
- `context`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `objectAssignee`
- `x_opencti_workflow_id`
- `confidence`
- `objectMarking`
- `severity`
- `priority`
- `_score`


**enum CaseTemplatesOrdering**  
Defined in `src/modules/case/case-template/case-template.graphql`

Values:
- `name`
- `description`
- `created`
- `_score`


**enum CasesOrdering**  
Defined in `src/modules/case/case.graphql`

Values:
- `name`
- `created`
- `modified`
- `context`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `x_opencti_workflow_id`
- `confidence`
- `objectMarking`
- `_score`


**enum FeedbacksOrdering**  
Defined in `src/modules/case/feedback/feedback.graphql`

Values:
- `name`
- `created`
- `modified`
- `context`
- `rating`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `x_opencti_workflow_id`
- `confidence`
- `objectMarking`
- `_score`


</details>


#### input (5)
<a id="input-5"></a>

| Name | File |
| --- | --- |
| `CaseIncidentAddInput` | `src/modules/case/case-incident/case-incident.graphql` |
| `CaseRfiAddInput` | `src/modules/case/case-rfi/case-rfi.graphql` |
| `CaseRftAddInput` | `src/modules/case/case-rft/case-rft.graphql` |
| `CaseTemplateAddInput` | `src/modules/case/case-template/case-template.graphql` |
| `FeedbackAddInput` | `src/modules/case/feedback/feedback.graphql` |

<details><summary>Show definition details</summary>


**input CaseIncidentAddInput**  
Defined in `src/modules/case/case-incident/case-incident.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `severity` | `String` |
| `priority` | `String` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `objects` | `[String]` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectAssignee` | `[String]` |
| `objectParticipant` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `response_types` | `[String!]` |
| `caseTemplates` | `[String!]` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `authorized_members` | `[MemberAccessInput!]` |
| `upsertOperations` | `[EditInput!]` |


**input CaseRfiAddInput**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `severity` | `String` |
| `priority` | `String` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `objects` | `[String]` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectAssignee` | `[String]` |
| `objectParticipant` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `information_types` | `[String!]` |
| `caseTemplates` | `[String!]` |
| `authorized_members` | `[MemberAccessInput!]` |
| `x_opencti_request_access` | `String` |
| `upsertOperations` | `[EditInput!]` |


**input CaseRftAddInput**  
Defined in `src/modules/case/case-rft/case-rft.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `severity` | `String` |
| `priority` | `String` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `objects` | `[String]` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectAssignee` | `[String]` |
| `objectParticipant` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `clientMutationId` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `update` | `Boolean` |
| `takedown_types` | `[String!]` |
| `caseTemplates` | `[String!]` |
| `authorized_members` | `[MemberAccessInput!]` |
| `upsertOperations` | `[EditInput!]` |


**input CaseTemplateAddInput**  
Defined in `src/modules/case/case-template/case-template.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `created` | `DateTime` |
| `tasks` | `[StixRef!]!` |


**input FeedbackAddInput**  
Defined in `src/modules/case/feedback/feedback.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `objects` | `[String]` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectAssignee` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `rating` | `Int` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### interface (1)
<a id="interface-1"></a>

| Name | File |
| --- | --- |
| `Case` | `src/modules/case/case.graphql` |

<details><summary>Show definition details</summary>


**interface Case**  
Defined in `src/modules/case/case.graphql`

_Referenced by:_
- `type` `CaseEdge` (case) in `src/modules/case/case.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `tasks` | `TaskConnection!` |


</details>


#### type (18)
<a id="type-18"></a>

| Name | File |
| --- | --- |
| `CaseConnection` | `src/modules/case/case.graphql` |
| `CaseEdge` | `src/modules/case/case.graphql` |
| `CaseIncident` | `src/modules/case/case-incident/case-incident.graphql` |
| `CaseIncidentConnection` | `src/modules/case/case-incident/case-incident.graphql` |
| `CaseIncidentEdge` | `src/modules/case/case-incident/case-incident.graphql` |
| `CaseRfi` | `src/modules/case/case-rfi/case-rfi.graphql` |
| `CaseRfiConnection` | `src/modules/case/case-rfi/case-rfi.graphql` |
| `CaseRfiEdge` | `src/modules/case/case-rfi/case-rfi.graphql` |
| `CaseRft` | `src/modules/case/case-rft/case-rft.graphql` |
| `CaseRftConnection` | `src/modules/case/case-rft/case-rft.graphql` |
| `CaseRftEdge` | `src/modules/case/case-rft/case-rft.graphql` |
| `CaseTemplate` | `src/modules/case/case-template/case-template.graphql` |
| `CaseTemplateConnection` | `src/modules/case/case-template/case-template.graphql` |
| `CaseTemplateEdge` | `src/modules/case/case-template/case-template.graphql` |
| `Feedback` | `src/modules/case/feedback/feedback.graphql` |
| `FeedbackConnection` | `src/modules/case/feedback/feedback.graphql` |
| `FeedbackEdge` | `src/modules/case/feedback/feedback.graphql` |
| `RfiRequestAccessConfiguration` | `src/modules/case/case-rfi/case-rfi.graphql` |

<details><summary>Show definition details</summary>


**type CaseConnection**  
Defined in `src/modules/case/case.graphql`

_Referenced by:_
- `type` `AdministrativeArea` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `Channel` (channel) in `src/modules/channel/channel.graphql`
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`
- `type` `Event` (event) in `src/modules/event/event.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `Language` (language) in `src/modules/language/language.graphql`
- `type` `MalwareAnalysis` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- `type` `SecurityCoverage` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`
- `type` `SecurityPlatform` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`
- `type` `Task` (task) in `src/modules/task/task.graphql`
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CaseEdge]` |


**type CaseEdge**  
Defined in `src/modules/case/case.graphql`

_Referenced by:_
- `type` `CaseConnection` (case) in `src/modules/case/case.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Case!` |


**type CaseIncident**  
Defined in `src/modules/case/case-incident/case-incident.graphql`

_Referenced by:_
- `type` `CaseIncidentEdge` (case) in `src/modules/case/case-incident/case-incident.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `securityCoverage` | `SecurityCoverage` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `tasks` | `TaskConnection!` |
| `rating` | `Int` |
| `response_types` | `[String!]` |
| `severity` | `String` |
| `priority` | `String` |


**type CaseIncidentConnection**  
Defined in `src/modules/case/case-incident/case-incident.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CaseIncidentEdge]` |


**type CaseIncidentEdge**  
Defined in `src/modules/case/case-incident/case-incident.graphql`

_Referenced by:_
- `type` `CaseIncidentConnection` (case) in `src/modules/case/case-incident/case-incident.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `CaseIncident!` |


**type CaseRfi**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

_Referenced by:_
- `type` `CaseRfiEdge` (case) in `src/modules/case/case-rfi/case-rfi.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `tasks` | `TaskConnection!` |
| `information_types` | `[String!]` |
| `severity` | `String` |
| `priority` | `String` |
| `x_opencti_request_access` | `String` |
| `requestAccessConfiguration` | `RfiRequestAccessConfiguration` |
| `x_opencti_workflow_id` | `String` |


**type CaseRfiConnection**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CaseRfiEdge]` |


**type CaseRfiEdge**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

_Referenced by:_
- `type` `CaseRfiConnection` (case) in `src/modules/case/case-rfi/case-rfi.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `CaseRfi!` |


**type CaseRft**  
Defined in `src/modules/case/case-rft/case-rft.graphql`

_Referenced by:_
- `type` `CaseRftEdge` (case) in `src/modules/case/case-rft/case-rft.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `tasks` | `TaskConnection!` |
| `takedown_types` | `[String!]` |
| `severity` | `String` |
| `priority` | `String` |


**type CaseRftConnection**  
Defined in `src/modules/case/case-rft/case-rft.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CaseRftEdge]` |


**type CaseRftEdge**  
Defined in `src/modules/case/case-rft/case-rft.graphql`

_Referenced by:_
- `type` `CaseRftConnection` (case) in `src/modules/case/case-rft/case-rft.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `CaseRft!` |


**type CaseTemplate**  
Defined in `src/modules/case/case-template/case-template.graphql`

_Referenced by:_
- `type` `CaseTemplateEdge` (case) in `src/modules/case/case-template/case-template.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `tasks` | `TaskTemplateConnection!` |


**type CaseTemplateConnection**  
Defined in `src/modules/case/case-template/case-template.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CaseTemplateEdge!]!` |


**type CaseTemplateEdge**  
Defined in `src/modules/case/case-template/case-template.graphql`

_Referenced by:_
- `type` `CaseTemplateConnection` (case) in `src/modules/case/case-template/case-template.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `CaseTemplate!` |


**type Feedback**  
Defined in `src/modules/case/feedback/feedback.graphql`

_Referenced by:_
- `type` `FeedbackEdge` (case) in `src/modules/case/feedback/feedback.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `tasks` | `TaskConnection!` |
| `rating` | `Int` |


**type FeedbackConnection**  
Defined in `src/modules/case/feedback/feedback.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[FeedbackEdge]` |


**type FeedbackEdge**  
Defined in `src/modules/case/feedback/feedback.graphql`

_Referenced by:_
- `type` `FeedbackConnection` (case) in `src/modules/case/feedback/feedback.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Feedback!` |


**type RfiRequestAccessConfiguration**  
Defined in `src/modules/case/case-rfi/case-rfi.graphql`

_Referenced by:_
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`

| Field | Type |
| --- | --- |
| `configuration` | `RequestAccessConfiguration` |
| `isUserCanAction` | `Boolean!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **43**
- Definition count: **170**


#### Expanded: enum (16)
<a id="expanded-enum-16"></a>

<details><summary>Show names</summary>

- `CaseIncidentsOrdering`
- `CaseRfisOrdering`
- `CaseRftsOrdering`
- `CaseTemplatesOrdering`
- `CasesOrdering`
- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `FeedbacksOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (8)
<a id="expanded-input-8"></a>

<details><summary>Show names</summary>

- `CaseIncidentAddInput`
- `CaseRfiAddInput`
- `CaseRftAddInput`
- `CaseTemplateAddInput`
- `EditInput`
- `FeedbackAddInput`
- `MemberAccessInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (133)
<a id="expanded-type-133"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `CaseIncident`
- `CaseIncidentConnection`
- `CaseIncidentEdge`
- `CaseRfi`
- `CaseRfiConnection`
- `CaseRfiEdge`
- `CaseRft`
- `CaseRftConnection`
- `CaseRftEdge`
- `CaseTemplate`
- `CaseTemplateConnection`
- `CaseTemplateEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `Feedback`
- `FeedbackConnection`
- `FeedbackEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `RequestAccessConfiguration`
- `RequestAccessMember`
- `RfiRequestAccessConfiguration`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TaskTemplate`
- `TaskTemplateConnection`
- `TaskTemplateEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: catalog
<a id="module-family-catalog"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 3 |
| Mutation fields | 0 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/catalog/catalog.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (3)
<a id="query-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| catalog | `Catalog` | `catalog(id: String!): Catalog @auth(for: [INGESTION])` |
| catalogs | `[Catalog!]!` | `catalogs: [Catalog!]! @auth(for: [INGESTION])` |
| contract | `ExtendedContract` | `contract(slug: String!): ExtendedContract @auth(for: [INGESTION])` |


#### Mutation (0)
<a id="mutation-0"></a>

_None._


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `CatalogsOrdering` | `src/modules/catalog/catalog.graphql` |

<details><summary>Show definition details</summary>


**enum CatalogsOrdering**  
Defined in `src/modules/catalog/catalog.graphql`

Values:
- `name`
- `_score`


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `Catalog` | `src/modules/catalog/catalog.graphql` |
| `CatalogConnection` | `src/modules/catalog/catalog.graphql` |
| `CatalogEdge` | `src/modules/catalog/catalog.graphql` |
| `ExtendedContract` | `src/modules/catalog/catalog.graphql` |

<details><summary>Show definition details</summary>


**type Catalog**  
Defined in `src/modules/catalog/catalog.graphql`

_Referenced by:_
- `type` `CatalogEdge` (catalog) in `src/modules/catalog/catalog.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `description` | `String!` |
| `contracts` | `[String!]!` |


**type CatalogConnection**  
Defined in `src/modules/catalog/catalog.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CatalogEdge!]!` |


**type CatalogEdge**  
Defined in `src/modules/catalog/catalog.graphql`

_Referenced by:_
- `type` `CatalogConnection` (catalog) in `src/modules/catalog/catalog.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Catalog!` |


**type ExtendedContract**  
Defined in `src/modules/catalog/catalog.graphql`

| Field | Type |
| --- | --- |
| `catalog_id` | `String!` |
| `contract` | `String!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **6**
- Definition count: **7**


#### Expanded: enum (1)
<a id="expanded-enum-1"></a>

<details><summary>Show names</summary>

- `CatalogsOrdering`

</details>


#### Expanded: type (6)
<a id="expanded-type-6"></a>

<details><summary>Show names</summary>

- `Catalog`
- `CatalogConnection`
- `CatalogEdge`
- `ExtendedContract`
- `Metric`
- `PageInfo`

</details>



## Module family: channel
<a id="module-family-channel"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/channel/channel.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| channel | `Channel` | `channel(id: String!): Channel @auth(for: [KNOWLEDGE])` |
| channels | `ChannelConnection` | `channels( first: Int after: ID orderBy: ChannelsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): ChannelConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| channelAdd | `Channel` | `channelAdd(input: ChannelAddInput!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelContextClean | `Channel` | `channelContextClean(id: ID!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelContextPatch | `Channel` | `channelContextPatch(id: ID!, input: EditContext!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelDelete | `ID` | `channelDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| channelFieldPatch | `Channel` | `channelFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelRelationAdd | `StixRefRelationship` | `channelRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| channelRelationDelete | `Channel` | `channelRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Channel @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `ChannelsOrdering` | `src/modules/channel/channel.graphql` |

<details><summary>Show definition details</summary>


**enum ChannelsOrdering**  
Defined in `src/modules/channel/channel.graphql`

Values:
- `name`
- `channel_types`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `objectMarking`
- `objectLabel`
- `x_opencti_workflow_id`
- `confidence`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `ChannelAddInput` | `src/modules/channel/channel.graphql` |

<details><summary>Show definition details</summary>


**input ChannelAddInput**  
Defined in `src/modules/channel/channel.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `channel_types` | `[String]` |
| `aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Channel` | `src/modules/channel/channel.graphql` |
| `ChannelConnection` | `src/modules/channel/channel.graphql` |
| `ChannelEdge` | `src/modules/channel/channel.graphql` |

<details><summary>Show definition details</summary>


**type Channel**  
Defined in `src/modules/channel/channel.graphql`

_Referenced by:_
- `type` `ChannelEdge` (channel) in `src/modules/channel/channel.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `description` | `String` |
| `channel_types` | `[String]` |
| `aliases` | `[String]` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type ChannelConnection**  
Defined in `src/modules/channel/channel.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[ChannelEdge]` |


**type ChannelEdge**  
Defined in `src/modules/channel/channel.graphql`

_Referenced by:_
- `type` `ChannelConnection` (channel) in `src/modules/channel/channel.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Channel!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ChannelsOrdering`
- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `ChannelAddInput`
- `EditContext`
- `EditInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `Channel`
- `ChannelConnection`
- `ChannelEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: dataComponent
<a id="module-family-datacomponent"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/dataComponent/dataComponent.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| dataComponent | `DataComponent` | `dataComponent(id: String!): DataComponent @auth(for: [KNOWLEDGE])` |
| dataComponents | `DataComponentConnection` | `dataComponents( first: Int after: ID orderBy: DataComponentsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DataComponentConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| dataComponentAdd | `DataComponent` | `dataComponentAdd(input: DataComponentAddInput!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentContextClean | `DataComponent` | `dataComponentContextClean(id: ID!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentContextPatch | `DataComponent` | `dataComponentContextPatch(id: ID!, input: EditContext!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentDelete | `ID` | `dataComponentDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataComponentFieldPatch | `DataComponent` | `dataComponentFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentRelationAdd | `StixRefRelationship` | `dataComponentRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataComponentRelationDelete | `DataComponent` | `dataComponentRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): DataComponent @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `DataComponentsOrdering` | `src/modules/dataComponent/dataComponent.graphql` |

<details><summary>Show definition details</summary>


**enum DataComponentsOrdering**  
Defined in `src/modules/dataComponent/dataComponent.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `x_opencti_workflow_id`
- `confidence`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `DataComponentAddInput` | `src/modules/dataComponent/dataComponent.graphql` |

<details><summary>Show definition details</summary>


**input DataComponentAddInput**  
Defined in `src/modules/dataComponent/dataComponent.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `objectOrganization` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `dataSource` | `String` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `DataComponent` | `src/modules/dataComponent/dataComponent.graphql` |
| `DataComponentConnection` | `src/modules/dataComponent/dataComponent.graphql` |
| `DataComponentEdge` | `src/modules/dataComponent/dataComponent.graphql` |

<details><summary>Show definition details</summary>


**type DataComponent**  
Defined in `src/modules/dataComponent/dataComponent.graphql`

_Referenced by:_
- `type` `DataComponentEdge` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `dataSource` | `DataSource` |
| `attackPatterns` | `AttackPatternConnection` |


**type DataComponentConnection**  
Defined in `src/modules/dataComponent/dataComponent.graphql`

_Referenced by:_
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DataComponentEdge]` |


**type DataComponentEdge**  
Defined in `src/modules/dataComponent/dataComponent.graphql`

_Referenced by:_
- `type` `DataComponentConnection` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DataComponent!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **151**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DataComponentsOrdering`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `DataComponentAddInput`
- `EditContext`
- `EditInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (123)
<a id="expanded-type-123"></a>

<details><summary>Show names</summary>

- `Assignee`
- `AttackPattern`
- `AttackPatternConnection`
- `AttackPatternEdge`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CourseOfAction`
- `CourseOfActionConnection`
- `CourseOfActionEdge`
- `CoverageResult`
- `Creator`
- `DataComponent`
- `DataComponentConnection`
- `DataComponentEdge`
- `DataSource`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: dataSource
<a id="module-family-datasource"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 9 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/dataSource/dataSource.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| dataSource | `DataSource` | `dataSource(id: String!): DataSource @auth(for: [KNOWLEDGE])` |
| dataSources | `DataSourceConnection` | `dataSources( first: Int after: ID orderBy: DataSourcesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DataSourceConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (9)
<a id="mutation-9"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| dataSourceAdd | `DataSource` | `dataSourceAdd(input: DataSourceAddInput!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceContextClean | `DataSource` | `dataSourceContextClean(id: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceContextPatch | `DataSource` | `dataSourceContextPatch(id: ID!, input: EditContext!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceDataComponentAdd | `DataSource` | `dataSourceDataComponentAdd(id: ID!, dataComponentId: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceDataComponentDelete | `DataSource` | `dataSourceDataComponentDelete(id: ID!, dataComponentId: ID!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceDelete | `ID` | `dataSourceDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| dataSourceFieldPatch | `DataSource` | `dataSourceFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceRelationAdd | `StixRefRelationship` | `dataSourceRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| dataSourceRelationDelete | `DataSource` | `dataSourceRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): DataSource @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `DataSourcesOrdering` | `src/modules/dataSource/dataSource.graphql` |

<details><summary>Show definition details</summary>


**enum DataSourcesOrdering**  
Defined in `src/modules/dataSource/dataSource.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `x_opencti_workflow_id`
- `confidence`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `DataSourceAddInput` | `src/modules/dataSource/dataSource.graphql` |

<details><summary>Show definition details</summary>


**input DataSourceAddInput**  
Defined in `src/modules/dataSource/dataSource.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `objectOrganization` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `x_mitre_platforms` | `[String!]` |
| `collection_layers` | `[String!]` |
| `dataComponents` | `[String]` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `DataSource` | `src/modules/dataSource/dataSource.graphql` |
| `DataSourceConnection` | `src/modules/dataSource/dataSource.graphql` |
| `DataSourceEdge` | `src/modules/dataSource/dataSource.graphql` |

<details><summary>Show definition details</summary>


**type DataSource**  
Defined in `src/modules/dataSource/dataSource.graphql`

_Referenced by:_
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSourceEdge` (dataSource) in `src/modules/dataSource/dataSource.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `x_mitre_platforms` | `[String!]` |
| `collection_layers` | `[String!]` |
| `dataComponents` | `DataComponentConnection` |


**type DataSourceConnection**  
Defined in `src/modules/dataSource/dataSource.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DataSourceEdge]` |


**type DataSourceEdge**  
Defined in `src/modules/dataSource/dataSource.graphql`

_Referenced by:_
- `type` `DataSourceConnection` (dataSource) in `src/modules/dataSource/dataSource.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DataSource!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **153**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DataSourcesOrdering`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `DataSourceAddInput`
- `EditContext`
- `EditInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (125)
<a id="expanded-type-125"></a>

<details><summary>Show names</summary>

- `Assignee`
- `AttackPattern`
- `AttackPatternConnection`
- `AttackPatternEdge`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CourseOfAction`
- `CourseOfActionConnection`
- `CourseOfActionEdge`
- `CoverageResult`
- `Creator`
- `DataComponent`
- `DataComponentConnection`
- `DataComponentEdge`
- `DataSource`
- `DataSourceConnection`
- `DataSourceEdge`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: decayRule
<a id="module-family-decayrule"></a>

| Metric | Value |
| --- | --- |
| SDL files | 2 |
| Query fields | 4 |
| Mutation fields | 6 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/decayRule/decayRule.graphql` |
| `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (4)
<a id="query-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| decayExclusionRule | `DecayExclusionRule` | `decayExclusionRule(id: String!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRules | `DecayExclusionRuleConnection` | `decayExclusionRules( first: Int after: ID orderBy: DecayExclusionRuleOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DecayExclusionRuleConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRule | `DecayRule` | `decayRule(id: String!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRules | `DecayRuleConnection` | `decayRules( first: Int after: ID orderBy: DecayRuleOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DecayRuleConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Mutation (6)
<a id="mutation-6"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| decayExclusionRuleAdd | `DecayExclusionRule` | `decayExclusionRuleAdd(input: DecayExclusionRuleAddInput!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRuleDelete | `ID` | `decayExclusionRuleDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayExclusionRuleFieldPatch | `DecayExclusionRule` | `decayExclusionRuleFieldPatch(id: ID!, input: [EditInput!]!): DecayExclusionRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleAdd | `DecayRule` | `decayRuleAdd(input: DecayRuleAddInput!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleDelete | `ID` | `decayRuleDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| decayRuleFieldPatch | `DecayRule` | `decayRuleFieldPatch(id: ID!, input: [EditInput!]!): DecayRule @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (2)
<a id="enum-2"></a>

| Name | File |
| --- | --- |
| `DecayExclusionRuleOrdering` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |
| `DecayRuleOrdering` | `src/modules/decayRule/decayRule.graphql` |

<details><summary>Show definition details</summary>


**enum DecayExclusionRuleOrdering**  
Defined in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

Values:
- `name`
- `active`
- `_score`


**enum DecayRuleOrdering**  
Defined in `src/modules/decayRule/decayRule.graphql`

Values:
- `name`
- `order`
- `_score`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `DecayExclusionRuleAddInput` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |
| `DecayRuleAddInput` | `src/modules/decayRule/decayRule.graphql` |

<details><summary>Show definition details</summary>


**input DecayExclusionRuleAddInput**  
Defined in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `decay_exclusion_filters` | `String!` |
| `active` | `Boolean!` |


**input DecayRuleAddInput**  
Defined in `src/modules/decayRule/decayRule.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `order` | `Int!` |
| `active` | `Boolean!` |
| `decay_lifetime` | `Int!` |
| `decay_pound` | `Float!` |
| `decay_points` | `[Int!]` |
| `decay_revoke_score` | `Int!` |
| `decay_observable_types` | `[String!]` |


</details>


#### type (7)
<a id="type-7"></a>

| Name | File |
| --- | --- |
| `DecayData` | `src/modules/decayRule/decayRule.graphql` |
| `DecayExclusionRule` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |
| `DecayExclusionRuleConnection` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |
| `DecayExclusionRuleEdge` | `src/modules/decayRule/exclusions/decayExclusionRule.graphql` |
| `DecayRule` | `src/modules/decayRule/decayRule.graphql` |
| `DecayRuleConnection` | `src/modules/decayRule/decayRule.graphql` |
| `DecayRuleEdge` | `src/modules/decayRule/decayRule.graphql` |

<details><summary>Show definition details</summary>


**type DecayData**  
Defined in `src/modules/decayRule/decayRule.graphql`

_Referenced by:_
- `type` `DecayRule` (decayRule) in `src/modules/decayRule/decayRule.graphql`

| Field | Type |
| --- | --- |
| `live_score_serie` | `[DecayHistory!]` |


**type DecayExclusionRule**  
Defined in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

_Referenced by:_
- `type` `DecayExclusionRuleEdge` (decayRule) in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `name` | `String!` |
| `description` | `String` |
| `decay_exclusion_filters` | `String!` |
| `active` | `Boolean` |


**type DecayExclusionRuleConnection**  
Defined in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DecayExclusionRuleEdge!]!` |


**type DecayExclusionRuleEdge**  
Defined in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

_Referenced by:_
- `type` `DecayExclusionRuleConnection` (decayRule) in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DecayExclusionRule!` |


**type DecayRule**  
Defined in `src/modules/decayRule/decayRule.graphql`

_Referenced by:_
- `type` `DecayRuleEdge` (decayRule) in `src/modules/decayRule/decayRule.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `order` | `Int!` |
| `active` | `Boolean!` |
| `built_in` | `Boolean` |
| `appliedIndicatorsCount` | `Int!` |
| `decay_lifetime` | `Int!` |
| `decay_pound` | `Float!` |
| `decay_points` | `[Int!]` |
| `decay_revoke_score` | `Int!` |
| `decay_observable_types` | `[String!]` |
| `decaySettingsChartData` | `DecayData` |


**type DecayRuleConnection**  
Defined in `src/modules/decayRule/decayRule.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DecayRuleEdge!]!` |


**type DecayRuleEdge**  
Defined in `src/modules/decayRule/decayRule.graphql`

_Referenced by:_
- `type` `DecayRuleConnection` (decayRule) in `src/modules/decayRule/decayRule.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DecayRule!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **16**
- Definition count: **18**


#### Expanded: enum (3)
<a id="expanded-enum-3"></a>

<details><summary>Show names</summary>

- `DecayExclusionRuleOrdering`
- `DecayRuleOrdering`
- `EditOperation`

</details>


#### Expanded: input (3)
<a id="expanded-input-3"></a>

<details><summary>Show names</summary>

- `DecayExclusionRuleAddInput`
- `DecayRuleAddInput`
- `EditInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`

</details>


#### Expanded: type (10)
<a id="expanded-type-10"></a>

<details><summary>Show names</summary>

- `DecayData`
- `DecayExclusionRule`
- `DecayExclusionRuleConnection`
- `DecayExclusionRuleEdge`
- `DecayHistory`
- `DecayRule`
- `DecayRuleConnection`
- `DecayRuleEdge`
- `Metric`
- `PageInfo`

</details>



## Module family: deleteOperation
<a id="module-family-deleteoperation"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 2 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/deleteOperation/deleteOperation.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| deleteOperation | `DeleteOperation` | `deleteOperation(id: String!): DeleteOperation @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| deleteOperations | `DeleteOperationConnection` | `deleteOperations( first: Int after: ID orderBy: DeleteOperationOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DeleteOperationConnection @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### Mutation (2)
<a id="mutation-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| deleteOperationConfirm | `ID` | `deleteOperationConfirm(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| deleteOperationRestore | `ID` | `deleteOperationRestore(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `DeleteOperationOrdering` | `src/modules/deleteOperation/deleteOperation.graphql` |

<details><summary>Show definition details</summary>


**enum DeleteOperationOrdering**  
Defined in `src/modules/deleteOperation/deleteOperation.graphql`

Values:
- `main_entity_name`
- `created_at`
- `deletedBy`
- `objectMarking`
- `_score`


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `DeleteOperation` | `src/modules/deleteOperation/deleteOperation.graphql` |
| `DeleteOperationConnection` | `src/modules/deleteOperation/deleteOperation.graphql` |
| `DeleteOperationEdge` | `src/modules/deleteOperation/deleteOperation.graphql` |
| `DeletedElement` | `src/modules/deleteOperation/deleteOperation.graphql` |

<details><summary>Show definition details</summary>


**type DeleteOperation**  
Defined in `src/modules/deleteOperation/deleteOperation.graphql`

_Referenced by:_
- `type` `DeleteOperationEdge` (deleteOperation) in `src/modules/deleteOperation/deleteOperation.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `objectMarking` | `[MarkingDefinition!]` |
| `confidence` | `Int` |
| `created_at` | `DateTime` |
| `deletedBy` | `Creator` |
| `main_entity_type` | `String!` |
| `main_entity_id` | `String!` |
| `main_entity_name` | `String!` |
| `deleted_elements` | `[DeletedElement!]!` |


**type DeleteOperationConnection**  
Defined in `src/modules/deleteOperation/deleteOperation.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DeleteOperationEdge!]!` |


**type DeleteOperationEdge**  
Defined in `src/modules/deleteOperation/deleteOperation.graphql`

_Referenced by:_
- `type` `DeleteOperationConnection` (deleteOperation) in `src/modules/deleteOperation/deleteOperation.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DeleteOperation!` |


**type DeletedElement**  
Defined in `src/modules/deleteOperation/deleteOperation.graphql`

_Referenced by:_
- `type` `DeleteOperation` (deleteOperation) in `src/modules/deleteOperation/deleteOperation.graphql`

| Field | Type |
| --- | --- |
| `id` | `String!` |
| `source_index` | `String!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **8**
- Definition count: **22**


#### Expanded: enum (3)
<a id="expanded-enum-3"></a>

<details><summary>Show names</summary>

- `DeleteOperationOrdering`
- `DraftOperation`
- `Version`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `DateTime`
- `StixId`

</details>


#### Expanded: type (16)
<a id="expanded-type-16"></a>

<details><summary>Show names</summary>

- `Creator`
- `DeleteOperation`
- `DeleteOperationConnection`
- `DeleteOperationEdge`
- `DeletedElement`
- `Display`
- `DisplayStep`
- `DraftVersion`
- `EditUserContext`
- `Inference`
- `InferenceAttribute`
- `MarkingDefinition`
- `Metric`
- `PageInfo`
- `Representative`
- `Rule`

</details>


#### Expanded: union (1)
<a id="expanded-union-1"></a>

<details><summary>Show names</summary>

- `StixObjectOrStixRelationship`

</details>



## Module family: disseminationList
<a id="module-family-disseminationlist"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/disseminationList/disseminationList.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| disseminationList | `DisseminationList` | `disseminationList(id: ID!): DisseminationList @auth(for: [KNOWLEDGE_KNDISSEMINATION, SETTINGS_SETDISSEMINATION])` |
| disseminationLists | `DisseminationListConnection` | `disseminationLists( first: Int after: ID orderBy: DisseminationListOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DisseminationListConnection @auth(for: [KNOWLEDGE_KNDISSEMINATION, SETTINGS_SETDISSEMINATION])` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| disseminationListAdd | `DisseminationList` | `disseminationListAdd(input: DisseminationListAddInput!): DisseminationList @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListDelete | `ID` | `disseminationListDelete(id: ID!): ID @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListFieldPatch | `DisseminationList` | `disseminationListFieldPatch(id: ID!, input: [EditInput!]!): DisseminationList @auth(for: [SETTINGS_SETDISSEMINATION])` |
| disseminationListSend | `Boolean` | `disseminationListSend(id: ID!, input: DisseminationListSendInput!): Boolean @auth(for: [KNOWLEDGE_KNDISSEMINATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `DisseminationListOrdering` | `src/modules/disseminationList/disseminationList.graphql` |

<details><summary>Show definition details</summary>


**enum DisseminationListOrdering**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

Values:
- `name`
- `_score`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `DisseminationListAddInput` | `src/modules/disseminationList/disseminationList.graphql` |
| `DisseminationListSendInput` | `src/modules/disseminationList/disseminationList.graphql` |

<details><summary>Show definition details</summary>


**input DisseminationListAddInput**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `emails` | `[String!]!` |
| `description` | `String` |


**input DisseminationListSendInput**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

| Field | Type |
| --- | --- |
| `entity_id` | `ID!` |
| `use_octi_template` | `Boolean!` |
| `email_object` | `String!` |
| `email_body` | `String!` |
| `email_attachment_ids` | `[ID!]!` |
| `html_to_body_file_id` | `ID` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `DisseminationList` | `src/modules/disseminationList/disseminationList.graphql` |
| `DisseminationListConnection` | `src/modules/disseminationList/disseminationList.graphql` |
| `DisseminationListEdge` | `src/modules/disseminationList/disseminationList.graphql` |

<details><summary>Show definition details</summary>


**type DisseminationList**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

_Referenced by:_
- `type` `DisseminationListEdge` (disseminationList) in `src/modules/disseminationList/disseminationList.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `emails` | `[String!]!` |
| `description` | `String` |


**type DisseminationListConnection**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DisseminationListEdge!]!` |


**type DisseminationListEdge**  
Defined in `src/modules/disseminationList/disseminationList.graphql`

_Referenced by:_
- `type` `DisseminationListConnection` (disseminationList) in `src/modules/disseminationList/disseminationList.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DisseminationList!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **10**
- Definition count: **12**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `DisseminationListOrdering`
- `EditOperation`

</details>


#### Expanded: input (3)
<a id="expanded-input-3"></a>

<details><summary>Show names</summary>

- `DisseminationListAddInput`
- `DisseminationListSendInput`
- `EditInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`

</details>


#### Expanded: type (5)
<a id="expanded-type-5"></a>

<details><summary>Show names</summary>

- `DisseminationList`
- `DisseminationListConnection`
- `DisseminationListEdge`
- `Metric`
- `PageInfo`

</details>



## Module family: draftWorkspace
<a id="module-family-draftworkspace"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 6 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/draftWorkspace/draftWorkspace.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (6)
<a id="query-6"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| draftWorkspace | `DraftWorkspace` | `draftWorkspace(id: String!): DraftWorkspace @auth(for: [KNOWLEDGE])` |
| draftWorkspaceEntities | `StixCoreObjectConnection` | `draftWorkspaceEntities( draftId: String!, types: [String] first: Int after: ID orderBy: StixCoreObjectsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixCoreObjectConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaceRelationships | `StixRelationshipConnection` | `draftWorkspaceRelationships( draftId: String!, types: [String] first: Int after: ID orderBy: StixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixRelationshipConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaceSightingRelationships | `StixSightingRelationshipConnection` | `draftWorkspaceSightingRelationships( draftId: String!, types: [String] first: Int after: ID orderBy: StixSightingRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): StixSightingRelationshipConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspaces | `DraftWorkspaceConnection` | `draftWorkspaces( first: Int after: ID orderBy: DraftWorkspacesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DraftWorkspaceConnection @auth(for: [KNOWLEDGE])` |
| draftWorkspacesRestricted | `DraftWorkspaceConnection` | `draftWorkspacesRestricted( first: Int after: ID orderBy: DraftWorkspacesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): DraftWorkspaceConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| draftWorkspaceAdd | `DraftWorkspace` | `draftWorkspaceAdd(input: DraftWorkspaceAddInput!): DraftWorkspace @auth(for: [KNOWLEDGE])` |
| draftWorkspaceDelete | `ID` | `draftWorkspaceDelete(id: ID!): ID @auth(for: [KNOWLEDGE])` |
| draftWorkspaceEditAuthorizedMembers | `DraftWorkspace` | `draftWorkspaceEditAuthorizedMembers(id: ID!, input: [MemberAccessInput!]): DraftWorkspace @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| draftWorkspaceValidate | `Work` | `draftWorkspaceValidate(id: ID!): Work @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (2)
<a id="enum-2"></a>

| Name | File |
| --- | --- |
| `DraftStatus` | `src/modules/draftWorkspace/draftWorkspace.graphql` |
| `DraftWorkspacesOrdering` | `src/modules/draftWorkspace/draftWorkspace.graphql` |

<details><summary>Show definition details</summary>


**enum DraftStatus**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

_Referenced by:_
- `type` `DraftWorkspace` (draftWorkspace) in `src/modules/draftWorkspace/draftWorkspace.graphql`

Values:
- `open`
- `validated`


**enum DraftWorkspacesOrdering**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

Values:
- `name`
- `created_at`
- `creator`
- `draft_status`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `DraftWorkspaceAddInput` | `src/modules/draftWorkspace/draftWorkspace.graphql` |

<details><summary>Show definition details</summary>


**input DraftWorkspaceAddInput**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `entity_id` | `String` |
| `authorized_members` | `[MemberAccessInput!]` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `DraftObjectsCount` | `src/modules/draftWorkspace/draftWorkspace.graphql` |
| `DraftWorkspace` | `src/modules/draftWorkspace/draftWorkspace.graphql` |
| `DraftWorkspaceConnection` | `src/modules/draftWorkspace/draftWorkspace.graphql` |
| `DraftWorkspaceEdge` | `src/modules/draftWorkspace/draftWorkspace.graphql` |

<details><summary>Show definition details</summary>


**type DraftObjectsCount**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

_Referenced by:_
- `type` `DraftWorkspace` (draftWorkspace) in `src/modules/draftWorkspace/draftWorkspace.graphql`

| Field | Type |
| --- | --- |
| `totalCount` | `Int!` |
| `entitiesCount` | `Int!` |
| `observablesCount` | `Int!` |
| `relationshipsCount` | `Int!` |
| `sightingsCount` | `Int!` |
| `containersCount` | `Int!` |


**type DraftWorkspace**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

_Referenced by:_
- `type` `DraftWorkspaceEdge` (draftWorkspace) in `src/modules/draftWorkspace/draftWorkspace.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `created_at` | `DateTime!` |
| `creators` | `[Creator!]` |
| `entity_id` | `String` |
| `objectsCount` | `DraftObjectsCount!` |
| `draft_status` | `DraftStatus!` |
| `processingCount` | `Int!` |
| `works(first: Int)` | `[Work!]` |
| `validationWork` | `Work` |
| `authorizedMembers` | `[MemberAccess!]!` |
| `currentUserAccessRight` | `String` |


**type DraftWorkspaceConnection**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[DraftWorkspaceEdge!]!` |


**type DraftWorkspaceEdge**  
Defined in `src/modules/draftWorkspace/draftWorkspace.graphql`

_Referenced by:_
- `type` `DraftWorkspaceConnection` (draftWorkspace) in `src/modules/draftWorkspace/draftWorkspace.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `DraftWorkspace!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **18**
- Definition count: **148**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `DraftStatus`
- `DraftWorkspacesOrdering`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `DraftWorkspaceAddInput`
- `MemberAccessInput`

</details>


#### Expanded: interface (7)
<a id="expanded-interface-7"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixCoreObject`
- `StixDomainObject`
- `StixObject`
- `StixRelationship`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `DateTime`
- `StixId`

</details>


#### Expanded: type (123)
<a id="expanded-type-123"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftObjectsCount`
- `DraftVersion`
- `DraftWorkspace`
- `DraftWorkspaceConnection`
- `DraftWorkspaceEdge`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreObjectConnection`
- `StixCoreObjectEdge`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRelationshipConnection`
- `StixRelationshipEdge`
- `StixSightingRelationship`
- `StixSightingRelationshipConnection`
- `StixSightingRelationshipsEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: emailTemplate
<a id="module-family-emailtemplate"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/emailTemplate/emailTemplate.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| emailTemplate | `EmailTemplate` | `emailTemplate(id: ID!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplates | `EmailTemplateConnection` | `emailTemplates ( first: Int after: ID orderBy: EmailTemplateOrdering orderMode: OrderingMode filters: FilterGroup search: String ): EmailTemplateConnection @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| emailTemplateAdd | `EmailTemplate` | `emailTemplateAdd(input: EmailTemplateAddInput!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateDelete | `ID` | `emailTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateFieldPatch | `EmailTemplate` | `emailTemplateFieldPatch(id: ID!, input: [EditInput!]!): EmailTemplate @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |
| emailTemplateTestSend | `Boolean` | `emailTemplateTestSend(id: ID!): Boolean @auth(for: [SETTINGS_SETACCESSES, VIRTUAL_ORGANIZATION_ADMIN])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `EmailTemplateOrdering` | `src/modules/emailTemplate/emailTemplate.graphql` |

<details><summary>Show definition details</summary>


**enum EmailTemplateOrdering**  
Defined in `src/modules/emailTemplate/emailTemplate.graphql`

Values:
- `name`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `EmailTemplateAddInput` | `src/modules/emailTemplate/emailTemplate.graphql` |

<details><summary>Show definition details</summary>


**input EmailTemplateAddInput**  
Defined in `src/modules/emailTemplate/emailTemplate.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `email_object` | `String!` |
| `sender_email` | `String!` |
| `template_body` | `String!` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `EmailTemplate` | `src/modules/emailTemplate/emailTemplate.graphql` |
| `EmailTemplateConnection` | `src/modules/emailTemplate/emailTemplate.graphql` |
| `EmailTemplateEdge` | `src/modules/emailTemplate/emailTemplate.graphql` |

<details><summary>Show definition details</summary>


**type EmailTemplate**  
Defined in `src/modules/emailTemplate/emailTemplate.graphql`

_Referenced by:_
- `type` `EmailTemplateEdge` (emailTemplate) in `src/modules/emailTemplate/emailTemplate.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `description` | `String` |
| `email_object` | `String!` |
| `sender_email` | `String!` |
| `template_body` | `String!` |


**type EmailTemplateConnection**  
Defined in `src/modules/emailTemplate/emailTemplate.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[EmailTemplateEdge!]!` |


**type EmailTemplateEdge**  
Defined in `src/modules/emailTemplate/emailTemplate.graphql`

_Referenced by:_
- `type` `EmailTemplateConnection` (emailTemplate) in `src/modules/emailTemplate/emailTemplate.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `EmailTemplate!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **9**
- Definition count: **10**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `EmailTemplateOrdering`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `EmailTemplateAddInput`

</details>


#### Expanded: scalar (1)
<a id="expanded-scalar-1"></a>

<details><summary>Show names</summary>

- `Any`

</details>


#### Expanded: type (5)
<a id="expanded-type-5"></a>

<details><summary>Show names</summary>

- `EmailTemplate`
- `EmailTemplateConnection`
- `EmailTemplateEdge`
- `Metric`
- `PageInfo`

</details>



## Module family: entitySetting
<a id="module-family-entitysetting"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 3 |
| Mutation fields | 1 |
| Subscription fields | 1 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/entitySetting/entitySetting.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (3)
<a id="query-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| entitySetting | `EntitySetting` | `entitySetting(id: String!): EntitySetting @auth` |
| entitySettingByType | `EntitySetting` | `entitySettingByType(targetType: String!): EntitySetting @auth` |
| entitySettings | `EntitySettingConnection` | `entitySettings( first: Int after: ID orderBy: EntitySettingsOrdering orderMode: OrderingMode filters: FilterGroup search: String includeObservables: Boolean ): EntitySettingConnection @auth` |


#### Mutation (1)
<a id="mutation-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| entitySettingsFieldPatch | `[EntitySetting]` | `entitySettingsFieldPatch(ids: [ID!]!, input: [EditInput!]!, commitMessage: String, references: [String]): [EntitySetting] @auth(for: [SETTINGS_SETCUSTOMIZATION, SETTINGS_SETPARAMETERS])` |


#### Subscription (1)
<a id="subscription-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| entitySetting | `EntitySetting` | `entitySetting(id: ID!): EntitySetting @auth` |


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `EntitySettingsOrdering` | `src/modules/entitySetting/entitySetting.graphql` |

<details><summary>Show definition details</summary>


**enum EntitySettingsOrdering**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

Values:
- `target_type`
- `_score`


</details>


#### type (8)
<a id="type-8"></a>

| Name | File |
| --- | --- |
| `DefaultValue` | `src/modules/entitySetting/entitySetting.graphql` |
| `DefaultValueAttribute` | `src/modules/entitySetting/entitySetting.graphql` |
| `EntitySetting` | `src/modules/entitySetting/entitySetting.graphql` |
| `EntitySettingConnection` | `src/modules/entitySetting/entitySetting.graphql` |
| `EntitySettingEdge` | `src/modules/entitySetting/entitySetting.graphql` |
| `OverviewWidgetCustomization` | `src/modules/entitySetting/entitySetting.graphql` |
| `ScaleAttribute` | `src/modules/entitySetting/entitySetting.graphql` |
| `TypeAttribute` | `src/modules/entitySetting/entitySetting.graphql` |

<details><summary>Show definition details</summary>


**type DefaultValue**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `DefaultValueAttribute` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`
- `type` `TypeAttribute` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`
- `type` `CsvMapperRepresentationAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `CsvMapperSchemaAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `JsonMapperRepresentationAttribute` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `String!` |
| `name` | `String!` |


**type DefaultValueAttribute**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `type` | `String!` |
| `defaultValues` | `[DefaultValue!]!` |


**type EntitySetting**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySettingEdge` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `target_type` | `String!` |
| `platform_entity_files_ref` | `Boolean` |
| `platform_hidden_type` | `Boolean` |
| `enforce_reference` | `Boolean` |
| `attributes_configuration` | `String` |
| `attributesDefinitions` | `[TypeAttribute!]!` |
| `mandatoryAttributes` | `[String!]!` |
| `scaleAttributes` | `[ScaleAttribute!]!` |
| `defaultValuesAttributes` | `[DefaultValueAttribute!]!` |
| `availableSettings` | `[String!]!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `overview_layout_customization` | `[OverviewWidgetCustomization!]` |
| `fintelTemplates(first: Int after: ID orderBy: FintelTemplateOrdering orderMode: OrderingMode search: String)` | `FintelTemplateConnection` |
| `requestAccessConfiguration` | `RequestAccessConfiguration` |


**type EntitySettingConnection**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[EntitySettingEdge!]!` |


**type EntitySettingEdge**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySettingConnection` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `EntitySetting!` |


**type OverviewWidgetCustomization**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `key` | `String!` |
| `width` | `Int!` |
| `label` | `String!` |


**type ScaleAttribute**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `scale` | `String!` |


**type TypeAttribute**  
Defined in `src/modules/entitySetting/entitySetting.graphql`

_Referenced by:_
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `type` | `String!` |
| `mandatory` | `Boolean!` |
| `mandatoryType` | `String!` |
| `editDefault` | `Boolean!` |
| `multiple` | `Boolean` |
| `upsert` | `Boolean!` |
| `label` | `String` |
| `defaultValues` | `[DefaultValue!]` |
| `scale` | `String` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **30**


#### Expanded: enum (3)
<a id="expanded-enum-3"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `EntitySettingsOrdering`
- `WidgetPerspective`

</details>


#### Expanded: input (1)
<a id="expanded-input-1"></a>

<details><summary>Show names</summary>

- `EditInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`

</details>


#### Expanded: type (24)
<a id="expanded-type-24"></a>

<details><summary>Show names</summary>

- `DefaultValue`
- `DefaultValueAttribute`
- `EditUserContext`
- `EntitySetting`
- `EntitySettingConnection`
- `EntitySettingEdge`
- `FintelTemplate`
- `FintelTemplateConnection`
- `FintelTemplateEdge`
- `FintelTemplateWidget`
- `Metric`
- `OverviewWidgetCustomization`
- `PageInfo`
- `RequestAccessConfiguration`
- `RequestAccessMember`
- `ScaleAttribute`
- `Status`
- `StatusTemplate`
- `TypeAttribute`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`

</details>



## Module family: event
<a id="module-family-event"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/event/event.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| event | `Event` | `event(id: String!): Event @auth(for: [KNOWLEDGE])` |
| events | `EventConnection` | `events( first: Int after: ID orderBy: EventsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): EventConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| eventAdd | `Event` | `eventAdd(input: EventAddInput!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventContextClean | `Event` | `eventContextClean(id: ID!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventContextPatch | `Event` | `eventContextPatch(id: ID!, input: EditContext!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventDelete | `ID` | `eventDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| eventFieldPatch | `Event` | `eventFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventRelationAdd | `StixRefRelationship` | `eventRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| eventRelationDelete | `Event` | `eventRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Event @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `EventsOrdering` | `src/modules/event/event.graphql` |

<details><summary>Show definition details</summary>


**enum EventsOrdering**  
Defined in `src/modules/event/event.graphql`

Values:
- `name`
- `event_types`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `objectMarking`
- `objectLabel`
- `x_opencti_workflow_id`
- `start_time`
- `stop_time`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `EventAddInput` | `src/modules/event/event.graphql` |

<details><summary>Show definition details</summary>


**input EventAddInput**  
Defined in `src/modules/event/event.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `event_types` | `[String!]` |
| `start_time` | `DateTime` |
| `stop_time` | `DateTime` |
| `aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Event` | `src/modules/event/event.graphql` |
| `EventConnection` | `src/modules/event/event.graphql` |
| `EventEdge` | `src/modules/event/event.graphql` |

<details><summary>Show definition details</summary>


**type Event**  
Defined in `src/modules/event/event.graphql`

_Referenced by:_
- `type` `EventEdge` (event) in `src/modules/event/event.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `description` | `String` |
| `event_types` | `[String]` |
| `start_time` | `DateTime` |
| `stop_time` | `DateTime` |
| `aliases` | `[String]` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type EventConnection**  
Defined in `src/modules/event/event.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[EventEdge]` |


**type EventEdge**  
Defined in `src/modules/event/event.graphql`

_Referenced by:_
- `type` `EventConnection` (event) in `src/modules/event/event.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Event!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `EventsOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `EventAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `Event`
- `EventConnection`
- `EventEdge`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: exclusionList
<a id="module-family-exclusionlist"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 3 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/exclusionList/exclusionList.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (3)
<a id="query-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| exclusionList | `ExclusionList` | `exclusionList(id: String!): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListCacheStatus | `ExclusionListCacheStatus` | `exclusionListCacheStatus: ExclusionListCacheStatus @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionLists | `ExclusionListConnection` | `exclusionLists( first: Int after: ID orderBy: ExclusionListOrdering orderMode: OrderingMode filters: FilterGroup search: String ): ExclusionListConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| exclusionListDelete | `ID` | `exclusionListDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListFieldPatch | `ExclusionList` | `exclusionListFieldPatch(id: ID!, input: [EditInput!], file: Upload): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| exclusionListFileAdd | `ExclusionList` | `exclusionListFileAdd(input: ExclusionListFileAddInput!): ExclusionList @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `ExclusionListOrdering` | `src/modules/exclusionList/exclusionList.graphql` |

<details><summary>Show definition details</summary>


**enum ExclusionListOrdering**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

Values:
- `name`
- `created_at`
- `enabled`
- `exclusion_list_values_count`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `ExclusionListFileAddInput` | `src/modules/exclusionList/exclusionList.graphql` |

<details><summary>Show definition details</summary>


**input ExclusionListFileAddInput**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `exclusion_list_entity_types` | `[String!]!` |
| `file` | `Upload!` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `ExclusionList` | `src/modules/exclusionList/exclusionList.graphql` |
| `ExclusionListCacheStatus` | `src/modules/exclusionList/exclusionList.graphql` |
| `ExclusionListConnection` | `src/modules/exclusionList/exclusionList.graphql` |
| `ExclusionListEdge` | `src/modules/exclusionList/exclusionList.graphql` |

<details><summary>Show definition details</summary>


**type ExclusionList**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

_Referenced by:_
- `type` `ExclusionListEdge` (exclusionList) in `src/modules/exclusionList/exclusionList.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |
| `description` | `String` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `enabled` | `Boolean!` |
| `exclusion_list_entity_types` | `[String!]!` |
| `exclusion_list_values_count` | `Int` |
| `file_id` | `String!` |
| `exclusion_list_file_size` | `Int` |


**type ExclusionListCacheStatus**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

| Field | Type |
| --- | --- |
| `refreshVersion` | `String!` |
| `cacheVersion` | `String!` |
| `isCacheRebuildInProgress` | `Boolean!` |


**type ExclusionListConnection**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[ExclusionListEdge!]` |


**type ExclusionListEdge**  
Defined in `src/modules/exclusionList/exclusionList.graphql`

_Referenced by:_
- `type` `ExclusionListConnection` (exclusionList) in `src/modules/exclusionList/exclusionList.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `ExclusionList!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **11**
- Definition count: **13**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `ExclusionListOrdering`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `ExclusionListFileAddInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `Upload`

</details>


#### Expanded: type (6)
<a id="expanded-type-6"></a>

<details><summary>Show names</summary>

- `ExclusionList`
- `ExclusionListCacheStatus`
- `ExclusionListConnection`
- `ExclusionListEdge`
- `Metric`
- `PageInfo`

</details>



## Module family: fintelDesign
<a id="module-family-finteldesign"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/fintelDesign/fintelDesign.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| fintelDesign | `FintelDesign` | `fintelDesign(id: String!): FintelDesign @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |
| fintelDesigns | `FintelDesignConnection` | `fintelDesigns( first: Int after: ID orderBy: FintelDesignOrdering orderMode: OrderingMode filters: FilterGroup search: String ): FintelDesignConnection @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| fintelDesignAdd | `FintelDesign` | `fintelDesignAdd(input: FintelDesignAddInput!): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignContextPatch | `FintelDesign` | `fintelDesignContextPatch(id: ID!, input: EditContext!): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignDelete | `ID` | `fintelDesignDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelDesignFieldPatch | `FintelDesign` | `fintelDesignFieldPatch(id: ID!, input: [EditInput!], file: Upload): FintelDesign @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `FintelDesignOrdering` | `src/modules/fintelDesign/fintelDesign.graphql` |

<details><summary>Show definition details</summary>


**enum FintelDesignOrdering**  
Defined in `src/modules/fintelDesign/fintelDesign.graphql`

Values:
- `_score`
- `name`
- `created_at`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `FintelDesignAddInput` | `src/modules/fintelDesign/fintelDesign.graphql` |

<details><summary>Show definition details</summary>


**input FintelDesignAddInput**  
Defined in `src/modules/fintelDesign/fintelDesign.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `gradiantFromColor` | `String` |
| `gradiantToColor` | `String` |
| `textColor` | `String` |
| `file` | `Upload` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `FintelDesign` | `src/modules/fintelDesign/fintelDesign.graphql` |
| `FintelDesignConnection` | `src/modules/fintelDesign/fintelDesign.graphql` |
| `FintelDesignEdge` | `src/modules/fintelDesign/fintelDesign.graphql` |

<details><summary>Show definition details</summary>


**type FintelDesign**  
Defined in `src/modules/fintelDesign/fintelDesign.graphql`

_Referenced by:_
- `type` `FintelDesignEdge` (fintelDesign) in `src/modules/fintelDesign/fintelDesign.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `description` | `String` |
| `file_id` | `String` |
| `gradiantFromColor` | `String` |
| `gradiantToColor` | `String` |
| `textColor` | `String` |


**type FintelDesignConnection**  
Defined in `src/modules/fintelDesign/fintelDesign.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[FintelDesignEdge!]!` |


**type FintelDesignEdge**  
Defined in `src/modules/fintelDesign/fintelDesign.graphql`

_Referenced by:_
- `type` `FintelDesignConnection` (fintelDesign) in `src/modules/fintelDesign/fintelDesign.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `FintelDesign` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **11**
- Definition count: **12**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `FintelDesignOrdering`

</details>


#### Expanded: input (3)
<a id="expanded-input-3"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `FintelDesignAddInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `Upload`

</details>


#### Expanded: type (5)
<a id="expanded-type-5"></a>

<details><summary>Show names</summary>

- `FintelDesign`
- `FintelDesignConnection`
- `FintelDesignEdge`
- `Metric`
- `PageInfo`

</details>



## Module family: fintelTemplate
<a id="module-family-finteltemplate"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 1 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/fintelTemplate/fintelTemplate.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (1)
<a id="query-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| fintelTemplate | `FintelTemplate` | `fintelTemplate(id: ID!): FintelTemplate @auth(for: [KNOWLEDGE, SETTINGS_SETCUSTOMIZATION])` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| fintelTemplateAdd | `FintelTemplate` | `fintelTemplateAdd(input: FintelTemplateAddInput!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateConfigurationImport | `FintelTemplate` | `fintelTemplateConfigurationImport(file: Upload!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateDelete | `ID` | `fintelTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| fintelTemplateFieldPatch | `FintelTemplate` | `fintelTemplateFieldPatch(id: ID!, input: [EditInput!]!): FintelTemplate @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `FintelTemplateOrdering` | `src/modules/fintelTemplate/fintelTemplate.graphql` |

<details><summary>Show definition details</summary>


**enum FintelTemplateOrdering**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

Values:
- `name`
- `start_date`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `FintelTemplateAddInput` | `src/modules/fintelTemplate/fintelTemplate.graphql` |
| `FintelTemplateWidgetAddInput` | `src/modules/fintelTemplate/fintelTemplate.graphql` |

<details><summary>Show definition details</summary>


**input FintelTemplateAddInput**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `settings_types` | `[String!]!` |
| `instance_filters` | `String` |
| `template_content` | `String` |
| `start_date` | `DateTime` |
| `fintel_template_widgets` | `[FintelTemplateWidgetAddInput!]` |


**input FintelTemplateWidgetAddInput**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

_Referenced by:_
- `input` `FintelTemplateAddInput` (fintelTemplate) in `src/modules/fintelTemplate/fintelTemplate.graphql`

| Field | Type |
| --- | --- |
| `variable_name` | `String!` |
| `widget` | `Any!` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `FintelTemplate` | `src/modules/fintelTemplate/fintelTemplate.graphql` |
| `FintelTemplateConnection` | `src/modules/fintelTemplate/fintelTemplate.graphql` |
| `FintelTemplateEdge` | `src/modules/fintelTemplate/fintelTemplate.graphql` |
| `FintelTemplateWidget` | `src/modules/fintelTemplate/fintelTemplate.graphql` |

<details><summary>Show definition details</summary>


**type FintelTemplate**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

_Referenced by:_
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `FintelTemplateEdge` (fintelTemplate) in `src/modules/fintelTemplate/fintelTemplate.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Task` (task) in `src/modules/task/task.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `settings_types` | `[String!]!` |
| `description` | `String` |
| `instance_filters` | `String` |
| `template_content` | `String!` |
| `start_date` | `DateTime` |
| `fintel_template_widgets` | `[FintelTemplateWidget!]!` |
| `toConfigurationExport` | `String!` |


**type FintelTemplateConnection**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

_Referenced by:_
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[FintelTemplateEdge!]!` |


**type FintelTemplateEdge**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

_Referenced by:_
- `type` `FintelTemplateConnection` (fintelTemplate) in `src/modules/fintelTemplate/fintelTemplate.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `FintelTemplate!` |


**type FintelTemplateWidget**  
Defined in `src/modules/fintelTemplate/fintelTemplate.graphql`

_Referenced by:_
- `type` `FintelTemplate` (fintelTemplate) in `src/modules/fintelTemplate/fintelTemplate.graphql`

| Field | Type |
| --- | --- |
| `variable_name` | `String!` |
| `widget` | `Widget!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **10**
- Definition count: **20**


#### Expanded: enum (3)
<a id="expanded-enum-3"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `FintelTemplateOrdering`
- `WidgetPerspective`

</details>


#### Expanded: input (3)
<a id="expanded-input-3"></a>

<details><summary>Show names</summary>

- `EditInput`
- `FintelTemplateAddInput`
- `FintelTemplateWidgetAddInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `Upload`

</details>


#### Expanded: type (11)
<a id="expanded-type-11"></a>

<details><summary>Show names</summary>

- `FintelTemplate`
- `FintelTemplateConnection`
- `FintelTemplateEdge`
- `FintelTemplateWidget`
- `Metric`
- `PageInfo`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`

</details>



## Module family: form
<a id="module-family-form"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 5 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/form/form.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| form | `Form` | `form(id: ID!): Form @auth(for: [KNOWLEDGE_KNUPDATE])` |
| forms | `FormConnection` | `forms( search: String first: Int after: ID orderBy: FormsOrdering orderMode: OrderingMode filters: FilterGroup ): FormConnection @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Mutation (5)
<a id="mutation-5"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| formAdd | `Form` | `formAdd(input: FormAddInput!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formDelete | `ID` | `formDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| formFieldPatch | `Form` | `formFieldPatch(id: ID!, input: [EditInput!]!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formImport | `Form` | `formImport(file: Upload!): Form @auth(for: [INGESTION_SETINGESTIONS])` |
| formSubmit | `FormSubmissionResponse` | `formSubmit(input: FormSubmissionInput!, isDraft: Boolean! = false): FormSubmissionResponse @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `FormsOrdering` | `src/modules/form/form.graphql` |

<details><summary>Show definition details</summary>


**enum FormsOrdering**  
Defined in `src/modules/form/form.graphql`

Values:
- `_score`
- `name`
- `created_at`
- `updated_at`
- `active`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `FormAddInput` | `src/modules/form/form.graphql` |
| `FormSubmissionInput` | `src/modules/form/form.graphql` |

<details><summary>Show definition details</summary>


**input FormAddInput**  
Defined in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String!` |
| `form_schema` | `String!` |
| `active` | `Boolean` |


**input FormSubmissionInput**  
Defined in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `formId` | `String!` |
| `values` | `String!` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `Form` | `src/modules/form/form.graphql` |
| `FormConnection` | `src/modules/form/form.graphql` |
| `FormEdge` | `src/modules/form/form.graphql` |
| `FormSubmissionResponse` | `src/modules/form/form.graphql` |

<details><summary>Show definition details</summary>


**type Form**  
Defined in `src/modules/form/form.graphql`

_Referenced by:_
- `type` `FormEdge` (form) in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `name` | `String!` |
| `description` | `String!` |
| `form_schema` | `String!` |
| `active` | `Boolean!` |
| `toConfigurationExport` | `String!` |


**type FormConnection**  
Defined in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[FormEdge!]!` |


**type FormEdge**  
Defined in `src/modules/form/form.graphql`

_Referenced by:_
- `type` `FormConnection` (form) in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Form!` |


**type FormSubmissionResponse**  
Defined in `src/modules/form/form.graphql`

| Field | Type |
| --- | --- |
| `success` | `Boolean!` |
| `bundleId` | `String` |
| `message` | `String` |
| `entityId` | `String` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **12**
- Definition count: **14**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `FormsOrdering`

</details>


#### Expanded: input (3)
<a id="expanded-input-3"></a>

<details><summary>Show names</summary>

- `EditInput`
- `FormAddInput`
- `FormSubmissionInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `Upload`

</details>


#### Expanded: type (6)
<a id="expanded-type-6"></a>

<details><summary>Show names</summary>

- `Form`
- `FormConnection`
- `FormEdge`
- `FormSubmissionResponse`
- `Metric`
- `PageInfo`

</details>



## Module family: grouping
<a id="module-family-grouping"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 6 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/grouping/grouping.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (6)
<a id="query-6"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| grouping | `Grouping` | `grouping(id: String!): Grouping @auth(for: [KNOWLEDGE])` |
| groupingContainsStixObjectOrStixRelationship | `Boolean` | `groupingContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| groupings | `GroupingConnection` | `groupings( first: Int after: ID orderBy: GroupingsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): GroupingConnection @auth(for: [KNOWLEDGE])` |
| groupingsDistribution | `[Distribution]` | `groupingsDistribution( objectId: String authorId: String field: String! operation: StatsOperation! limit: Int order: String startDate: DateTime endDate: DateTime dateAttribute: String filters: FilterGroup search: String ): [Distribution] @auth(for: [KNOWLEDGE, EXPLORE])` |
| groupingsNumber | `Number` | `groupingsNumber( groupingContext: String objectId: String authorId: String endDate: DateTime filters: FilterGroup ): Number @auth(for: [KNOWLEDGE, EXPLORE])` |
| groupingsTimeSeries | `[TimeSeries]` | `groupingsTimeSeries( objectId: String authorId: String groupingType: String field: String! operation: StatsOperation! startDate: DateTime! endDate: DateTime! interval: String! filters: FilterGroup search: String ): [TimeSeries] @auth(for: [KNOWLEDGE, EXPLORE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| groupingAdd | `Grouping` | `groupingAdd(input: GroupingAddInput!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingContextClean | `Grouping` | `groupingContextClean(id: ID!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingContextPatch | `Grouping` | `groupingContextPatch(id: ID!, input: EditContext): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingDelete | `ID` | `groupingDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| groupingFieldPatch | `Grouping` | `groupingFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingRelationAdd | `StixRefRelationship` | `groupingRelationAdd(id: ID!, input: StixRefRelationshipAddInput): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| groupingRelationDelete | `Grouping` | `groupingRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Grouping @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `GroupingsOrdering` | `src/modules/grouping/grouping.graphql` |

<details><summary>Show definition details</summary>


**enum GroupingsOrdering**  
Defined in `src/modules/grouping/grouping.graphql`

Values:
- `name`
- `created`
- `modified`
- `context`
- `created_at`
- `updated_at`
- `createdBy`
- `objectMarking`
- `x_opencti_workflow_id`
- `creator`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `GroupingAddInput` | `src/modules/grouping/grouping.graphql` |

<details><summary>Show definition details</summary>


**input GroupingAddInput**  
Defined in `src/modules/grouping/grouping.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `context` | `String!` |
| `x_opencti_aliases` | `[String]` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `confidence` | `Int` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `objectOrganization` | `[String]` |
| `externalReferences` | `[String]` |
| `objects` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `authorized_members` | `[MemberAccessInput!]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Grouping` | `src/modules/grouping/grouping.graphql` |
| `GroupingConnection` | `src/modules/grouping/grouping.graphql` |
| `GroupingEdge` | `src/modules/grouping/grouping.graphql` |

<details><summary>Show definition details</summary>


**type Grouping**  
Defined in `src/modules/grouping/grouping.graphql`

_Referenced by:_
- `type` `GroupingEdge` (grouping) in `src/modules/grouping/grouping.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `name` | `String!` |
| `description` | `String` |
| `content` | `String` |
| `content_mapping` | `String` |
| `context` | `String!` |
| `x_opencti_aliases` | `[String]` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `securityCoverage` | `SecurityCoverage` |


**type GroupingConnection**  
Defined in `src/modules/grouping/grouping.graphql`

_Referenced by:_
- `type` `AdministrativeArea` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `Channel` (channel) in `src/modules/channel/channel.graphql`
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`
- `type` `Event` (event) in `src/modules/event/event.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `Language` (language) in `src/modules/language/language.graphql`
- `type` `MalwareAnalysis` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- `type` `SecurityCoverage` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`
- `type` `SecurityPlatform` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`
- `type` `Task` (task) in `src/modules/task/task.graphql`
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[GroupingEdge]` |


**type GroupingEdge**  
Defined in `src/modules/grouping/grouping.graphql`

_Referenced by:_
- `type` `GroupingConnection` (grouping) in `src/modules/grouping/grouping.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Grouping!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **20**
- Definition count: **143**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `GroupingsOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (5)
<a id="expanded-input-5"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `GroupingAddInput`
- `MemberAccessInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (114)
<a id="expanded-type-114"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TimeSeries`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: indicator
<a id="module-family-indicator"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 5 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/indicator/indicator.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (5)
<a id="query-5"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| indicator | `Indicator` | `indicator(id: String!): Indicator @auth(for: [KNOWLEDGE])` |
| indicators | `IndicatorConnection` | `indicators( first: Int after: ID orderBy: IndicatorsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): IndicatorConnection @auth(for: [KNOWLEDGE])` |
| indicatorsDistribution | `[Distribution]` | `indicatorsDistribution( objectId: String field: String! operation: StatsOperation! limit: Int order: String startDate: DateTime endDate: DateTime dateAttribute: String ): [Distribution] @auth(for: [KNOWLEDGE])` |
| indicatorsNumber | `Number` | `indicatorsNumber(pattern_type: String, objectId: String, endDate: DateTime): Number @auth(for: [KNOWLEDGE])` |
| indicatorsTimeSeries | `[TimeSeries]` | `indicatorsTimeSeries( objectId: String field: String! operation: StatsOperation! startDate: DateTime! endDate: DateTime! interval: String! filters: FilterGroup ): [TimeSeries] @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| indicatorAdd | `Indicator` | `indicatorAdd(input: IndicatorAddInput!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorContextClean | `Indicator` | `indicatorContextClean(id: ID!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorContextPatch | `Indicator` | `indicatorContextPatch(id: ID!, input: EditContext): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorDelete | `ID` | `indicatorDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| indicatorFieldPatch | `Indicator` | `indicatorFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorRelationAdd | `StixRefRelationship` | `indicatorRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| indicatorRelationDelete | `Indicator` | `indicatorRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Indicator @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `IndicatorsOrdering` | `src/modules/indicator/indicator.graphql` |

<details><summary>Show definition details</summary>


**enum IndicatorsOrdering**  
Defined in `src/modules/indicator/indicator.graphql`

Values:
- `pattern_type`
- `pattern_version`
- `pattern`
- `name`
- `indicator_types`
- `valid_from`
- `valid_until`
- `x_opencti_score`
- `x_opencti_detection`
- `confidence`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `x_opencti_workflow_id`
- `objectMarking`
- `creator`
- `createdBy`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `IndicatorAddInput` | `src/modules/indicator/indicator.graphql` |

<details><summary>Show definition details</summary>


**input IndicatorAddInput**  
Defined in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId!]` |
| `pattern_type` | `String!` |
| `pattern_version` | `String` |
| `pattern` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `indicator_types` | `[String!]` |
| `valid_from` | `DateTime` |
| `valid_until` | `DateTime` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `x_opencti_score` | `Int` |
| `x_opencti_detection` | `Boolean` |
| `x_opencti_main_observable_type` | `String` |
| `x_mitre_platforms` | `[String!]` |
| `killChainPhases` | `[String!]` |
| `createdBy` | `String` |
| `objectMarking` | `[String!]` |
| `objectLabel` | `[String!]` |
| `objectOrganization` | `[String!]` |
| `externalReferences` | `[String!]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `createObservables` | `Boolean` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `basedOn` | `[String!]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (9)
<a id="type-9"></a>

| Name | File |
| --- | --- |
| `DecayChartData` | `src/modules/indicator/indicator.graphql` |
| `DecayHistory` | `src/modules/indicator/indicator.graphql` |
| `DecayLiveDetails` | `src/modules/indicator/indicator.graphql` |
| `Indicator` | `src/modules/indicator/indicator.graphql` |
| `IndicatorConnection` | `src/modules/indicator/indicator.graphql` |
| `IndicatorDecayExclusionRule` | `src/modules/indicator/indicator.graphql` |
| `IndicatorDecayRule` | `src/modules/indicator/indicator.graphql` |
| `IndicatorEdge` | `src/modules/indicator/indicator.graphql` |
| `ObservablesValues` | `src/modules/indicator/indicator.graphql` |

<details><summary>Show definition details</summary>


**type DecayChartData**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `live_score_serie` | `[DecayHistory!]` |


**type DecayHistory**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `DecayData` (decayRule) in `src/modules/decayRule/decayRule.graphql`
- `type` `DecayChartData` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `DecayLiveDetails` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `score` | `Int!` |


**type DecayLiveDetails**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `live_score` | `Int!` |
| `live_points` | `[DecayHistory!]` |


**type Indicator**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `IndicatorEdge` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `pattern_type` | `String` |
| `pattern_version` | `String` |
| `pattern` | `String` |
| `name` | `String!` |
| `description` | `String` |
| `indicator_types` | `[String]` |
| `valid_from` | `DateTime` |
| `valid_until` | `DateTime` |
| `x_opencti_score` | `Int` |
| `x_opencti_detection` | `Boolean` |
| `x_opencti_main_observable_type` | `String` |
| `x_opencti_observable_values` | `[ObservablesValues!]` |
| `x_mitre_platforms` | `[String]` |
| `killChainPhases` | `[KillChainPhase!]` |
| `observables(first: Int)` | `StixCyberObservableConnection` |
| `decay_base_score` | `Int` |
| `decay_base_score_date` | `DateTime` |
| `decay_applied_rule` | `IndicatorDecayRule` |
| `decay_exclusion_applied_rule` | `IndicatorDecayExclusionRule` |
| `decay_history` | `[DecayHistory!]` |
| `decayLiveDetails` | `DecayLiveDetails` |
| `decayChartData` | `DecayChartData` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type IndicatorConnection**  
Defined in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IndicatorEdge]` |


**type IndicatorDecayExclusionRule**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `decay_exclusion_id` | `String!` |
| `decay_exclusion_name` | `String!` |
| `decay_exclusion_created_at` | `DateTime!` |
| `decay_exclusion_filters` | `String!` |


**type IndicatorDecayRule**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `decay_rule_id` | `String` |
| `decay_lifetime` | `Int!` |
| `decay_pound` | `Float!` |
| `decay_points` | `[Int!]` |
| `decay_revoke_score` | `Int!` |


**type IndicatorEdge**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `IndicatorConnection` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Indicator!` |


**type ObservablesValues**  
Defined in `src/modules/indicator/indicator.graphql`

_Referenced by:_
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`

| Field | Type |
| --- | --- |
| `type` | `String` |
| `value` | `String` |
| `hashes` | `[Hash!]` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **25**
- Definition count: **155**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `IndicatorsOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `IndicatorAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (6)
<a id="expanded-interface-6"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixCyberObservable`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (126)
<a id="expanded-type-126"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DecayChartData`
- `DecayHistory`
- `DecayLiveDetails`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Hash`
- `Indicator`
- `IndicatorConnection`
- `IndicatorDecayExclusionRule`
- `IndicatorDecayRule`
- `IndicatorEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservablesValues`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixCyberObservableConnection`
- `StixCyberObservableEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TimeSeries`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: ingestion
<a id="module-family-ingestion"></a>

| Metric | Value |
| --- | --- |
| SDL files | 5 |
| Query fields | 14 |
| Mutation fields | 23 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/ingestion/ingestion-csv.graphql` |
| `src/modules/ingestion/ingestion-json.graphql` |
| `src/modules/ingestion/ingestion-rss.graphql` |
| `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `src/modules/ingestion/ingestion-taxii.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (14)
<a id="query-14"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| csvFeedAddInputFromImport | `CSVFeedAddInputFromImport!` | `csvFeedAddInputFromImport( file: Upload! ): CSVFeedAddInputFromImport! @auth(for: [INGESTION, CSVMAPPERS])` |
| defaultIngestionGroupCount | `Int` | `defaultIngestionGroupCount: Int @auth(for: [INGESTION])` |
| ingestionCsv | `IngestionCsv` | `ingestionCsv(id: String!): IngestionCsv @auth(for: [INGESTION])` |
| ingestionCsvs | `IngestionCsvConnection` | `ingestionCsvs( first: Int after: ID orderBy: IngestionCsvOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionCsvConnection @auth(for: [INGESTION])` |
| ingestionJson | `IngestionJson` | `ingestionJson(id: String!): IngestionJson @auth(for: [INGESTION])` |
| ingestionJsons | `IngestionJsonConnection` | `ingestionJsons( first: Int after: ID orderBy: IngestionJsonOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionJsonConnection @auth(for: [INGESTION])` |
| ingestionRss | `IngestionRss` | `ingestionRss(id: String!): IngestionRss @auth(for: [INGESTION])` |
| ingestionRsss | `IngestionRssConnection` | `ingestionRsss( first: Int after: ID orderBy: IngestionRssOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionRssConnection @auth(for: [INGESTION])` |
| ingestionTaxii | `IngestionTaxii` | `ingestionTaxii(id: String!): IngestionTaxii @auth(for: [INGESTION])` |
| ingestionTaxiiCollection | `IngestionTaxiiCollection` | `ingestionTaxiiCollection(id: String!): IngestionTaxiiCollection @auth(for: [INGESTION])` |
| ingestionTaxiiCollections | `IngestionTaxiiCollectionConnection` | `ingestionTaxiiCollections( first: Int after: ID orderBy: IngestionTaxiiCollectionOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionTaxiiCollectionConnection @auth(for: [INGESTION])` |
| ingestionTaxiis | `IngestionTaxiiConnection` | `ingestionTaxiis( first: Int after: ID orderBy: IngestionTaxiiOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): IngestionTaxiiConnection @auth(for: [INGESTION])` |
| taxiiFeedAddInputFromImport | `TaxiiFeedAddInputFromImport!` | `taxiiFeedAddInputFromImport( file: Upload! ): TaxiiFeedAddInputFromImport! @auth(for: [INGESTION])` |
| userAlreadyExists | `Boolean` | `userAlreadyExists(name: String!): Boolean @auth(for: [INGESTION])` |


#### Mutation (23)
<a id="mutation-23"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| ingestionCsvAdd | `IngestionCsv` | `ingestionCsvAdd(input: IngestionCsvAddInput!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvAddAutoUser | `IngestionCsv` | `ingestionCsvAddAutoUser(id: ID!, input: IngestionCsvAddAutoUserInput!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvDelete | `ID` | `ingestionCsvDelete(id: ID!): ID @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvFieldPatch | `IngestionCsv` | `ingestionCsvFieldPatch(id: ID!, input: [EditInput!]!): IngestionCsv @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionCsvResetState | `IngestionCsv` | `ingestionCsvResetState(id: ID!): IngestionCsv @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionCsvTester | `CsvMapperTestResult` | `ingestionCsvTester(input: IngestionCsvAddInput!): CsvMapperTestResult @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonAdd | `IngestionJson` | `ingestionJsonAdd(input: IngestionJsonAddInput!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonDelete | `ID` | `ingestionJsonDelete(id: ID!): ID @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonEdit | `IngestionJson` | `ingestionJsonEdit(id: ID!, input: IngestionJsonAddInput!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonFieldPatch | `IngestionJson` | `ingestionJsonFieldPatch(id: ID!, input: [EditInput!]!): IngestionJson @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionJsonResetState | `IngestionJson` | `ingestionJsonResetState(id: ID!): IngestionJson @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionJsonTester | `JsonMapperTestResult` | `ingestionJsonTester(input: IngestionJsonAddInput!): JsonMapperTestResult @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| ingestionRssAdd | `IngestionRss` | `ingestionRssAdd(input: IngestionRssAddInput!): IngestionRss @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionRssDelete | `ID` | `ingestionRssDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionRssFieldPatch | `IngestionRss` | `ingestionRssFieldPatch(id: ID!, input: [EditInput!]!): IngestionRss @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiAdd | `IngestionTaxii` | `ingestionTaxiiAdd(input: IngestionTaxiiAddInput!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiAddAutoUser | `IngestionTaxii` | `ingestionTaxiiAddAutoUser(id: ID!, input: IngestionTaxiiAddAutoUserInput!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionAdd | `IngestionTaxiiCollection` | `ingestionTaxiiCollectionAdd(input: IngestionTaxiiCollectionAddInput!): IngestionTaxiiCollection @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionDelete | `ID` | `ingestionTaxiiCollectionDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiCollectionFieldPatch | `IngestionTaxiiCollection` | `ingestionTaxiiCollectionFieldPatch(id: ID!, input: [EditInput!]!): IngestionTaxiiCollection @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiDelete | `ID` | `ingestionTaxiiDelete(id: ID!): ID @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiFieldPatch | `IngestionTaxii` | `ingestionTaxiiFieldPatch(id: ID!, input: [EditInput!]!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |
| ingestionTaxiiResetState | `IngestionTaxii` | `ingestionTaxiiResetState(id: ID!): IngestionTaxii @auth(for: [INGESTION_SETINGESTIONS])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (8)
<a id="enum-8"></a>

| Name | File |
| --- | --- |
| `IngestionAuthType` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `IngestionCsvMapperType` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionCsvOrdering` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionJsonOrdering` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionRssOrdering` | `src/modules/ingestion/ingestion-rss.graphql` |
| `IngestionTaxiiCollectionOrdering` | `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `IngestionTaxiiOrdering` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `TaxiiVersion` | `src/modules/ingestion/ingestion-taxii.graphql` |

<details><summary>Show definition details</summary>


**enum IngestionAuthType**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

_Referenced by:_
- `input` `IngestionCsvAddInput` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `input` `IngestionJsonAddInput` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`
- `input` `IngestionTaxiiAddInput` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`
- `type` `IngestionCsv` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `type` `IngestionJson` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`
- `type` `IngestionTaxii` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`

Values:
- `none`
- `basic`
- `bearer`
- `certificate`


**enum IngestionCsvMapperType**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

_Referenced by:_
- `input` `IngestionCsvAddInput` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `type` `CSVFeedAddInputFromImport` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `type` `IngestionCsv` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`

Values:
- `inline`
- `id`


**enum IngestionCsvOrdering**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `uri`
- `mapper`
- `_score`


**enum IngestionJsonOrdering**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `uri`
- `mapper`
- `_score`


**enum IngestionRssOrdering**  
Defined in `src/modules/ingestion/ingestion-rss.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `uri`
- `_score`


**enum IngestionTaxiiCollectionOrdering**  
Defined in `src/modules/ingestion/ingestion-taxii-collection.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `_score`


**enum IngestionTaxiiOrdering**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `uri`
- `version`
- `_score`


**enum TaxiiVersion**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

_Referenced by:_
- `input` `IngestionTaxiiAddInput` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`
- `type` `IngestionTaxii` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`

Values:
- `v1`
- `v2`
- `v21`


</details>


#### input (9)
<a id="input-9"></a>

| Name | File |
| --- | --- |
| `HeaderInput` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionCsvAddAutoUserInput` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionCsvAddInput` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionJsonAddInput` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionRssAddInput` | `src/modules/ingestion/ingestion-rss.graphql` |
| `IngestionTaxiiAddAutoUserInput` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `IngestionTaxiiAddInput` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `IngestionTaxiiCollectionAddInput` | `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `QueryAttribute` | `src/modules/ingestion/ingestion-json.graphql` |

<details><summary>Show definition details</summary>


**input HeaderInput**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `input` `IngestionJsonAddInput` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `value` | `String!` |


**input IngestionCsvAddAutoUserInput**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `user_name` | `String!` |
| `confidence_level` | `Int!` |


**input IngestionCsvAddInput**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `current_state_date` | `DateTime` |
| `uri` | `String!` |
| `csv_mapper_id` | `String` |
| `csv_mapper` | `String` |
| `csv_mapper_type` | `IngestionCsvMapperType` |
| `ingestion_running` | `Boolean` |
| `user_id` | `String!` |
| `automatic_user` | `Boolean` |
| `confidence_level` | `Int` |
| `markings` | `[String!]` |


**input IngestionJsonAddInput**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `current_state_date` | `DateTime` |
| `uri` | `String!` |
| `verb` | `String!` |
| `body` | `String` |
| `pagination_with_sub_page` | `Boolean` |
| `pagination_with_sub_page_attribute_path` | `String` |
| `pagination_with_sub_page_query_verb` | `String` |
| `headers` | `[HeaderInput!]` |
| `query_attributes` | `[QueryAttribute!]` |
| `json_mapper_id` | `String!` |
| `ingestion_running` | `Boolean` |
| `user_id` | `String!` |
| `markings` | `[String!]` |


**input IngestionRssAddInput**  
Defined in `src/modules/ingestion/ingestion-rss.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `uri` | `String!` |
| `current_state_date` | `DateTime` |
| `ingestion_running` | `Boolean` |
| `user_id` | `String` |
| `created_by_ref` | `String` |
| `object_marking_refs` | `[String!]` |
| `report_types` | `[String!]` |


**input IngestionTaxiiAddAutoUserInput**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `user_name` | `String!` |
| `confidence_level` | `Int!` |


**input IngestionTaxiiAddInput**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `version` | `TaxiiVersion!` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `uri` | `String!` |
| `collection` | `String!` |
| `added_after_start` | `DateTime` |
| `ingestion_running` | `Boolean` |
| `confidence_to_score` | `Boolean` |
| `user_id` | `String!` |
| `automatic_user` | `Boolean` |
| `confidence_level` | `Int` |


**input IngestionTaxiiCollectionAddInput**  
Defined in `src/modules/ingestion/ingestion-taxii-collection.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `user_id` | `String` |
| `confidence_to_score` | `Boolean` |
| `authorized_members` | `[MemberAccessInput!]!` |


**input QueryAttribute**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `input` `IngestionJsonAddInput` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `type` | `String` |
| `from` | `String` |
| `to` | `String` |
| `data_operation` | `String` |
| `state_operation` | `String` |
| `default` | `String` |
| `exposed` | `String` |


</details>


#### type (19)
<a id="type-19"></a>

| Name | File |
| --- | --- |
| `CSVFeedAddInputFromImport` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionCsv` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionCsvConnection` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionCsvEdge` | `src/modules/ingestion/ingestion-csv.graphql` |
| `IngestionHeader` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionJson` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionJsonConnection` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionJsonEdge` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionQueryAttribute` | `src/modules/ingestion/ingestion-json.graphql` |
| `IngestionRss` | `src/modules/ingestion/ingestion-rss.graphql` |
| `IngestionRssConnection` | `src/modules/ingestion/ingestion-rss.graphql` |
| `IngestionRssEdge` | `src/modules/ingestion/ingestion-rss.graphql` |
| `IngestionTaxii` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `IngestionTaxiiCollection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `IngestionTaxiiCollectionConnection` | `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `IngestionTaxiiCollectionEdge` | `src/modules/ingestion/ingestion-taxii-collection.graphql` |
| `IngestionTaxiiConnection` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `IngestionTaxiiEdge` | `src/modules/ingestion/ingestion-taxii.graphql` |
| `TaxiiFeedAddInputFromImport` | `src/modules/ingestion/ingestion-taxii.graphql` |

<details><summary>Show definition details</summary>


**type CSVFeedAddInputFromImport**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String!` |
| `uri` | `String!` |
| `authentication_type` | `String!` |
| `markings` | `[String!]!,` |
| `authentication_value` | `String!,` |
| `csvMapper` | `CsvMapperAddInputFromImport!` |
| `csv_mapper_type` | `IngestionCsvMapperType` |
| `scheduling_period` | `String` |


**type IngestionCsv**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

_Referenced by:_
- `type` `IngestionCsvEdge` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `uri` | `String!` |
| `csv_mapper_type` | `IngestionCsvMapperType` |
| `csvMapper` | `CsvMapper!` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `user_id` | `String!` |
| `user` | `Creator` |
| `ingestion_running` | `Boolean` |
| `current_state_hash` | `String` |
| `current_state_date` | `DateTime` |
| `last_execution_date` | `DateTime` |
| `markings` | `[String!]` |
| `toConfigurationExport` | `String!` |
| `duplicateCsvMapper` | `CsvMapper!` |


**type IngestionCsvConnection**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IngestionCsvEdge!]!` |


**type IngestionCsvEdge**  
Defined in `src/modules/ingestion/ingestion-csv.graphql`

_Referenced by:_
- `type` `IngestionCsvConnection` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `IngestionCsv!` |


**type IngestionHeader**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `type` `IngestionJson` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `value` | `String!` |


**type IngestionJson**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `type` `IngestionJsonEdge` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `connector_id` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `uri` | `String!` |
| `verb` | `String!` |
| `body` | `String` |
| `pagination_with_sub_page` | `Boolean` |
| `pagination_with_sub_page_attribute_path` | `String` |
| `pagination_with_sub_page_query_verb` | `String` |
| `headers` | `[IngestionHeader!]` |
| `query_attributes` | `[IngestionQueryAttribute!]` |
| `jsonMapper` | `JsonMapper!` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `user_id` | `String!` |
| `user` | `Creator` |
| `ingestion_running` | `Boolean` |
| `last_execution_date` | `DateTime` |
| `markings` | `[String!]` |


**type IngestionJsonConnection**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IngestionJsonEdge!]!` |


**type IngestionJsonEdge**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `type` `IngestionJsonConnection` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `IngestionJson!` |


**type IngestionQueryAttribute**  
Defined in `src/modules/ingestion/ingestion-json.graphql`

_Referenced by:_
- `type` `IngestionJson` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`

| Field | Type |
| --- | --- |
| `type` | `String` |
| `from` | `String` |
| `to` | `String` |
| `data_operation` | `String` |
| `state_operation` | `String` |
| `default` | `String` |
| `exposed` | `String` |


**type IngestionRss**  
Defined in `src/modules/ingestion/ingestion-rss.graphql`

_Referenced by:_
- `type` `IngestionRssEdge` (ingestion) in `src/modules/ingestion/ingestion-rss.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `scheduling_period` | `String` |
| `uri` | `String!` |
| `user` | `Creator` |
| `defaultCreatedBy` | `Identity` |
| `defaultMarkingDefinitions` | `[MarkingDefinition]` |
| `report_types` | `[String!]` |
| `current_state_date` | `DateTime` |
| `last_execution_date` | `DateTime` |
| `ingestion_running` | `Boolean` |


**type IngestionRssConnection**  
Defined in `src/modules/ingestion/ingestion-rss.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IngestionRssEdge!]!` |


**type IngestionRssEdge**  
Defined in `src/modules/ingestion/ingestion-rss.graphql`

_Referenced by:_
- `type` `IngestionRssConnection` (ingestion) in `src/modules/ingestion/ingestion-rss.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `IngestionRss!` |


**type IngestionTaxii**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

_Referenced by:_
- `type` `IngestionTaxiiEdge` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `uri` | `String!` |
| `collection` | `String!` |
| `version` | `TaxiiVersion!` |
| `authentication_type` | `IngestionAuthType!` |
| `authentication_value` | `String` |
| `user_id` | `String` |
| `user` | `Creator` |
| `current_state_cursor` | `String` |
| `added_after_start` | `DateTime` |
| `ingestion_running` | `Boolean` |
| `last_execution_date` | `DateTime` |
| `confidence_to_score` | `Boolean` |
| `toConfigurationExport` | `String!` |


**type IngestionTaxiiCollection**  
Defined in `src/modules/ingestion/ingestion-taxii-collection.graphql`

_Referenced by:_
- `type` `IngestionTaxiiCollectionEdge` (ingestion) in `src/modules/ingestion/ingestion-taxii-collection.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `ingestion_running` | `Boolean` |
| `confidence_to_score` | `Boolean` |
| `description` | `String` |
| `user_id` | `String` |
| `user` | `Creator` |
| `authorized_members` | `[MemberAccess!]` |


**type IngestionTaxiiCollectionConnection**  
Defined in `src/modules/ingestion/ingestion-taxii-collection.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IngestionTaxiiCollectionEdge!]!` |


**type IngestionTaxiiCollectionEdge**  
Defined in `src/modules/ingestion/ingestion-taxii-collection.graphql`

_Referenced by:_
- `type` `IngestionTaxiiCollectionConnection` (ingestion) in `src/modules/ingestion/ingestion-taxii-collection.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `IngestionTaxiiCollection!` |


**type IngestionTaxiiConnection**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[IngestionTaxiiEdge!]!` |


**type IngestionTaxiiEdge**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

_Referenced by:_
- `type` `IngestionTaxiiConnection` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `IngestionTaxii!` |


**type TaxiiFeedAddInputFromImport**  
Defined in `src/modules/ingestion/ingestion-taxii.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String!` |
| `uri` | `String!` |
| `version` | `String!` |
| `collection` | `String!` |
| `authentication_type` | `String!` |
| `authentication_value` | `String!` |
| `added_after_start` | `String` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **49**
- Definition count: **199**


#### Expanded: enum (21)
<a id="expanded-enum-21"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `CsvMapperOperator`
- `CsvMapperRepresentationType`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `IngestionAuthType`
- `IngestionCsvMapperType`
- `IngestionCsvOrdering`
- `IngestionJsonOrdering`
- `IngestionRssOrdering`
- `IngestionTaxiiCollectionOrdering`
- `IngestionTaxiiOrdering`
- `JsonMapperRepresentationType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `TaxiiVersion`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (11)
<a id="expanded-input-11"></a>

<details><summary>Show names</summary>

- `EditInput`
- `HeaderInput`
- `IngestionCsvAddAutoUserInput`
- `IngestionCsvAddInput`
- `IngestionJsonAddInput`
- `IngestionRssAddInput`
- `IngestionTaxiiAddAutoUserInput`
- `IngestionTaxiiAddInput`
- `IngestionTaxiiCollectionAddInput`
- `MemberAccessInput`
- `QueryAttribute`

</details>


#### Expanded: interface (6)
<a id="expanded-interface-6"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `JsonAttributeColumnConfiguration`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (4)
<a id="expanded-scalar-4"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `Upload`

</details>


#### Expanded: type (154)
<a id="expanded-type-154"></a>

<details><summary>Show names</summary>

- `Assignee`
- `AttributeBasedOn`
- `AttributeColumn`
- `AttributeColumnConfiguration`
- `AttributePath`
- `AttributeRef`
- `CSVFeedAddInputFromImport`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ComplexPath`
- `ComplexVariable`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `CsvMapper`
- `CsvMapperAddInputFromImport`
- `CsvMapperRepresentation`
- `CsvMapperRepresentationAttribute`
- `CsvMapperRepresentationTarget`
- `CsvMapperRepresentationTargetColumn`
- `CsvMapperTestResult`
- `DefaultMarking`
- `DefaultValue`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `IngestionCsv`
- `IngestionCsvConnection`
- `IngestionCsvEdge`
- `IngestionHeader`
- `IngestionJson`
- `IngestionJsonConnection`
- `IngestionJsonEdge`
- `IngestionQueryAttribute`
- `IngestionRss`
- `IngestionRssConnection`
- `IngestionRssEdge`
- `IngestionTaxii`
- `IngestionTaxiiCollection`
- `IngestionTaxiiCollectionConnection`
- `IngestionTaxiiCollectionEdge`
- `IngestionTaxiiConnection`
- `IngestionTaxiiEdge`
- `JsonComplexPath`
- `JsonComplexPathConfiguration`
- `JsonMapper`
- `JsonMapperRepresentation`
- `JsonMapperRepresentationAttribute`
- `JsonMapperRepresentationTarget`
- `JsonMapperTestResult`
- `JsonMapperVariable`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TaxiiFeedAddInputFromImport`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: internal
<a id="module-family-internal"></a>

| Metric | Value |
| --- | --- |
| SDL files | 3 |
| Query fields | 7 |
| Mutation fields | 9 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/internal/csvMapper/csvMapper.graphql` |
| `src/modules/internal/csvMapper/deprecated/csvMapper.graphql` |
| `src/modules/internal/jsonMapper/jsonMapper.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (7)
<a id="query-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| csvMapper | `CsvMapper` | `csvMapper(id: ID!): CsvMapper @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| csvMapperAddInputFromImport | `CsvMapperAddInputFromImport!` | `csvMapperAddInputFromImport( file: Upload! ): CsvMapperAddInputFromImport! @auth(for: [CSVMAPPERS])` |
| csvMapperSchemaAttributes | `[CsvMapperSchemaAttributes!]!` | `csvMapperSchemaAttributes: [CsvMapperSchemaAttributes!]! @auth(for: [KNOWLEDGE, CSVMAPPERS])` |
| csvMapperTest | `CsvMapperTestResult` | csvMapperTest( configuration: String! content: String! ): CsvMapperTestResult @auth(for: [CSVMAPPERS]) @deprecated(reason: "[>=6.4 & <6.7]. Use `csvMapperTest mutation`.") |
| csvMappers | `CsvMapperConnection` | `csvMappers( first: Int after: ID orderBy: CsvMapperOrdering orderMode: OrderingMode filters: FilterGroup search: String ): CsvMapperConnection @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| jsonMapper | `JsonMapper` | `jsonMapper(id: ID!): JsonMapper @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |
| jsonMappers | `JsonMapperConnection` | `jsonMappers( first: Int after: ID orderBy: JsonMapperOrdering orderMode: OrderingMode filters: FilterGroup search: String ): JsonMapperConnection @auth(for: [CSVMAPPERS, INGESTION_SETINGESTIONS])` |


#### Mutation (9)
<a id="mutation-9"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| csvMapperAdd | `CsvMapper` | `csvMapperAdd(input: CsvMapperAddInput!): CsvMapper @auth(for: [CSVMAPPERS])` |
| csvMapperDelete | `ID` | `csvMapperDelete(id: ID!): ID @auth(for: [CSVMAPPERS])` |
| csvMapperFieldPatch | `CsvMapper` | `csvMapperFieldPatch(id: ID!, input: [EditInput!]!): CsvMapper @auth(for: [CSVMAPPERS])` |
| csvMapperTest | `CsvMapperTestResult` | `csvMapperTest(configuration: String!, file: Upload!): CsvMapperTestResult @auth(for: [CSVMAPPERS])` |
| jsonMapperAdd | `JsonMapper` | `jsonMapperAdd(input: JsonMapperAddInput!): JsonMapper @auth(for: [CSVMAPPERS])` |
| jsonMapperDelete | `ID` | `jsonMapperDelete(id: ID!): ID @auth(for: [CSVMAPPERS])` |
| jsonMapperFieldPatch | `JsonMapper` | `jsonMapperFieldPatch(id: ID!, input: [EditInput!]!): JsonMapper @auth(for: [CSVMAPPERS])` |
| jsonMapperImport | `String!` | `jsonMapperImport(file: Upload!): String! @auth(for: [CSVMAPPERS])` |
| jsonMapperTest | `JsonMapperTestResult` | `jsonMapperTest(configuration: String!, file: Upload!): JsonMapperTestResult @auth(for: [CSVMAPPERS])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (5)
<a id="enum-5"></a>

| Name | File |
| --- | --- |
| `CsvMapperOperator` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperOrdering` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationType` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonMapperOrdering` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonMapperRepresentationType` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |

<details><summary>Show definition details</summary>


**enum CsvMapperOperator**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationTargetColumnInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `CsvMapperRepresentationTargetColumn` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

Values:
- `eq`
- `not_eq`


**enum CsvMapperOrdering**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

Values:
- `name`
- `_score`


**enum CsvMapperRepresentationType**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `CsvMapperRepresentation` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

Values:
- `entity`
- `relationship`


**enum JsonMapperOrdering**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

Values:
- `name`
- `_score`


**enum JsonMapperRepresentationType**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperRepresentation` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

Values:
- `entity`
- `relationship`


</details>


#### input (10)
<a id="input-10"></a>

| Name | File |
| --- | --- |
| `AttributeBasedOnInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributeColumnConfigurationInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributeColumnInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributeRefInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperAddInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationAttributeInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationTargetColumnInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationTargetInput` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonMapperAddInput` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |

<details><summary>Show definition details</summary>


**input AttributeBasedOnInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationAttributeInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `representations` | `[String]` |


**input AttributeColumnConfigurationInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `AttributeColumnInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `separator` | `String` |
| `pattern_date` | `String` |
| `timezone` | `String` |


**input AttributeColumnInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationAttributeInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `column_name` | `String` |
| `configuration` | `AttributeColumnConfigurationInput` |


**input AttributeRefInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationAttributeInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `multiple` | `Boolean` |
| `id` | `String` |
| `ids` | `[String]` |


**input CsvMapperAddInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `has_header` | `Boolean!` |
| `separator` | `String!` |
| `representations` | `String!` |
| `skipLineChar` | `String` |


**input CsvMapperRepresentationAttributeInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `key` | `String` |
| `column` | `AttributeColumnInput` |
| `based_on` | `AttributeBasedOnInput` |
| `ref` | `AttributeRefInput` |


**input CsvMapperRepresentationInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `type` | `CsvMapperRepresentationType!` |
| `target` | `CsvMapperRepresentationTargetInput!` |
| `attributes` | `[CsvMapperRepresentationAttributeInput]!` |
| `from` | `String` |
| `to` | `String` |


**input CsvMapperRepresentationTargetColumnInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationTargetInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `column_reference` | `String` |
| `operator` | `CsvMapperOperator` |
| `value` | `String` |


**input CsvMapperRepresentationTargetInput**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `input` `CsvMapperRepresentationInput` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `entity_type` | `String!` |
| `column_based` | `CsvMapperRepresentationTargetColumnInput` |


**input JsonMapperAddInput**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `representations` | `String!` |


</details>


#### interface (1)
<a id="interface-1"></a>

| Name | File |
| --- | --- |
| `JsonAttributeColumnConfiguration` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |

<details><summary>Show definition details</summary>


**interface JsonAttributeColumnConfiguration**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonComplexPath` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `separator` | `String` |
| `pattern_date` | `String` |
| `timezone` | `String` |


</details>


#### type (30)
<a id="type-30"></a>

| Name | File |
| --- | --- |
| `AttributeBasedOn` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributeColumn` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributeColumnConfiguration` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `AttributePath` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `AttributeRef` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `ComplexPath` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `ComplexVariable` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `CsvMapper` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperAddInputFromImport` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperConnection` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperEdge` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentation` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationAttribute` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationTarget` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperRepresentationTargetColumn` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperSchemaAttribute` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperSchemaAttributes` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `CsvMapperTestResult` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonComplexPath` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonComplexPathConfiguration` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonComplexPathVariable` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapper` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperAddInputFromImport` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperConnection` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonMapperEdge` | `src/modules/internal/csvMapper/csvMapper.graphql` |
| `JsonMapperRepresentation` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperRepresentationAttribute` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperRepresentationTarget` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperTestResult` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |
| `JsonMapperVariable` | `src/modules/internal/jsonMapper/jsonMapper.graphql` |

<details><summary>Show definition details</summary>


**type AttributeBasedOn**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentationAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `JsonMapperRepresentationAttribute` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `identifier` | `String` |
| `representations` | `[String]` |


**type AttributeColumn**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentationAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `column_name` | `String` |
| `configuration` | `AttributeColumnConfiguration` |


**type AttributeColumnConfiguration**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `AttributeColumn` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `AttributePath` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`
- `type` `ComplexPath` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `separator` | `String` |
| `pattern_date` | `String` |
| `timezone` | `String` |


**type AttributePath**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperRepresentationAttribute` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `path` | `String!` |
| `independent` | `Boolean` |
| `configuration` | `AttributeColumnConfiguration` |


**type AttributeRef**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentationAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `multiple` | `Boolean` |
| `id` | `String` |
| `ids` | `[String]` |


**type ComplexPath**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperRepresentationAttribute` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `formula` | `String!` |
| `variables` | `[ComplexVariable!]` |
| `configuration` | `AttributeColumnConfiguration` |


**type ComplexVariable**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `ComplexPath` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `path` | `String!` |
| `variable` | `String!` |
| `independent` | `Boolean` |


**type CsvMapper**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `IngestionCsv` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `type` `CsvMapperEdge` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `has_header` | `Boolean!` |
| `separator` | `String!` |
| `skipLineChar` | `String` |
| `representations` | `[CsvMapperRepresentation!]!` |
| `errors` | `String` |
| `toConfigurationExport` | `String!` |


**type CsvMapperAddInputFromImport**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CSVFeedAddInputFromImport` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `has_header` | `Boolean!` |
| `separator` | `String!` |
| `representations` | `[CsvMapperRepresentation!]!` |
| `skipLineChar` | `String` |


**type CsvMapperConnection**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[CsvMapperEdge!]!` |


**type CsvMapperEdge**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperConnection` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `CsvMapper!` |


**type CsvMapperRepresentation**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapper` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `CsvMapperAddInputFromImport` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `type` | `CsvMapperRepresentationType!` |
| `target` | `CsvMapperRepresentationTarget!` |
| `attributes` | `[CsvMapperRepresentationAttribute!]!` |
| `from` | `String` |
| `to` | `String` |


**type CsvMapperRepresentationAttribute**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentation` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `key` | `String!` |
| `column` | `AttributeColumn` |
| `based_on` | `AttributeBasedOn` |
| `ref` | `AttributeRef` |
| `default_values` | `[DefaultValue!]` |


**type CsvMapperRepresentationTarget**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentation` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `entity_type` | `String!` |
| `column_based` | `CsvMapperRepresentationTargetColumn` |


**type CsvMapperRepresentationTargetColumn**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperRepresentationTarget` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `column_reference` | `String` |
| `operator` | `CsvMapperOperator` |
| `value` | `String` |


**type CsvMapperSchemaAttribute**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `CsvMapperSchemaAttribute` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `CsvMapperSchemaAttributes` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `type` | `String!` |
| `mandatory` | `Boolean!` |
| `mandatoryType` | `String!` |
| `editDefault` | `Boolean!` |
| `multiple` | `Boolean!` |
| `defaultValues` | `[DefaultValue!]` |
| `label` | `String!` |
| `mappings` | `[CsvMapperSchemaAttribute!]` |


**type CsvMapperSchemaAttributes**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `attributes` | `[CsvMapperSchemaAttribute!]!` |


**type CsvMapperTestResult**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `objects` | `String!` |
| `nbRelationships` | `Int!` |
| `nbEntities` | `Int!` |


**type JsonComplexPath**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperVariable` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `complex` | `JsonComplexPathConfiguration` |
| `configuration` | `JsonAttributeColumnConfiguration` |


**type JsonComplexPathConfiguration**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonComplexPath` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`
- `type` `JsonComplexPathConfiguration` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `complex` | `JsonComplexPathConfiguration` |
| `formula` | `String` |


**type JsonComplexPathVariable**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `path` | `String` |
| `variable` | `String` |
| `independent` | `Boolean` |


**type JsonMapper**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `IngestionJson` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`
- `type` `JsonMapperEdge` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `variables` | `[JsonMapperVariable!]` |
| `representations` | `[JsonMapperRepresentation!]!` |
| `errors` | `String` |
| `toConfigurationExport` | `String!` |


**type JsonMapperAddInputFromImport**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `representations` | `[JsonMapperRepresentation!]!` |


**type JsonMapperConnection**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[JsonMapperEdge!]!` |


**type JsonMapperEdge**  
Defined in `src/modules/internal/csvMapper/csvMapper.graphql`

_Referenced by:_
- `type` `JsonMapperConnection` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `JsonMapper!` |


**type JsonMapperRepresentation**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapper` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`
- `type` `JsonMapperAddInputFromImport` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `type` | `JsonMapperRepresentationType!` |
| `target` | `JsonMapperRepresentationTarget!` |
| `identifier` | `String` |
| `attributes` | `[JsonMapperRepresentationAttribute!]!` |
| `from` | `String` |
| `to` | `String` |


**type JsonMapperRepresentationAttribute**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperRepresentation` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `key` | `String!` |
| `mode` | `String!` |
| `attr_path` | `AttributePath` |
| `complex_path` | `ComplexPath` |
| `based_on` | `AttributeBasedOn` |
| `default_values` | `[DefaultValue!]` |


**type JsonMapperRepresentationTarget**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapperRepresentation` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `entity_type` | `String!` |
| `path` | `String!` |


**type JsonMapperTestResult**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `objects` | `String!` |
| `nbRelationships` | `Int!` |
| `nbEntities` | `Int!` |
| `state` | `String!` |


**type JsonMapperVariable**  
Defined in `src/modules/internal/jsonMapper/jsonMapper.graphql`

_Referenced by:_
- `type` `JsonMapper` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `path` | `JsonComplexPath` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **53**
- Definition count: **53**


#### Expanded: enum (6)
<a id="expanded-enum-6"></a>

<details><summary>Show names</summary>

- `CsvMapperOperator`
- `CsvMapperOrdering`
- `CsvMapperRepresentationType`
- `EditOperation`
- `JsonMapperOrdering`
- `JsonMapperRepresentationType`

</details>


#### Expanded: input (11)
<a id="expanded-input-11"></a>

<details><summary>Show names</summary>

- `AttributeBasedOnInput`
- `AttributeColumnConfigurationInput`
- `AttributeColumnInput`
- `AttributeRefInput`
- `CsvMapperAddInput`
- `CsvMapperRepresentationAttributeInput`
- `CsvMapperRepresentationInput`
- `CsvMapperRepresentationTargetColumnInput`
- `CsvMapperRepresentationTargetInput`
- `EditInput`
- `JsonMapperAddInput`

</details>


#### Expanded: interface (1)
<a id="expanded-interface-1"></a>

<details><summary>Show names</summary>

- `JsonAttributeColumnConfiguration`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `Upload`

</details>


#### Expanded: type (33)
<a id="expanded-type-33"></a>

<details><summary>Show names</summary>

- `AttributeBasedOn`
- `AttributeColumn`
- `AttributeColumnConfiguration`
- `AttributePath`
- `AttributeRef`
- `ComplexPath`
- `ComplexVariable`
- `CsvMapper`
- `CsvMapperAddInputFromImport`
- `CsvMapperConnection`
- `CsvMapperEdge`
- `CsvMapperRepresentation`
- `CsvMapperRepresentationAttribute`
- `CsvMapperRepresentationTarget`
- `CsvMapperRepresentationTargetColumn`
- `CsvMapperSchemaAttribute`
- `CsvMapperSchemaAttributes`
- `CsvMapperTestResult`
- `DefaultValue`
- `JsonComplexPath`
- `JsonComplexPathConfiguration`
- `JsonComplexPathVariable`
- `JsonMapper`
- `JsonMapperAddInputFromImport`
- `JsonMapperConnection`
- `JsonMapperEdge`
- `JsonMapperRepresentation`
- `JsonMapperRepresentationAttribute`
- `JsonMapperRepresentationTarget`
- `JsonMapperTestResult`
- `JsonMapperVariable`
- `Metric`
- `PageInfo`

</details>



## Module family: language
<a id="module-family-language"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/language/language.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| language | `Language` | `language(id: String!): Language @auth(for: [KNOWLEDGE])` |
| languages | `LanguageConnection` | `languages( first: Int after: ID orderBy: LanguagesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): LanguageConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| languageAdd | `Language` | `languageAdd(input: LanguageAddInput!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageContextClean | `Language` | `languageContextClean(id: ID!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageContextPatch | `Language` | `languageContextPatch(id: ID!, input: EditContext!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageDelete | `ID` | `languageDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| languageFieldPatch | `Language` | `languageFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageRelationAdd | `StixRefRelationship` | `languageRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| languageRelationDelete | `Language` | `languageRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Language @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `LanguagesOrdering` | `src/modules/language/language.graphql` |

<details><summary>Show definition details</summary>


**enum LanguagesOrdering**  
Defined in `src/modules/language/language.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `objectMarking`
- `objectLabel`
- `x_opencti_workflow_id`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `LanguageAddInput` | `src/modules/language/language.graphql` |

<details><summary>Show definition details</summary>


**input LanguageAddInput**  
Defined in `src/modules/language/language.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Language` | `src/modules/language/language.graphql` |
| `LanguageConnection` | `src/modules/language/language.graphql` |
| `LanguageEdge` | `src/modules/language/language.graphql` |

<details><summary>Show definition details</summary>


**type Language**  
Defined in `src/modules/language/language.graphql`

_Referenced by:_
- `type` `LanguageEdge` (language) in `src/modules/language/language.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `aliases` | `[String]` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type LanguageConnection**  
Defined in `src/modules/language/language.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[LanguageEdge]` |


**type LanguageEdge**  
Defined in `src/modules/language/language.graphql`

_Referenced by:_
- `type` `LanguageConnection` (language) in `src/modules/language/language.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Language!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `LanguagesOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `LanguageAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `Language`
- `LanguageConnection`
- `LanguageEdge`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: malwareAnalysis
<a id="module-family-malwareanalysis"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/malwareAnalysis/malwareAnalysis.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| malwareAnalyses | `MalwareAnalysisConnection` | `malwareAnalyses( first: Int after: ID orderBy: MalwareAnalysesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): MalwareAnalysisConnection @auth(for: [KNOWLEDGE])` |
| malwareAnalysis | `MalwareAnalysis` | `malwareAnalysis(id: String!): MalwareAnalysis @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| malwareAnalysisAdd | `MalwareAnalysis` | `malwareAnalysisAdd(input: MalwareAnalysisAddInput!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisContextClean | `MalwareAnalysis` | `malwareAnalysisContextClean(id: ID!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisContextPatch | `MalwareAnalysis` | `malwareAnalysisContextPatch(id: ID!, input: EditContext!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisDelete | `ID` | `malwareAnalysisDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| malwareAnalysisFieldPatch | `MalwareAnalysis` | `malwareAnalysisFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisRelationAdd | `StixRefRelationship` | `malwareAnalysisRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| malwareAnalysisRelationDelete | `MalwareAnalysis` | `malwareAnalysisRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): MalwareAnalysis @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `MalwareAnalysesOrdering` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` |

<details><summary>Show definition details</summary>


**enum MalwareAnalysesOrdering**  
Defined in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

Values:
- `result_name`
- `product`
- `operatingSystem`
- `creator`
- `createdBy`
- `objectLabel`
- `submitted`
- `objectMarking`
- `x_opencti_workflow_id`
- `confidence`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `MalwareAnalysisAddInput` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` |

<details><summary>Show definition details</summary>


**input MalwareAnalysisAddInput**  
Defined in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `product` | `String!` |
| `version` | `String` |
| `hostVm` | `String` |
| `operatingSystem` | `String` |
| `installedSoftware` | `[String]` |
| `configuration_version` | `String` |
| `modules` | `[String]` |
| `analysis_engine_version` | `String` |
| `analysis_definition_version` | `String` |
| `submitted` | `DateTime` |
| `analysis_started` | `DateTime` |
| `analysis_ended` | `DateTime` |
| `result_name` | `String!` |
| `result` | `String` |
| `analysisSco` | `[String]` |
| `analysisSample` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectAssignee` | `[String]` |
| `externalReferences` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `MalwareAnalysis` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` |
| `MalwareAnalysisConnection` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` |
| `MalwareAnalysisEdge` | `src/modules/malwareAnalysis/malwareAnalysis.graphql` |

<details><summary>Show definition details</summary>


**type MalwareAnalysis**  
Defined in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

_Referenced by:_
- `type` `MalwareAnalysisEdge` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `x_opencti_inferences` | `[Inference]` |
| `draftVersion` | `DraftVersion` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `product` | `String!` |
| `version` | `String` |
| `hostVm` | `Software` |
| `operatingSystem` | `Software` |
| `installedSoftware` | `SoftwareConnection` |
| `configuration_version` | `String` |
| `modules` | `[String!]` |
| `analysis_engine_version` | `String` |
| `analysis_definition_version` | `String` |
| `submitted` | `DateTime` |
| `analysis_started` | `DateTime` |
| `analysis_ended` | `DateTime` |
| `result_name` | `String!` |
| `result` | `String` |
| `analysisSco` | `StixCyberObservableConnection` |
| `sample` | `StixCyberObservable` |


**type MalwareAnalysisConnection**  
Defined in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[MalwareAnalysisEdge!]` |


**type MalwareAnalysisEdge**  
Defined in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

_Referenced by:_
- `type` `MalwareAnalysisConnection` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `MalwareAnalysis!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **164**


#### Expanded: enum (12)
<a id="expanded-enum-12"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `MalwareAnalysesOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `StixCyberObservablesOrdering`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `MalwareAnalysisAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (6)
<a id="expanded-interface-6"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixCyberObservable`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (134)
<a id="expanded-type-134"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DecayChartData`
- `DecayHistory`
- `DecayLiveDetails`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Hash`
- `Indicator`
- `IndicatorConnection`
- `IndicatorDecayExclusionRule`
- `IndicatorDecayRule`
- `IndicatorEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `MalwareAnalysis`
- `MalwareAnalysisConnection`
- `MalwareAnalysisEdge`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservablesValues`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Software`
- `SoftwareConnection`
- `SoftwareEdge`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixCyberObservableConnection`
- `StixCyberObservableEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Vulnerability`
- `VulnerabilityConnection`
- `VulnerabilityEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: managerConfiguration
<a id="module-family-managerconfiguration"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 1 |
| Subscription fields | 1 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/managerConfiguration/managerConfiguration.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| managerConfiguration | `ManagerConfiguration` | `managerConfiguration(id: String!): ManagerConfiguration @auth(for: [SETTINGS_FILEINDEXING])` |
| managerConfigurationByManagerId | `ManagerConfiguration` | `managerConfigurationByManagerId(managerId: String!): ManagerConfiguration @auth` |


#### Mutation (1)
<a id="mutation-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| managerConfigurationFieldPatch | `ManagerConfiguration` | `managerConfigurationFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): ManagerConfiguration @auth(for: [SETTINGS_FILEINDEXING])` |


#### Subscription (1)
<a id="subscription-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| managerConfiguration | `ManagerConfiguration` | `managerConfiguration(id: ID!): ManagerConfiguration @auth` |


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### type (1)
<a id="type-1"></a>

| Name | File |
| --- | --- |
| `ManagerConfiguration` | `src/modules/managerConfiguration/managerConfiguration.graphql` |

<details><summary>Show definition details</summary>


**type ManagerConfiguration**  
Defined in `src/modules/managerConfiguration/managerConfiguration.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `manager_id` | `String!` |
| `manager_running` | `Boolean` |
| `last_run_start_date` | `DateTime` |
| `last_run_end_date` | `DateTime` |
| `manager_setting` | `JSON` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **4**
- Definition count: **7**


#### Expanded: enum (1)
<a id="expanded-enum-1"></a>

<details><summary>Show names</summary>

- `EditOperation`

</details>


#### Expanded: input (1)
<a id="expanded-input-1"></a>

<details><summary>Show names</summary>

- `EditInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `JSON`

</details>


#### Expanded: type (2)
<a id="expanded-type-2"></a>

<details><summary>Show names</summary>

- `ManagerConfiguration`
- `Metric`

</details>



## Module family: metrics
<a id="module-family-metrics"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 1 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/metrics/metrics.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (1)
<a id="mutation-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| metricPatch | `BasicObject` | `metricPatch(id: ID!, input: PatchMetricInput!): BasicObject @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `PatchMetricInput` | `src/modules/metrics/metrics.graphql` |

<details><summary>Show definition details</summary>


**input PatchMetricInput**  
Defined in `src/modules/metrics/metrics.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `value` | `Float!` |


</details>


#### type (1)
<a id="type-1"></a>

| Name | File |
| --- | --- |
| `Metric` | `src/modules/metrics/metrics.graphql` |

<details><summary>Show definition details</summary>


**type Metric**  
Defined in `src/modules/metrics/metrics.graphql`

_Referenced by:_
- `type` `AdministrativeArea` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `CaseTemplate` (case) in `src/modules/case/case-template/case-template.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `Catalog` (catalog) in `src/modules/catalog/catalog.graphql`
- `type` `Channel` (channel) in `src/modules/channel/channel.graphql`
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`
- `type` `DecayExclusionRule` (decayRule) in `src/modules/decayRule/exclusions/decayExclusionRule.graphql`
- `type` `DecayRule` (decayRule) in `src/modules/decayRule/decayRule.graphql`
- `type` `DeleteOperation` (deleteOperation) in `src/modules/deleteOperation/deleteOperation.graphql`
- `type` `DisseminationList` (disseminationList) in `src/modules/disseminationList/disseminationList.graphql`
- `type` `DraftWorkspace` (draftWorkspace) in `src/modules/draftWorkspace/draftWorkspace.graphql`
- `type` `EmailTemplate` (emailTemplate) in `src/modules/emailTemplate/emailTemplate.graphql`
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`
- `type` `Event` (event) in `src/modules/event/event.graphql`
- `type` `ExclusionList` (exclusionList) in `src/modules/exclusionList/exclusionList.graphql`
- `type` `FintelDesign` (fintelDesign) in `src/modules/fintelDesign/fintelDesign.graphql`
- `type` `FintelTemplate` (fintelTemplate) in `src/modules/fintelTemplate/fintelTemplate.graphql`
- `type` `Form` (form) in `src/modules/form/form.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `IngestionCsv` (ingestion) in `src/modules/ingestion/ingestion-csv.graphql`
- `type` `IngestionJson` (ingestion) in `src/modules/ingestion/ingestion-json.graphql`
- `type` `IngestionRss` (ingestion) in `src/modules/ingestion/ingestion-rss.graphql`
- `type` `IngestionTaxii` (ingestion) in `src/modules/ingestion/ingestion-taxii.graphql`
- `type` `IngestionTaxiiCollection` (ingestion) in `src/modules/ingestion/ingestion-taxii-collection.graphql`
- `type` `CsvMapper` (internal) in `src/modules/internal/csvMapper/csvMapper.graphql`
- `type` `JsonMapper` (internal) in `src/modules/internal/jsonMapper/jsonMapper.graphql`
- `type` `Language` (language) in `src/modules/language/language.graphql`
- `type` `MalwareAnalysis` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`
- `type` `ManagerConfiguration` (managerConfiguration) in `src/modules/managerConfiguration/managerConfiguration.graphql`
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`
- `type` `Notification` (notification) in `src/modules/notification/notification.graphql`
- `type` `Trigger` (notification) in `src/modules/notification/notification.graphql`
- `type` `Notifier` (notifier) in `src/modules/notifier/notifier.graphql`
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- …

| Field | Type |
| --- | --- |
| `name` | `ID!` |
| `value` | `Float!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **4**
- Definition count: **3**


#### Expanded: input (1)
<a id="expanded-input-1"></a>

<details><summary>Show names</summary>

- `PatchMetricInput`

</details>


#### Expanded: interface (1)
<a id="expanded-interface-1"></a>

<details><summary>Show names</summary>

- `BasicObject`

</details>


#### Expanded: type (1)
<a id="expanded-type-1"></a>

<details><summary>Show names</summary>

- `Metric`

</details>



## Module family: narrative
<a id="module-family-narrative"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/narrative/narrative.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| narrative | `Narrative` | `narrative(id: String!): Narrative @auth(for: [KNOWLEDGE])` |
| narratives | `NarrativeConnection` | `narratives( first: Int after: ID orderBy: NarrativesOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NarrativeConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| narrativeAdd | `Narrative` | `narrativeAdd(input: NarrativeAddInput!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeContextClean | `Narrative` | `narrativeContextClean(id: ID!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeContextPatch | `Narrative` | `narrativeContextPatch(id: ID!, input: EditContext!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeDelete | `ID` | `narrativeDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| narrativeFieldPatch | `Narrative` | `narrativeFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeRelationAdd | `StixRefRelationship` | `narrativeRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| narrativeRelationDelete | `Narrative` | `narrativeRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Narrative @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `NarrativesOrdering` | `src/modules/narrative/narrative.graphql` |

<details><summary>Show definition details</summary>


**enum NarrativesOrdering**  
Defined in `src/modules/narrative/narrative.graphql`

Values:
- `name`
- `narrative_types`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `objectMarking`
- `objectLabel`
- `x_opencti_workflow_id`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `NarrativeAddInput` | `src/modules/narrative/narrative.graphql` |

<details><summary>Show definition details</summary>


**input NarrativeAddInput**  
Defined in `src/modules/narrative/narrative.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `narrative_types` | `[String!]` |
| `aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Narrative` | `src/modules/narrative/narrative.graphql` |
| `NarrativeConnection` | `src/modules/narrative/narrative.graphql` |
| `NarrativeEdge` | `src/modules/narrative/narrative.graphql` |

<details><summary>Show definition details</summary>


**type Narrative**  
Defined in `src/modules/narrative/narrative.graphql`

_Referenced by:_
- `type` `NarrativeEdge` (narrative) in `src/modules/narrative/narrative.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `description` | `String` |
| `narrative_types` | `[String]` |
| `aliases` | `[String]` |
| `parentNarratives` | `NarrativeConnection` |
| `subNarratives` | `NarrativeConnection` |
| `isSubNarrative` | `Boolean` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type NarrativeConnection**  
Defined in `src/modules/narrative/narrative.graphql`

_Referenced by:_
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[NarrativeEdge!]!` |


**type NarrativeEdge**  
Defined in `src/modules/narrative/narrative.graphql`

_Referenced by:_
- `type` `NarrativeConnection` (narrative) in `src/modules/narrative/narrative.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Narrative!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `NarrativesOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `NarrativeAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Narrative`
- `NarrativeConnection`
- `NarrativeEdge`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: notification
<a id="module-family-notification"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 10 |
| Mutation fields | 10 |
| Subscription fields | 2 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/notification/notification.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (10)
<a id="query-10"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| myNotifications | `NotificationConnection` | `myNotifications( first: Int after: ID orderBy: NotificationsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotificationConnection @auth` |
| myUnreadNotificationsCount | `Int` | `myUnreadNotificationsCount: Int @auth` |
| notification | `Notification` | `notification(id: String!): Notification @auth` |
| notifications | `NotificationConnection` | `notifications( first: Int after: ID orderBy: NotificationsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotificationConnection @auth(for: [SETTINGS_SETACCESSES])` |
| triggerActivity | `Trigger` | `triggerActivity(id: String!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerKnowledge | `Trigger` | `triggerKnowledge(id: String!): Trigger @auth` |
| triggers | `TriggerConnection` | `triggers( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): TriggerConnection @auth` |
| triggersActivity | `TriggerConnection` | `triggersActivity( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup search: String ): TriggerConnection @auth(for: [SETTINGS_SECURITYACTIVITY]) # Notifications` |
| triggersKnowledge | `TriggerConnection` | `triggersKnowledge( first: Int after: ID orderBy: TriggersOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): TriggerConnection @auth` |
| triggersKnowledgeCount | `Int` | `triggersKnowledgeCount(filters: FilterGroup, includeAuthorities: Boolean, search: String): Int @auth # Alerts` |


#### Mutation (10)
<a id="mutation-10"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| notificationDelete | `ID` | `notificationDelete(id: ID!): ID @auth` |
| notificationMarkRead | `Notification` | `notificationMarkRead(id: ID!, read: Boolean!): Notification @auth` |
| triggerActivityDelete | `ID` | `triggerActivityDelete(id: ID!): ID @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerActivityDigestAdd | `Trigger` | `triggerActivityDigestAdd(input: TriggerActivityDigestAddInput!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY]) # Notifications` |
| triggerActivityFieldPatch | `Trigger` | `triggerActivityFieldPatch(id: ID!, input: [EditInput!]!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerActivityLiveAdd | `Trigger` | `triggerActivityLiveAdd(input: TriggerActivityLiveAddInput!): Trigger @auth(for: [SETTINGS_SECURITYACTIVITY])` |
| triggerKnowledgeDelete | `ID` | `triggerKnowledgeDelete(id: ID!): ID @auth` |
| triggerKnowledgeDigestAdd | `Trigger` | `triggerKnowledgeDigestAdd(input: TriggerDigestAddInput!): Trigger @auth # Alerts` |
| triggerKnowledgeFieldPatch | `Trigger` | `triggerKnowledgeFieldPatch(id: ID!, input: [EditInput!]!): Trigger @auth` |
| triggerKnowledgeLiveAdd | `Trigger` | `triggerKnowledgeLiveAdd(input: TriggerLiveAddInput!): Trigger @auth` |


#### Subscription (2)
<a id="subscription-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| notification | `Notification` | `notification: Notification @auth` |
| notificationsNumber | `NotificationCount` | `notificationsNumber: NotificationCount @auth` |


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (6)
<a id="enum-6"></a>

| Name | File |
| --- | --- |
| `DigestPeriod` | `src/modules/notification/notification.graphql` |
| `NotificationsOrdering` | `src/modules/notification/notification.graphql` |
| `TriggerActivityEventType` | `src/modules/notification/notification.graphql` |
| `TriggerEventType` | `src/modules/notification/notification.graphql` |
| `TriggerType` | `src/modules/notification/notification.graphql` |
| `TriggersOrdering` | `src/modules/notification/notification.graphql` |

<details><summary>Show definition details</summary>


**enum DigestPeriod**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `input` `TriggerActivityDigestAddInput` (notification) in `src/modules/notification/notification.graphql`
- `input` `TriggerDigestAddInput` (notification) in `src/modules/notification/notification.graphql`
- `type` `Trigger` (notification) in `src/modules/notification/notification.graphql`

Values:
- `hour`
- `day`
- `week`
- `month`


**enum NotificationsOrdering**  
Defined in `src/modules/notification/notification.graphql`

Values:
- `name`
- `created`
- `_score`


**enum TriggerActivityEventType**  
Defined in `src/modules/notification/notification.graphql`

Values:
- `authentication`
- `read`
- `mutation`
- `file`
- `command`


**enum TriggerEventType**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `input` `TriggerLiveAddInput` (notification) in `src/modules/notification/notification.graphql`

Values:
- `create`
- `update`
- `delete`


**enum TriggerType**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `Trigger` (notification) in `src/modules/notification/notification.graphql`

Values:
- `live`
- `digest`


**enum TriggersOrdering**  
Defined in `src/modules/notification/notification.graphql`

Values:
- `name`
- `created`
- `event_types`
- `trigger_type`
- `notifiers`
- `_score`


</details>


#### input (4)
<a id="input-4"></a>

| Name | File |
| --- | --- |
| `TriggerActivityDigestAddInput` | `src/modules/notification/notification.graphql` |
| `TriggerActivityLiveAddInput` | `src/modules/notification/notification.graphql` |
| `TriggerDigestAddInput` | `src/modules/notification/notification.graphql` |
| `TriggerLiveAddInput` | `src/modules/notification/notification.graphql` |

<details><summary>Show definition details</summary>


**input TriggerActivityDigestAddInput**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `trigger_ids` | `[String!]!` |
| `period` | `DigestPeriod!` |
| `trigger_time` | `String` |
| `notifiers` | `[StixRef!]!` |
| `recipients` | `[String!]!` |


**input TriggerActivityLiveAddInput**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `notifiers` | `[StixRef!]` |
| `filters` | `String` |
| `recipients` | `[String!]!` |


**input TriggerDigestAddInput**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `trigger_ids` | `[String!]!` |
| `period` | `DigestPeriod!` |
| `trigger_time` | `String` |
| `notifiers` | `[StixRef!]!` |
| `recipients` | `[String!]` |


**input TriggerLiveAddInput**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `event_types` | `[TriggerEventType!]!` |
| `notifiers` | `[StixRef!]` |
| `instance_trigger` | `Boolean!` |
| `filters` | `String` |
| `recipients` | `[String!]` |


</details>


#### type (10)
<a id="type-10"></a>

| Name | File |
| --- | --- |
| `Notification` | `src/modules/notification/notification.graphql` |
| `NotificationConnection` | `src/modules/notification/notification.graphql` |
| `NotificationContent` | `src/modules/notification/notification.graphql` |
| `NotificationCount` | `src/modules/notification/notification.graphql` |
| `NotificationEdge` | `src/modules/notification/notification.graphql` |
| `NotificationEvent` | `src/modules/notification/notification.graphql` |
| `ResolvedInstanceFilter` | `src/modules/notification/notification.graphql` |
| `Trigger` | `src/modules/notification/notification.graphql` |
| `TriggerConnection` | `src/modules/notification/notification.graphql` |
| `TriggerEdge` | `src/modules/notification/notification.graphql` |

<details><summary>Show definition details</summary>


**type Notification**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `NotificationEdge` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `created` | `DateTime` |
| `name` | `String!` |
| `notification_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `notification_content` | `[NotificationContent!]!` |
| `is_read` | `Boolean!` |
| `user_id` | `String` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |


**type NotificationConnection**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[NotificationEdge]` |


**type NotificationContent**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `Notification` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `title` | `String!` |
| `events` | `[NotificationEvent!]!` |


**type NotificationCount**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `user_id` | `String` |
| `count` | `Int` |


**type NotificationEdge**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `NotificationConnection` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Notification!` |


**type NotificationEvent**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `NotificationContent` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `message` | `String!` |
| `instance_id` | `String` |
| `operation` | `String!` |


**type ResolvedInstanceFilter**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `id` | `String!` |
| `valid` | `Boolean!` |
| `value` | `String` |


**type Trigger**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `Trigger` (notification) in `src/modules/notification/notification.graphql`
- `type` `TriggerEdge` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `trigger_type` | `TriggerType!` |
| `event_types` | `[String!]` |
| `filters` | `String` |
| `notifiers` | `[Notifier!]` |
| `trigger_ids` | `[String]` |
| `triggers` | `[Trigger]` |
| `recipients` | `[Member!]` |
| `period` | `DigestPeriod` |
| `trigger_time` | `String` |
| `isDirectAdministrator` | `Boolean` |
| `currentUserAccessRight` | `String` |
| `instance_trigger` | `Boolean` |


**type TriggerConnection**  
Defined in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[TriggerEdge!]!` |


**type TriggerEdge**  
Defined in `src/modules/notification/notification.graphql`

_Referenced by:_
- `type` `TriggerConnection` (notification) in `src/modules/notification/notification.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Trigger!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **29**
- Definition count: **160**


#### Expanded: enum (18)
<a id="expanded-enum-18"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DigestPeriod`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `FilterMode`
- `FilterOperator`
- `NotificationsOrdering`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `TriggerActivityEventType`
- `TriggerEventType`
- `TriggerType`
- `TriggersOrdering`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (7)
<a id="expanded-input-7"></a>

<details><summary>Show names</summary>

- `EditInput`
- `Filter`
- `FilterGroup`
- `TriggerActivityDigestAddInput`
- `TriggerActivityLiveAddInput`
- `TriggerDigestAddInput`
- `TriggerLiveAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (4)
<a id="expanded-scalar-4"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`

</details>


#### Expanded: type (123)
<a id="expanded-type-123"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `Member`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notification`
- `NotificationConnection`
- `NotificationContent`
- `NotificationCount`
- `NotificationEdge`
- `NotificationEvent`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `ResolvedInstanceFilter`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `Trigger`
- `TriggerConnection`
- `TriggerEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: notifier
<a id="module-family-notifier"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 4 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/notifier/notifier.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (4)
<a id="query-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| notificationNotifiers | `[Notifier!]!` | `notificationNotifiers: [Notifier!]! @auth` |
| notifier | `Notifier` | `notifier(id: String!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierTest | `String` | `notifierTest(input: NotifierTestInput!): String @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifiers | `NotifierConnection` | `notifiers( first: Int after: ID orderBy: NotifierOrdering orderMode: OrderingMode filters: FilterGroup search: String ): NotifierConnection @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| notifierAdd | `Notifier` | `notifierAdd(input: NotifierAddInput!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierDelete | `ID` | `notifierDelete(id: ID!): ID @auth(for: [SETTINGS_SETCUSTOMIZATION])` |
| notifierFieldPatch | `Notifier` | `notifierFieldPatch(id: ID!, input: [EditInput!]!): Notifier @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `NotifierOrdering` | `src/modules/notifier/notifier.graphql` |

<details><summary>Show definition details</summary>


**enum NotifierOrdering**  
Defined in `src/modules/notifier/notifier.graphql`

Values:
- `name`
- `created`
- `connector`
- `_score`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `NotifierAddInput` | `src/modules/notifier/notifier.graphql` |
| `NotifierTestInput` | `src/modules/notifier/notifier.graphql` |

<details><summary>Show definition details</summary>


**input NotifierAddInput**  
Defined in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `notifier_connector_id` | `String!` |
| `notifier_configuration` | `String!` |
| `authorized_members` | `[MemberAccessInput!]` |


**input NotifierTestInput**  
Defined in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `notifier_test_id` | `String!` |
| `notifier_connector_id` | `String!` |
| `notifier_configuration` | `String!` |


</details>


#### type (5)
<a id="type-5"></a>

| Name | File |
| --- | --- |
| `Notifier` | `src/modules/notifier/notifier.graphql` |
| `NotifierConnection` | `src/modules/notifier/notifier.graphql` |
| `NotifierConnector` | `src/modules/notifier/notifier.graphql` |
| `NotifierEdge` | `src/modules/notifier/notifier.graphql` |
| `NotifierParameter` | `src/modules/notifier/notifier.graphql` |

<details><summary>Show definition details</summary>


**type Notifier**  
Defined in `src/modules/notifier/notifier.graphql`

_Referenced by:_
- `type` `Trigger` (notification) in `src/modules/notification/notification.graphql`
- `type` `NotifierEdge` (notifier) in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |
| `notifier_connector` | `NotifierConnector!` |
| `notifier_connector_id` | `String!` |
| `notifier_configuration` | `String!` |
| `authorized_members` | `[MemberAccess!]` |


**type NotifierConnection**  
Defined in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[NotifierEdge]` |


**type NotifierConnector**  
Defined in `src/modules/notifier/notifier.graphql`

_Referenced by:_
- `type` `Notifier` (notifier) in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |
| `connector_type` | `String` |
| `connector_schema` | `String` |
| `connector_schema_ui` | `String` |
| `built_in` | `Boolean` |


**type NotifierEdge**  
Defined in `src/modules/notifier/notifier.graphql`

_Referenced by:_
- `type` `NotifierConnection` (notifier) in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Notifier!` |


**type NotifierParameter**  
Defined in `src/modules/notifier/notifier.graphql`

| Field | Type |
| --- | --- |
| `key` | `String` |
| `value` | `String` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **12**
- Definition count: **17**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `NotifierOrdering`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditInput`
- `MemberAccessInput`
- `NotifierAddInput`
- `NotifierTestInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`

</details>


#### Expanded: type (9)
<a id="expanded-type-9"></a>

<details><summary>Show names</summary>

- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Notifier`
- `NotifierConnection`
- `NotifierConnector`
- `NotifierEdge`
- `NotifierParameter`
- `PageInfo`

</details>



## Module family: organization
<a id="module-family-organization"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 3 |
| Mutation fields | 10 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/organization/organization.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (3)
<a id="query-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| organization | `Organization` | `organization(id: String!): Organization @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATION_ADMIN])` |
| organizations | `OrganizationConnection` | `organizations( first: Int after: ID orderBy: OrganizationsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): OrganizationConnection @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATION_ADMIN])` |
| securityOrganizations | `OrganizationConnection` | `securityOrganizations( first: Int after: ID orderBy: OrganizationsOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): OrganizationConnection @auth(for: [SETTINGS_SETACCESSES, SETTINGS_SECURITYACTIVITY, VIRTUAL_ORGANIZATION_ADMIN])` |


#### Mutation (10)
<a id="mutation-10"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| organizationAdd | `Organization` | `organizationAdd(input: OrganizationAddInput!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationAdminAdd | `Organization` | `organizationAdminAdd(id: ID!, memberId: String!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationAdminRemove | `Organization` | `organizationAdminRemove(id: ID!, memberId: String!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationContextClean | `Organization` | `organizationContextClean(id: ID!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationContextPatch | `Organization` | `organizationContextPatch(id: ID!, input: EditContext!): Organization @auth(for: [KNOWLEDGE_KNUPDATE, SETTINGS_SETACCESSES])` |
| organizationDelete | `ID` | `organizationDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| organizationEditAuthorizedAuthorities | `Organization` | `organizationEditAuthorizedAuthorities(id: ID!, input: [String!]!): Organization @auth(for: [SETTINGS_SETACCESSES])` |
| organizationFieldPatch | `Organization` | `organizationFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): Organization @auth(for: [KNOWLEDGE_KNUPDATE, SETTINGS_SETACCESSES])` |
| organizationRelationAdd | `StixRefRelationship` | `organizationRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| organizationRelationDelete | `Organization` | `organizationRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Organization @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `OrganizationsOrdering` | `src/modules/organization/organization.graphql` |

<details><summary>Show definition details</summary>


**enum OrganizationsOrdering**  
Defined in `src/modules/organization/organization.graphql`

Values:
- `name`
- `confidence`
- `created`
- `created_at`
- `modified`
- `updated_at`
- `x_opencti_organization_type`
- `x_opencti_workflow_id`
- `x_opencti_score`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `OrganizationAddInput` | `src/modules/organization/organization.graphql` |

<details><summary>Show definition details</summary>


**input OrganizationAddInput**  
Defined in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `contact_information` | `String` |
| `roles` | `[String]` |
| `x_opencti_aliases` | `[String]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `x_opencti_organization_type` | `String` |
| `x_opencti_reliability` | `String` |
| `x_opencti_score` | `Int` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_modified_at` | `DateTime` |
| `x_opencti_workflow_id` | `String` |
| `clientMutationId` | `String` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (6)
<a id="type-6"></a>

| Name | File |
| --- | --- |
| `MeOrganization` | `src/modules/organization/organization.graphql` |
| `MeOrganizationConnection` | `src/modules/organization/organization.graphql` |
| `MeOrganizationEdge` | `src/modules/organization/organization.graphql` |
| `Organization` | `src/modules/organization/organization.graphql` |
| `OrganizationConnection` | `src/modules/organization/organization.graphql` |
| `OrganizationEdge` | `src/modules/organization/organization.graphql` |

<details><summary>Show definition details</summary>


**type MeOrganization**  
Defined in `src/modules/organization/organization.graphql`

_Referenced by:_
- `type` `MeOrganizationEdge` (organization) in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |


**type MeOrganizationConnection**  
Defined in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[MeOrganizationEdge!]!` |


**type MeOrganizationEdge**  
Defined in `src/modules/organization/organization.graphql`

_Referenced by:_
- `type` `MeOrganizationConnection` (organization) in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `MeOrganization!` |


**type Organization**  
Defined in `src/modules/organization/organization.graphql`

_Referenced by:_
- `type` `AdministrativeArea` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `Channel` (channel) in `src/modules/channel/channel.graphql`
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`
- `type` `Event` (event) in `src/modules/event/event.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `Language` (language) in `src/modules/language/language.graphql`
- `type` `MalwareAnalysis` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- `type` `OrganizationEdge` (organization) in `src/modules/organization/organization.graphql`
- `type` `SecurityCoverage` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`
- `type` `SecurityPlatform` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`
- `type` `Task` (task) in `src/modules/task/task.graphql`
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `identity_class` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `contact_information` | `String` |
| `roles` | `[String]` |
| `x_opencti_aliases` | `[String]` |
| `x_opencti_reliability` | `String` |
| `x_opencti_organization_type` | `String` |
| `x_opencti_score` | `Int` |
| `sectors` | `SectorConnection` |
| `members(first: Int after: ID orderBy: UsersOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `UserConnection` |
| `authorized_authorities` | `[String]` |
| `grantable_groups` | `[Group!]` |
| `subOrganizations` | `OrganizationConnection` |
| `parentOrganizations` | `OrganizationConnection` |
| `default_dashboard` | `Workspace` |
| `default_hidden_types` | `[String!]` |
| `restrict_access` | `Boolean` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type OrganizationConnection**  
Defined in `src/modules/organization/organization.graphql`

_Referenced by:_
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[OrganizationEdge!]!` |


**type OrganizationEdge**  
Defined in `src/modules/organization/organization.graphql`

_Referenced by:_
- `type` `OrganizationConnection` (organization) in `src/modules/organization/organization.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Organization!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **16**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `OrganizationsOrdering`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `OrganizationAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MeOrganization`
- `MeOrganizationConnection`
- `MeOrganizationEdge`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: pir
<a id="module-family-pir"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 6 |
| Mutation fields | 6 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/pir/pir.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (6)
<a id="query-6"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| pir | `Pir` | `pir(id: ID!): Pir @auth(for: [PIRAPI])` |
| pirLogs | `LogConnection` | `pirLogs( pirId: ID! first: Int after: ID orderBy: LogsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): LogConnection @auth(for: [KNOWLEDGE, PIRAPI])` |
| pirRelationships | `PirRelationshipConnection` | `pirRelationships( pirId: ID! first: Int after: ID orderBy: PirRelationshipOrdering orderMode: OrderingMode fromId: [String] fromRole: String fromTypes: [String] startTimeStart: DateTime startTimeStop: DateTime stopTimeStart: DateTime stopTimeStop: DateTime firstSeenStart: DateTi…` |
| pirRelationshipsDistribution | `[Distribution]` | `pirRelationshipsDistribution( pirId: ID! field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String fromId: [String] fromTypes: [String] relationship_type: [String] search: String filters: FilterGr…` |
| pirRelationshipsMultiTimeSeries | `[MultiTimeSeries]` | `pirRelationshipsMultiTimeSeries( operation: StatsOperation! startDate: DateTime! endDate: DateTime interval: String! onlyInferred: Boolean timeSeriesParameters: [PirRelationshipsTimeSeriesParameters!]! relationship_type: [String!] ): [MultiTimeSeries] @auth(for: [KNOWLEDGE, PIRA…` |
| pirs | `PirConnection` | `pirs( first: Int after: ID orderBy: PirOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PirConnection @auth(for: [PIRAPI])` |


#### Mutation (6)
<a id="mutation-6"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| pirAdd | `Pir` | `pirAdd(input: PirAddInput!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirDelete | `ID` | `pirDelete(id: ID!): ID @auth(for: [PIRAPI_PIRUPDATE])` |
| pirEditAuthorizedMembers | `Pir` | `pirEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirFieldPatch | `Pir` | `pirFieldPatch(id: ID!, input: [EditInput!]!): Pir @auth(for: [PIRAPI_PIRUPDATE])` |
| pirFlagElement | `ID` | `pirFlagElement(id: ID!, input: PirFlagElementInput!): ID @auth(for: [BYPASS])` |
| pirUnflagElement | `ID` | `pirUnflagElement(id: ID!, input: PirUnflagElementInput!): ID @auth(for: [BYPASS])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (3)
<a id="enum-3"></a>

| Name | File |
| --- | --- |
| `PirOrdering` | `src/modules/pir/pir.graphql` |
| `PirRelationshipOrdering` | `src/modules/pir/pir.graphql` |
| `PirType` | `src/modules/pir/pir.graphql` |

<details><summary>Show definition details</summary>


**enum PirOrdering**  
Defined in `src/modules/pir/pir.graphql`

Values:
- `_score`
- `name`
- `created_at`
- `updated_at`
- `creator`


**enum PirRelationshipOrdering**  
Defined in `src/modules/pir/pir.graphql`

Values:
- `created_at`
- `updated_at`
- `pir_score`


**enum PirType**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `input` `PirAddInput` (pir) in `src/modules/pir/pir.graphql`
- `type` `Pir` (pir) in `src/modules/pir/pir.graphql`

Values:
- `THREAT_LANDSCAPE`
- `THREAT_ORIGIN`
- `THREAT_CUSTOM`


</details>


#### input (7)
<a id="input-7"></a>

| Name | File |
| --- | --- |
| `PirAddInput` | `src/modules/pir/pir.graphql` |
| `PirCriterionInput` | `src/modules/pir/pir.graphql` |
| `PirDependencyInput` | `src/modules/pir/pir.graphql` |
| `PirExplanationInput` | `src/modules/pir/pir.graphql` |
| `PirFlagElementInput` | `src/modules/pir/pir.graphql` |
| `PirRelationshipsTimeSeriesParameters` | `src/modules/pir/pir.graphql` |
| `PirUnflagElementInput` | `src/modules/pir/pir.graphql` |

<details><summary>Show definition details</summary>


**input PirAddInput**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `pir_type` | `PirType!` |
| `description` | `String` |
| `pir_rescan_days` | `Int!` |
| `pir_criteria` | `[PirCriterionInput!]!` |
| `pir_filters` | `FilterGroup!` |
| `authorized_members` | `[MemberAccessInput!]` |


**input PirCriterionInput**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `input` `PirAddInput` (pir) in `src/modules/pir/pir.graphql`
- `input` `PirExplanationInput` (pir) in `src/modules/pir/pir.graphql`
- `input` `PirFlagElementInput` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `weight` | `Int!` |
| `filters` | `FilterGroup!` |


**input PirDependencyInput**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `input` `PirExplanationInput` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `element_id` | `ID!` |
| `author_id` | `ID` |


**input PirExplanationInput**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `dependencies` | `[PirDependencyInput!]!` |
| `criterion` | `PirCriterionInput!` |


**input PirFlagElementInput**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `relationshipId` | `ID!` |
| `sourceId` | `ID!` |
| `matchingCriteria` | `[PirCriterionInput!]!` |
| `relationshipAuthorId` | `ID` |


**input PirRelationshipsTimeSeriesParameters**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `pirId` | `ID!` |
| `field` | `String!` |
| `elementWithTargetTypes` | `[String]` |
| `fromId` | `[String]` |
| `fromTypes` | `[String]` |
| `relationship_type` | `[String]` |
| `confidences` | `[Int]` |
| `search` | `String` |
| `filters` | `FilterGroup` |
| `dynamicFrom` | `FilterGroup` |


**input PirUnflagElementInput**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `relationshipId` | `ID!` |
| `sourceId` | `ID!` |


</details>


#### type (11)
<a id="type-11"></a>

| Name | File |
| --- | --- |
| `Pir` | `src/modules/pir/pir.graphql` |
| `PirConnection` | `src/modules/pir/pir.graphql` |
| `PirCriterion` | `src/modules/pir/pir.graphql` |
| `PirDependency` | `src/modules/pir/pir.graphql` |
| `PirEdge` | `src/modules/pir/pir.graphql` |
| `PirExplanation` | `src/modules/pir/pir.graphql` |
| `PirInformation` | `src/modules/pir/pir.graphql` |
| `PirRelationship` | `src/modules/pir/pir.graphql` |
| `PirRelationshipConnection` | `src/modules/pir/pir.graphql` |
| `PirRelationshipEdge` | `src/modules/pir/pir.graphql` |
| `PirScore` | `src/modules/pir/pir.graphql` |

<details><summary>Show definition details</summary>


**type Pir**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirEdge` (pir) in `src/modules/pir/pir.graphql`
- `type` `PirRelationship` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `creators` | `[Creator!]` |
| `name` | `String!` |
| `pir_type` | `PirType!` |
| `description` | `String` |
| `pir_rescan_days` | `Int!` |
| `pir_criteria` | `[PirCriterion!]!` |
| `pir_filters` | `String!` |
| `lastEventId` | `String!` |
| `authorizedMembers` | `[MemberAccess!]!` |
| `currentUserAccessRight` | `String` |
| `pirContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String)` | `ContainerConnection` |
| `queue_messages` | `Int!` |


**type PirConnection**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[PirEdge!]!` |


**type PirCriterion**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `Pir` (pir) in `src/modules/pir/pir.graphql`
- `type` `PirExplanation` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `filters` | `String!` |
| `weight` | `Int!` |


**type PirDependency**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirExplanation` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `element_id` | `ID!` |
| `author_id` | `ID` |


**type PirEdge**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirConnection` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Pir!` |


**type PirExplanation**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirInformation` (pir) in `src/modules/pir/pir.graphql`
- `type` `PirRelationship` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `dependencies` | `[PirDependency!]!` |
| `criterion` | `PirCriterion!` |


**type PirInformation**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `AdministrativeArea` (administrativeArea) in `src/modules/administrativeArea/administrativeArea.graphql`
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`
- `type` `Channel` (channel) in `src/modules/channel/channel.graphql`
- `type` `DataComponent` (dataComponent) in `src/modules/dataComponent/dataComponent.graphql`
- `type` `DataSource` (dataSource) in `src/modules/dataSource/dataSource.graphql`
- `type` `Event` (event) in `src/modules/event/event.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `Indicator` (indicator) in `src/modules/indicator/indicator.graphql`
- `type` `Language` (language) in `src/modules/language/language.graphql`
- `type` `MalwareAnalysis` (malwareAnalysis) in `src/modules/malwareAnalysis/malwareAnalysis.graphql`
- `type` `Narrative` (narrative) in `src/modules/narrative/narrative.graphql`
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- `type` `SecurityCoverage` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`
- `type` `SecurityPlatform` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`
- `type` `Task` (task) in `src/modules/task/task.graphql`
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `pir_score` | `Int!` |
| `last_pir_score_date` | `DateTime!` |
| `pir_explanation` | `[PirExplanation!]!` |


**type PirRelationship**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirRelationshipEdge` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `fromRole` | `String` |
| `toRole` | `String` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `refreshed_at` | `DateTime` |
| `fromId` | `String!` |
| `toId` | `String!` |
| `fromType` | `String!` |
| `toType` | `String!` |
| `from` | `StixDomainObject` |
| `to` | `Pir` |
| `creators` | `[Creator!]` |
| `pir_explanation` | `[PirExplanation!]` |
| `pir_score` | `Int` |


**type PirRelationshipConnection**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[PirRelationshipEdge!]!` |


**type PirRelationshipEdge**  
Defined in `src/modules/pir/pir.graphql`

_Referenced by:_
- `type` `PirRelationshipConnection` (pir) in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `PirRelationship!` |


**type PirScore**  
Defined in `src/modules/pir/pir.graphql`

| Field | Type |
| --- | --- |
| `pir_id` | `ID!` |
| `pir_score` | `Int!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **32**
- Definition count: **164**


#### Expanded: enum (15)
<a id="expanded-enum-15"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `FilterMode`
- `FilterOperator`
- `OrderingMode`
- `PirOrdering`
- `PirRelationshipOrdering`
- `PirType`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (11)
<a id="expanded-input-11"></a>

<details><summary>Show names</summary>

- `EditInput`
- `Filter`
- `FilterGroup`
- `MemberAccessInput`
- `PirAddInput`
- `PirCriterionInput`
- `PirDependencyInput`
- `PirExplanationInput`
- `PirFlagElementInput`
- `PirRelationshipsTimeSeriesParameters`
- `PirUnflagElementInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (4)
<a id="expanded-scalar-4"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `JSON`
- `StixId`

</details>


#### Expanded: type (126)
<a id="expanded-type-126"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `Change`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `ContextData`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `Log`
- `LogConnection`
- `LogEdge`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `MultiTimeSeries`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `Pir`
- `PirConnection`
- `PirCriterion`
- `PirDependency`
- `PirEdge`
- `PirExplanation`
- `PirInformation`
- `PirRelationship`
- `PirRelationshipConnection`
- `PirRelationshipEdge`
- `PirScore`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TimeSeries`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: playbook
<a id="module-family-playbook"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 4 |
| Mutation fields | 14 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/playbook/playbook.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (4)
<a id="query-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| playbook | `Playbook` | `playbook(id: String!): Playbook @auth(for: [AUTOMATION])` |
| playbookComponents | `[PlaybookComponent]!` | `playbookComponents: [PlaybookComponent]! @auth(for: [AUTOMATION])` |
| playbooks | `PlaybookConnection` | `playbooks( first: Int after: ID orderBy: PlaybooksOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PlaybookConnection @auth(for: [AUTOMATION])` |
| playbooksForEntity | `[Playbook]` | `playbooksForEntity(id: String!): [Playbook] @auth(for: [AUTOMATION])` |


#### Mutation (14)
<a id="mutation-14"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| playbookAdd | `Playbook` | `playbookAdd(input: PlaybookAddInput!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookAddLink | `String!` | `playbookAddLink(id: ID!, input: PlaybookAddLinkInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookAddNode | `String!` | `playbookAddNode(id: ID!, input: PlaybookAddNodeInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDelete | `ID` | `playbookDelete(id: ID!): ID @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDeleteLink | `Playbook` | `playbookDeleteLink(id: ID!, linkId: ID!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDeleteNode | `Playbook` | `playbookDeleteNode(id: ID!, nodeId: ID!): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookDuplicate | `String!` | `playbookDuplicate(id: ID!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookExecute | `Boolean` | `playbookExecute(id: ID!, entityId: String!): Boolean @auth(for: [AUTOMATION])` |
| playbookFieldPatch | `Playbook` | `playbookFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Playbook @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookImport | `String!` | `playbookImport(file: Upload!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookInsertNode | `PlaybookInsertResult!` | `playbookInsertNode(id: ID!, parentNodeId: ID!, parentPortId: ID!, childNodeId: ID!, input: PlaybookAddNodeInput!): PlaybookInsertResult! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookReplaceNode | `String!` | `playbookReplaceNode(id: ID!, nodeId: ID!, input: PlaybookAddNodeInput!): String! @auth(for: [AUTOMATION_AUTMANAGE])` |
| playbookStepExecution | `Boolean` | `playbookStepExecution(execution_id: ID!, event_id: ID!, execution_start: DateTime!, data_instance_id: ID!, playbook_id: ID!, previous_step_id: ID!, step_id: ID!, previous_bundle: String!, bundle: String!): Boolean @auth(for: [CONNECTORAPI])` |
| playbookUpdatePositions | `ID` | `playbookUpdatePositions(id: ID!, positions: String!): ID @auth(for: [AUTOMATION_AUTMANAGE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `PlaybooksOrdering` | `src/modules/playbook/playbook.graphql` |

<details><summary>Show definition details</summary>


**enum PlaybooksOrdering**  
Defined in `src/modules/playbook/playbook.graphql`

Values:
- `name`
- `playbook_running`
- `_score`


</details>


#### input (4)
<a id="input-4"></a>

| Name | File |
| --- | --- |
| `PlaybookAddInput` | `src/modules/playbook/playbook.graphql` |
| `PlaybookAddLinkInput` | `src/modules/playbook/playbook.graphql` |
| `PlaybookAddNodeInput` | `src/modules/playbook/playbook.graphql` |
| `PositionInput` | `src/modules/playbook/playbook.graphql` |

<details><summary>Show definition details</summary>


**input PlaybookAddInput**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |


**input PlaybookAddLinkInput**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `from_node` | `String!` |
| `from_port` | `String!` |
| `to_node` | `String!` |


**input PlaybookAddNodeInput**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `component_id` | `String!` |
| `position` | `PositionInput!` |
| `configuration` | `String` |


**input PositionInput**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `input` `PlaybookAddNodeInput` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `x` | `Float!` |
| `y` | `Float!` |


</details>


#### type (8)
<a id="type-8"></a>

| Name | File |
| --- | --- |
| `PlayBookExecution` | `src/modules/playbook/playbook.graphql` |
| `PlayBookExecutionStep` | `src/modules/playbook/playbook.graphql` |
| `Playbook` | `src/modules/playbook/playbook.graphql` |
| `PlaybookComponent` | `src/modules/playbook/playbook.graphql` |
| `PlaybookComponentPort` | `src/modules/playbook/playbook.graphql` |
| `PlaybookConnection` | `src/modules/playbook/playbook.graphql` |
| `PlaybookEdge` | `src/modules/playbook/playbook.graphql` |
| `PlaybookInsertResult` | `src/modules/playbook/playbook.graphql` |

<details><summary>Show definition details</summary>


**type PlayBookExecution**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `type` `Playbook` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `playbook_id` | `ID!` |
| `execution_start` | `String` |
| `steps` | `[PlayBookExecutionStep!]` |


**type PlayBookExecutionStep**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `type` `PlayBookExecution` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `message` | `String` |
| `status` | `String` |
| `in_timestamp` | `String` |
| `out_timestamp` | `String` |
| `duration` | `Int` |
| `bundle_or_patch` | `String` |
| `error` | `String` |


**type Playbook**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `type` `PlaybookEdge` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `description` | `String` |
| `playbook_running` | `Boolean` |
| `playbook_definition` | `String` |
| `last_executions` | `[PlayBookExecution!]` |
| `queue_messages` | `Int!` |
| `toConfigurationExport` | `String!` |


**type PlaybookComponent**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |
| `description` | `String!` |
| `icon` | `String!` |
| `is_entry_point` | `Boolean` |
| `is_internal` | `Boolean` |
| `configuration_schema` | `String` |
| `ports` | `[PlaybookComponentPort!]!` |


**type PlaybookComponentPort**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `type` `PlaybookComponent` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `type` | `String!` |


**type PlaybookConnection**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[PlaybookEdge!]!` |


**type PlaybookEdge**  
Defined in `src/modules/playbook/playbook.graphql`

_Referenced by:_
- `type` `PlaybookConnection` (playbook) in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Playbook!` |


**type PlaybookInsertResult**  
Defined in `src/modules/playbook/playbook.graphql`

| Field | Type |
| --- | --- |
| `nodeId` | `String!` |
| `linkId` | `String!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **20**
- Definition count: **20**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `PlaybooksOrdering`

</details>


#### Expanded: input (5)
<a id="expanded-input-5"></a>

<details><summary>Show names</summary>

- `EditInput`
- `PlaybookAddInput`
- `PlaybookAddLinkInput`
- `PlaybookAddNodeInput`
- `PositionInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `Upload`

</details>


#### Expanded: type (10)
<a id="expanded-type-10"></a>

<details><summary>Show names</summary>

- `Metric`
- `PageInfo`
- `PlayBookExecution`
- `PlayBookExecutionStep`
- `Playbook`
- `PlaybookComponent`
- `PlaybookComponentPort`
- `PlaybookConnection`
- `PlaybookEdge`
- `PlaybookInsertResult`

</details>



## Module family: publicDashboard
<a id="module-family-publicdashboard"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 12 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/publicDashboard/publicDashboard.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (12)
<a id="query-12"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| publicBookmarks | `StixDomainObjectConnection` | `publicBookmarks( uriKey: String! widgetId : String! ): StixDomainObjectConnection @public` |
| publicDashboard | `PublicDashboard` | `publicDashboard(id: String!): PublicDashboard @auth(for: [EXPLORE])` |
| publicDashboardByUriKey | `PublicDashboard` | `publicDashboardByUriKey(uri_key: String!): PublicDashboard @public` |
| publicDashboards | `PublicDashboardConnection` | `publicDashboards( first: Int after: ID orderBy: PublicDashboardsOrdering orderMode: OrderingMode filters: FilterGroup search: String ): PublicDashboardConnection @auth(for: [EXPLORE])` |
| publicStixCoreObjects | `StixCoreObjectConnection` | `publicStixCoreObjects( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): StixCoreObjectConnection @public` |
| publicStixCoreObjectsDistribution | `[PublicDistribution]` | `publicStixCoreObjectsDistribution( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [PublicDistribution] @public` |
| publicStixCoreObjectsMultiTimeSeries | `[MultiTimeSeries]` | `publicStixCoreObjectsMultiTimeSeries( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [MultiTimeSeries] @public` |
| publicStixCoreObjectsNumber | `Number` | `publicStixCoreObjectsNumber( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): Number @public` |
| publicStixRelationships | `StixRelationshipConnection` | `publicStixRelationships( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): StixRelationshipConnection @public` |
| publicStixRelationshipsDistribution | `[PublicDistribution]` | `publicStixRelationshipsDistribution( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [PublicDistribution] @public` |
| publicStixRelationshipsMultiTimeSeries | `[MultiTimeSeries]` | `publicStixRelationshipsMultiTimeSeries( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): [MultiTimeSeries] @public` |
| publicStixRelationshipsNumber | `Number` | `publicStixRelationshipsNumber( uriKey: String! widgetId : String! startDate: DateTime endDate: DateTime ): Number @public` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| publicDashboardAdd | `PublicDashboard` | `publicDashboardAdd(input: PublicDashboardAddInput!): PublicDashboard @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |
| publicDashboardDelete | `ID` | `publicDashboardDelete(id: ID!): ID @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |
| publicDashboardFieldPatch | `PublicDashboard` | `publicDashboardFieldPatch(id: ID!, input: [EditInput!]!): PublicDashboard @auth(for: [EXPLORE_EXUPDATE_PUBLISH])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `PublicDashboardsOrdering` | `src/modules/publicDashboard/publicDashboard.graphql` |

<details><summary>Show definition details</summary>


**enum PublicDashboardsOrdering**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `user_id`
- `enabled`
- `dashboard`
- `uri_key`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `PublicDashboardAddInput` | `src/modules/publicDashboard/publicDashboard.graphql` |

<details><summary>Show definition details</summary>


**input PublicDashboardAddInput**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `uri_key` | `String!` |
| `description` | `String` |
| `dashboard_id` | `String!` |
| `allowed_markings_ids` | `[String!]` |
| `enabled` | `Boolean!` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `PublicDashboard` | `src/modules/publicDashboard/publicDashboard.graphql` |
| `PublicDashboardConnection` | `src/modules/publicDashboard/publicDashboard.graphql` |
| `PublicDashboardEdge` | `src/modules/publicDashboard/publicDashboard.graphql` |
| `PublicDistribution` | `src/modules/publicDashboard/publicDashboard.graphql` |

<details><summary>Show definition details</summary>


**type PublicDashboard**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

_Referenced by:_
- `type` `PublicDashboardEdge` (publicDashboard) in `src/modules/publicDashboard/publicDashboard.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `owner` | `Creator` |
| `description` | `String` |
| `dashboard_id` | `String!` |
| `dashboard` | `Workspace!` |
| `user_id` | `String!` |
| `public_manifest` | `String` |
| `private_manifest` | `String` |
| `uri_key` | `String!` |
| `allowed_markings_ids` | `[String!]` |
| `allowed_markings` | `[MarkingDefinitionShort!]` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `editContext` | `[EditUserContext!]` |
| `enabled` | `Boolean!` |


**type PublicDashboardConnection**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[PublicDashboardEdge!]!` |


**type PublicDashboardEdge**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

_Referenced by:_
- `type` `PublicDashboardConnection` (publicDashboard) in `src/modules/publicDashboard/publicDashboard.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `PublicDashboard!` |


**type PublicDistribution**  
Defined in `src/modules/publicDashboard/publicDashboard.graphql`

| Field | Type |
| --- | --- |
| `label` | `String!` |
| `entity` | `StixObjectOrStixRelationshipOrCreator` |
| `value` | `Int` |
| `breakdownDistribution` | `[Distribution]` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **17**
- Definition count: **151**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `PublicDashboardsOrdering`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `PublicDashboardAddInput`

</details>


#### Expanded: interface (7)
<a id="expanded-interface-7"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixCoreObject`
- `StixDomainObject`
- `StixObject`
- `StixRelationship`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`

</details>


#### Expanded: type (125)
<a id="expanded-type-125"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MarkingDefinitionShort`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `MultiTimeSeries`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `PublicDashboard`
- `PublicDashboardConnection`
- `PublicDashboardEdge`
- `PublicDistribution`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreObjectConnection`
- `StixCoreObjectEdge`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixDomainObjectConnection`
- `StixDomainObjectEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRelationshipConnection`
- `StixRelationshipEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TimeSeries`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: requestAccess
<a id="module-family-requestaccess"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 2 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/requestAccess/requestAccess.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (2)
<a id="mutation-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| requestAccessAdd | `ID` | `requestAccessAdd(input: RequestAccessAddInput!): ID @auth(for: [KNOWLEDGE])` |
| requestAccessConfigure | `RequestAccessConfiguration` | `requestAccessConfigure(input: RequestAccessConfigureInput!): RequestAccessConfiguration @auth(for: [SETTINGS_SETCUSTOMIZATION])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `RequestAccessType` | `src/modules/requestAccess/requestAccess.graphql` |

<details><summary>Show definition details</summary>


**enum RequestAccessType**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

_Referenced by:_
- `input` `RequestAccessAddInput` (requestAccess) in `src/modules/requestAccess/requestAccess.graphql`

Values:
- `organization_sharing`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `RequestAccessAddInput` | `src/modules/requestAccess/requestAccess.graphql` |
| `RequestAccessConfigureInput` | `src/modules/requestAccess/requestAccess.graphql` |

<details><summary>Show definition details</summary>


**input RequestAccessAddInput**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

| Field | Type |
| --- | --- |
| `request_access_reason` | `String` |
| `request_access_entities` | `[ID!]!` |
| `request_access_members` | `[ID!]!` |
| `request_access_type` | `RequestAccessType` |


**input RequestAccessConfigureInput**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

| Field | Type |
| --- | --- |
| `approved_status_id` | `ID` |
| `declined_status_id` | `ID` |
| `approval_admin` | `[ID]` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `RequestAccessConfiguration` | `src/modules/requestAccess/requestAccess.graphql` |
| `RequestAccessMember` | `src/modules/requestAccess/requestAccess.graphql` |
| `RequestAccessStatus` | `src/modules/requestAccess/requestAccess.graphql` |
| `RequestAccessWorkflow` | `src/modules/requestAccess/requestAccess.graphql` |

<details><summary>Show definition details</summary>


**type RequestAccessConfiguration**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

_Referenced by:_
- `type` `RfiRequestAccessConfiguration` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `EntitySetting` (entitySetting) in `src/modules/entitySetting/entitySetting.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `approved_status` | `Status` |
| `declined_status` | `Status` |
| `approval_admin` | `[RequestAccessMember]` |


**type RequestAccessMember**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

_Referenced by:_
- `type` `RequestAccessConfiguration` (requestAccess) in `src/modules/requestAccess/requestAccess.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |


**type RequestAccessStatus**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `template_id` | `String` |
| `statusTemplate` | `[StatusTemplate]` |


**type RequestAccessWorkflow**  
Defined in `src/modules/requestAccess/requestAccess.graphql`

| Field | Type |
| --- | --- |
| `approved_workflow_id` | `String` |
| `declined_workflow_id` | `String` |
| `approval_admin` | `[ID]` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **8**
- Definition count: **10**


#### Expanded: enum (1)
<a id="expanded-enum-1"></a>

<details><summary>Show names</summary>

- `RequestAccessType`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `RequestAccessAddInput`
- `RequestAccessConfigureInput`

</details>


#### Expanded: type (7)
<a id="expanded-type-7"></a>

<details><summary>Show names</summary>

- `EditUserContext`
- `RequestAccessConfiguration`
- `RequestAccessMember`
- `RequestAccessStatus`
- `RequestAccessWorkflow`
- `Status`
- `StatusTemplate`

</details>



## Module family: savedFilter
<a id="module-family-savedfilter"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 1 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/savedFilter/savedFilter.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (1)
<a id="query-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| savedFilters | `SavedFilterConnection` | `savedFilters( first: Int after: ID orderBy: SavedFilterOrdering orderMode: OrderingMode filters: FilterGroup search: String ): SavedFilterConnection @auth` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| savedFilterAdd | `SavedFilter` | `savedFilterAdd(input: SavedFilterAddInput!): SavedFilter @auth` |
| savedFilterDelete | `ID` | `savedFilterDelete(id: ID!): ID @auth` |
| savedFilterFieldPatch | `SavedFilter` | `savedFilterFieldPatch(id: ID!, input: [EditInput!]): SavedFilter @auth` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `SavedFilterOrdering` | `src/modules/savedFilter/savedFilter.graphql` |

<details><summary>Show definition details</summary>


**enum SavedFilterOrdering**  
Defined in `src/modules/savedFilter/savedFilter.graphql`

Values:
- `name`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `SavedFilterAddInput` | `src/modules/savedFilter/savedFilter.graphql` |

<details><summary>Show definition details</summary>


**input SavedFilterAddInput**  
Defined in `src/modules/savedFilter/savedFilter.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `filters` | `String!` |
| `scope` | `String!` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `SavedFilter` | `src/modules/savedFilter/savedFilter.graphql` |
| `SavedFilterConnection` | `src/modules/savedFilter/savedFilter.graphql` |
| `SavedFilterEdge` | `src/modules/savedFilter/savedFilter.graphql` |

<details><summary>Show definition details</summary>


**type SavedFilter**  
Defined in `src/modules/savedFilter/savedFilter.graphql`

_Referenced by:_
- `type` `SavedFilterEdge` (savedFilter) in `src/modules/savedFilter/savedFilter.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `name` | `String!` |
| `filters` | `String!` |
| `scope` | `String!` |


**type SavedFilterConnection**  
Defined in `src/modules/savedFilter/savedFilter.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[SavedFilterEdge!]` |


**type SavedFilterEdge**  
Defined in `src/modules/savedFilter/savedFilter.graphql`

_Referenced by:_
- `type` `SavedFilterConnection` (savedFilter) in `src/modules/savedFilter/savedFilter.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `SavedFilter!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **8**
- Definition count: **10**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `SavedFilterOrdering`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `SavedFilterAddInput`

</details>


#### Expanded: scalar (1)
<a id="expanded-scalar-1"></a>

<details><summary>Show names</summary>

- `Any`

</details>


#### Expanded: type (5)
<a id="expanded-type-5"></a>

<details><summary>Show names</summary>

- `Metric`
- `PageInfo`
- `SavedFilter`
- `SavedFilterConnection`
- `SavedFilterEdge`

</details>



## Module family: securityCoverage
<a id="module-family-securitycoverage"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/securityCoverage/securityCoverage.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| securityCoverage | `SecurityCoverage` | `securityCoverage(id: String!): SecurityCoverage @auth(for: [KNOWLEDGE])` |
| securityCoverages | `SecurityCoverageConnection` | `securityCoverages( first: Int after: ID orderBy: SecurityCoverageOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): SecurityCoverageConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| securityCoverageAdd | `SecurityCoverage` | `securityCoverageAdd(input: SecurityCoverageAddInput!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageContextClean | `SecurityCoverage` | `securityCoverageContextClean(id: ID!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageContextPatch | `SecurityCoverage` | `securityCoverageContextPatch(id: ID!, input: EditContext!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageDelete | `ID` | `securityCoverageDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| securityCoverageFieldPatch | `SecurityCoverage` | `securityCoverageFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageRelationAdd | `StixRefRelationship` | `securityCoverageRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityCoverageRelationDelete | `SecurityCoverage` | `securityCoverageRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): SecurityCoverage @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `SecurityCoverageOrdering` | `src/modules/securityCoverage/securityCoverage.graphql` |

<details><summary>Show definition details</summary>


**enum SecurityCoverageOrdering**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

Values:
- `name`
- `confidence`
- `created`
- `created_at`
- `creator`
- `modified`
- `updated_at`
- `objectMarking`
- `coverage_last_result`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `SecurityCoverageAddInput` | `src/modules/securityCoverage/securityCoverage.graphql` |

<details><summary>Show definition details</summary>


**input SecurityCoverageAddInput**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `confidence` | `Int` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `objectCovered` | `String!` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `revoked` | `Boolean` |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `external_uri` | `String` |
| `externalReferences` | `[String]` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `update` | `Boolean` |
| `auto_enrichment_disable` | `Boolean!` |
| `periodicity` | `String` |
| `duration` | `String` |
| `type_affinity` | `String` |
| `platforms_affinity` | `[String]` |
| `coverage_last_result` | `DateTime` |
| `coverage_valid_from` | `DateTime` |
| `coverage_valid_to` | `DateTime` |
| `coverage_information` | `[SecurityCoverageExpectation!]` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `CoverageResult` | `src/modules/securityCoverage/securityCoverage.graphql` |
| `SecurityCoverage` | `src/modules/securityCoverage/securityCoverage.graphql` |
| `SecurityCoverageConnection` | `src/modules/securityCoverage/securityCoverage.graphql` |
| `SecurityCoverageEdge` | `src/modules/securityCoverage/securityCoverage.graphql` |

<details><summary>Show definition details</summary>


**type CoverageResult**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

_Referenced by:_
- `type` `SecurityCoverage` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`

| Field | Type |
| --- | --- |
| `coverage_name` | `String!` |
| `coverage_score` | `Int!` |


**type SecurityCoverage**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

_Referenced by:_
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `Grouping` (grouping) in `src/modules/grouping/grouping.graphql`
- `type` `SecurityCoverageEdge` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `refreshed_at` | `DateTime` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `identity_class` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `contact_information` | `String` |
| `roles` | `[String]` |
| `x_opencti_aliases` | `[String]` |
| `x_opencti_reliability` | `String` |
| `security_platform_type` | `String` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `coverage_last_result` | `DateTime` |
| `coverage_valid_from` | `DateTime` |
| `coverage_valid_to` | `DateTime` |
| `coverage_information` | `[CoverageResult!]` |
| `external_uri` | `String` |
| `periodicity` | `String` |
| `duration` | `String` |
| `type_affinity` | `String` |
| `platforms_affinity` | `[String!]` |
| `auto_enrichment_disable` | `Boolean` |
| `objectCovered` | `StixDomainObject` |
| `toStixBundle` | `String` |


**type SecurityCoverageConnection**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[SecurityCoverageEdge!]!` |


**type SecurityCoverageEdge**  
Defined in `src/modules/securityCoverage/securityCoverage.graphql`

_Referenced by:_
- `type` `SecurityCoverageConnection` (securityCoverage) in `src/modules/securityCoverage/securityCoverage.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `SecurityCoverage!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **14**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `SecurityCoverageOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (5)
<a id="expanded-input-5"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `SecurityCoverageAddInput`
- `SecurityCoverageExpectation`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (115)
<a id="expanded-type-115"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SecurityCoverageConnection`
- `SecurityCoverageEdge`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: securityPlatform
<a id="module-family-securityplatform"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/securityPlatform/securityPlatform.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| securityPlatform | `SecurityPlatform` | `securityPlatform(id: String!): SecurityPlatform @auth(for: [KNOWLEDGE])` |
| securityPlatforms | `SecurityPlatformConnection` | `securityPlatforms( first: Int after: ID orderBy: SecurityPlatformOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): SecurityPlatformConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| securityPlatformAdd | `SecurityPlatform` | `securityPlatformAdd(input: SecurityPlatformAddInput!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformContextClean | `SecurityPlatform` | `securityPlatformContextClean(id: ID!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformContextPatch | `SecurityPlatform` | `securityPlatformContextPatch(id: ID!, input: EditContext!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformDelete | `ID` | `securityPlatformDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| securityPlatformFieldPatch | `SecurityPlatform` | `securityPlatformFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformRelationAdd | `StixRefRelationship` | `securityPlatformRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| securityPlatformRelationDelete | `SecurityPlatform` | `securityPlatformRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): SecurityPlatform @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `SecurityPlatformOrdering` | `src/modules/securityPlatform/securityPlatform.graphql` |

<details><summary>Show definition details</summary>


**enum SecurityPlatformOrdering**  
Defined in `src/modules/securityPlatform/securityPlatform.graphql`

Values:
- `name`
- `confidence`
- `created`
- `created_at`
- `modified`
- `updated_at`
- `security_platform_type`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `SecurityPlatformAddInput` | `src/modules/securityPlatform/securityPlatform.graphql` |

<details><summary>Show definition details</summary>


**input SecurityPlatformAddInput**  
Defined in `src/modules/securityPlatform/securityPlatform.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `security_platform_type` | `String` |
| `confidence` | `Int` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectLabel` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `revoked` | `Boolean` |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `externalReferences` | `[String]` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `SecurityPlatform` | `src/modules/securityPlatform/securityPlatform.graphql` |
| `SecurityPlatformConnection` | `src/modules/securityPlatform/securityPlatform.graphql` |
| `SecurityPlatformEdge` | `src/modules/securityPlatform/securityPlatform.graphql` |

<details><summary>Show definition details</summary>


**type SecurityPlatform**  
Defined in `src/modules/securityPlatform/securityPlatform.graphql`

_Referenced by:_
- `type` `SecurityPlatformEdge` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `identity_class` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `contact_information` | `String` |
| `roles` | `[String]` |
| `x_opencti_aliases` | `[String]` |
| `x_opencti_reliability` | `String` |
| `security_platform_type` | `String` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |


**type SecurityPlatformConnection**  
Defined in `src/modules/securityPlatform/securityPlatform.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[SecurityPlatformEdge!]!` |


**type SecurityPlatformEdge**  
Defined in `src/modules/securityPlatform/securityPlatform.graphql`

_Referenced by:_
- `type` `SecurityPlatformConnection` (securityPlatform) in `src/modules/securityPlatform/securityPlatform.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `SecurityPlatform!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **13**
- Definition count: **144**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `SecurityPlatformOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `SecurityPlatformAddInput`
- `StixRefRelationshipAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SecurityPlatform`
- `SecurityPlatformConnection`
- `SecurityPlatformEdge`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: stixCyberObservable
<a id="module-family-stixcyberobservable"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 0 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (0)
<a id="mutation-0"></a>

_None._


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### type (1)
<a id="type-1"></a>

| Name | File |
| --- | --- |
| `StixCyberObservableEditMutations` | `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql` |

<details><summary>Show definition details</summary>


**type StixCyberObservableEditMutations**  
Defined in `src/modules/stixCyberObservable/deprecated/stixCyberObservable.graphql`

| Field | Type |
| --- | --- |
| `promote` | `StixCyberObservable` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **1**
- Definition count: **145**


#### Expanded: enum (9)
<a id="expanded-enum-9"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: interface (6)
<a id="expanded-interface-6"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixCyberObservable`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `DateTime`
- `StixId`

</details>


#### Expanded: type (125)
<a id="expanded-type-125"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DecayChartData`
- `DecayHistory`
- `DecayLiveDetails`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Hash`
- `Indicator`
- `IndicatorConnection`
- `IndicatorDecayExclusionRule`
- `IndicatorDecayRule`
- `IndicatorEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservablesValues`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixCyberObservableConnection`
- `StixCyberObservableEdge`
- `StixCyberObservableEditMutations`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: support
<a id="module-family-support"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/support/support.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| supportPackage | `SupportPackage` | `supportPackage(id: String!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |
| supportPackages | `SupportPackageConnection` | `supportPackages( first: Int after: ID orderBy: SupportPackageOrdering orderMode: OrderingMode filters: FilterGroup search: String ): SupportPackageConnection @auth(for: [SETTINGS_SUPPORT])` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| supportPackageAdd | `SupportPackage` | `supportPackageAdd(input: SupportPackageAddInput!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |
| supportPackageDelete | `ID` | `supportPackageDelete(id: ID!): ID @auth(for: [SETTINGS_SUPPORT])` |
| supportPackageForceZip | `SupportPackage` | `supportPackageForceZip(input: SupportPackageForceZipInput!): SupportPackage @auth(for: [SETTINGS_SUPPORT])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (2)
<a id="enum-2"></a>

| Name | File |
| --- | --- |
| `PackageStatus` | `src/modules/support/support.graphql` |
| `SupportPackageOrdering` | `src/modules/support/support.graphql` |

<details><summary>Show definition details</summary>


**enum PackageStatus**  
Defined in `src/modules/support/support.graphql`

_Referenced by:_
- `type` `SupportPackage` (support) in `src/modules/support/support.graphql`

Values:
- `IN_PROGRESS`
- `READY`
- `IN_ERROR`


**enum SupportPackageOrdering**  
Defined in `src/modules/support/support.graphql`

Values:
- `name`
- `created_at`
- `package_status`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `SupportPackageAddInput` | `src/modules/support/support.graphql` |
| `SupportPackageForceZipInput` | `src/modules/support/support.graphql` |

<details><summary>Show definition details</summary>


**input SupportPackageAddInput**  
Defined in `src/modules/support/support.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |


**input SupportPackageForceZipInput**  
Defined in `src/modules/support/support.graphql`

| Field | Type |
| --- | --- |
| `id` | `String!` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `SupportPackage` | `src/modules/support/support.graphql` |
| `SupportPackageConnection` | `src/modules/support/support.graphql` |
| `SupportPackageEdge` | `src/modules/support/support.graphql` |

<details><summary>Show definition details</summary>


**type SupportPackage**  
Defined in `src/modules/support/support.graphql`

_Referenced by:_
- `type` `SupportPackageEdge` (support) in `src/modules/support/support.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `name` | `String!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created_at` | `DateTime!` |
| `package_status` | `PackageStatus!` |
| `package_url` | `String` |
| `package_upload_dir` | `String` |
| `nodes_count` | `Int!` |
| `createdBy` | `Individual` |
| `creators` | `[Creator!]` |


**type SupportPackageConnection**  
Defined in `src/modules/support/support.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[SupportPackageEdge!]!` |


**type SupportPackageEdge**  
Defined in `src/modules/support/support.graphql`

_Referenced by:_
- `type` `SupportPackageConnection` (support) in `src/modules/support/support.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `SupportPackage!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **10**
- Definition count: **139**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `PackageStatus`
- `RolesOrdering`
- `State`
- `SupportPackageOrdering`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `SupportPackageAddInput`
- `SupportPackageForceZipInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `DateTime`
- `StixId`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Individual`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `SupportPackage`
- `SupportPackageConnection`
- `SupportPackageEdge`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: task
<a id="module-family-task"></a>

| Metric | Value |
| --- | --- |
| SDL files | 2 |
| Query fields | 5 |
| Mutation fields | 8 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/task/task-template/task-template.graphql` |
| `src/modules/task/task.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (5)
<a id="query-5"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| task | `Task` | `task(id: String!): Task @auth` |
| taskContainsStixObjectOrStixRelationship | `Boolean` | `taskContainsStixObjectOrStixRelationship(id: String!, stixObjectOrStixRelationshipId: String!): Boolean @auth(for: [KNOWLEDGE])` |
| taskTemplate | `TaskTemplate` | `taskTemplate(id: String!): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplates | `TaskTemplateConnection` | `taskTemplates( first: Int after: ID orderBy: TaskTemplatesOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): TaskTemplateConnection @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| tasks | `TaskConnection` | `tasks( first: Int after: ID orderBy: TasksOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): TaskConnection @auth` |


#### Mutation (8)
<a id="mutation-8"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| taskAdd | `Task` | `taskAdd(input: TaskAddInput!): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskDelete | `ID` | `taskDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| taskFieldPatch | `Task` | `taskFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskRelationAdd | `StixRefRelationship` | `taskRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskRelationDelete | `Task` | `taskRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): Task @auth(for: [KNOWLEDGE_KNUPDATE])` |
| taskTemplateAdd | `TaskTemplate` | `taskTemplateAdd(input: TaskTemplateAddInput!): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplateDelete | `ID` | `taskTemplateDelete(id: ID!): ID @auth(for: [SETTINGS_SETCASETEMPLATES])` |
| taskTemplateFieldPatch | `TaskTemplate` | `taskTemplateFieldPatch(id: ID!, input: [EditInput!]!, commitMessage: String, references: [String]): TaskTemplate @auth(for: [SETTINGS_SETCASETEMPLATES])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (2)
<a id="enum-2"></a>

| Name | File |
| --- | --- |
| `TaskTemplatesOrdering` | `src/modules/task/task-template/task-template.graphql` |
| `TasksOrdering` | `src/modules/task/task.graphql` |

<details><summary>Show definition details</summary>


**enum TaskTemplatesOrdering**  
Defined in `src/modules/task/task-template/task-template.graphql`

Values:
- `name`
- `description`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `creator`
- `_score`


**enum TasksOrdering**  
Defined in `src/modules/task/task.graphql`

Values:
- `name`
- `description`
- `created`
- `modified`
- `context`
- `created_at`
- `updated_at`
- `creator`
- `createdBy`
- `x_opencti_workflow_id`
- `confidence`
- `due_date`
- `objectAssignee`
- `_score`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `TaskAddInput` | `src/modules/task/task.graphql` |
| `TaskTemplateAddInput` | `src/modules/task/task-template/task-template.graphql` |

<details><summary>Show definition details</summary>


**input TaskAddInput**  
Defined in `src/modules/task/task.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |
| `created` | `DateTime` |
| `due_date` | `DateTime` |
| `objectAssignee` | `[String]` |
| `objectParticipant` | `[String]` |
| `objectLabel` | `[String]` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `createdBy` | `String` |
| `objects` | `[String]` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


**input TaskTemplateAddInput**  
Defined in `src/modules/task/task-template/task-template.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `description` | `String` |


</details>


#### type (6)
<a id="type-6"></a>

| Name | File |
| --- | --- |
| `Task` | `src/modules/task/task.graphql` |
| `TaskConnection` | `src/modules/task/task.graphql` |
| `TaskEdge` | `src/modules/task/task.graphql` |
| `TaskTemplate` | `src/modules/task/task-template/task-template.graphql` |
| `TaskTemplateConnection` | `src/modules/task/task-template/task-template.graphql` |
| `TaskTemplateEdge` | `src/modules/task/task-template/task-template.graphql` |

<details><summary>Show definition details</summary>


**type Task**  
Defined in `src/modules/task/task.graphql`

_Referenced by:_
- `type` `TaskEdge` (task) in `src/modules/task/task.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `authorized_members` | `[MemberAccess!]` |
| `authorized_members_activation_date` | `DateTime` |
| `currentUserAccessRight` | `String` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `relatedContainers(first: Int after: ID orderBy: ContainersOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] viaTypes: [String])` | `ContainerConnection` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `filesFromTemplate(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `fintelTemplates` | `[FintelTemplate!]` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `name` | `String!` |
| `description` | `String` |
| `due_date` | `DateTime` |
| `content_mapping` | `String` |


**type TaskConnection**  
Defined in `src/modules/task/task.graphql`

_Referenced by:_
- `interface` `Case` (case) in `src/modules/case/case.graphql`
- `type` `CaseIncident` (case) in `src/modules/case/case-incident/case-incident.graphql`
- `type` `CaseRfi` (case) in `src/modules/case/case-rfi/case-rfi.graphql`
- `type` `CaseRft` (case) in `src/modules/case/case-rft/case-rft.graphql`
- `type` `Feedback` (case) in `src/modules/case/feedback/feedback.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[TaskEdge!]!` |


**type TaskEdge**  
Defined in `src/modules/task/task.graphql`

_Referenced by:_
- `type` `TaskConnection` (task) in `src/modules/task/task.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Task!` |


**type TaskTemplate**  
Defined in `src/modules/task/task-template/task-template.graphql`

_Referenced by:_
- `type` `TaskTemplateEdge` (task) in `src/modules/task/task-template/task-template.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `name` | `String!` |
| `description` | `String` |


**type TaskTemplateConnection**  
Defined in `src/modules/task/task-template/task-template.graphql`

_Referenced by:_
- `type` `CaseTemplate` (case) in `src/modules/case/case-template/case-template.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[TaskTemplateEdge!]!` |


**type TaskTemplateEdge**  
Defined in `src/modules/task/task-template/task-template.graphql`

_Referenced by:_
- `type` `TaskTemplateConnection` (task) in `src/modules/task/task-template/task-template.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `TaskTemplate!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **19**
- Definition count: **145**


#### Expanded: enum (12)
<a id="expanded-enum-12"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `TaskTemplatesOrdering`
- `TasksOrdering`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (4)
<a id="expanded-input-4"></a>

<details><summary>Show names</summary>

- `EditInput`
- `StixRefRelationshipAddInput`
- `TaskAddInput`
- `TaskTemplateAddInput`

</details>


#### Expanded: interface (5)
<a id="expanded-interface-5"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (116)
<a id="expanded-type-116"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `TaskTemplate`
- `TaskTemplateConnection`
- `TaskTemplateEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: theme
<a id="module-family-theme"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 4 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/theme/theme.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| theme | `Theme` | `theme(id: ID!): Theme @public` |
| themes | `ThemeConnection` | `themes( first: Int after: ID orderBy: ThemeOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): ThemeConnection @public` |


#### Mutation (4)
<a id="mutation-4"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| themeAdd | `Theme` | `themeAdd(input: ThemeAddInput!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |
| themeDelete | `ID` | `themeDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| themeFieldPatch | `Theme` | `themeFieldPatch(id: ID!,input: [EditInput!]!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |
| themeImport | `Theme` | `themeImport(file: Upload!): Theme @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `ThemeOrdering` | `src/modules/theme/theme.graphql` |

<details><summary>Show definition details</summary>


**enum ThemeOrdering**  
Defined in `src/modules/theme/theme.graphql`

Values:
- `name`
- `created_at`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `ThemeAddInput` | `src/modules/theme/theme.graphql` |

<details><summary>Show definition details</summary>


**input ThemeAddInput**  
Defined in `src/modules/theme/theme.graphql`

| Field | Type |
| --- | --- |
| `name` | `String!` |
| `theme_background` | `String!` |
| `theme_paper` | `String!` |
| `theme_nav` | `String!` |
| `theme_primary` | `String!` |
| `theme_secondary` | `String!` |
| `theme_accent` | `String!` |
| `theme_logo` | `String` |
| `theme_logo_collapsed` | `String` |
| `theme_logo_login` | `String` |
| `theme_text_color` | `String!` |
| `built_in` | `Boolean` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Theme` | `src/modules/theme/theme.graphql` |
| `ThemeConnection` | `src/modules/theme/theme.graphql` |
| `ThemeEdge` | `src/modules/theme/theme.graphql` |

<details><summary>Show definition details</summary>


**type Theme**  
Defined in `src/modules/theme/theme.graphql`

_Referenced by:_
- `type` `ThemeEdge` (theme) in `src/modules/theme/theme.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `name` | `String!` |
| `theme_background` | `String!` |
| `theme_paper` | `String!` |
| `theme_nav` | `String!` |
| `theme_primary` | `String!` |
| `theme_secondary` | `String!` |
| `theme_accent` | `String!` |
| `theme_logo` | `String` |
| `theme_logo_collapsed` | `String` |
| `theme_logo_login` | `String` |
| `theme_text_color` | `String!` |
| `toConfigurationExport` | `String!` |
| `built_in` | `Boolean` |
| `metrics` | `[Metric]` |


**type ThemeConnection**  
Defined in `src/modules/theme/theme.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[ThemeEdge!]!` |


**type ThemeEdge**  
Defined in `src/modules/theme/theme.graphql`

_Referenced by:_
- `type` `ThemeConnection` (theme) in `src/modules/theme/theme.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Theme!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **9**
- Definition count: **11**


#### Expanded: enum (2)
<a id="expanded-enum-2"></a>

<details><summary>Show names</summary>

- `EditOperation`
- `ThemeOrdering`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `ThemeAddInput`

</details>


#### Expanded: scalar (2)
<a id="expanded-scalar-2"></a>

<details><summary>Show names</summary>

- `Any`
- `Upload`

</details>


#### Expanded: type (5)
<a id="expanded-type-5"></a>

<details><summary>Show names</summary>

- `Metric`
- `PageInfo`
- `Theme`
- `ThemeConnection`
- `ThemeEdge`

</details>



## Module family: threatActorIndividual
<a id="module-family-threatactorindividual"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 7 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/threatActorIndividual/threatActorIndividual.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| threatActorIndividual | `ThreatActorIndividual` | `threatActorIndividual(id: String!): ThreatActorIndividual @auth(for: [KNOWLEDGE])` |
| threatActorsIndividuals | `ThreatActorIndividualConnection` | `threatActorsIndividuals( first: Int after: ID orderBy: ThreatActorsIndividualOrdering orderMode: OrderingMode filters: FilterGroup search: String toStix: Boolean ): ThreatActorIndividualConnection @auth(for: [KNOWLEDGE])` |


#### Mutation (7)
<a id="mutation-7"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| threatActorIndividualAdd | `ThreatActorIndividual` | `threatActorIndividualAdd(input: ThreatActorIndividualAddInput!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualContextClean | `ThreatActorIndividual` | `threatActorIndividualContextClean(id: ID!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualContextPatch | `ThreatActorIndividual` | `threatActorIndividualContextPatch(id: ID!, input: EditContext): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualDelete | `ID` | `threatActorIndividualDelete(id: ID!): ID @auth(for: [KNOWLEDGE_KNUPDATE_KNDELETE])` |
| threatActorIndividualFieldPatch | `ThreatActorIndividual` | `threatActorIndividualFieldPatch(id: ID!, input: [EditInput]!, commitMessage: String, references: [String]): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualRelationAdd | `StixRefRelationship` | `threatActorIndividualRelationAdd(id: ID!, input: StixRefRelationshipAddInput!): StixRefRelationship @auth(for: [KNOWLEDGE_KNUPDATE])` |
| threatActorIndividualRelationDelete | `ThreatActorIndividual` | `threatActorIndividualRelationDelete(id: ID!, toId: StixRef!, relationship_type: String!): ThreatActorIndividual @auth(for: [KNOWLEDGE_KNUPDATE])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `ThreatActorsIndividualOrdering` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |

<details><summary>Show definition details</summary>


**enum ThreatActorsIndividualOrdering**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

Values:
- `name`
- `created`
- `modified`
- `created_at`
- `updated_at`
- `x_opencti_workflow_id`
- `sophistication`
- `resource_level`
- `confidence`
- `_score`
- `objectMarking`
- `threat_actor_types`


</details>


#### input (2)
<a id="input-2"></a>

| Name | File |
| --- | --- |
| `MeasureInput` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |
| `ThreatActorIndividualAddInput` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |

<details><summary>Show definition details</summary>


**input MeasureInput**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

_Referenced by:_
- `input` `ThreatActorIndividualAddInput` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `measure` | `Float` |
| `date_seen` | `DateTime` |


**input ThreatActorIndividualAddInput**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `threat_actor_types` | `[String]` |
| `first_seen` | `DateTime` |
| `last_seen` | `DateTime` |
| `roles` | `[String]` |
| `goals` | `[String]` |
| `sophistication` | `String` |
| `resource_level` | `String` |
| `primary_motivation` | `String` |
| `secondary_motivations` | `[String]` |
| `personal_motivations` | `[String]` |
| `date_of_birth` | `DateTime` |
| `gender` | `String` |
| `job_title` | `String` |
| `marital_status` | `String` |
| `eye_color` | `String` |
| `hair_color` | `String` |
| `height` | `[MeasureInput!]` |
| `weight` | `[MeasureInput!]` |
| `confidence` | `Int` |
| `revoked` | `Boolean` |
| `lang` | `String` |
| `createdBy` | `String` |
| `objectMarking` | `[String]` |
| `objectOrganization` | `[String]` |
| `objectAssignee` | `[String]` |
| `objectLabel` | `[String]` |
| `bornIn` | `String` |
| `ethnicity` | `String` |
| `externalReferences` | `[String]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `clientMutationId` | `String` |
| `x_opencti_workflow_id` | `String` |
| `x_opencti_modified_at` | `DateTime` |
| `update` | `Boolean` |
| `file` | `Upload` |
| `fileMarkings` | `[String]` |
| `files` | `[Upload]` |
| `filesMarkings` | `[[String]]` |
| `upsertOperations` | `[EditInput!]` |


</details>


#### type (4)
<a id="type-4"></a>

| Name | File |
| --- | --- |
| `Measure` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |
| `ThreatActorIndividual` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |
| `ThreatActorIndividualConnection` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |
| `ThreatActorIndividualEdge` | `src/modules/threatActorIndividual/threatActorIndividual.graphql` |

<details><summary>Show definition details</summary>


**type Measure**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

_Referenced by:_
- `type` `ThreatActorIndividual` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `index` | `Int` |
| `measure` | `Float` |
| `date_seen` | `DateTime` |


**type ThreatActorIndividual**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

_Referenced by:_
- `type` `ThreatActorIndividualEdge` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `createdBy` | `Identity` |
| `numberOfConnectedElement` | `Int!` |
| `objectMarking` | `[MarkingDefinition!]` |
| `objectOrganization` | `[Organization!]` |
| `objectLabel` | `[Label!]` |
| `externalReferences(first: Int)` | `ExternalReferenceConnection` |
| `containersNumber` | `Number` |
| `containers(first: Int, entityTypes: [String!])` | `ContainerConnection` |
| `reports(first: Int)` | `ReportConnection` |
| `notes(first: Int)` | `NoteConnection` |
| `opinions(first: Int)` | `OpinionConnection` |
| `observedData(first: Int)` | `ObservedDataConnection` |
| `groupings(first: Int)` | `GroupingConnection` |
| `cases(first: Int)` | `CaseConnection` |
| `stixCoreRelationships(first: Int after: ID orderBy: StixCoreRelationshipsOrdering orderMode: OrderingMode fromId: StixRef toId: StixRef fromTypes: [String] toTypes: [String] relationship_type: String startTimeStart: DateTime startTimeStop:…` | `StixCoreRelationshipConnection` |
| `stixCoreObjectsDistribution(relationship_type: [String] toTypes: [String] field: String! startDate: DateTime endDate: DateTime dateAttribute: String operation: StatsOperation! limit: Int order: String types: [String] filters: FilterGroup s…` | `[Distribution]` |
| `stixCoreRelationshipsDistribution(field: String! operation: StatsOperation! startDate: DateTime endDate: DateTime dateAttribute: String isTo: Boolean limit: Int order: String elementWithTargetTypes: [String] fromId: [String] fromRole: Stri…` | `[Distribution]` |
| `opinions_metrics` | `OpinionsMetrics` |
| `revoked` | `Boolean!` |
| `confidence` | `Int` |
| `lang` | `String` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `x_opencti_graph_data` | `String` |
| `objectAssignee` | `[Assignee!]` |
| `objectParticipant` | `[Participant!]` |
| `avatar` | `OpenCtiFile` |
| `name` | `String!` |
| `description` | `String` |
| `aliases` | `[String]` |
| `threat_actor_types` | `[String]` |
| `first_seen` | `DateTime` |
| `last_seen` | `DateTime` |
| `roles` | `[String]` |
| `goals` | `[String]` |
| `sophistication` | `String` |
| `resource_level` | `String` |
| `primary_motivation` | `String` |
| `secondary_motivations` | `[String]` |
| `personal_motivations` | `[String]` |
| `locations` | `LocationConnection` |
| `countries` | `CountryConnection` |
| `date_of_birth` | `DateTime` |
| `gender` | `String` |
| `job_title` | `String` |
| `marital_status` | `String` |
| `eye_color` | `String` |
| `hair_color` | `String` |
| `height` | `[Measure!]` |
| `weight` | `[Measure!]` |
| `bornIn` | `Country` |
| `ethnicity` | `Country` |
| `creators` | `[Creator!]` |
| `toStix(version: Version)` | `String` |
| `importFiles(first: Int prefixMimeType: String after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `pendingFiles(first: Int after: ID orderBy: FileOrdering orderMode: OrderingMode search: String filters: FilterGroup)` | `FileConnection` |
| `exportFiles(first: Int)` | `FileConnection` |
| `editContext` | `[EditUserContext!]` |
| `connectors(onlyAlive: Boolean)` | `[Connector]` |
| `jobs(first: Int)` | `[Work]` |
| `status` | `Status` |
| `workflowEnabled` | `Boolean` |
| `pirInformation(pirId: ID!)` | `PirInformation` |
| `securityCoverage` | `SecurityCoverage` |


**type ThreatActorIndividualConnection**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[ThreatActorIndividualEdge]` |


**type ThreatActorIndividualEdge**  
Defined in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

_Referenced by:_
- `type` `ThreatActorIndividualConnection` (threatActorIndividual) in `src/modules/threatActorIndividual/threatActorIndividual.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `ThreatActorIndividual!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **15**
- Definition count: **155**


#### Expanded: enum (11)
<a id="expanded-enum-11"></a>

<details><summary>Show names</summary>

- `ConnectorPriorityGroup`
- `DraftOperation`
- `EditOperation`
- `EffectiveConfidenceLevelSourceType`
- `OrderingMode`
- `RolesOrdering`
- `State`
- `ThreatActorsIndividualOrdering`
- `UnitSystem`
- `Version`
- `WidgetPerspective`

</details>


#### Expanded: input (5)
<a id="expanded-input-5"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `MeasureInput`
- `StixRefRelationshipAddInput`
- `ThreatActorIndividualAddInput`

</details>


#### Expanded: interface (6)
<a id="expanded-interface-6"></a>

<details><summary>Show names</summary>

- `Case`
- `Container`
- `Identity`
- `Location`
- `StixDomainObject`
- `StixObject`

</details>


#### Expanded: scalar (5)
<a id="expanded-scalar-5"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `StixRef`
- `Upload`

</details>


#### Expanded: type (125)
<a id="expanded-type-125"></a>

<details><summary>Show names</summary>

- `Assignee`
- `Capability`
- `CaseConnection`
- `CaseEdge`
- `ConfidenceLevel`
- `ConfidenceLevelOverride`
- `Connector`
- `ConnectorConfig`
- `ConnectorConfiguration`
- `ConnectorHealthMetrics`
- `ConnectorInfo`
- `ConnectorQueueDetails`
- `ContainerConnection`
- `ContainerEdge`
- `Country`
- `CountryConnection`
- `CountryEdge`
- `CoverageResult`
- `Creator`
- `DefaultMarking`
- `Display`
- `DisplayStep`
- `Distribution`
- `DraftVersion`
- `EditUserContext`
- `EffectiveConfidenceLevel`
- `EffectiveConfidenceLevelOverride`
- `EffectiveConfidenceLevelSource`
- `ExternalReference`
- `ExternalReferenceConnection`
- `ExternalReferenceEdge`
- `File`
- `FileConnection`
- `FileEdge`
- `FileMetadata`
- `FintelTemplate`
- `FintelTemplateWidget`
- `Group`
- `GroupConnection`
- `GroupEdge`
- `Grouping`
- `GroupingConnection`
- `GroupingEdge`
- `Inference`
- `InferenceAttribute`
- `KillChainPhase`
- `Label`
- `LocationConnection`
- `LocationEdge`
- `ManagerContractConfiguration`
- `ManagerContractExcerpt`
- `MarkingDefinition`
- `Measure`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `Note`
- `NoteConnection`
- `NoteEdge`
- `Notifier`
- `NotifierConnector`
- `Number`
- `ObservedData`
- `ObservedDataConnection`
- `ObservedDataEdge`
- `OpenCtiFile`
- `Opinion`
- `OpinionConnection`
- `OpinionEdge`
- `OpinionsMetrics`
- `Organization`
- `OrganizationConnection`
- `OrganizationEdge`
- `PageInfo`
- `Participant`
- `PirCriterion`
- `PirDependency`
- `PirExplanation`
- `PirInformation`
- `RabbitMQConnection`
- `Region`
- `RegionConnection`
- `RegionEdge`
- `Report`
- `ReportConnection`
- `ReportEdge`
- `Representative`
- `Role`
- `RoleConnection`
- `RoleEdge`
- `Rule`
- `S3Connection`
- `Sector`
- `SectorConnection`
- `SectorEdge`
- `SecurityCoverage`
- `SessionDetail`
- `Status`
- `StatusTemplate`
- `StixCoreRelationship`
- `StixCoreRelationshipConnection`
- `StixCoreRelationshipEdge`
- `StixObjectOrStixRelationshipConnection`
- `StixObjectOrStixRelationshipEdge`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `StixRefRelationship`
- `Task`
- `TaskConnection`
- `TaskEdge`
- `ThreatActorIndividual`
- `ThreatActorIndividualConnection`
- `ThreatActorIndividualEdge`
- `User`
- `UserConnection`
- `UserEdge`
- `Widget`
- `WidgetColumn`
- `WidgetDataSelection`
- `WidgetLayout`
- `WidgetParameters`
- `Work`
- `WorkMessage`
- `WorkTracking`
- `Workspace`

</details>


#### Expanded: union (3)
<a id="expanded-union-3"></a>

<details><summary>Show names</summary>

- `EffectiveConfidenceLevelSourceObject`
- `StixObjectOrStixRelationship`
- `StixObjectOrStixRelationshipOrCreator`

</details>



## Module family: vocabulary
<a id="module-family-vocabulary"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 3 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/vocabulary/vocabulary.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (3)
<a id="query-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| vocabularies | `VocabularyConnection` | `vocabularies( category: VocabularyCategory first: Int after: ID orderBy: VocabularyOrdering orderMode: OrderingMode filters: FilterGroup search: String ): VocabularyConnection @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SETVOCABULARIES, INGESTION_SETINGESTIONS, INVESTI…` |
| vocabulary | `Vocabulary` | `vocabulary(id: String!): Vocabulary @auth(for: [KNOWLEDGE, SETTINGS_SETACCESSES, SETTINGS_SETVOCABULARIES])` |
| vocabularyCategories | `[VocabularyDefinition!]!` | `vocabularyCategories: [VocabularyDefinition!]! @auth` |


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| vocabularyAdd | `Vocabulary` | `vocabularyAdd(input: VocabularyAddInput!): Vocabulary @auth(for: [SETTINGS_SETVOCABULARIES])` |
| vocabularyDelete | `ID` | `vocabularyDelete(id: ID!): ID @auth(for: [SETTINGS_SETVOCABULARIES])` |
| vocabularyFieldPatch | `Vocabulary` | `vocabularyFieldPatch(id: ID!, input: [EditInput!]!): Vocabulary @auth(for: [SETTINGS_SETVOCABULARIES])` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (2)
<a id="enum-2"></a>

| Name | File |
| --- | --- |
| `VocabularyCategory` | `src/modules/vocabulary/vocabulary.graphql` |
| `VocabularyOrdering` | `src/modules/vocabulary/vocabulary.graphql` |

<details><summary>Show definition details</summary>


**enum VocabularyCategory**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

_Referenced by:_
- `input` `VocabularyAddInput` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`
- `type` `VocabularyDefinition` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`

Values:
- `account_type_ov`
- `attack_motivation_ov`
- `attack_resource_level_ov`
- `case_severity_ov`
- `case_priority_ov`
- `channel_types_ov`
- `collection_layers_ov`
- `event_type_ov`
- `grouping_context_ov`
- `implementation_language_ov`
- `incident_response_types_ov`
- `incident_type_ov`
- `incident_severity_ov`
- `indicator_type_ov`
- `infrastructure_type_ov`
- `integrity_level_ov`
- `malware_capabilities_ov`
- `malware_result_ov`
- `malware_type_ov`
- `platforms_ov`
- `opinion_ov`
- `organization_type_ov`
- `pattern_type_ov`
- `permissions_ov`
- `processor_architecture_ov`
- `reliability_ov`
- `report_types_ov`
- `request_for_information_types_ov`
- `request_for_takedown_types_ov`
- `security_platform_type_ov`
- `service_status_ov`
- `service_type_ov`
- `start_type_ov`
- `key_type_ov`
- `threat_actor_group_type_ov`
- `threat_actor_group_role_ov`
- `threat_actor_group_sophistication_ov`
- `threat_actor_individual_type_ov`
- `threat_actor_individual_role_ov`
- `threat_actor_individual_sophistication_ov`
- `tool_types_ov`
- `note_types_ov`
- `gender_ov`
- `marital_status_ov`
- `hair_color_ov`
- `eye_color_ov`
- `persona_type_ov`
- `coverage_ov`


**enum VocabularyOrdering**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

Values:
- `name`
- `category`
- `description`
- `order`
- `_score`


</details>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `VocabularyAddInput` | `src/modules/vocabulary/vocabulary.graphql` |

<details><summary>Show definition details</summary>


**input VocabularyAddInput**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `stix_id` | `StixId` |
| `x_opencti_stix_ids` | `[StixId]` |
| `name` | `String!` |
| `description` | `String` |
| `category` | `VocabularyCategory!` |
| `order` | `Int` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `aliases` | `[String!]` |
| `update` | `Boolean` |


</details>


#### type (5)
<a id="type-5"></a>

| Name | File |
| --- | --- |
| `Vocabulary` | `src/modules/vocabulary/vocabulary.graphql` |
| `VocabularyConnection` | `src/modules/vocabulary/vocabulary.graphql` |
| `VocabularyDefinition` | `src/modules/vocabulary/vocabulary.graphql` |
| `VocabularyEdge` | `src/modules/vocabulary/vocabulary.graphql` |
| `VocabularyFieldDefinition` | `src/modules/vocabulary/vocabulary.graphql` |

<details><summary>Show definition details</summary>


**type Vocabulary**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

_Referenced by:_
- `type` `VocabularyEdge` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `standard_id` | `String!` |
| `entity_type` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `representative` | `Representative!` |
| `creators` | `[Creator!]` |
| `x_opencti_stix_ids` | `[StixId]` |
| `is_inferred` | `Boolean!` |
| `spec_version` | `String!` |
| `created_at` | `DateTime!` |
| `updated_at` | `DateTime!` |
| `x_opencti_modified_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `draftVersion` | `DraftVersion` |
| `x_opencti_inferences` | `[Inference]` |
| `created` | `DateTime` |
| `modified` | `DateTime` |
| `category` | `VocabularyDefinition!` |
| `name` | `String!` |
| `description` | `String` |
| `usages` | `Int!` |
| `aliases` | `[String!]` |
| `builtIn` | `Boolean` |
| `is_hidden` | `Boolean` |
| `order` | `Int` |


**type VocabularyConnection**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[VocabularyEdge!]!` |


**type VocabularyDefinition**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

_Referenced by:_
- `type` `Vocabulary` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `key` | `VocabularyCategory!` |
| `description` | `String` |
| `entity_types` | `[String!]!` |
| `fields` | `[VocabularyFieldDefinition!]!` |


**type VocabularyEdge**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

_Referenced by:_
- `type` `VocabularyConnection` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Vocabulary!` |


**type VocabularyFieldDefinition**  
Defined in `src/modules/vocabulary/vocabulary.graphql`

_Referenced by:_
- `type` `VocabularyDefinition` (vocabulary) in `src/modules/vocabulary/vocabulary.graphql`

| Field | Type |
| --- | --- |
| `key` | `String!` |
| `required` | `Boolean!` |
| `multiple` | `Boolean!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **12**
- Definition count: **28**


#### Expanded: enum (5)
<a id="expanded-enum-5"></a>

<details><summary>Show names</summary>

- `DraftOperation`
- `EditOperation`
- `Version`
- `VocabularyCategory`
- `VocabularyOrdering`

</details>


#### Expanded: input (2)
<a id="expanded-input-2"></a>

<details><summary>Show names</summary>

- `EditInput`
- `VocabularyAddInput`

</details>


#### Expanded: scalar (3)
<a id="expanded-scalar-3"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`

</details>


#### Expanded: type (17)
<a id="expanded-type-17"></a>

<details><summary>Show names</summary>

- `Creator`
- `Display`
- `DisplayStep`
- `DraftVersion`
- `EditUserContext`
- `Inference`
- `InferenceAttribute`
- `MarkingDefinition`
- `Metric`
- `PageInfo`
- `Representative`
- `Rule`
- `Vocabulary`
- `VocabularyConnection`
- `VocabularyDefinition`
- `VocabularyEdge`
- `VocabularyFieldDefinition`

</details>


#### Expanded: union (1)
<a id="expanded-union-1"></a>

<details><summary>Show names</summary>

- `StixObjectOrStixRelationship`

</details>



## Module family: workspace
<a id="module-family-workspace"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 2 |
| Mutation fields | 9 |
| Subscription fields | 1 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/workspace/workspace.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (2)
<a id="query-2"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| workspace | `Workspace` | `workspace(id: String!): Workspace @auth(for: [EXPLORE, SETTINGS_SETACCESSES, INVESTIGATION])` |
| workspaces | `WorkspaceConnection` | `workspaces( first: Int after: ID orderBy: WorkspacesOrdering orderMode: OrderingMode filters: FilterGroup includeAuthorities: Boolean search: String ): WorkspaceConnection @auth(for: [EXPLORE, SETTINGS_SETACCESSES, INVESTIGATION])` |


#### Mutation (9)
<a id="mutation-9"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| workspaceAdd | `Workspace` | `workspaceAdd(input: WorkspaceAddInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceConfigurationImport | `String!` | `workspaceConfigurationImport(file: Upload!): String! @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceContextClean | `Workspace` | `workspaceContextClean(id: ID!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceContextPatch | `Workspace` | `workspaceContextPatch(id: ID!, input: EditContext!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceDelete | `ID` | `workspaceDelete(id: ID!): ID @auth(for: [EXPLORE_EXUPDATE_EXDELETE, INVESTIGATION_INUPDATE_INDELETE])` |
| workspaceDuplicate | `Workspace` | `workspaceDuplicate(input: WorkspaceDuplicateInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceEditAuthorizedMembers | `Workspace` | `workspaceEditAuthorizedMembers(id: ID!, input:[MemberAccessInput!]!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceFieldPatch | `Workspace` | `workspaceFieldPatch(id: ID!, input: [EditInput!]!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |
| workspaceWidgetConfigurationImport | `Workspace` | `workspaceWidgetConfigurationImport(id: ID!, input: ImportConfigurationInput!): Workspace @auth(for: [EXPLORE_EXUPDATE, INVESTIGATION_INUPDATE])` |


#### Subscription (1)
<a id="subscription-1"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| workspace | `Workspace` | `workspace(id: ID!): Workspace @auth(for: [EXPLORE, INVESTIGATION])` |


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### enum (1)
<a id="enum-1"></a>

| Name | File |
| --- | --- |
| `WorkspacesOrdering` | `src/modules/workspace/workspace.graphql` |

<details><summary>Show definition details</summary>


**enum WorkspacesOrdering**  
Defined in `src/modules/workspace/workspace.graphql`

Values:
- `name`
- `created_at`
- `updated_at`
- `creator`
- `_score`


</details>


#### input (4)
<a id="input-4"></a>

| Name | File |
| --- | --- |
| `ImportConfigurationInput` | `src/modules/workspace/workspace.graphql` |
| `ImportWidgetInput` | `src/modules/workspace/workspace.graphql` |
| `WorkspaceAddInput` | `src/modules/workspace/workspace.graphql` |
| `WorkspaceDuplicateInput` | `src/modules/workspace/workspace.graphql` |

<details><summary>Show definition details</summary>


**input ImportConfigurationInput**  
Defined in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `importType` | `String!` |
| `file` | `Upload!` |
| `dashboardManifest` | `String` |


**input ImportWidgetInput**  
Defined in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `file` | `Upload!` |
| `dashboardManifest` | `String` |


**input WorkspaceAddInput**  
Defined in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `type` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `tags` | `[String!]` |
| `authorized_members` | `[MemberAccessInput!]` |
| `investigated_entities_ids` | `[String]` |


**input WorkspaceDuplicateInput**  
Defined in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `type` | `String!` |
| `name` | `String!` |
| `description` | `String` |
| `manifest` | `String` |
| `tags` | `[String!]` |


</details>


#### type (3)
<a id="type-3"></a>

| Name | File |
| --- | --- |
| `Workspace` | `src/modules/workspace/workspace.graphql` |
| `WorkspaceConnection` | `src/modules/workspace/workspace.graphql` |
| `WorkspaceEdge` | `src/modules/workspace/workspace.graphql` |

<details><summary>Show definition details</summary>


**type Workspace**  
Defined in `src/modules/workspace/workspace.graphql`

_Referenced by:_
- `type` `Organization` (organization) in `src/modules/organization/organization.graphql`
- `type` `PublicDashboard` (publicDashboard) in `src/modules/publicDashboard/publicDashboard.graphql`
- `type` `WorkspaceEdge` (workspace) in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `id` | `ID!` |
| `entity_type` | `String!` |
| `standard_id` | `String!` |
| `parent_types` | `[String!]!` |
| `metrics` | `[Metric]` |
| `type` | `String` |
| `name` | `String!` |
| `description` | `String` |
| `owner` | `Creator` |
| `tags` | `[String!]` |
| `manifest` | `String` |
| `created_at` | `DateTime` |
| `updated_at` | `DateTime` |
| `refreshed_at` | `DateTime` |
| `editContext` | `[EditUserContext!]` |
| `investigated_entities_ids` | `[String]` |
| `objects(first: Int after: ID orderBy: StixObjectOrStixRelationshipsOrdering orderMode: OrderingMode filters: FilterGroup search: String types: [String] all: Boolean)` | `StixObjectOrStixRelationshipRefConnection` |
| `graph_data` | `String` |
| `authorizedMembers` | `[MemberAccess!]!` |
| `currentUserAccessRight` | `String` |
| `toStixReportBundle` | `String` |
| `toConfigurationExport` | `String!` |
| `toWidgetExport(widgetId: ID!)` | `String!` |
| `isShared` | `Boolean` |


**type WorkspaceConnection**  
Defined in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `pageInfo` | `PageInfo!` |
| `edges` | `[WorkspaceEdge!]!` |


**type WorkspaceEdge**  
Defined in `src/modules/workspace/workspace.graphql`

_Referenced by:_
- `type` `WorkspaceConnection` (workspace) in `src/modules/workspace/workspace.graphql`

| Field | Type |
| --- | --- |
| `cursor` | `String!` |
| `node` | `Workspace!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **15**
- Definition count: **35**


#### Expanded: enum (4)
<a id="expanded-enum-4"></a>

<details><summary>Show names</summary>

- `DraftOperation`
- `EditOperation`
- `Version`
- `WorkspacesOrdering`

</details>


#### Expanded: input (7)
<a id="expanded-input-7"></a>

<details><summary>Show names</summary>

- `EditContext`
- `EditInput`
- `ImportConfigurationInput`
- `ImportWidgetInput`
- `MemberAccessInput`
- `WorkspaceAddInput`
- `WorkspaceDuplicateInput`

</details>


#### Expanded: scalar (4)
<a id="expanded-scalar-4"></a>

<details><summary>Show names</summary>

- `Any`
- `DateTime`
- `StixId`
- `Upload`

</details>


#### Expanded: type (19)
<a id="expanded-type-19"></a>

<details><summary>Show names</summary>

- `Creator`
- `Display`
- `DisplayStep`
- `DraftVersion`
- `EditUserContext`
- `Inference`
- `InferenceAttribute`
- `MarkingDefinition`
- `MemberAccess`
- `MemberGroupRestriction`
- `Metric`
- `PageInfo`
- `Representative`
- `Rule`
- `StixObjectOrStixRelationshipRefConnection`
- `StixObjectOrStixRelationshipRefEdge`
- `Workspace`
- `WorkspaceConnection`
- `WorkspaceEdge`

</details>


#### Expanded: union (1)
<a id="expanded-union-1"></a>

<details><summary>Show names</summary>

- `StixObjectOrStixRelationship`

</details>



## Module family: xtm
<a id="module-family-xtm"></a>

| Metric | Value |
| --- | --- |
| SDL files | 1 |
| Query fields | 0 |
| Mutation fields | 3 |
| Subscription fields | 0 |


### Files
<a id="files"></a>

| SDL file (relative) |
| --- |
| `src/modules/xtm/hub/xtm-hub.graphql` |


### Root fields
<a id="root-fields"></a>


#### Query (0)
<a id="query-0"></a>

_None._


#### Mutation (3)
<a id="mutation-3"></a>

| Field | Returns | Signature |
| --- | --- | --- |
| autoRegisterOpenCTI | `Success!` | `autoRegisterOpenCTI(input: AutoRegisterInput!): Success! @auth(for: [SETTINGS_SETPARAMETERS])` |
| checkXTMHubConnectivity | `CheckXTMHubConnectivityResponse!` | `checkXTMHubConnectivity: CheckXTMHubConnectivityResponse! @auth(for: [SETTINGS_SETPARAMETERS])` |
| contactUsXtmHub | `Success!` | `contactUsXtmHub: Success! @auth` |


#### Subscription (0)
<a id="subscription-0"></a>

_None._


### SDL definitions (declared in this family)
<a id="sdl-definitions-declared-in-this-family"></a>


#### input (1)
<a id="input-1"></a>

| Name | File |
| --- | --- |
| `AutoRegisterInput` | `src/modules/xtm/hub/xtm-hub.graphql` |

<details><summary>Show definition details</summary>


**input AutoRegisterInput**  
Defined in `src/modules/xtm/hub/xtm-hub.graphql`

| Field | Type |
| --- | --- |
| `platform_token` | `String!` |


</details>


#### type (2)
<a id="type-2"></a>

| Name | File |
| --- | --- |
| `CheckXTMHubConnectivityResponse` | `src/modules/xtm/hub/xtm-hub.graphql` |
| `Success` | `src/modules/xtm/hub/xtm-hub.graphql` |

<details><summary>Show definition details</summary>


**type CheckXTMHubConnectivityResponse**  
Defined in `src/modules/xtm/hub/xtm-hub.graphql`

| Field | Type |
| --- | --- |
| `status` | `XTMHubRegistrationStatus` |


**type Success**  
Defined in `src/modules/xtm/hub/xtm-hub.graphql`

| Field | Type |
| --- | --- |
| `success` | `Boolean!` |


</details>


### Expanded reachability graph (from module roots)
<a id="expanded-reachability-graph-from-module-roots"></a>

- Depth: **100**
- Seed count: **3**
- Definition count: **4**


#### Expanded: enum (1)
<a id="expanded-enum-1"></a>

<details><summary>Show names</summary>

- `XTMHubRegistrationStatus`

</details>


#### Expanded: input (1)
<a id="expanded-input-1"></a>

<details><summary>Show names</summary>

- `AutoRegisterInput`

</details>


#### Expanded: type (2)
<a id="expanded-type-2"></a>

<details><summary>Show names</summary>

- `CheckXTMHubConnectivityResponse`
- `Success`

</details>
