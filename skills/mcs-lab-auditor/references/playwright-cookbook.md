# Playwright cookbook for mcs-labs portals

This document captures known quirks, sign-in flows, and selectors for the five portals that bootcamp labs target. The primary skill reads it when deciding how to navigate, when to wait, and how to detect mid-run auth expiry.

## Tool mapping

| Step kind | Primary tool | Notes |
|---|---|---|
| navigate | `mcp__plugin_playwright_playwright__browser_navigate` | URL from step text or scene hint |
| click | `_browser_snapshot` → `_browser_click` | Click by snapshot ref. Never use raw CSS selectors. |
| type | `_browser_type` | Use `slowly: false` unless the field has client-side validation that rate-limits |
| fill form | `_browser_fill_form` | When multiple inputs need to be filled at once |
| select | `_browser_select_option` | For native `<select>` only; M365/Power Platform combos are usually clickable comboboxes — use `_browser_click` |
| keyboard | `_browser_press_key` | Enter, Escape, Tab |
| wait | `_browser_wait_for` | Prefer `text:` over `selector:` |
| inspect | `_browser_snapshot` + `_browser_take_screenshot` | Capture both for the judge |
| diagnostics | `_browser_console_messages`, `_browser_network_requests` | Read after a failed step to enrich the finding |
| evaluate | `_browser_evaluate` | Used sparingly (cookie/localStorage extraction, expiry-page scraping). Restricted by judge-config. |

## Portal map

| Portal | URL prefix | Auth-required probe |
|---|---|---|
| Copilot Studio | `https://copilotstudio.microsoft.com/` | `/environments` |
| M365 Copilot | `https://m365.cloud.microsoft/chat/` | `/chat/` |
| Power Platform admin | `https://admin.powerplatform.microsoft.com/` | `/environments` |
| Azure portal | `https://portal.azure.com/` | `/#home` |
| SharePoint | tenant-specific `https://<tenant>.sharepoint.com/` | tenant root |

All five federate to AAD — a single sign-in at `https://login.microsoftonline.com` cascades to all of them via SSO. In Playwright MCP, the orchestrator reuses the same browser session across turns/subagents; it does not persist auth via `storage-state.json`.

## Sign-in flow (run-start)

After decrypting the cached credential blob:

1. `_browser_navigate` to `https://login.microsoftonline.com/`.
2. `_browser_snapshot` to find the username input (label "Email, phone, or Skype" or "Sign in").
3. `_browser_type` the username.
4. Click `Next` (button text).
5. Wait for the password field (label "Password").
6. `_browser_type` the password.
7. Click `Sign in`.
8. **First-login password change**: if the URL navigates to `…/password/Change` or a "Update your password" page appears, abort the run with status `error, reason: first_login_password_change_required` — the workshop-issued account needs to be initialized once interactively before the plugin can use it.
9. **"Stay signed in?" prompt**: click `Yes`. This is required to ensure cookies persist long enough for the run.
10. **MFA prompt**: if challenged, abort with `error, reason: mfa_required` — workshop accounts should be exempt; if they're not, the workshop org didn't configure them correctly.
11. Wait for redirect to either the M365 home page or `office.com`. Confirm the signed-in landing page is reached and keep the current MCP browser session open for downstream portal steps.
12. **Dismiss first-run welcome modals.** Workshop-issued accounts are fresh tenants, so any portal the run touches will show a one-time welcome dialog. Call the `Welcome-to-Copilot-Studio modal handler` below to clear it before the first lab step. The handler is idempotent — if the modal isn't there, it's a no-op.

That active MCP browser session covers all federated portals. The orchestrator does NOT need to re-sign-in per portal.

## Welcome-to-Copilot-Studio modal handler

The first time a workshop-issued account visits `https://copilotstudio.microsoft.com/`, a one-time **"Welcome to Microsoft Copilot Studio"** modal pops up over the home page. It contains a "Choose your country/region" combobox and a primary **Get Started** button, plus an unchecked marketing-opt-in checkbox. Until it is dismissed, the rest of the Copilot Studio UI is non-interactable, which means the scene-boundary auth probe — and every lab step that touches Copilot Studio — would otherwise get stuck behind it.

The handler is **always set to `United States`** and **always clicks Get Started**, regardless of what AAD pre-populated. Forcing US keeps audit runs deterministic across workshop venues; we never check the marketing opt-in.

### Pseudocode

