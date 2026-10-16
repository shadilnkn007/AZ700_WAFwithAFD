# AZ-700 Student Lab: Azure Front Door WAF with Two App Services

> **Audience:** Students
> **Lab type:** Azure portal only\
> **Estimated duration:** 2--3 hours\
> **Required tier:** Azure Front Door **Premium** for managed WAF rules\
> **Required outcome:** Route requests through Azure Front Door,
> associate a WAF policy, test Detection and Prevention modes, inspect
> logs, and verify custom and managed rules.

------------------------------------------------------------------------

## 1. Scenario

You are deploying a public web application that must be available
through a global entry point and protected by Azure Web Application
Firewall (WAF). Two Azure App Services act as origins. Azure Front Door
routes requests to the origin group, while a WAF policy inspects
incoming HTTP requests.

In this lab, you will:

-   Deploy two simple App Services in separate Azure regions.
-   Create Azure Front Door Premium with an endpoint, origin group,
    origins, and route.
-   Create a WAF policy and associate it with the Front Door endpoint
    domain.
-   Add a custom query-string block rule and an IP block rule.
-   Enable the managed Default Rule Set.
-   Compare Detection and Prevention modes.
-   Send Front Door WAF and access logs to Log Analytics and inspect
    events.
-   Optionally configure rate limiting as the final exercise.

## 2. Architecture

``` text
Student browser
      |
      | HTTPS
      v
Azure Front Door Premium endpoint
      |
      +-- WAF security policy association
      |      |
      |      +-- Custom rule: block test query string
      |      +-- Custom rule: block a selected client IP
      |      +-- Managed Default Rule Set (DRS)
      |      +-- Optional rate-limit rule
      |
      v
Origin group: og-az700-waf
      |
      +-- App Service A (Central India)
      |
      +-- App Service B (South India)

Azure Front Door diagnostics
      |
      v
Log Analytics workspace
```

The WAF protects requests that pass through the Front Door domain. A
direct request to an App Service hostname does **not** pass through
Front Door's WAF. For this introductory lab, origins remain publicly
reachable to keep setup simple; in production, restrict origin access so
clients cannot bypass Front Door.

## 3. AZ-700 objectives

  -----------------------------------------------------------------------
  Objective                           How the lab covers it
  ----------------------------------- -----------------------------------
  Design a WAF deployment             Choose Front Door as the global
                                      entry point and design its origin
                                      group

  Configure Detection or Prevention   Test the same custom rule in both
  mode                                modes

  Configure WAF rule sets for Azure   Enable the managed Default Rule Set
  Front Door                          and configure custom rules

  Implement a WAF policy              Create and configure policy
                                      settings and rules

  Associate a WAF policy              Associate the policy with the Front
                                      Door endpoint domain using a
                                      security policy
  -----------------------------------------------------------------------

## 4. Prerequisites and cost

-   An Azure subscription that can create App Services, App Service
    plans, Azure Front Door Premium, a WAF policy, and a Log Analytics
    workspace.
-   Permission to create and configure these resources.
-   A modern browser.
-   A way to determine your current public IPv4 address for the IP rule.
    If you use a VPN, corporate proxy, or shared NAT, the address seen
    by Front Door may differ from your device's local IP.
-   No application code, command-line tools, or local software is
    required.

**Cost warning:** Front Door Premium, App Service plans, and Log
Analytics ingestion can incur charges. Check current pricing and quotas
before starting. Complete the lab in a scheduled session and delete the
resource group afterward.

## 5. Resource naming plan

Use one resource group for all resources. Replace `<unique>` with a
short unique suffix where shown. Names must satisfy each Azure service's
naming rules.

  Resource                  Example name / setting
  ------------------------- -----------------------------
  Resource group            `rg-az700-waf-lab`
  App Service A             `app-az700-waf-a-<unique>`
  App Service B             `app-az700-waf-b-<unique>`
  Front Door profile        `afd-az700-waf-<unique>`
  Front Door endpoint       `afd-az700-waf-<unique>-ep`
  Origin group              `og-az700-waf`
  WAF policy                `waf-az700-lab`
  Security policy           `sec-az700-waf`
  Log Analytics workspace   `law-az700-waf-<unique>`

