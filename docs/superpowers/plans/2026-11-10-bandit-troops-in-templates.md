# Bandit Troops in Recruitment Templates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let players pick bandit troops as recruitment-template targets, and warn at template creation when a bandit template cannot work because troop conversion is off.

**Architecture:** Remove the two `IsBandit` filters that hide bandit troops (picker list and template resolution), make bandit cultures not count toward a template's settlement-culture restriction so "Recruit from Template" orders keep a target, and add a creation-time warning popup. Bandit troops enter parties through the existing troop-conversion swap, so no recruiting logic changes.

**Tech Stack:** C# (netstandard2.0), Bannerlord game API (`TaleWorlds.*`), Harmony. Spec: `docs/superpowers/specs/2026-11-10-bandit-troops-in-templates-design.md`.

**Testing note:** The repo has no test project, and compiling needs the Bannerlord game assemblies, which are not available in the development environment (no `dotnet` either). There is no TDD cycle. Each task is verified with a `grep` check on the source, and the final task is a build plus manual in-game verification on a machine with the game installed. All source paths are relative to the repo root.

---

## File Structure

| File | Change |
|---|---|
| `Bannerlord.PartyAI/Domain/Recruitment.cs` | Picker list includes bandit troops (Task 1) |
| `Bannerlord.PartyAI/PAICustomTemplate.cs` | Template resolution includes bandits; `TroopCultures` ignores bandit cultures; new `ContainsBanditTroops` property (Tasks 2-3) |
| `Bannerlord.PartyAI/ViewModels/Dialogs/NewTemplateModal.cs` | Creation-time warning (Task 4) |
| `Bannerlord.PartyAI/_Module/ModuleData/Languages/EN/str_english.xml` | Two new strings for the warning (Task 4) |

Commit style follows upstream: short plain-sentence messages, ending with the attribution line below.

---

### Task 1: Show bandit troops in the template picker

**Files:**
- Modify: `Bannerlord.PartyAI/Domain/Recruitment.cs:162-176` (`GetAllTopTierTroops`)

- [ ] **Step 1: Allow the Bandit occupation**

In `GetAllTopTierTroops`, replace

```csharp
        Occupation[] occupations = [Occupation.Soldier, Occupation.Mercenary, Occupation.CaravanGuard];
```

with

```csharp
        Occupation[] occupations = [Occupation.Soldier, Occupation.Mercenary, Occupation.CaravanGuard, Occupation.Bandit];
```

- [ ] **Step 2: Remove the bandit-culture exclusion**

Replace

```csharp
            if (!characters.Contains(troop) && !troop.IsHero && troop.Culture != null
                && !troop.Culture.IsBandit && occupations.Contains(troop.Occupation))
```

with

```csharp
            if (!characters.Contains(troop) && !troop.IsHero && troop.Culture != null
                && occupations.Contains(troop.Occupation))
```

- [ ] **Step 3: Verify the filter is gone**

Run: `grep -n "IsBandit\|Occupation.Bandit" Bannerlord.PartyAI/Domain/Recruitment.cs`
Expected: exactly one line, the occupations array containing `Occupation.Bandit`. No `IsBandit` match.

- [ ] **Step 4: Commit**

```bash
git add Bannerlord.PartyAI/Domain/Recruitment.cs
git commit -m "Show bandit troops in the template troop picker

Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Resolve bandit troops as part of a template

**Files:**
- Modify: `Bannerlord.PartyAI/PAICustomTemplate.cs:39-54` (`ResolveTroops`)

- [ ] **Step 1: Remove the bandit-culture exclusion**

In `ResolveTroops`, replace

```csharp
                if (!troop.IsHero && (!troop.Culture?.IsBandit ?? false) && UpgradesTo(troop, e.Character))
```

with

```csharp
                if (!troop.IsHero && UpgradesTo(troop, e.Character))
```

- [ ] **Step 2: Verify**

Run: `grep -n "IsBandit" Bannerlord.PartyAI/PAICustomTemplate.cs`
Expected: no output (the helper added in Task 3 does not exist yet).

- [ ] **Step 3: Commit**

```bash
git add Bannerlord.PartyAI/PAICustomTemplate.cs
git commit -m "Count bandit troops as part of a template

Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Keep recruit-order targets for bandit templates

`SetPartyTemplate` (`Models/PartyAiEntitySettings.cs:212-214`) clears a recruit order's target unless the template's `TroopCultures` contains the settlement's culture, and already treats an empty set as unrestricted. Ignoring bandit cultures in `TroopCultures` makes a bandit-only template unrestricted and a mixed template restrict to its faction cultures. `PartyAiEntitySettings.cs` itself does not change.

This task also adds `ContainsBanditTroops`, used by Task 4.

