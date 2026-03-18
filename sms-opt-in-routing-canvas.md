# Braze SMS Opt-In Routing Canvas — Sequence Diagrams

This document describes the **single Routing Layer Canvas** that acts as the entry point for all SMS sign-ups. The diagrams below cover the double opt-in orchestration, attribute logging, and downstream routing logic across five key scenarios.

---

## Participants

| Actor | Description |
|---|---|
| **Sign-up Source** | External trigger origin — API events from Facebook, iOS, Android, Web, or an inbound keyword |
| **End User** | The person signing up or texting a keyword |
| **Routing Canvas (Braze)** | The single entry-point canvas that manages opt-in checks, double opt-in, attribute logging, and downstream routing |
| **Custom Keyword Canvas** | Downstream canvas for keyword-specific content (e.g., AMAZON coupon) |
| **SMS Onboarding Canvas** | Downstream canvas for general SMS onboarding sequences |

---

## Comprehensive Overview

This diagram shows the complete routing logic with all branching paths.

```mermaid
sequenceDiagram
    autonumber

    participant SS as Sign-up Source
    participant EU as End User
    participant RC as Routing Canvas<br/>(Braze)
    participant CKC as Custom Keyword<br/>Canvas
    participant SOC as SMS Onboarding<br/>Canvas

    Note over SS,SOC: ENTRY — API event or inbound keyword

    alt API Sign-up (Facebook, iOS, Android, Web)
        SS->>RC: SMS sign-up event via API
    else Inbound Keyword (e.g., AMAZON, WALMART)
        EU->>RC: User texts keyword to short code
    end

    activate RC
    Note over RC: Routing Canvas triggered

    RC->>RC: Check if phone is in<br/>SMS subscription group

    alt User NOT in SMS subscription group (net new or existing unsubscribed)
        rect rgb(255, 235, 238)
        Note over RC: User is not subscribed
        RC->>RC: Add to suppression segment<br/>(block campaign messages until confirmed)
        RC->>EU: Send double opt-in SMS<br/>("Reply Y to confirm")
        EU->>RC: Replies "Y"
        RC->>RC: Confirm SMS subscription
        RC->>RC: Remove from suppression segment
        end
    else User IS in SMS subscription group (already subscribed)
        rect rgb(232, 245, 233)
        Note over RC: Already subscribed —<br/>skip double opt-in,<br/>no suppression needed
        end
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log custom attributes:<br/>• entry_source (API or keyword)<br/>• inbound_keyword (e.g., AMAZON)<br/>• sign_up_platform (e.g., Facebook)
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire custom event with properties:<br/>• source_type (keyword | api_signup)<br/>• keyword (if applicable)

    alt source_type = keyword
        RC->>CKC: Trigger Custom Keyword Canvas
        Note over CKC: Delivers keyword-specific content<br/>(e.g., AMAZON coupon message)
    else source_type = api_signup
        RC->>SOC: Trigger SMS Onboarding Canvas
        Note over SOC: Begins onboarding drip sequence
    end
    end

    deactivate RC

    Note over SS,SOC: Exactly one downstream canvas is triggered<br/>based on event properties
```

---

## Scenario Diagrams

### Scenario 1 — New user texts coupon keyword before double opt-in

A brand-new user (not in Braze or not in the SMS subscription group) texts a keyword like **AMAZON**. They must complete the double opt-in before receiving keyword content.

```mermaid
sequenceDiagram
    autonumber

    participant EU as End User<br/>(New)
    participant RC as Routing Canvas<br/>(Braze)
    participant CKC as Custom Keyword<br/>Canvas

    EU->>RC: Texts "AMAZON" to short code

    activate RC
    Note over RC: Routing Canvas triggered<br/>(inbound keyword entry)

    RC->>RC: Check SMS subscription group
    Note over RC: ❌ User NOT subscribed

    rect rgb(255, 235, 238)
    Note over RC: Double Opt-In Flow
    RC->>RC: Suppress from campaign messaging
    RC->>EU: "Reply Y to confirm SMS subscription"
    EU->>RC: Replies "Y"
    RC->>RC: Add to SMS subscription group
    RC->>RC: Remove suppression
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log attributes:<br/>• entry_source = keyword<br/>• inbound_keyword = AMAZON
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire event: source_type = keyword
    RC->>CKC: Trigger Custom Keyword Canvas
    end
    deactivate RC

    Note over CKC: Deliver AMAZON coupon content
```

---

### Scenario 2 — New user subscribes through standard API double opt-in

A new user signs up via an API source (e.g., Facebook lead form, website form). They go through double opt-in and enter the general onboarding sequence.