The Front Door endpoint name must be globally unique. Azure may reject
names already in use.

------------------------------------------------------------------------

# Part A --- Deploy the two App Services

## 6. Create the resource group

1.  Sign in to the [Azure portal](https://portal.azure.com/).
2.  Search for **Resource groups** and select **Create**.
3.  Select your subscription.
4.  Resource group name: `rg-az700-waf-lab`.
5.  Choose a region for the resource group's metadata, such as **Central
    India**.
6.  Select **Review + create**, then **Create**.

## 7. Create App Service A

1.  Search for **App Services** and select **Create**.
2.  Select your subscription and resource group `rg-az700-waf-lab`.
3.  Enter a globally unique name such as `app-az700-waf-a-<unique>`.
4.  Publish: **Code**.
5.  Runtime stack: choose a currently supported runtime offered by the
    portal. The default starter page is sufficient; no custom code is
    required.
6.  Operating system: choose Windows or Linux, consistent with the
    runtime you selected.
7.  Region: **Central India**.
8.  Create a new App Service plan using the smallest suitable plan
    available to your subscription. Review its pricing before creating
    it.
9.  Select **Review + create**, validate, and select **Create**.
10. After deployment, open the App Service and select **Default domain**
    or **Browse**.
11. Confirm that the application responds in a browser. Copy the
    hostname, for example `app-az700-waf-a-<unique>.azurewebsites.net`.

## 8. Create App Service B

Repeat the preceding process with these changes:

-   Name: `app-az700-waf-b-<unique>`.
-   Region: **South India**.
-   Reuse an appropriate App Service plan if the portal and region allow
    it; otherwise create a suitable plan in South India.
-   Open the default domain and confirm that the application responds.
-   Copy the hostname.

**Checkpoint:** Both App Services must work directly before you
configure Front Door. If one does not work, resolve that issue before
proceeding.

------------------------------------------------------------------------

# Part B --- Create Azure Front Door

## 9. Create a Front Door Premium profile and endpoint

1.  In the Azure portal, search for **Front Door and CDN profiles** and
    select **Create**.
2.  Select the option that lets you configure the profile and its
    components (often labelled **Custom create**).
3.  Select subscription and resource group `rg-az700-waf-lab`.
4.  Profile name: `afd-az700-waf-<unique>`.
5.  Tier: **Premium**.
6.  Add an endpoint with a globally unique name, for example
    `afd-az700-waf-<unique>-ep`.
7.  Do not enable caching for this lab; dynamic or default App Service
    pages do not require it.
8.  Continue to the origin group and route configuration described
    below. If your portal flow creates the profile first, finish the
    profile deployment and then open the profile to add the remaining
    components.

## 10. Create the origin group

1.  Open the Front Door profile.
2.  Under **Settings**, select **Origin groups** (the navigation label
    may vary slightly).
3.  Select **+ Add**.
4.  Name: `og-az700-waf`.
5.  Add the first origin:
    -   Origin type: **App Services** / **App Service**, if offered.
    -   Origin name: `origin-app-a`.
    -   Host name: select or enter App Service A's default hostname.
    -   Origin host header: use the App Service hostname if the portal
        asks for it.
    -   Origin enabled: enabled.
    -   Priority: `1`.
    -   Weight: `1000`.
6.  Add the second origin:
    -   Origin name: `origin-app-b`.
    -   Host name: App Service B's default hostname.
    -   Origin host header: use the App Service hostname if requested.
    -   Origin enabled: enabled.
    -   Priority: `1`.
    -   Weight: `1000`.
7.  Configure a health probe:
    -   Protocol: HTTPS.
    -   Method: GET.
    -   Path: `/`.
    -   Choose an interval offered by the portal, such as 30 seconds.
8.  Save the origin group.

**Why use equal priorities?** Equal priority and equal weight allow
Front Door to distribute requests across healthy origins. If you instead
set one origin to a lower numeric priority value than the other, the
lower-numbered origin is preferred and the other can act as a failover
origin.

## 11. Create a route

1.  In the Front Door profile, open **Front Door manager** or
    **Endpoints** and select your endpoint.
2.  Add a route.
3.  Route name: `route-web`.
4.  Domains: select the default Front Door endpoint domain.
5.  Patterns to match: `/*` (or the portal's all-paths option).
6.  Accepted protocols: HTTPS; optionally accept HTTP and enable
    **Redirect all traffic to HTTPS**.
7.  Origin group: `og-az700-waf`.
8.  Forwarding protocol: **HTTPS only** or **Match incoming request**,
    if available. HTTPS is preferred for this lab.
9.  Leave origin path empty.
10. Leave caching disabled.
11. Save the route and wait for deployment to complete.

## 12. Verify Front Door routing

1.  Open the Front Door endpoint domain in a browser using `https://`.
2.  Confirm that the default App Service page loads.
3.  Refresh several times. With equal-priority origins, requests can be
    served by either healthy origin; the default pages may look
    identical.
4.  If the Front Door URL does not work, wait several minutes for
    configuration propagation and check the origin health state, route
    domain, origin hostnames, HTTPS settings, and health probe.

Record the endpoint hostname. Use it for all WAF tests; do not use the
direct App Service hostnames when validating WAF behavior.

------------------------------------------------------------------------

# Part C --- Create the WAF policy and associate it

## 13. Create a Front Door WAF policy

1.  In the portal, search for **Web Application Firewall (WAF)**.
2.  Select **Create**.
3.  For policy type / policy for, select **Global WAF (Front Door)**.
4.  Select the same subscription and resource group as the Front Door
    profile.
5.  Policy name: `waf-az700-lab`.
6.  Front Door tier: **Premium**. It must match the Front Door profile.
7.  Policy state: **Enabled**.
8.  Policy mode: **Detection**.
9.  Continue to **Managed rules**.
10. Enable the available Microsoft **Default Rule Set (DRS)** version
    offered by the portal. Do not hard-code a version from an old
    screenshot; available versions can change.
11. Keep default rule actions and exclusions unchanged for this
    introductory lab.
12. Continue through **Policy settings**. Leave defaults unless you want
    to set a simple custom blocked-response body for the demonstration.
13. Continue to **Custom rules**. You will add the rules in the next
    section.
14. Save or create the policy. If the wizard includes an **Association**
    page, you can associate it there; the following section describes
    the security-policy workflow.

**Concept:** A WAF policy contains settings, custom rules, and (on
Premium) managed rule sets. Creating the policy is not enough: it must
be associated with the Front Door domain that receives traffic.

## 14. Add custom rule 1 --- block a test query string

This rule provides a predictable demonstration that does not depend on
the student's IP address.

1.  Open WAF policy `waf-az700-lab`.

2.  Open **Custom rules** and select **Add custom rule**.

3.  Configure the rule:

      Setting                            Value
      ---------------------------------- --------------------
      Name                               `Block-Test-Query`
      Status                             Enabled
      Rule type                          Match
      Priority                           `10`
      Match condition / match variable   Query string
      Operator                           Contains
      Match value                        `waf-test-block`
      Transformations                    None
      Action                             Block

4.  Save the rule.

Portal labels can vary slightly. Select the query-string match variable,
not the full request URI, so the test focuses on the query string.

### Test the query-string rule in Detection mode

1.  Open the Front Door URL normally, for example
    `https://<front-door-endpoint-hostname>/`.
2.  Confirm that the application loads.
3.  Open `https://<front-door-endpoint-hostname>/?test=waf-test-block`.
4.  In Detection mode, the matching custom rule is logged but its Block
    action is not enforced; the request should still reach the origin.
5.  If the page is not available, check the route and origin health
    before troubleshooting the WAF.

Do not expect to see a WAF log until diagnostic settings have been
enabled and logs have had time to arrive. Logging is configured in Part
E.

## 15. Add custom rule 2 --- block a client IP

This rule demonstrates IP-based access control. Use it temporarily so
you do not accidentally block yourself for the rest of the lab.

1.  Open **Custom rules** and select **Add custom rule**.

2.  Configure:

      Setting          Value
      ---------------- ---------------------------------
      Name             `Block-Student-IP`
      Status           Enabled
      Rule type        Match
      Priority         `20`
      Match variable   Remote address (`RemoteAddr`)
      Operator         IP match / IP address
      Match value      `<your-public-IPv4-address>/32`
      Action           Block

3.  Replace the placeholder with the public IPv4 address that Front Door
    will see. `/32` means one IPv4 address.

4.  Save the rule.

**Important:** Some networks use a proxy, VPN, or shared NAT. The source
IP seen by Front Door may not be the IP shown by your device's network
configuration. Use the RemoteAddr/client-address semantics in the portal
and logs. If the test locks you out, disable this rule or remove your IP
from the match values.

For a safe classroom sequence, add this rule while in Detection mode,
verify its log entry, and then temporarily disable it while testing the
query-string rule in Prevention mode. Re-enable it only when you are
ready to demonstrate IP blocking.

## 16. Associate the WAF policy with Front Door

If you did not associate the policy in the WAF creation wizard:

1.  Open the Azure Front Door profile.
2.  Select **Security policies** (or the equivalent security-policy area
    in Front Door manager).
3.  Select **+ Add**.
4.  Name: `sec-az700-waf`.
5.  Select the WAF policy `waf-az700-lab`.
6.  Select the default Front Door endpoint domain created earlier.
7.  Apply the policy to the selected domain and all paths (`/*`) if a
    path selector is presented.
8.  Save the security policy.
9.  Wait for deployment/propagation to complete.

**Verification:** Reopen the security policy and confirm that the
correct WAF policy and Front Door endpoint domain are associated.

------------------------------------------------------------------------

# Part D --- Detection vs. Prevention

## 17. Configure and test Detection mode

1.  Open `waf-az700-lab`.
2.  Confirm **Policy state = Enabled** and **Policy mode = Detection**.
3.  Ensure the security policy association is deployed.
4.  Browse to the normal Front Door URL. Expected: application loads.
5.  Browse to
    `https://<front-door-endpoint-hostname>/?test=waf-test-block`.
    Expected: request is not blocked by the custom Block action while in
    Detection mode.
6.  Check WAF logs using Part E. Look for the custom rule name or rule
    ID and a match/action indicating detection.

**Explain to students:** Detection mode evaluates and logs matching
requests but does not enforce the configured Block action. It is useful
for observing potential false positives before moving to Prevention.

## 18. Configure and test Prevention mode

1.  In the WAF policy, change **Policy mode** to **Prevention**.
2.  Save the change and wait for propagation.
3.  Make sure `Block-Student-IP` is disabled for the query-string test
    so it does not interfere.
4.  Open the normal Front Door URL. Expected: the application loads.
5.  Open `https://<front-door-endpoint-hostname>/?test=waf-test-block`.
    Expected: Front Door blocks the request, commonly returning HTTP
    `403`.
6.  Check WAF logs and identify the matching custom rule.
7.  Re-enable `Block-Student-IP` only if you are ready to test it.

**Explain to students:** Prevention mode enforces configured actions. A
matching Block action stops the request at the WAF rather than
forwarding it to the origin.

## 19. Test the IP-blocking rule

1.  Keep the policy in Prevention mode.
2.  Enable `Block-Student-IP` with your current public IPv4 `/32`.
3.  Browse to the normal Front Door URL.
4.  Expected: requests from that IP are blocked. The exact behavior can
    vary if the network uses a proxy or changes public IP.
5.  Disable the rule or remove your IP from the match values immediately
    after the test.
6.  Confirm that the normal URL works again.

Do not use an allow rule with a broad match in this lab. Custom rule
actions can terminate evaluation, so priorities and match conditions
matter.

------------------------------------------------------------------------

# Part E --- Configure diagnostics and inspect WAF logs

Logging is an important part of this lab. It is required to demonstrate
Detection mode properly and to show students how to troubleshoot matches
and false positives. Configure diagnostics before relying on the
log-based exercises.

## 20. Create a Log Analytics workspace

1.  In the Azure portal, search for **Log Analytics workspaces**.
2.  Select **Create**.
3.  Subscription: your lab subscription.
4.  Resource group: `rg-az700-waf-lab`.
5.  Name: `law-az700-waf-<unique>`.
6.  Region: choose an available region suitable for the lab; for a
    simple lab, use the same region as the resource group if available.
7.  Select **Review + create**, then **Create**.
8.  Open the workspace after deployment.

## 21. Enable Front Door diagnostic settings

1.  Open the **Azure Front Door profile** (not the App Service and not
    just the WAF policy).
2.  Under **Monitoring**, select **Diagnostic settings**.
3.  Select **+ Add diagnostic setting**.
4.  Name it `diag-afd-waf-lab`.
5.  Select the available log categories:
    -   **FrontDoorWebApplicationFirewallLog** --- essential for WAF
        matches and blocks.
    -   **FrontDoorAccessLog** --- useful for request and routing
        context.
6.  Under destination details, select **Send to Log Analytics
    workspace**.
7.  Select the subscription and workspace `law-az700-waf-<unique>`.
8.  If the portal asks for a destination/table mode, choose the default
    offered for the lab and note it, because it affects query table
    names.
9.  Save the diagnostic setting.

The category names and available destination/table options can vary by
resource and portal experience. Select the WAF log category that is
actually offered by your Front Door profile.

## 22. Generate events and query the logs

1.  Wait a few minutes after enabling diagnostics.
2.  Send several requests through the Front Door hostname:
    -   Normal request: `https://<front-door-endpoint-hostname>/`
    -   Query test:
        `https://<front-door-endpoint-hostname>/?test=waf-test-block`
    -   If configured, a request from the IP listed in
        `Block-Student-IP`.
3.  Open the Log Analytics workspace and select **Logs**.
4.  If diagnostics use **Azure Diagnostics** mode, start with the
    following query:

``` kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.CDN"
| where Category contains "WebApplicationFirewall"
| sort by TimeGenerated desc
| take 50
```

5.  If the query returns no rows:
    -   Confirm the diagnostic setting was saved on the Front Door
        profile.
    -   Confirm `FrontDoorWebApplicationFirewallLog` is selected.
    -   Confirm the correct Log Analytics workspace was selected.
    -   Generate new test requests and wait for ingestion.
    -   Check whether the workspace is using resource-specific tables
        instead of `AzureDiagnostics`. In that case, use the table list
        in the Logs pane and select the Front Door WAF log table offered
        by the workspace.
6.  Open a returned record and inspect fields such as the rule name/ID,
    action, request URI, client address, and timestamp. Exact field
    names vary by table schema.

**Expected learning:** Detection mode produces match events without
enforcing Block; Prevention mode produces block events when a blocking
rule matches. Logs also help investigate false positives and validate
that the correct policy is associated.

------------------------------------------------------------------------

# Part F --- Managed Default Rule Set exercise

## 23. Verify the managed rules

1.  Open `waf-az700-lab`.
2.  Open **Managed rules**.
3.  Confirm that the available Microsoft **Default Rule Set (DRS)** is
    enabled.
4.  Expand the rule groups and inspect the descriptions and enabled
    state.
5.  Keep the default settings for this introductory lab.
6.  Ensure the policy is in Detection mode if you want to observe
    managed-rule matches without blocking requests.

Managed rules are maintained by Microsoft and cover common web attack
patterns. They are different from custom rules, which you define for
requirements such as blocking a specific IP or query-string value.

**Optional demonstration:** Only if time permits, use a harmless,
non-destructive test request documented by Microsoft for the selected
DRS version. Do not assume that every sample payload will trigger a
managed rule; results depend on the rule version, request format, and
configuration. Never run destructive tests against production
applications.

------------------------------------------------------------------------

# Part G --- Optional final exercise: Rate limiting

> **Optional extension:** Complete this only after the required
> query-string, IP-blocking, mode, association, and logging exercises.

Rate limiting is a custom WAF rule type that limits the number of
requests from a client socket IP within a fixed time window. It can help
mitigate excessive request rates, but it is not a complete DDoS
solution.

## 24. Create a rate-limit rule

1.  Open WAF policy `waf-az700-lab`.

2.  Open **Custom rules** and select **Add custom rule**.

3.  Configure a rule similar to the following. Portal labels and
    available thresholds may vary:

      -----------------------------------------------------------------------
      Setting                             Suggested value
      ----------------------------------- -----------------------------------
      Name                                `RateLimit-Lab`

      Status                              Enabled

      Rule type                           Rate limit

      Priority                            `30`

      Rate limit threshold                `20` requests

      Time window                         `1 minute`

      Match condition                     Request URI contains `/` (or
                                          another supported condition that
                                          matches your test path)

      Action                              Block
      -----------------------------------------------------------------------

4.  Save the rule.

5.  Confirm the policy is in **Prevention** mode for a block
    demonstration.

6.  From one client, refresh the Front Door URL repeatedly within the
    configured window. You may use browser refresh; do not use an
    aggressive load-testing tool.

7.  Observe whether requests are blocked after the threshold is reached.
    Check WAF logs for rate-limit events.

8.  Wait for the rate-limit window to expire and test again.

**Important notes:**

-   Rate limits are applied per client socket IP and are enforced across
    Front Door edge servers, so a small amount of traffic above a low
    threshold may still pass.
-   A corporate proxy or shared NAT may cause multiple students to share
    one source IP. Do not run this exercise for the whole class at once
    with a very low threshold.
-   Do not set the threshold so low that ordinary student testing
    becomes confusing.
-   Disable or remove this rule when finished.

------------------------------------------------------------------------

# Part H --- Validation and troubleshooting

## 25. Expected results

  ------------------------------------------------------------------------
  Test                     Detection mode          Prevention mode
  ------------------------ ----------------------- -----------------------
  Normal Front Door        Allowed                 Allowed
  request                                          

  URL includes             Allowed and logged      Blocked by
  `?test=waf-test-block`                           `Block-Test-Query`

  Request from IP in       Allowed and logged      Blocked, if Front Door
  `Block-Student-IP`                               sees that client IP

  Request matching a       Logged; not blocked by  Action depends on
  managed rule             Detection mode          managed-rule
                                                   configuration

  Rate-limit threshold     Logged/evaluated        Requests may be blocked
  exceeded                 without enforcing the   for the rest of the
                           Block action            fixed rate-limit window

  Direct App Service       Bypasses Front Door WAF Bypasses Front Door WAF
  hostname                                         
  ------------------------------------------------------------------------

## 26. Troubleshooting checklist

### Front Door URL does not load

-   Confirm the endpoint and route are enabled.
-   Confirm the route includes the default endpoint domain and `/*`.
-   Confirm both origins have correct App Service hostnames.
-   Check origin health, probe path, and HTTPS configuration.
-   Wait for Front Door configuration propagation.

### WAF rule does not block the request

-   Confirm the WAF policy is **Enabled**.
-   Confirm mode is **Prevention**.
-   Confirm the security policy associates the policy with the exact
    Front Door domain being tested.
-   Confirm the custom rule is enabled and its match condition/value are
    correct.
-   Confirm another higher-priority terminating rule is not changing the
    result.
-   Test through the Front Door hostname, not directly against App
    Service.

### WAF logs do not appear

-   Confirm `FrontDoorWebApplicationFirewallLog` is selected in
    diagnostic settings.
-   Confirm the diagnostic setting sends to the correct Log Analytics
    workspace.
-   Generate fresh requests and allow time for ingestion.
-   Check the workspace's available tables and whether it uses Azure
    Diagnostics or resource-specific mode.
-   Check the selected time range in Logs.

### IP rule does not match

-   Confirm the public IP seen by Front Door.
-   Check whether VPN, proxy, or NAT changes the source IP.
-   Use the intended address match variable (`RemoteAddr` or
    `SocketAddr`) and the correct CIDR.
-   Disable the rule after the exercise to restore access.

------------------------------------------------------------------------

# Part I --- Cleanup

## 27. Delete lab resources

When the lab and review are complete:

1.  Open **Resource groups**.
2.  Select `rg-az700-waf-lab`.
3.  Verify that it contains only resources created for this lab.
4.  Select **Delete resource group**.
5.  Enter the resource group name to confirm, then delete.

Deleting the resource group removes the lab's Front Door profile, WAF
policy, App Services, plans, and Log Analytics workspace if they were
all created in this group. Ensure you have not placed unrelated
resources in the same group.

------------------------------------------------------------------------

# Part J --- Review questions

1.  Why is Front Door a suitable WAF attachment point for a globally
    accessible web application?
2.  What is the difference between a WAF policy and a security policy
    association?
3.  What happens to a matching Block rule in Detection mode? What
    changes in Prevention mode?
4.  What is the difference between a custom rule and the managed Default
    Rule Set?
5.  Why should a WAF policy be evaluated in Detection mode before
    enforcing Prevention in production?
6.  Why does testing an App Service's direct hostname not validate the
    Front Door WAF?
7.  What information can the WAF logs provide when a legitimate request
    is blocked?
8.  How does a rate-limit rule differ from a simple match rule?
9.  Why can a very low rate limit be unreliable or disruptive in a
    classroom using shared NAT?
10. What additional control would help prevent clients from bypassing
    Front Door and accessing the origins directly?

## Suggested answers for the trainer

1.  Front Door is a global HTTP(S) entry point and can apply WAF
    inspection before routing requests to origins.
2.  The WAF policy defines rules and mode. The security policy
    associates the WAF policy with a Front Door domain (and, where
    applicable, scope).
3.  Detection logs a match without enforcing the configured block;
    Prevention enforces the rule action.
4.  Custom rules address specific application or access requirements;
    managed rules provide Microsoft-maintained protections against
    common web exploits.
5.  To identify false positives and tune rules before blocking
    legitimate users.
6.  The direct hostname bypasses the Front Door endpoint and its WAF
    policy.
7.  Matched rule, action, request details, client address, timestamp,
    and other schema-dependent fields.
8.  A match rule acts on conditions in a request; a rate-limit rule
    considers request volume from a client over a time window.
9.  Shared source IPs can cause students to affect one another, and low
    thresholds may not be enforced identically across edge servers.
10. Restrict origin access using supported private connectivity or
    origin access controls and configure the application to accept
    traffic only from the intended Front Door path.

------------------------------------------------------------------------

## Microsoft Learn references

-   [Quickstart: Create an Azure Front Door using the Azure
    portal](https://learn.microsoft.com/en-us/azure/frontdoor/create-front-door-portal)
-   [Azure Web Application Firewall on Azure Front Door ---
    overview](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/afds-overview)
-   [Custom rules for Azure WAF on Front
    Door](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-custom-rules)
-   [Configure an IP restriction WAF
    rule](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-configure-ip-restriction)
-   [Rate limiting for Azure Front Door
    WAF](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-rate-limit)
-   [Best practices for Azure Front Door
    WAF](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-best-practices)
-   [Policy settings for Azure Front Door
    WAF](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-policy-settings)

