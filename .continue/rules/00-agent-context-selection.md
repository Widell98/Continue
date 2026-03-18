---
name: Agent Context Selection
description: >
  Använd denna när användaren ställer breda eller öppna frågor utan att
  specifikt nämna eller @-markera filer. Guida agenten att själv välja
  relevanta filer för kontext.
alwaysApply: true
---

# Agent Context Selection — breda frågor

När användaren ställer en **bred fråga** (t.ex. "Hur fungerar X?", "Förklara Y", "Vad är mönstret för Z?") utan att ange specifika filer:

1. **Proaktivt sök och läs** relevanta filer med tillgängliga sök- och filläsningsverktyg
2. **Välj filer utifrån frågans tema** — inkludera filer som matchar reglernas globs så att rätt regler aktiveras
3. **Bygg kontext före svar** — läs in tillräckligt med kod innan du svarar

## Regel–fil-mappning (globs → vilka regler som aktiveras)

| Frågetema | Filer att inkludera | Regler som aktiveras |
|-----------|---------------------|----------------------|
| ViewModels, MVVM, commands, ObservableCollection, BaseViewModel | `**/*ViewModel.cs`, `**/*VM.cs`, `**/*ViewModelDesign.cs` | ViewModel Rules |
| UI, XAML, bindings, styles, UserControls | `**/*.xaml`, `**/*.xaml.cs` | UI Rules |
| Services, repositories, DI, IOrderService | `**/*Service.cs`, `**/*Repository.cs`, `**/I*Service.cs`, `**/I*Repository.cs` | Service Rules |
| DbContext, EF Core, migrationer, databas | `**/*Context.cs`, `**/*Migration*.cs`, `**/Migrations/*.cs`, `**/*Configuration.cs` | Data Rules |

## Flöde för breda frågor

1. Tolka frågans tema (ViewModel, UI, Service, Data, eller kombination)
2. Sök efter matchande filer med glob eller grep
3. Läs in representativa filer som täcker frågan
4. Svara med stöd av den inlästa koden och de regler som nu är aktiverade

## Exempel

**Fråga:** "Hur hanterar vi laddning och fel i listvyer?"

→ Tema: ViewModel + eventuellt UI. Inkludera: `*ViewModel.cs`, `*ListViewModel.cs`, relevanta XAML-filer. Reglerna för IsLoading, LoadCommand, ErrorMessage aktiveras via ViewModel-globs.

**Fråga:** "Hur pratar ViewModels med databasen?"

→ Tema: ViewModel + Service + Data. Inkludera: ViewModel-filer, Service-filer, DbContext/Configuration. Alla tre regeluppsättningar blir tillgängliga.
