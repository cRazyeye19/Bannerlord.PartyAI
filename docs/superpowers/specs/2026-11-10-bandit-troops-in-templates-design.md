# Bandit troops in recruitment templates

## Goal

Let players pick bandit troops (looters, sea raiders, forest/mountain/desert/steppe bandits, and their upgrade trees) as targets in recruitment templates, the same way faction and mercenary troops can be picked today.

## Decisions

- **Scope:** all bandit troop trees are selectable. No mod-options toggle.
- **Sourcing:** conversion only. Bandit troops enter a party through the existing troop-conversion step (`PartyAITroopRecruiter.ExchangeClanTroops`), which swaps a recruited volunteer (normally a settlement-culture troop) for the nearest-tier template troop. No new recruitment source.
- **Out of scope (possible future change):** recruiting captured bandit prisoners into template slots, which is how bandits are recruited in vanilla.

## Current behavior that blocks bandits

1. `Recruitment.GetAllTopTierTroops` (picker list) skips troops whose culture `IsBandit`, and only walks trees rooted in `Occupation.Soldier`, `Mercenary` or `CaravanGuard`.
2. `PAICustomTemplate.ResolveTroops` skips troops whose culture `IsBandit`, so they would not count as part of a template.
3. `PartyAiEntitySettings.SetPartyTemplate` clears a "Recruit from Template" order's target settlement unless the template's `TroopCultures` contains the settlement's culture. A bandit culture never matches a settlement.

## Changes

### 1. Picker list (`Domain/Recruitment.cs`, `GetAllTopTierTroops`)

- Remove the `!troop.Culture.IsBandit` condition.
- Add `Occupation.Bandit` to the allowed occupations.
- Bandit top-tier troops then appear in the picker, ordered by culture like the existing entries.

### 2. Template resolution (`PAICustomTemplate.cs`, `ResolveTroops`)

- Remove the `IsBandit` condition.
- This only runs when a template is created. Existing saved templates keep their stored `Troops` list (saveable property), so saves are unaffected.

### 3. Recruit-order targeting (`PAICustomTemplate.cs`, `TroopCultures`)

- Skip bandit cultures when building the culture set.
- A bandit-only template yields an empty set, which `SetPartyTemplate` already treats as unrestricted (any settlement). The order target is no longer cleared.
- A mixed template (for example Vlandian plus bandit) still restricts to settlements of the faction cultures, which supply the volunteers that get converted.

## How it works at runtime

With troop conversion enabled, an AI or companion party recruits an ordinary volunteer from a notable. The recruit event triggers `ExchangeClanTroops`, which replaces it with a random template troop of the nearest tier (limited by the daily conversion cap) and adjusts gold when tier or elite status differs.

Conversion is off by default for lord and companion parties (`AllowTroopConversion = false`) and on by default for caravans and garrisons. With conversion off, `Recruitment.ShouldRecruit` only accepts volunteers that can upgrade into a template troop. A faction recruit never upgrades into a bandit troop, so a bandit-only template recruits nothing. The UI should warn about this when a template contains bandit troops and conversion is off.

## Risks to verify in-game

- Bandit characters may fail `IsValidEncyclopediaItem` in `GetAllTopTierTroops`. If so, add a bandit exception to that check.
- `Recruitment.IsEliteTroop` starts from `Culture.EliteBasicTroop`, which is probably null for bandit cultures. `TraverseTree(null)` returns a single-null list, so this should not crash, and bandit troops simply count as non-elite. `DetermineReplacement` falls back across elite and non-elite.
- The troop picker UI (`NewTemplateModal` and related view models) may render bandit entries poorly (culture headings, names, icons).

## Testing

The repo has no test project, and building needs the Bannerlord game assemblies. Verification is manual in-game:

1. Create a template containing bandit top-tier troops (bandit-only and mixed).
2. Enable troop conversion, assign the template to a companion party, give it a "Recruit from Template" order, and confirm it picks a settlement and that recruits become template troops.
3. Confirm that with conversion off, the warning shows for bandit templates.
4. Load an older save with existing templates and confirm they are unchanged.
