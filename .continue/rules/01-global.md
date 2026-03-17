---
name: Global Rules
alwaysApply: true
---

# Globala regler — gäller alltid

Vi jobbar i en stor C# .NET MVVM-kodbas med många utvecklare och domäner.

- Engelska i kod, svenska i kommentarer
- Constructor injection only — inga statiska klasser eller ServiceLocator
- Aldrig `async void` — undantag: event handlers
- Aldrig `.Result` eller `.Wait()` på Task
- Aldrig `Thread.Sleep` eller blockerande anrop
- Aldrig instansiera tjänster i ViewModel: `new ConcreteService()`
- Aldrig `MessageBox.Show` eller direkta UI-anrop i ViewModel
- Alla tjänster registreras i DI-containern vid uppstart — aldrig new:as manuellt