```
# Precondition: we just navigated to https://copilotstudio.microsoft.com/ (or its preview redirect).
# This handler is idempotent and safe to call after every navigation that lands on copilotstudio.microsoft.com.

_browser_wait_for(text: "Welcome to Microsoft Copilot Studio", time: 10)
# If the wait times out: no modal — already dismissed for this tenant. Return silently.

_browser_snapshot()

# 1) Force "United States" in the country/region dropdown.
#    The control is a native-looking dropdown labeled "Choose your country/region".
#    Try _browser_select_option first; if it's a custom combobox, fall back to click-open + click "United States".
let region_ref = ref_of_label("Choose your country/region")
try:
    _browser_select_option(ref: region_ref, value: "United States")
catch (not_a_native_select):
    _browser_click(ref: region_ref)
    _browser_snapshot()
    _browser_click(ref: ref_of_option("United States"))

# 2) Do NOT check the marketing opt-in checkbox ("I will receive information, tips, and offers...").
#    Leave it unchecked; that's the audit's intent.

# 3) Click "Get Started".
_browser_click(ref: ref_of_button("Get Started"))

# 4) Confirm dismissal.
_browser_wait_for(textGone: "Welcome to Microsoft Copilot Studio", time: 15)
_browser_snapshot()
# Page should now show the Copilot Studio home with the "Agents" left-nav item visible.
```

### When to call the handler

- **Run-start sign-in flow**, after the M365 home redirect (step 12 above). Call it once before any per-UC subagent starts.
- **Scene-boundary auth probe**, right after the probe URL settles successfully (see below).
- **Post-redemption**, in `workshop-redemption.md` §5.5 — the redemption flow is the very first time the account hits Copilot Studio.

The handler never emits a finding. The modal is a Microsoft product onboarding screen, not a lab issue; if a lab's instructions tell the learner to dismiss it, the static phase will flag that separately.

### Variants we don't handle (yet)

- M365 Copilot Chat shows a different "first-run" carousel inside the chat pane. Labs that target `m365.cloud.microsoft/chat/` work around it by sending an explicit message; no dedicated handler is needed.
- Azure portal has its own "Welcome to Azure" splash; see the Azure portal quirks section.

## Scene-boundary auth probe

At the start of each scene (h4 heading), navigate to `config/workshop.yml#auth_probe_url` (default Copilot Studio environments page). If the URL ends up at `login.microsoftonline.com/...`, the session expired:

1. Halt the run.
2. Mark the current lab `status: error, reason: auth_expired` in `manifest.yml`.
3. Append the run entry to `audit-history.yml` with the same status.
4. Tell the user to run `/audit-account redeem` and `/audit-bootcamp --resume <run-id>`.

If the probe succeeds (`copilotstudio.microsoft.com/environments` or similar in the URL), **invoke the Welcome-to-Copilot-Studio modal handler** above before proceeding with the scene. The handler is a no-op once the modal has been dismissed for the tenant, so it's safe to call at every probe.

## Known portal quirks

### Copilot Studio

- **First-run welcome modal** ("Welcome to Microsoft Copilot Studio" with a country/region dropdown and Get Started button) blocks all interaction on the first visit per tenant. See the `Welcome-to-Copilot-Studio modal handler` section above — every entry point that lands on `copilotstudio.microsoft.com` must call it.
- **Environment selector** appears in the top-right; after sign-in, the default environment may not be the one the lab expects. The judge should not flag this as `broken` — instead, flag as `cannot_verify` with a hint to switch environments.
- **Publish modal** appears slowly. After clicking "Publish", wait up to 30s for the confirmation modal — the actual publish operation can be 10-20s.
- **Generated topic regenerates** when you click "Generate". Lab screenshots from before regen may not match. This is `non_deterministic` territory.

#### Custom Prompt tools (Prompt Builder) — the `(Replace this text)` pattern

When a lab instructs the learner to create a **Custom Prompt tool** (Add a tool → New tool → Prompt) and the prompt body contains a placeholder like `(Replace this text)`, **the placeholder is literal — it must be selected and replaced by inserting a typed text-input variable chip via `Add content → Text`, not typed over with free-form text.** Skipping this pattern causes two real failures we've seen in audits:

1. **InputContentFiltered error [105]** — when boundary queries hit Azure OpenAI's content filter. The raw user query gets concatenated into the *system prompt*, which is stricter than the user-content channel. Symptoms: queries like "Who is the president?" or "How tall is the Empire State Building?" return error code 105 instead of a friendly chit-chat refusal.
2. **Prompt fails to receive the user's actual message** — even when the filter doesn't fire, the model responds to whatever literal text was typed instead of the live `Activity.Text`.

