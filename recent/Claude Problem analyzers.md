I found Gearset's own public list. A few notes on accuracy before the list: their Help Center's "Problem analyzers" collection currently documents **50 named analyzers** (51 articles total, one of which is the general intro). Their in-app CI template UI separately states "up to 45 (currently)" analyzers to toggle. Neither of those numbers is exactly "70+" — that figure likely either rounds up, has grown since the help docs were last updated, or (more likely) blends in Gearset's separate **Static Code Analysis** layer, which runs the PMD Apex ruleset (Best Practices, Code Style, Design, Performance, Security, Error-Prone categories) as a series of rulesets used to detect bugs and bad practices — that's a different, additive system from the metadata Problem Analyzers, with its own large rule count. So "70+" is probably Problem Analyzers + a chunk of the default PMD ruleset combined, not 70 metadata-dependency rules alone. I'd treat the marketing number with that caveat rather than as a literal single-list count.

Here's the real, documented list, grouped by what it actually checks:

**Dependency & reference integrity**
- Missing dependencies
- Unresolved missing dependencies
- Deleted dependencies
- Changed dependencies
- Remove references to objects which are not yet in the target
- New or changed reference user
- Deleted owner
- New items with permissions that are not included in the deployment

**Flows**
- Deleted active Flows
- Flow Definitions can't be deleted
- Flow Definitions being used to set active state of Flows
- Must delete all versions of a Flow
- Deploying Flow translations with Problem Analyzer

**Record types & picklists**
- RecordType picklist values must exist on target
- Remove picklists from RecordType
- Remove record type deletions
- Object has more than one person account record type default
- Object has more than one default record type visibility
- Object has no default record type visibility

**Profiles, permissions & connected apps**
- Connected apps must have a unique consumerKey
- Standard tab settings that do not exist on the target
- Standard applications cannot be added or deleted

**Layouts & UI metadata**
- Invalid action sort orders within layouts
- Remove horizontal alignment
- Items with names in different cases
- Metadata sections must be grouped together
- Action overrides must be present on target

**Data model**
- Custom field has changed type
- Master-detail relationships

**List views**
- Cannot remove all list views

**Queues, skills & routing**
- Cannot delete Skills
- Cannot deploy more than 16 queues with routing configurations
- Routing config must be set to blank when removing from queue

**Workflow rules**
- Removing Workflow Rule Actions
- WorkflowOutboundMessages have different values for endpointUrl in source and target

**Sites / Communities / Network**
- Different Network "picassoSite" tags
- Different Site.com "label" tags
- Swap the site administrator reference of a new CustomSite

**Managed packages & platform features**
- Feature not enabled in target
- Features that might not be enabled in the source or target
- Remove components added to standard objects by managed packages
- Exclude Isotope Subscription
- ActivateRss not defined

**CPQ / OmniStudio**
- Changed reusable OmniScript dependency
- Enabling or disabling quotes

**AI / Agentforce**
- Prompt Templates that reference deprecated LLMs

**Misc**
- Apex API version mismatch
- History tracking settings
- TASK.WHAT_NAME field
- Already active StageAssignment rules

