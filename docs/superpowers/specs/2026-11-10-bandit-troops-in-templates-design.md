# Bandit troops in recruitment templates

All source paths below are relative to `Bannerlord.PartyAI/` (the project folder).

## Goal

Let players pick bandit troops (looters, sea raiders, forest/mountain/desert/steppe bandits, and their upgrade trees) as targets in recruitment templates, the same way faction and mercenary troops can be picked today.

## Decisions

- **Scope:** all bandit troop trees are selectable. No mod-options toggle.
- **Sourcing:** conversion only. Bandit troops enter a party through the existing troop-conversion step (`PartyAITroopRecruiter.ExchangeClanTroops`), which swaps a recruited volunteer (normally a settlement-culture troop) for the nearest-tier template troop. No new recruitment source.
- **Warning:** when a template is created and contains bandit troops while troop conversion is off, show a warning at creation time (change 4).
- **Out of scope (possible future change):** recruiting captured bandit prisoners into template slots, which is how bandits are recruited in vanilla.

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

### 4. Conversion-off warning (`ViewModels/Dialogs/NewTemplateModal.cs`, `ConfirmCreation`)

- **When:** in the party-screen completion callback, right after `AddPartyTemplate(newTemplate)`, if `newTemplate.Troops` contains any troop whose culture `IsBandit` **and** `SubModule.PartySettingsManager.AllowTroopConversion` is false.
- **Condition scope:** it checks only the global `AllowTroopConversion` flag, because that is the flag the recruitment gates check (see Known limitations). It does not look at the caravan or garrison flags.
- **What:** a one-button information popup (`InformationManager.ShowInquiry` with only the affirmative option, as the mod already does in `PAInformationManager`). Suggested text: "This template contains bandit troops. Bandit troops are only obtained through troop conversion, which is currently disabled, so parties using this template won't recruit them. Enable troop conversion in the mod options." Title: the template name or a short "Bandit troops" heading.
- **Strings:** add the title and body to `_Module/ModuleData/Languages/EN/str_english.xml` as new `PAI`-prefixed string IDs, referenced with `{=ID}Text` like the existing strings.
- **Non-blocking:** the template is still created. The warning is informational only.
- **Fallback:** if showing a popup from inside the party-screen callback misbehaves in-game (see Risks), use `InformationManager.DisplayMessage` with a colored chat line, which the mod already uses for similar notices.

## How it works at runtime

With troop conversion enabled, an AI or companion party recruits an ordinary volunteer from a notable. The recruit event triggers `ExchangeClanTroops`, which replaces it with a random template troop of the nearest tier (limited by the daily conversion cap) and adjusts gold when tier or elite status differs.

## Known limitations

- **Conversion must be on.** Lord and companion parties have conversion off by default (`AllowTroopConversion = false`). `Recruitment.ShouldRecruit` and the two recruitment patches (`Patches/RecruitmentCampaignBehaviorPatches.cs`, `Patches/AiVisitSettlementBehaviorPatches.cs`) check only that global flag. With it off, only volunteers that can upgrade into a template troop are accepted, and a faction recruit never upgrades into a bandit troop, so a bandit-only template recruits nothing.
- **Caravans and garrisons.** `AllowTroopConversionForCaravans` and `AllowTroopConversionForGarrisons` default to on, but the recruitment gates above ignore them. Garrisons are converted by a daily tick independent of those gates, so they should still receive template troops. A caravan with a bandit-only template and the global flag off may recruit nothing. Whether to make the gates honor the per-type flags is a separate decision and is not part of this change.
- **The warning only fires at creation.** A player who creates a bandit template while conversion is on, then turns conversion off later, or who assigns the template to a different party afterward, gets no further warning. Warning at assignment time is not part of this change.

## Risks to verify in-game

- `Occupation.Bandit` may not be the occupation bandit troops carry, or may not exist under that name. The game assemblies are not available in the development environment, so change 1 is unverified until built. If wrong, find the correct value by inspecting `CharacterObject.Occupation` for a known bandit troop.
- Bandit characters may fail `IsValidEncyclopediaItem` in `GetAllTopTierTroops`. If so, add a bandit exception to that check.
- `Recruitment.IsEliteTroop` starts from `Culture.EliteBasicTroop`, which is probably null for bandit cultures. `TraverseTree(null)` returns a single-null list, so this should not crash, and bandit troops simply count as non-elite. `DetermineReplacement` falls back across elite and non-elite.
- Showing an inquiry from inside the party-screen completion callback may not display, or may appear behind or conflict with the closing party screen. If so, use the `DisplayMessage` fallback in change 4 or defer the popup to the next tick.
- The troop picker UI (`ViewModels/Dialogs/NewTemplateModal.cs` and the party screen it opens through `VMUtilities.OpenPartyScreen`) may render bandit entries poorly (culture headings, names, icons).

## Testing

The repo has no test project, and building needs the Bannerlord game assemblies. Verification is manual in-game:

1. With troop conversion off, open "Create New Template" and confirm bandit top-tier troops appear in the Eligible Troops list. Create one bandit-only and one mixed (faction plus bandit) template. Expect the warning popup after each, and that both templates exist afterward.
1a. Create a template with only faction troops (conversion still off). Expect no warning.
1b. Turn conversion on and create a bandit template. Expect no warning.
2. Enable troop conversion, assign the bandit-only template to a companion party, and give it a "Recruit from Template" order. Expect the order to keep a target settlement (not be cleared) and the party to travel there.
3. After the party recruits there, open its roster. Expect the new troops to be bandit troops from the template, not the settlement's faction troops, up to the daily conversion cap.
4. Repeat with the mixed template. Expect only settlements of the faction culture to be targeted, and a mix of faction and bandit troops in the roster.
5. With conversion off, assign the bandit-only template to a companion party. Expect no new recruits (the documented limitation).
6. Load a save made before this change that has existing templates. Expect their troop lists to be unchanged.