**Correct sequence** (audit this step-by-step against the live UI):

1. In Prompt Builder, after pasting the instruction text containing `(Replace this text)`:
   - **Select** the literal string `(Replace this text)` in the contenteditable.
   - Click **Add content → Text** in the prompt editor toolbar.
   - Set **Name** = `Query` (or whatever the lab specifies).
   - Set **Sample data** = realistic sample input (lab may provide).
   - Save the prompt. The selection is replaced by a typed chip displayed inline (e.g. `[Query]`).
2. After clicking **Save → Add and configure**, in the tool's **Inputs** section:
   - For the `Query` row, set **Fill using** = `Custom value`.
   - For **Value**, expand the variable picker and select **System → Activity.Text** (string).
3. Save the tool.

The runtime data path is: `user utterance → Activity.Text (system) → Custom value mapping → Query input variable → prompt's [Query] chip`. This is the only path that (a) avoids the system-prompt content filter and (b) delivers the user's live message to the model.

**This is a plugin/auditor obligation, not a lab requirement.** The mcs-tools lab (UC4) describes the correct sequence in its markdown. The audit run that prompted this cookbook section failed because the auditor typed over the placeholder with prose instead of inserting the Text chip — that's an auditor execution bug, not a lab content bug. The audit driver MUST follow this sequence whenever a Custom Prompt step appears, even if the lab text is correct, because typing over a placeholder is the path of least resistance and the failure mode is silent until boundary queries fire content filter [105].

If a *future* lab introduces a Custom Prompt step but omits these instructions, file a finding describing the missing guidance with a suggested correction that adds the Prompt-Builder `Add content → Text` step and the Inputs `System.Activity.Text` mapping. Do not file such a finding for labs (like `mcs-tools`) where the lab markdown already covers it.

#### Default Greeting topic intercepts orchestrator routing

The default **Greeting** topic on every Copilot Studio agent fires on any utterance the orchestrator classifies as a greeting ("How are you?", "Hello", "Hi", etc.). When a lab asks the learner to test a tool with a greeting-style utterance to verify orchestrator routing, the Greeting topic short-circuits routing and the tool never executes — so the lab appears broken when it isn't.

Two fixes, both should be in the lab text:

1. **Turn off the Greeting topic** on the Topics tab before testing. The toggle is on the topic row in the System topics list.
2. **Use non-greeting test utterances** as alternates ("Tell me about cats", "The weather today.") so the learner can confirm routing even if they forget step 1.

**When the lab markdown omits the "disable Greeting topic" step but the test utterances are greeting-shaped** — file a finding. As of audit run 2026-05-25T1545Z-7a2b, this is true of UC4 in `mcs-tools` (test utterances start with "Hey..." which trips the default Greeting topic) and an accompanying lab issue + PR was filed against `microsoft/mcs-labs`. Unlike the Custom Prompt pattern above, this *is* a lab content gap, not a plugin gap.

### M365 Copilot / Agent Builder

- Agent Builder generates a different first-message and configuration on every "Generate" click. Steps that say "your agent should look similar to this" are `non_deterministic` by definition.
- The Agent Builder pane is an iframe inside `m365.cloud.microsoft/chat/`. `_browser_snapshot` may need a brief wait after navigation for the iframe to load.

### Power Platform admin

- Most pages are server-rendered and stable. Solution explorer and environment variables panels are the main lab targets.

### Azure portal

- First navigation to the Azure portal after sign-in shows a "Welcome to Azure" splash for new accounts. Dismiss with the close button if present.
- Resource creation forms have lots of asynchronous validation. Always `_browser_wait_for` the "Validation passed" text before clicking "Review + create".

### SharePoint

- Tenant-specific URL — the lab must supply it via a step. Workshop accounts get tenant-scoped SharePoint at `<workshopId>.sharepoint.com`.
- "Create site" wizard has a multi-step flow with selector text that varies by tenant template. Lean on visible-text targeting (`Communication site` button), not CSS.

## Anti-patterns

- **Do not use raw CSS or XPath selectors.** Always go through `_browser_snapshot` and target by ref. The DOM in M365/Power Platform changes between releases; visible text is more stable.
- **Do not skip the snapshot before a click.** The snapshot ref must come from a snapshot taken in the same step; using a stale ref will throw.
- **Do not click "Cancel" or "Discard" if a step seems wrong.** Halt the lab as `error` instead — destructive actions can corrupt prior steps' state.