**Files:**
- Modify: `Bannerlord.PartyAI/PAICustomTemplate.cs:14-29`

- [ ] **Step 1: Skip bandit cultures in `TroopCultures` and add the helper and property**

Replace

```csharp
    private HashSet<CultureObject> _troopCultures = new();
    public HashSet<CultureObject> TroopCultures
    {
        get
        {
            _troopCultures ??= new();
            if (_troopCultures.Count == 0)
            {
                foreach (CharacterObject troop in Troops)
                {
                    _troopCultures.Add(troop.Culture);
                }
            }
            return _troopCultures;
        }
    }
```

with

```csharp
    private HashSet<CultureObject> _troopCultures = new();

    /// <summary>
    /// Cultures of the faction troops in this template. Bandit cultures are ignored: no
    /// settlement has a bandit culture, so counting them would make every settlement look
    /// ineligible. A bandit-only template therefore has an empty set, which callers treat as
    /// "any settlement".
    /// </summary>
    public HashSet<CultureObject> TroopCultures
    {
        get
        {
            _troopCultures ??= new();
            if (_troopCultures.Count == 0)
            {
                foreach (CharacterObject troop in Troops)
                {
                    if (IsBanditTroop(troop))
                    {
                        continue;
                    }

                    _troopCultures.Add(troop.Culture);
                }
            }
            return _troopCultures;
        }
    }

    public bool ContainsBanditTroops => Troops.Any(IsBanditTroop);

    private static bool IsBanditTroop(CharacterObject troop) => troop.Culture?.IsBandit ?? false;
```

(`System.Linq` is already imported in this file.)

- [ ] **Step 2: Verify**

Run: `grep -n "IsBanditTroop\|ContainsBanditTroops" Bannerlord.PartyAI/PAICustomTemplate.cs`
Expected: four lines: the `continue` check in the getter, the `ContainsBanditTroops` property, and the `IsBanditTroop` definition plus the `Any(IsBanditTroop)` use (the definition and the property each match once, the getter once, so at least three distinct lines containing `IsBanditTroop` and one containing `ContainsBanditTroops`).

- [ ] **Step 3: Commit**

```bash
git add Bannerlord.PartyAI/PAICustomTemplate.cs
git commit -m "Ignore bandit cultures when restricting recruit targets

Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Warn on creation when a bandit template cannot recruit

**Files:**
- Modify: `Bannerlord.PartyAI/_Module/ModuleData/Languages/EN/str_english.xml` (before the closing `</strings>`)
- Modify: `Bannerlord.PartyAI/ViewModels/Dialogs/NewTemplateModal.cs` (usings; `ConfirmCreation`)

- [ ] **Step 1: Add the strings**

In `str_english.xml`, directly after the last `<string>` line (`PAIAreaPatrolHint`) and before `</strings>`, add:

```xml
	<string id="PAIBanditTemplateTitle"             text="Bandit Troops"/>
	<string id="PAIBanditTemplateWarning"           text="This template contains bandit troops. Bandit troops are only obtained through troop conversion, which is currently disabled, so parties using this template won't recruit them. Enable troop conversion in the mod options."/>
```

(The file already mixes random and readable IDs, for example `PAIAreaPatrolHint`.)

- [ ] **Step 2: Add the usings**

In `NewTemplateModal.cs`, replace

```csharp
using System;
using TaleWorlds.Core;
using TaleWorlds.Localization;
```

with

```csharp
using System;
using TaleWorlds.Core;
using TaleWorlds.Library;
using TaleWorlds.Localization;
```

- [ ] **Step 3: Show the warning after the template is saved**

In `ConfirmCreation`, replace

```csharp
                SubModule.PartySettingsManager.AddPartyTemplate(newTemplate);

                return true;
```

with

```csharp
                SubModule.PartySettingsManager.AddPartyTemplate(newTemplate);

                if (newTemplate.ContainsBanditTroops && !SubModule.PartySettingsManager.AllowTroopConversion)
                {
                    ShowBanditConversionWarning();
                }

                return true;
```

- [ ] **Step 4: Add the warning method**

At the end of the `NewTemplateModal` class, after `ConfirmCreation`, add:

```csharp
    private static void ShowBanditConversionWarning()
    {
        InformationManager.ShowInquiry(new InquiryData(
            new TextObject("{=PAIBanditTemplateTitle}Bandit Troops").ToString(),
            new TextObject("{=PAIBanditTemplateWarning}This template contains bandit troops. Bandit troops are only obtained through troop conversion, which is currently disabled, so parties using this template won't recruit them. Enable troop conversion in the mod options.").ToString(),
            true, false, new TextObject("{=Y94H6XnK}Accept").ToString(), string.Empty,
            () => { }, null));
    }
