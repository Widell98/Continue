# domain-vm.md — ViewModel-domän (MVVM)

> Läses av AI-agenten när frågan rör ViewModels, kommandon, properties,
> notifieringar eller navigering i vår C# .NET MVVM-kodbas.

---

## 1. Grundregler

- **Ingen kod-bakom i View** — all logik tillhör ViewModel
- **ViewModel känner inte till View** — ingen referens till UI-kontroller
- **Konstruktorinjektion alltid** — inga statiska klasser eller ServiceLocator
- **Asynkront** — alltid `async`/`await`, aldrig `.Result` eller `.Wait()`
- **Dispose** — implementera `IDisposable` om VM prenumererar på events

---

## 2. BaseViewModel

Alla ViewModels ärver från `BaseViewModel`. Använd aldrig `INotifyPropertyChanged` direkt.

```csharp
public abstract class BaseViewModel : INotifyPropertyChanged, IDisposable
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));

    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }

    public virtual void Dispose() { }
}
```

---

## 3. Properties

### Enkelt värde
```csharp
private string _firstName = string.Empty;
public string FirstName
{
    get => _firstName;
    set => SetProperty(ref _firstName, value);
}
```

### Med sidoeffekt
```csharp
private bool _isLoading;
public bool IsLoading
{
    get => _isLoading;
    set
    {
        if (SetProperty(ref _isLoading, value))
            LoadCommand.NotifyCanExecuteChanged();
    }
}
```

### Beräknad (computed) — ingen backing field
```csharp
public bool HasItems => Items.Count > 0;

// Glöm inte notifiera vid beroendets ändring:
private void OnItemsChanged() => OnPropertyChanged(nameof(HasItems));
```

---

## 4. Kommandon

### RelayCommand (synkront)
```csharp
public IRelayCommand DeleteCommand { get; }

// I konstruktorn:
DeleteCommand = new RelayCommand(
    execute: DeleteSelected,
    canExecute: () => SelectedItem is not null
);

private void DeleteSelected()
{
    Items.Remove(SelectedItem!);
    SelectedItem = null;
}
```

### AsyncRelayCommand (asynkront) — föredras framför RelayCommand + async void
```csharp
public IAsyncRelayCommand LoadCommand { get; }

// I konstruktorn:
LoadCommand = new AsyncRelayCommand(
    execute: LoadDataAsync,
    canExecute: () => !IsLoading
);

private async Task LoadDataAsync(CancellationToken ct)
{
    IsLoading = true;
    try
    {
        var result = await _dataService.GetItemsAsync(ct);
        Items = new ObservableCollection<ItemViewModel>(result.Select(x => new ItemViewModel(x)));
    }
    catch (OperationCanceledException) { /* ignorera */ }
    catch (Exception ex)
    {
        ErrorMessage = ex.Message;
    }
    finally
    {
        IsLoading = false;
    }
}
```

### ❌ Undvik async void (utom i event handlers)
```csharp
// FEL — exceptions swallowas, testbart inte
public ICommand SaveCommand => new RelayCommand(async () => await SaveAsync());

// RÄTT
public IAsyncRelayCommand SaveCommand { get; } = new AsyncRelayCommand(SaveAsync);
```

---

## 5. Samlingar

### Använd alltid ObservableCollection
```csharp
private ObservableCollection<OrderViewModel> _orders = new();
public ObservableCollection<OrderViewModel> Orders
{
    get => _orders;
    private set => SetProperty(ref _orders, value);
}
```

### Bulk-uppdatering — ersätt hela samlingen istället för Add i loop
```csharp
// FEL — triggar OnPropertyChanged för varje item
foreach (var item in newItems)
    Orders.Add(new OrderViewModel(item));

// RÄTT
Orders = new ObservableCollection<OrderViewModel>(
    newItems.Select(x => new OrderViewModel(x))
);
```

---

## 6. Navigering

Navigering sker via `INavigationService` — ViewModel anropar aldrig `Frame.Navigate` direkt.