```mermaid
sequenceDiagram
    autonumber

    participant SS as Sign-up Source<br/>(e.g., Facebook)
    participant EU as End User<br/>(New)
    participant RC as Routing Canvas<br/>(Braze)
    participant SOC as SMS Onboarding<br/>Canvas

    SS->>RC: API event — SMS sign-up<br/>(source: Facebook)

    activate RC
    Note over RC: Routing Canvas triggered<br/>(API event entry)

    RC->>RC: Check SMS subscription group
    Note over RC: ❌ User NOT subscribed

    rect rgb(255, 235, 238)
    Note over RC: Double Opt-In Flow
    RC->>RC: Suppress from campaign messaging
    RC->>EU: "Reply Y to confirm SMS subscription"
    EU->>RC: Replies "Y"
    RC->>RC: Add to SMS subscription group
    RC->>RC: Remove suppression
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log attributes:<br/>• entry_source = api<br/>• sign_up_platform = Facebook
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire event: source_type = api_signup
    RC->>SOC: Trigger SMS Onboarding Canvas
    end
    deactivate RC

    Note over SOC: Begin onboarding drip sequence
```

---

### Scenario 3 — Existing user (not subscribed) subscribes through standard API double opt-in

An existing Braze user who is **not** in the SMS subscription group signs up via an API source. The flow mirrors Scenario 2 — subscription status drives behavior, not user age.

```mermaid
sequenceDiagram
    autonumber

    participant SS as Sign-up Source<br/>(e.g., Web)
    participant EU as End User<br/>(Existing, not subscribed)
    participant RC as Routing Canvas<br/>(Braze)
    participant SOC as SMS Onboarding<br/>Canvas

    SS->>RC: API event — SMS sign-up<br/>(source: Web)

    activate RC
    Note over RC: Routing Canvas triggered<br/>(API event entry)

    RC->>RC: Check SMS subscription group
    Note over RC: ❌ User NOT subscribed<br/>(exists in Braze but not in SMS group)

    rect rgb(255, 235, 238)
    Note over RC: Double Opt-In Flow
    RC->>RC: Suppress from campaign messaging
    RC->>EU: "Reply Y to confirm SMS subscription"
    EU->>RC: Replies "Y"
    RC->>RC: Add to SMS subscription group
    RC->>RC: Remove suppression
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log attributes:<br/>• entry_source = api<br/>• sign_up_platform = Web
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire event: source_type = api_signup
    RC->>SOC: Trigger SMS Onboarding Canvas
    end
    deactivate RC

    Note over SOC: Begin onboarding drip sequence
```

---

### Scenario 4 — Existing user (already subscribed) texts coupon keyword

An existing user who is **already subscribed** to SMS texts a keyword like **AMAZON**. They bypass double opt-in entirely and go straight to keyword content.

```mermaid
sequenceDiagram
    autonumber

    participant EU as End User<br/>(Existing, subscribed)
    participant RC as Routing Canvas<br/>(Braze)
    participant CKC as Custom Keyword<br/>Canvas

    EU->>RC: Texts "AMAZON" to short code

    activate RC
    Note over RC: Routing Canvas triggered<br/>(inbound keyword entry)

    RC->>RC: Check SMS subscription group
    Note over RC: ✅ User IS subscribed

    rect rgb(232, 245, 233)
    Note over RC: No double opt-in needed<br/>No suppression applied<br/>User proceeds immediately
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log attributes:<br/>• entry_source = keyword<br/>• inbound_keyword = AMAZON
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire event: source_type = keyword
    RC->>CKC: Trigger Custom Keyword Canvas
    end
    deactivate RC

    Note over CKC: Deliver AMAZON coupon content
```

---

### Scenario 5 — Existing user (not subscribed) texts coupon keyword before double opt-in

An existing Braze user who is **not subscribed** to SMS texts a keyword like **AMAZON**. They must complete double opt-in first, then receive keyword content.