```

This mirrors the existing one-button inquiry in `PAInformationManager.CheckContext`. The `{=Y94H6XnK}Accept` string already exists in the base game.

- [ ] **Step 5: Verify**

Run: `grep -n "ContainsBanditTroops\|ShowBanditConversionWarning\|PAIBanditTemplate" Bannerlord.PartyAI/ViewModels/Dialogs/NewTemplateModal.cs Bannerlord.PartyAI/_Module/ModuleData/Languages/EN/str_english.xml`
Expected: `ContainsBanditTroops` once and `ShowBanditConversionWarning` twice (call and definition) in `NewTemplateModal.cs`; `PAIBanditTemplateTitle` and `PAIBanditTemplateWarning` once each in the XML and once each in `NewTemplateModal.cs`.

- [ ] **Step 6: Commit**

```bash
git add Bannerlord.PartyAI/ViewModels/Dialogs/NewTemplateModal.cs Bannerlord.PartyAI/_Module/ModuleData/Languages/EN/str_english.xml
git commit -m "Warn when a bandit template is created with troop conversion off

Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Build and verify in-game (needs a machine with Bannerlord installed)

**Files:** none changed unless a risk below fires.

- [ ] **Step 1: Build**

Run: `dotnet build Bannerlord.PartyAI.slnx -c Debug`
Expected: build succeeds with no errors. This confirms `Occupation.Bandit` exists and the new code compiles. If the SDK cannot find the game, set the `GameFolder` MSBuild property to the Bannerlord install directory (the Bannerlord build SDK's convention; not verified here).
If `Occupation.Bandit` does not compile, find the right enum value for bandit troops in the game assemblies and use that in Task 1 step 1.

- [ ] **Step 2: Run the manual checks from the spec**

Follow the Testing section of the spec, in order:

1. Conversion off: create a bandit-only and a mixed template. Expect bandit top-tier troops in the picker, the warning after each creation, and both templates created.
2. Faction-only template, conversion off: expect no warning.
3. Conversion on: create a bandit template. Expect no warning.
4. Conversion on: assign the bandit-only template to a companion with a "Recruit from Template" order. Expect the order to keep a target settlement, the party to travel there, and its new troops to be bandit troops (up to the daily conversion cap).
5. Mixed template: only settlements of the faction culture are targeted, and the roster mixes faction and bandit troops.
6. Conversion off with the bandit-only template assigned: no new recruits (documented limitation).
7. Load a save from before this change: existing templates are unchanged.

- [ ] **Step 3: Handle the risks from the spec if they occur**

- Bandit troops missing from the picker even though the build succeeded: they probably fail `IsValidEncyclopediaItem` in `Recruitment.GetAllTopTierTroops` (the loop that calls `pageOf.IsValidEncyclopediaItem(c)`). Change that condition to `pageOf.IsValidEncyclopediaItem(c) || c.Occupation == Occupation.Bandit`, then re-run check 1.
- Warning popup doesn't show, or shows behind the closing party screen: in `ShowBanditConversionWarning`, replace the `ShowInquiry` call with
  ```csharp
  InformationManager.DisplayMessage(new InformationMessage(
      new TextObject("{=PAIBanditTemplateWarning}This template contains bandit troops. Bandit troops are only obtained through troop conversion, which is currently disabled, so parties using this template won't recruit them. Enable troop conversion in the mod options.").ToString(),
      Colors.Red));
  ```
- Bandit entries render badly in the picker (names, icons, headings): note exactly what is wrong. That needs its own follow-up, so do not fix it in this change.

- [ ] **Step 4: Commit any risk fixes**

Only if step 3 changed code:

```bash
git add -A Bannerlord.PartyAI
git commit -m "Fix bandit template issues found in-game

Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage**
- Change 1 (picker): Task 1. Change 2 (resolution): Task 2. Change 3 (`TroopCultures`): Task 3. Change 4 (warning): Task 4.
- Spec risks (`Occupation.Bandit`, encyclopedia filter, popup timing, picker rendering): Task 5 steps 1 and 3. The `IsEliteTroop` null-culture risk needs no change and is observed through check 4.
- Known limitations (conversion must be on, caravan flag, warning only at creation): documented in the spec, no code, covered by checks 3 and 6.
- Spec test plan: Task 5 step 2 (the spec's 1/1a/1b/2-6 map to checks 1-7 here).

**Placeholders:** none. Every code step shows the code.

**Names:** `IsBanditTroop` and `ContainsBanditTroops` (defined in Task 3) match their use in Task 4. `ShowBanditConversionWarning` is defined and called within Task 4. String IDs `PAIBanditTemplateTitle` and `PAIBanditTemplateWarning` match between the XML and the C# (and the fallback in Task 5).