```csharp
public class OrderListViewModel : BaseViewModel
{
    private readonly INavigationService _navigation;

    public OrderListViewModel(INavigationService navigation)
    {
        _navigation = navigation;
        OpenDetailCommand = new RelayCommand<OrderViewModel>(OpenDetail);
    }

    public IRelayCommand<OrderViewModel> OpenDetailCommand { get; }

    private void OpenDetail(OrderViewModel? order)
    {
        if (order is null) return;
        _navigation.NavigateTo<OrderDetailViewModel>(order.Id);
    }
}
```

---

## 7. Validering

Implementera `INotifyDataErrorInfo` via BaseViewModel eller CommunityToolkit.

```csharp
[ObservableProperty]
[NotifyDataErrorInfo]
[Required(ErrorMessage = "Namn krävs")]
[MaxLength(100)]
private string _name = string.Empty;
```

Om CommunityToolkit inte används, validera manuellt i setter:
```csharp
public string Email
{
    get => _email;
    set
    {
        SetProperty(ref _email, value);
        ValidateEmail(value);
    }
}

private void ValidateEmail(string value)
{
    ClearErrors(nameof(Email));
    if (!value.Contains('@'))
        AddError(nameof(Email), "Ogiltig e-postadress");
}
```

---

## 8. Designtidsstöd (Design-time data)

Varje ViewModel ska ha en design-time-variant för XAML-förhandsvisning.

```csharp
public class OrderListViewModelDesign : OrderListViewModel
{
    public OrderListViewModelDesign() : base(new NavigationServiceDesign())
    {
        Orders = new ObservableCollection<OrderViewModel>
        {
            new() { OrderNumber = "ORD-001", CustomerName = "Testbolaget AB", Total = 1250.00m },
            new() { OrderNumber = "ORD-002", CustomerName = "Demo AB", Total = 750.00m },
        };
    }
}
```

I XAML:
```xml
d:DataContext="{d:DesignInstance Type=vm:OrderListViewModelDesign, IsDesignTimeCreatable=True}"
```

---

## 9. Namngivning

| Koncept | Konvention | Exempel |
|---|---|---|
| Backing field | `_camelCase` | `_isLoading` |
| Public property | `PascalCase` | `IsLoading` |
| Kommando | `[Verb][Noun]Command` | `DeleteOrderCommand` |
| Async-metod | `[Verb]Async` | `LoadOrdersAsync` |
| ViewModel-klass | `[Feature]ViewModel` | `OrderListViewModel` |
| Design-time | `[Feature]ViewModelDesign` | `OrderListViewModelDesign` |

---

## 10. Dos och don'ts

### ✅ Gör
- Använd `CommunityToolkit.Mvvm` där det förenklar
- Injicera tjänster via konstruktorn
- Skriv unit-tester för kommandon och properties
- Använd `CancellationToken` i alla async-metoder
- Håll ViewModel tunn — flytta affärslogik till service-lagret

### ❌ Undvik
- `async void` (utom event handlers)
- Direktreferens till `Window`, `Frame` eller UI-kontroller
- Statiska fält eller singleton-state i ViewModel
- Logik i property getters (tunga beräkningar)
- `Thread.Sleep` eller blockerande anrop

---

## 11. Vanliga misstag i vår kodbas

```csharp
// ❌ FEL — glömt notifiera beroende property
private bool _isAdmin;
public bool IsAdmin { get => _isAdmin; set => SetProperty(ref _isAdmin, value); }
public bool CanDelete => IsAdmin && SelectedItem is not null;
// CanDelete uppdateras aldrig när IsAdmin ändras!

// ✅ RÄTT
public bool IsAdmin
{
    get => _isAdmin;
    set
    {
        if (SetProperty(ref _isAdmin, value))
            OnPropertyChanged(nameof(CanDelete));
    }
}
```

```csharp
// ❌ FEL — ViewModel vet om View
public void ShowError()
{
    MessageBox.Show("Fel!"); // Bryter MVVM, ej testbart
}

// ✅ RÄTT — via service
public void ShowError()
{
    _dialogService.ShowError("Fel uppstod vid sparande");
}
```

---

*Tillhör: `.continue/domain-vm.md` | Uppdateras av AI-arkitekten vid teambeslut*
