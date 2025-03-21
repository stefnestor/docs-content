---
navigation_title: "Elastic Cloud Serverless"
mapped_pages:
  - https://www.elastic.co/guide/en/serverless/current/serverless-changelog.html
---

# Elastic Cloud Serverless changelog [elastic-cloud-serverless-changelog]
Review the changes, fixes, and more to Elastic Cloud Serverless.

For serverless API changes, refer to [APIs Changelog](https://www.elastic.co/docs/api/changes).

For serverless changes in Cloud Console, check out [Elastic Cloud Hosted release notes](asciidocalypse://docs/cloud/docs/release-notes/cloud-hosted/index.md).

% Release notes include only features, enhancements, and fixes. Add breaking changes, deprecations, and known issues to the applicable release notes sections.

% ## version.next [elastic-cloud-serverless-changelog-releasedate]

% ### Features and enhancements [elastic-cloud-serverless-releasedate-features-enhancements]

% ### Fixes [elastic-cloud-serverless-releasedate-fixes]

## January 27, 2025 [serverless-changelog-01272025]

### Features and enhancements [elastic-cloud-serverless-01272025-features-enhancements]
* Breaks out timeline and note privileges in Elastic Security Serverless ({{kibana-pull}}201780[#201780]).
* Adds service enrichment to the detection engine in Elastic Security Serverless ({{kibana-pull}}206582[#206582]).
* Updates the Entity Store Dashboard to prompt for the Service Entity Type in Elastic Security Serverless ({{kibana-pull}}207336[#207336]).
* Adds `enrichPolicyExecutionInterval` to entity enablement and initialization APIs in Elastic Security Serverless ({{kibana-pull}}207374[#207374]).
* Introduces a lookback period configuration for the Entity Store in Elastic Security Serverless ({{kibana-pull}}206421[#206421]).
* Allows pre-configured connectors to opt into exposing their configurations by setting `exposeConfig` in Alerting ({{kibana-pull}}207654[#207654]).
* Adds selector syntax support to log source profiles in Elastic Observability Serverless ({{kibana-pull}}206937[#206937]).
* Displays stack traces in the logs overview tab in Elastic Observability Serverless ({{kibana-pull}}204521[#204521]).
* Enables the use of the rule form to create rules in Elastic Observability Serverless ({{kibana-pull}}206774[#206774]).
* Checks only read privileges of existing indices during rule execution in Elastic Security Serverless ({{kibana-pull}}177658[#177658]).
* Updates KNN search and query template autocompletion in Elasticsearch Serverless ({{kibana-pull}}207187[#207187]).
* Updates JSON schemas for code editors in Machine Learning ({{kibana-pull}}207706[#207706]).
* Reindexes the `.kibana_security_session_1` index to the 8.x format in Security ({{kibana-pull}}204097[#204097]).

### Fixes [elastic-cloud-serverless-01272025-fixes]
* Fixes editing alerts filters for multi-consumer rule types in Alerting ({{kibana-pull}}206848[#206848]).
* Resolves an issue where Chrome was no longer hidden for reports in Dashboards and Visualizations ({{kibana-pull}}206988[#206988]).
* Updates library transforms and duplicate functionality in Dashboards and Visualizations ({{kibana-pull}}206140[#206140]).
* Fixes an issue where drag previews are now absolutely positioned in Dashboards and Visualizations ({{kibana-pull}}208247[#208247]).
* Fixes an issue where an accessible label now appears on the range slider in Dashboards and Visualizations ({{kibana-pull}}205308[#205308]).
* Fixes a dropdown label sync issue when sorting by "Type" ({{kibana-pull}}206424[#206424]).
* Fixes an access bug related to user instructions in Elastic Observability Serverless ({{kibana-pull}}207069[#207069]).
* Fixes the Open Explore in Discover link to open in a new tab in Elastic Observability Serverless ({{kibana-pull}}207346[#207346]).
* Returns an empty object for tool arguments when none are provided in Elastic Observability Serverless ({{kibana-pull}}207943[#207943]).
* Ensures similar cases count is not fetched without the proper license in Elastic Security Serverless ({{kibana-pull}}207220[#207220]).
* Fixes table leading actions to use standardized colors in Elastic Security Serverless ({{kibana-pull}}207743[#207743]).
* Adds missing fields to the AWS S3 manifest in Elastic Security Serverless ({{kibana-pull}}208080[#208080]).
* Prevents redundant requests when loading Discover sessions and toggling chart visibility in ES|QL ({{kibana-pull}}206699[#206699]).
* Fixes a UI error when agents move to an orphaned state in Fleet ({{kibana-pull}}207746[#207746]).
* Restricts non-local Elasticsearch output types for agentless integrations and policies in Fleet ({{kibana-pull}}207296[#207296]).
* Fixes table responsiveness in the Notifications feature of Machine Learning ({{kibana-pull}}206956[#206956]).

## January 13, 2025 [serverless-changelog-01132025]

### Features and enhancements [elastic-cloud-serverless-01132025-features-enhancements]
* Adds last alert status change to Elastic Security Serverless flyout ({{kibana-pull}}205224[#205224]).
* Case templates are now GA ({{kibana-pull}}205940[#205940]).
* Adds format to JSON messages in Elastic Observability Serverless Logs profile ({{kibana-pull}}205666[#205666]).
* Adds inference connector in Elastic Security Serverless AI features ({{kibana-pull}}204505[#204505]).
* Adds inference connector for Auto Import in Elastic Security Serverless ({{kibana-pull}}206111[#206111]).
* Adds Feature Flag Support for Cloud Security Posture Plugin in Elastic Security Serverless ({{kibana-pull}}205438[#205438]).
* Adds the ability to sync Machine Learning saved objects to all spaces ({{kibana-pull}}202175[#202175]).
* Improves messages for recovered alerts in Machine Learning Transforms ({{kibana-pull}}205721[#205721]).

### Fixes [elastic-cloud-serverless-01132025-fixes]
* Fixes an issue where "KEEP" columns are not applied after an Elasticsearch error in Discover ({{kibana-pull}}205833[#205833]).
* Resolves padding issues in the document comparison table in Discover ({{kibana-pull}}205984[#205984]).
* Fixes a bug affecting bulk imports for the knowledge base in Elastic Observability Serverless ({{kibana-pull}}205075[#205075]).
* Enhances the Find API by adding cursor-based pagination (search_after) as an alternative to offset-based pagination ({{kibana-pull}}203712[#203712]).
* Updates Elastic Observability Serverless to use architecture-specific Elser models ({{kibana-pull}}205851[#205851]).
* Fixes dynamic batching in the timeline for Elastic Security Serverless ({{kibana-pull}}204034[#204034]).
* Resolves a race condition bug in Elastic Security Serverless related to OpenAI errors ({{kibana-pull}}205665[#205665]).
* Improves the integration display by ensuring all policies are listed in Elastic Security Serverless ({{kibana-pull}}205103[#205103]).
* Renames color variables in the user interface for better clarity and consistency  ({{kibana-pull}}204908[#204908]).
* Allows editor suggestions to remain visible when the inline documentation flyout is open in ES|QL ({{kibana-pull}}206064[#206064]).
* Ensures the same time range is applied to documents and the histogram in ES|QL ({{kibana-pull}}204694[#204694]).
* Fixes validation for the "required" field in multi-text input fields in Fleet ({{kibana-pull}}205768[#205768]).
* Fixes timeout issues for bulk actions in Fleet ({{kibana-pull}}205735[#205735]).
* Handles invalid RRule parameters to prevent infinite loops in alerts ({{kibana-pull}}205650[#205650]).
* Fixes privileges display for features and sub-features requiring "All Spaces" permissions in Fleet ({{kibana-pull}}204402[#204402]).
* Prevents password managers from modifying disabled input fields ({{kibana-pull}}204269[#204269]).
* Updates the listing control in the user interface ({{kibana-pull}}205914[#205914]).
* Improves consistency in the help dropdown design ({{kibana-pull}}206280[#206280]).

## January 6, 2025 [serverless-changelog-01062025]

### Features and enhancements [elastic-cloud-serverless-01062025-features-enhancements]
* Introduces case observables in Elastic Security Serverless ({{kibana-pull}}190237[#190237]).
* Adds a JSON field called "additional fields" to ServiceNow cases when sent using connector, containing the internal names of the ServiceNow table columns ({{kibana-pull}}201948[#201948]).
* Adds the ability to configure the appearance color mode to sync dark mode with the system value ({{kibana-pull}}203406[#203406]).
* Makes the "Copy" action visible on cell hover in Discover ({{kibana-pull}}204744[#204744]).
* Updates the `EnablementModalCallout` name to `AdditionalChargesMessage` in Elastic Security Serverless ({{kibana-pull}}203061[#203061]).
* Adds more control over which Elastic Security Serverless alerts in Attack Discovery are included as context to the large language model ({{kibana-pull}}205070[#205070]).
* Adds a consistent layout and other UI enhancements for {{ml}} pages ({{kibana-pull}}203813[#203813]).

### Fixes [elastic-cloud-serverless-01062025-fixes]
* Fixes an issue that caused dashboards to lag when dragging the time slider ({{kibana-pull}}201885[#201885]).
* Updates the CloudFormation template to the latest version and adjusts the documentation to reflect the use of a single Firehose stream created by the new template ({{kibana-pull}}204185[#204185]).
* Fixes Integration and Datastream name validation in Elastic Security Serverless ({{kibana-pull}}204943[#204943]).
* Fixes an issue in the Automatic Import process where there is now inclusion of the `@timestamp` field in ECS field mappings whenever possible ({{kibana-pull}}204931[#204931]).
* Allows Automatic Import to safely parse Painless field names that are not valid Painless identifiers in `if` contexts ({{kibana-pull}}205220[#205220]).
* Aligns the Box Native Connector configuration fields with the source of truth in the connectors codebase, correcting mismatches and removing unused configurations ({{kibana-pull}}203241[#203241]).
* Fixes the "Show all agent tags" option in Fleet when the agent list is filtered ({{kibana-pull}}205163[#205163]).
* Updates the Results Explorer flyout footer buttons alignment in Data Frame Analytics ({{kibana-pull}}204735[#204735]).
* Adds a missing space between lines in the Data Frame Analytics delete job modal ({{kibana-pull}}204732[#204732]).
* Fixes an issue where the Refresh button in the Anomaly Detection Datafeed counts table was unresponsive ({{kibana-pull}}204625[#204625]).
* Fixes the inference timeout check in File Upload ({{kibana-pull}}204722[#204722]).
* Fixes the side bar navigation for the Data Visualizer ({{kibana-pull}}205170[#205170]).

## December 16, 2024 [serverless-changelog-12162024]

### Features and enhancements [elastic-cloud-serverless-12162024-features-enhancements]
* Optimizes the Kibana Trained Models API ({{kibana-pull}}200977[#200977]).
* Adds a **Create Case** action to the **Log rate analysis** page ({{kibana-pull}}201549[#201549]).
* Improves AI Assistant’s response quality by giving it access to Elastic’s product documentation ({{kibana-pull}}199694[#199694]).
* Adds support for suppressing EQL sequence alerts ({{kibana-pull}}189725[#189725]).
* Adds an **Advanced settings** section to the SLO form ({{kibana-pull}}200822[#200822]).
* Adds a new sub-feature privilege under **Synthetics and Uptime** `Can manage private locations` ({{kibana-pull}}201100[#201100]).

### Fixes [elastic-cloud-serverless-12162024-fixes]
* Fixes point visibility regression ({{kibana-pull}}202358[#202358]).
* Improves help text of creator and view count features on dashboard listing page ({{kibana-pull}}202488[#202488]).
* Highlights matching field values when performing a KQL search on a keyword field ({{kibana-pull}}201952[#201952]).
* Supports "Inspect" in saved search embeddables ({{kibana-pull}}202947[#202947]).
* Fixes your ability to clear the user-specific system prompt ({{kibana-pull}}202279[#202279]).
* Fixes error when opening rule flyout ({{kibana-pull}}202386[#202386]).
* Fixes to Ops Genie as a default connector ({{kibana-pull}}201923[#201923]).
* Fixes actions on charts ({{kibana-pull}}202443[#202443]).
* Adds flyout to table view in Infrastructure Inventory ({{kibana-pull}}202646[#202646]).
* Fixes service names with spaces not being URL encoded properly for `context.viewInAppUrl` ({{kibana-pull}}202890[#202890]).
* Allows access query logic to handle user ID and name conditions ({{kibana-pull}}202833[#202833]).
* Fixes APM rule error message for invalid KQL filter ({{kibana-pull}}203096[#203096]).
* Rejects CEF logs from Automatic Import and redirects you to the CEF integration instead ({{kibana-pull}}201792[#201792]).
* Updates the install rules title and message ({{kibana-pull}}202226[#202226]).
* Fixes error on second entity engine init API call ({{kibana-pull}}202903[#202903]).
* Restricts unsupported log formats ({{kibana-pull}}202994[#202994]).
* Removes errors related to Enterprise Search nodes ({{kibana-pull}}202437[#202437]).
* Improves web crawler name consistency ({{kibana-pull}}202738[#202738]).
* Fixes editor cursor jumpiness ({{kibana-pull}}202389[#202389]).
* Fixes rollover datastreams on subobjects mapper exception ({{kibana-pull}}202689[#202689]).
* Fixes spaces sync to retrieve 10,000 trained models ({{kibana-pull}}202712[#202712]).
* Fixes log rate analysis embeddable error on the Alerts page ({{kibana-pull}}203093[#203093]).
* Fixes Slack API connectors not displayed under Slack connector type when adding new connector to rule ({{kibana-pull}}202315[#202315]).

## December 9, 2024 [serverless-changelog-12092024]

### Features and enhancements [elastic-cloud-serverless-12092024-features-enhancements]
* Elastic Observability Serverless adds a new sub-feature for managing private locations ({{kibana-pull}}201100[#201100]).
* Elastic Observability Serverless adds the ability to configure SLO advanced settings from the UI ({{kibana-pull}}200822[#200822]).
* Elastic Security Serverless adds support for suppressing EQL sequence alerts ({{kibana-pull}}189725[#189725]).
* Elastic Security Serverless adds a `/trained_models_list` endpoint to retrieve complete data for the Trained Model UI ({{kibana-pull}}200977[#200977]).
* Machine Learning adds an action to include log rate analysis in a case ({{kibana-pull}}199694[#199694]).
* Machine Learning enhances the Kibana API to optimize trained models ({{kibana-pull}}201549[#201549]).

### Fixes [elastic-cloud-serverless-12092020-fixes]
* Fixes Slack API connectors not being displayed under the Slack connector type when adding a new connector to a rule in Alerting ({{kibana-pull}}202315[#202315]).
* Fixes point visibility regression in dashboard visualizations ({{kibana-pull}}202358[#202358]).
* Improves help text for creator and view count features on the Dashboard listing page ({{kibana-pull}}202488[#202488]).
* Highlights matching field values when performing a KQL search on a keyword field in Discover ({{kibana-pull}}201952[#201952]).
* Adds support for the **Inspect** option in saved search embeddables in Discover ({{kibana-pull}}202947[#202947]).
* Enables the ability to clear user-specific system prompts in Elastic Observability Serverless ({{kibana-pull}}202279[#202279]).
* Fixes an error when opening the rule flyout in Elastic Observability Serverless ({{kibana-pull}}202386[#202386]).
* Improves handling of Opsgenie as the default connector in Elastic Observability Serverless ({{kibana-pull}}201923[#201923]).
* Fixes issues with actions on charts in Elastic Observability Serverless ({{kibana-pull}}202443[#202443]).
* Adds a flyout to the table view in Infrastructure Inventory in Elastic Observability Serverless ({{kibana-pull}}202646[#202646]).
* Fixes service names with spaces not being URL-encoded properly for `{{context.viewInAppUrl}}` in Elastic Observability Serverless ({{kibana-pull}}202890[#202890]).
* Enhances access query logic to handle user ID and name conditions in Elastic Observability Serverless ({{kibana-pull}}202833[#202833]).
* Fixes an APM rule error message when a KQL filter is invalid in Elastic Observability Serverless ({{kibana-pull}}203096[#203096]).
* Restricts and rejects CEF logs in automatic import and redirects them to the CEF integration in Elastic Security Serverless ({{kibana-pull}}201792[#201792]).
* Updates the copy of the install rules title and message in Elastic Security Serverless ({{kibana-pull}}202226[#202226]).
* Clears errors on the second entity engine initialization API call in Elastic Security Serverless ({{kibana-pull}}202903[#202903]).
* Restricts unsupported log formats in Elastic Security Serverless ({{kibana-pull}}202994[#202994]).
* Removes errors related to Enterprise Search nodes in Elasticsearch Serverless ({{kibana-pull}}202437[#202437]).
* Ensures consistency in web crawler naming in Elasticsearch Serverless ({{kibana-pull}}202738[#202738]).
* Fixes editor cursor jumpiness in ES|QL ({{kibana-pull}}202389[#202389]).
* Implements rollover of data streams on subobject mapper exceptions in Fleet ({{kibana-pull}}202689[#202689]).
* Fixes trained models to retrieve up to 10,000 models when spaces are synced in Machine Learning ({{kibana-pull}}202712[#202712]).
* Fixes a Log Rate Analysis embeddable error on the Alerts page in AiOps ({{kibana-pull}}203093[#203093]).

## December 3, 2024 [serverless-changelog-12032024]

### Features and enhancements [elastic-cloud-serverless-12032024-features-enhancements]
* Adds tabs for Import Entities and Engine Status to the Entity Store ({{kibana-pull}}201235[#201235]).
* Adds status tracking for agentless integrations to {{fleet}} ({{kibana-pull}}199567[#199567]).
* Adds a new {{ml}} module that can detect anomalous activity in host-based logs ({{kibana-pull}}195582[#195582]).
* Allows custom Mapbox Vector Tile sources to style map layers and provide custom legends ({{kibana-pull}}200656[#200656]).
* Excludes stale SLOs from counts of healthy and violated SLOs ({{kibana-pull}}201027[#201027]).
* Adds a **Continue without adding integrations** button to the {{elastic-sec}} Dashboards page that takes you to the Entity Analytics dashboard ({{kibana-pull}}201363[#201363]).
* Displays visualization descriptions under their titles ({{kibana-pull}}198816[#198816]).

### Fixes [elastic-cloud-serverless-12032024-fixes]
* Hides the **Clear** button when no filters are selected ({{kibana-pull}}200177[#200177]).
* Fixes a mismatch between how wildcards were handled in previews versus actual rule executions ({{kibana-pull}}201553[#201553]).
* Fixes incorrect Y-axis and hover values in the Service Inventory’s Log rate chart ({{kibana-pull}}201361[#201361]).
* Disables the **Add note** button in the alert details flyout for users who lack privileges ({{kibana-pull}}201707[#201707]).
* Fixes the descriptions of threshold rules that use cardinality ({{kibana-pull}}201162[#201162]).
* Disables the **Install All** button on the **Add Elastic Rules** page when rules are installing ({{kibana-pull}}201731[#201731]).
* Reintroduces a data usage warning on the Entity Analytics Enablement modal ({{kibana-pull}}201920[#201920]).
* Improves accessibility for the **Create a connector** page ({{kibana-pull}}201590[#201590]).
* Fixes a bug that could cause {{agents}} to get stuck updating during scheduled upgrades ({{kibana-pull}}202126[#202126]).
* Fixes a bug related to starting {{ml}} deployments with autoscaling and no active nodes ({{kibana-pull}}201256[#201256]).
* Initializes saved objects when the **Trained Model** page loads ({{kibana-pull}}201426[#201426]).
* Fixes the display of deployment stats for unallocated deployments of {{ml}} models ({{kibana-pull}}202005[#202005]).
* Enables the solution type search for instant deployments ({{kibana-pull}}201688[#201688]).
* Improves the consistency of alert counts across different views ({{kibana-pull}}202188[#202188]).