```mermaid
sequenceDiagram
    autonumber

    participant EU as End User<br/>(Existing, not subscribed)
    participant RC as Routing Canvas<br/>(Braze)
    participant CKC as Custom Keyword<br/>Canvas

    EU->>RC: Texts "AMAZON" to short code

    activate RC
    Note over RC: Routing Canvas triggered<br/>(inbound keyword entry)

    RC->>RC: Check SMS subscription group
    Note over RC: ❌ User NOT subscribed<br/>(exists in Braze but not in SMS group)

    rect rgb(255, 235, 238)
    Note over RC: Double Opt-In Flow
    RC->>RC: Suppress from campaign messaging
    RC->>EU: "Reply Y to confirm SMS subscription"
    EU->>RC: Replies "Y"
    RC->>RC: Add to SMS subscription group
    RC->>RC: Remove suppression
    end

    rect rgb(232, 240, 254)
    Note over RC: Attribute Logging
    RC->>RC: Log attributes:<br/>• entry_source = keyword<br/>• inbound_keyword = AMAZON
    end

    rect rgb(243, 229, 245)
    Note over RC: Downstream Routing
    RC->>RC: Fire event: source_type = keyword
    RC->>CKC: Trigger Custom Keyword Canvas
    end
    deactivate RC

    Note over CKC: Deliver AMAZON coupon content<br/>(same as Scenario 1 — subscription status<br/>drives opt-in, not user age)
```

---

## Scenario Summary Matrix

| # | User Status | SMS Subscribed? | Entry Type | Double Opt-In? | Downstream Canvas |
|---|---|---|---|---|---|
| 1 | New | No | Keyword (AMAZON) | Yes | Custom Keyword Canvas |
| 2 | New | No | API (Facebook) | Yes | SMS Onboarding Canvas |
| 3 | Existing | No | API (Web) | Yes | SMS Onboarding Canvas |
| 4 | Existing | Yes | Keyword (AMAZON) | No | Custom Keyword Canvas |
| 5 | Existing | No | Keyword (AMAZON) | Yes | Custom Keyword Canvas |

### Key Takeaway

The **SMS subscription status** — not whether the user is "new" or "existing" in Braze — determines whether the double opt-in flow is triggered. The **entry type** (keyword vs. API sign-up) determines which downstream canvas is activated.

---

## Next Steps

### 1. Create and Configure the Routing Canvas
- Build the single Routing Canvas in Braze with two entry triggers:
  - **API-triggered entry** for sign-ups from Facebook, iOS, Android, and Web
  - **Inbound keyword entry** for custom keyword responses
- Add a **Decision Split** step checking SMS subscription group membership
- Configure the **double opt-in message step** with a "Reply Y to confirm" template
- Add a **User Update** step to add confirmed users to the SMS subscription group
- Implement **suppression logic** (add to suppression segment on entry for unsubscribed users; remove upon confirmation)
- Add **Custom Attribute** steps to log `entry_source`, `inbound_keyword`, and `sign_up_platform`
- Add a **Custom Event** step to fire the downstream routing event with `source_type` and `keyword` properties

### 2. Update Existing Downstream Canvases
- **Custom Keyword Canvas**: Remove all single opt-in steps (contact card layers, opt-in confirmation, subscription updates). The Routing Canvas now handles all of this.
- **SMS Onboarding Canvas**: Same cleanup — remove opt-in logic. This canvas should assume the user is already double-opted-in when they arrive.

### 3. Set New Downstream Triggers
- Change the **Custom Keyword Canvas** trigger to: custom event from Routing Canvas where `source_type = keyword`
- Change the **SMS Onboarding Canvas** trigger to: custom event from Routing Canvas where `source_type = api_signup`
- Both triggers should be able to access event properties (e.g., `keyword`, `sign_up_platform`) for message personalization

### 4. Finalize Trigger Logic for Existing Subscribed Users
- Existing subscribed users who text a keyword must still flow through the Routing Canvas (to log attributes and fire the downstream event) but **bypass** the double opt-in entirely
- Verify that re-engagement with a keyword correctly triggers the Custom Keyword Canvas and delivers the expected content without any opt-in friction
- Consider rate-limiting or deduplication if a subscribed user texts the same keyword multiple times

---

## Custom Attributes & Event Properties Reference

### Custom Attributes (logged on user profile)

| Attribute | Type | Example Values | Purpose |
|---|---|---|---|
| `entry_source` | String | `keyword`, `api` | How the user entered the SMS program |
| `inbound_keyword` | String | `AMAZON`, `WALMART`, `DEALS` | The specific keyword texted (if applicable) |
| `sign_up_platform` | String | `Facebook`, `iOS`, `Android`, `Web` | The API source platform (if applicable) |
| `sms_opt_in_date` | Date | `2026-03-18` | When the user confirmed their subscription |

### Custom Event Properties (passed to downstream canvases)

| Property | Type | Example Values | Purpose |
|---|---|---|---|
| `source_type` | String | `keyword`, `api_signup` | Determines which downstream canvas to trigger |
| `keyword` | String | `AMAZON` | The keyword for content personalization |
| `sign_up_platform` | String | `Facebook` | The originating platform for analytics |