Source: [Gearset Help Center — Problem analyzers collection](https://docs.gearset.com/en/collections/10441556-problem-analyzers).

This list is a good proxy for scoping the "Tier B" effort I outlined earlier: each one of these is a distinct, hand-coded rule that knows a specific Salesforce metadata quirk (e.g. queue routing config limits, record-type default visibility, OmniScript dependency chains). It's a useful checklist if you want to prioritize which 8–10 to build first for your own MVP — I'd start with **Missing dependencies, Deleted dependencies, RecordType picklist values must exist on target, Custom field has changed type, and New items with permissions not included in the deployment**, since those map to the most common real-world deployment failures.

---

Effort legend used below: **Low** = hours–2 days · **Medium** = 3 days–2 weeks · **High** = 3–6+ weeks · **Very High** = months, with ongoing maintenance.

These "likely strategy" columns are my engineering inference from how each rule behaves (not confirmed Gearset internals — they haven't published their implementation).

## Dependency & reference integrity

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Missing dependencies | Build a reference graph by parsing each metadata file type for embedded refs, cross-check existence in target via Tooling API/describe | **High** — generic dependency-graph engine across many metadata types | Reactive: run `sf project deploy validate --dry-run`, parse `INVALID_FIELD`/component errors, auto re-add from source repo |
| Unresolved missing dependencies | Same graph, flags refs missing even in source | **High** | Same reactive validate + maintain a manual exception list |
| Deleted dependencies | Scan all active target metadata for usage before allowing a delete | **Medium–High** — needs org-wide "where used" scan, not just the package | Tooling API `MetadataComponentDependency` query for "where used" before destructive deploy |
| Changed dependencies | Diff field/type changes, check if dependents (formulas, reports) would break | **High** | Reactive: run full test deploy, capture compile errors from dependent Apex/Flow |
| Remove references to objects not yet in target | Parse Profile/PermissionSet XML, strip refs to objects absent from target | **Medium** | Multi-pass ordered deploy (objects first, profiles second) instead of filtering |
| New or changed reference user | Detect embedded username fields (dashboard/report running user, Flow run-as) and validate existence in target | **Medium** — need a catalog of metadata types with embedded usernames | Manual checklist + grep script querying target via `sf data query` |
| Deleted owner | Detect orphaned owner refs after deleting an owning component | **Medium** | Same grep+query script pattern |
| New items with permissions not included in deployment | Index every Profile/PermissionSet in source for grants on new fields/objects/classes, auto-suggest inclusion | **Medium–High** — combinatorial but mechanical | CI lint that fails if a new field has zero permission grants anywhere in the package |

## Flows

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Deleted active Flows | Encode the Metadata API quirk: must deactivate before delete, auto-sequence | **Low–Medium** — well-documented, mechanical | Hardcode the deactivate-then-delete two-phase logic |
| Flow Definitions can't be deleted | FlowDefinition vs Flow component-type quirk, auto-correct package type | **Low** | Hardcode rule |
| Flow Definitions being used to set active state | Normalize activeVersionNumber handling across API versions | **Low–Medium** | Hardcode per API version |
| Must delete all versions of a Flow | Query existing versions, ensure destructiveChanges covers all | **Low** | Same — cheap to hardcode |
| Deploying Flow translations with Problem Analyzer | Keep Flow + Translations metadata bundled/ordered | **Medium** | Manual sequencing checklist |

## Record types & picklists

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| RecordType picklist values must exist on target | Parse RecordType XML's picklist blocks, cross-check field describe in target | **Medium** | Reactive: catch Salesforce's own error, optionally auto-create missing value via Tooling API |
| Remove picklists from RecordType | Same cross-check, opposite direction | **Medium** | Same reactive parse |
| Remove record type deletions | Check layout/data dependency before allowing delete | **Medium–High** — needs a SOQL count check too | Reactive: let deploy fail with standard error, surface message |
| Object has >1 person account record type default | Cardinality rule encoded from platform docs | **Low** | Cheap XPath/regex assertion |
| Object has >1 default record type visibility | Same family | **Low** | Same |
| Object has no default record type visibility | Same family | **Low** | Same |

## Profiles, permissions & connected apps

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Connected apps must have unique consumerKey | Structural uniqueness check across package + target | **Low** | Simple comparison script |
| Standard tab settings not on target | Cross-reference standard tab list against target's enabled features/licenses | **Medium** — list needs upkeep across releases | Reactive: catch `INVALID_TYPE`, strip element |
| Standard applications cannot be added/deleted | Filter standard app names from add/delete ops | **Low** | Hardcoded filter list |

## Layouts & UI metadata

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Invalid action sort orders in layouts | Parse Layout XML action-list sortOrder values, auto-renumber | **Medium** — schema-specific | Reactive: catch deploy error, fix manually |
| Remove horizontal alignment | Strip a deprecated, no-longer-accepted attribute | **Low** | Regex find/replace pre-deploy — trivial |
| Items with names in different cases | Case-insensitive duplicate-name detection | **Low–Medium** | Simple linter script |
| Metadata sections must be grouped together | Canonical reordering of sibling XML element groups | **Medium** — schema-specific per type | Reuse the canonicalization script from the XML-diff discussion, applied as a fixer not just a differ |
| Action overrides must be present on target | Cross-check actionOverrides reference an existing Flexipage/VF/LWC in target | **Medium** | Reactive validate + strip override if missing |

## Data model

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Custom field has changed type | Compare source vs target field type, warn of conversion/data-loss risk | **Medium** — may need a record-count query too | Reactive: let Salesforce reject unsafe conversions, surface its error |
| Master-detail relationships | Encode platform rules on relationship creation order and existing data | **Medium–High** — data-dependent edge cases | Reactive: accept deploy failure messages |

## List views, skills, queues

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Cannot remove all list views | Count existing minus deleted, block if it hits zero | **Low–Medium** | Reactive error catch |
| Cannot delete Skills | Convert delete intent to deactivation (platform constraint) | **Low** | Hardcode rule |
| Cannot deploy >16 queues with routing config | Hardcoded platform limit, simple count+threshold | **Low** — cheapest item on the list | Trivial script |
| Routing config must be blank when removing from queue | Explicit-blank vs omission quirk in Metadata API | **Low** | Field-blanking script |

## Workflow rules & environment-specific values

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Removing Workflow Rule Actions | Ordering constraint between rule and its action refs | **Low–Medium** | Reactive, manual reorder on error |
| WorkflowOutboundMessages endpointUrl differs source/target | "Don't silently overwrite environment-specific value" pattern | **Low** | Generic "preserve target value" override list — reusable for Named Credentials, Remote Site Settings too |
| Different Network "picassoSite" tags | Same preserve-target-value pattern | **Low** | Same generic mechanism |
| Different Site.com "label" tags | Same pattern | **Low** | Same |
| Swap site administrator reference of new CustomSite | Substitute source siteAdmin username for a valid target user | **Low–Medium** | Config-mapped "default site admin per environment" substitution script |

## Managed packages & platform features

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Feature not enabled in target | Maintain feature→metadata mapping, check via org describe | **Medium–High** — ongoing list upkeep every release | Reactive: catch specific error code, friendlier message |
| Features that might not be enabled in source/target | Same, broader/fuzzier | **Medium–High** | Reactive |
| Remove components added to standard objects by managed packages | Filter namespace-prefixed components on retrieve | **Medium** | Regex namespace-prefix filter during package generation (well-known SFDX gotcha, several open-source delta tools already do this) |
| Exclude Isotope Subscription | Narrow vendor-specific hardcoded exclusion | **Low** | Same — manual exclusion list either way |
| ActivateRss not defined | Default-value normalization for InstalledPackage field | **Low** | Hardcoded default-value injection |

## OmniStudio / CPQ / AI

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Changed reusable OmniScript dependency | Bespoke parsing of OmniStudio's own metadata layer (non-standard) | **High** | Use OmniStudio's native versioning/deployment tools instead of building custom tracking |
| Enabling or disabling quotes | Detect QuoteTemplate dependency on org-wide Quote feature toggle | **Low–Medium** | Simple org-setting describe check pre-deploy |
| Prompt Templates referencing deprecated LLMs | Maintain current valid/deprecated model-ID list, cross-check | **Medium** — recurring list maintenance, not one-time | Reactive: let activation fail, surface Salesforce's deprecation error |

## Misc / platform quirks

| Rule | Likely Gearset strategy | DIY effort & approach | Alternative / cheaper strategy |
|---|---|---|---|
| Apex API version mismatch | Read `<apiVersion>` tag per class/trigger, flag mismatches | **Low** — easiest item on the entire list | A 10-line CI lint step; barely needs an "alternative," it's already near-free |
| History tracking settings | Parent object flag must be enabled before field-level tracking | **Low–Medium** | Simple parent-child flag dependency check |
| TASK.WHAT_NAME field | Exclude a known unsupported system field from certain deploy contexts | **Low** | Hardcoded exclusion list |
| Already active StageAssignment rules | Can't redeploy/reactivate an already-active assignment, needs sequencing | **Low–Medium** | Reactive, catch + retry |

### Reading this table
A clear pattern: roughly two-thirds of these rules (queue limits, Flow deletion quirks, Apex API version, standard-app filters, environment-value preservation) are **Low effort, mechanical, well-documented platform quirks** — genuinely cheap to replicate as a checklist script. The genuinely hard ones (Missing/Changed/Deleted dependencies, Feature-not-enabled, Master-detail relationships, OmniScript dependencies) are the ones that need a real org-aware dependency graph and ongoing release-to-release maintenance — that's where the bulk of Gearset's actual engineering investment is, and where the "reactive validate + parse error" fallback is almost always the pragmatic DIY answer instead of trying to out-build them.
