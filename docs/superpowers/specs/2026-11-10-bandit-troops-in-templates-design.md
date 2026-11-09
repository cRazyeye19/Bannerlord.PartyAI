# Bandit troops in recruitment templates

All source paths below are relative to `Bannerlord.PartyAI/` (the project folder).

## Goal

Let players pick bandit troops (looters, sea raiders, forest/mountain/desert/steppe bandits, and their upgrade trees) as targets in recruitment templates, the same way faction and mercenary troops can be picked today.

## Decisions

- **Scope:** all bandit troop trees are selectable. No mod-options toggle.
- **Sourcing:** conversion only. Bandit troops enter a party through the existing troop-conversion step (`PartyAITroopRecruiter.ExchangeClanTroops`), which swaps a recruited volunteer (normally a settlement-culture troop) for the nearest-tier template troop. No new recruitment source.
- **Out of scope (possible future changes):**
  - Recruiting captured bandit prisoners into template slots, which is how bandits are recruited in vanilla.
  - A UI warning when a template contains bandit troops but conversion is off (see Known limitations).

## Current behavior that blocks bandits

1. `Domain/Recruitment.cs`, `GetAllTopTierTroops` (picker list) skips troops whose culture `IsBandit`, and only walks trees rooted in `Occupation.Soldier`, `Mercenary` or `CaravanGuard`.
2. `PAICustomTemplate.cs`, `ResolveTroops` skips troops whose culture `IsBandit`, so they would not count as part of a template.
3. `Models/PartyAiEntitySettings.cs`, `SetPartyTemplate` clears a "Recruit from Template" order's target settlement unless the template's `TroopCultures` contains the settlement's culture. A bandit culture never matches a settlement.

These are the only `IsBandit` filters in the project. The other bandit reference, `IsBanditFaction` in `PartyAIClanPartySettingsManager.IsAIHeroManageable`, concerns clans and is unrelated.

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

## Known limitations

- **Conversion must be on.** Lord and companion parties have conversion off by default (`AllowTroopConversion = false`). `Recruitment.ShouldRecruit` and the two recruitment patches (`Patches/RecruitmentCampaignBehaviorPatches.cs`, `Patches/AiVisitSettlementBehaviorPatches.cs`) check only that global flag. With it off, only volunteers that can upgrade into a template troop are accepted, and a faction recruit never upgrades into a bandit troop, so a bandit-only template recruits nothing.
- **Caravans and garrisons.** `AllowTroopConversionForCaravans` and `AllowTroopConversionForGarrisons` default to on, but the recruitment gates above ignore them. Garrisons are converted by a daily tick independent of those gates, so they should still receive template troops. A caravan with a bandit-only template and the global flag off may recruit nothing. Whether to make the gates honor the per-type flags is a separate decision and is not part of this change.
- **No UI warning.** Nothing tells the player that a bandit template needs conversion on. A natural place for one is the existing pattern in `ViewModels/MenuOptionVMs/PartyAIPartyOptionsVM.cs`, which disables a toggle with an explanatory hint when conversion is on. Deferred.

## Risks to verify in-game

- `Occupation.Bandit` may not be the occupation bandit troops carry, or may not exist under that name. The game assemblies are not available in the development environment, so change 1 is unverified until built. If wrong, find the correct value by inspecting `CharacterObject.Occupation` for a known bandit troop.
- Bandit characters may fail `IsValidEncyclopediaItem` in `GetAllTopTierTroops`. If so, add a bandit exception to that check.
- `Recruitment.IsEliteTroop` starts from `Culture.EliteBasicTroop`, which is probably null for bandit cultures. `TraverseTree(null)` returns a single-null list, so this should not crash, and bandit troops simply count as non-elite. `DetermineReplacement` falls back across elite and non-elite.
- The troop picker UI (`ViewModels/Dialogs/NewTemplateModal.cs` and the party screen it opens through `VMUtilities.OpenPartyScreen`) may render bandit entries poorly (culture headings, names, icons).

## Testing

The repo has no test project, and building needs the Bannerlord game assemblies. Verification is manual in-game:

1. Open "Create New Template" and confirm bandit top-tier troops appear in the Eligible Troops list. Create one bandit-only and one mixed (faction plus bandit) template.
2. Enable troop conversion, assign the bandit-only template to a companion party, and give it a "Recruit from Template" order. Expect the order to keep a target settlement (not be cleared) and the party to travel there.
3. After the party recruits there, open its roster. Expect the new troops to be bandit troops from the template, not the settlement's faction troops, up to the daily conversion cap.
4. Repeat with the mixed template. Expect only settlements of the faction culture to be targeted, and a mix of faction and bandit troops in the roster.
5. With conversion off, assign the bandit-only template to a companion party. Expect no new recruits (the documented limitation).
6. Load a save made before this change that has existing templates. Expect their troop lists to be unchanged.
