---
name: Global Rules
alwaysApply: true
---

Vi jobbar i en stor C# .NET MVVM-kodbas med många utvecklare och domäner.

# Globala regler
Before every response:
- Check which domain the question belongs to
- Load and confirm the matching rule file
- Start response with which rule files were loaded
- English in code, Swedish in comments
- Constructor injection only — no static classes or ServiceLocator
- Never use async void — except event handlers
- Never call .Result or .Wait() on any Task
- Never use Thread.Sleep or blocking calls
- Never instantiate services inside ViewModel: new ConcreteService()
- Never use MessageBox.Show or any direct UI calls in ViewModel
- Always read the matching domain rule file before responding