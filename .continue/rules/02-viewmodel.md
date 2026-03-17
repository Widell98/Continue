---
name: ViewModel Rules
description: >
  Använd denna när frågan handlar om ViewModels, commands, properties,
  INotifyPropertyChanged, ObservableCollection, IsLoading, RelayCommand,
  AsyncRelayCommand, MVVM-mönster, BaseViewModel, design-time data eller
  navigering från ViewModel.
alwaysApply: false
globs:
  - "**/*ViewModel.cs"
  - "**/*VM.cs"
  - "**/*ViewModelDesign.cs"
---

# ViewModel-regler

## BaseViewModel
Alla ViewModels ärver från BaseViewModel:

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

## Properties
- Backingfält: `_camelCase`
- Publika properties: `PascalCase`
- Använd alltid `SetProperty(ref _field, value)`
- Om ändring påverkar canExecute: anropa `NotifyCanExecuteChanged` i settern
- Om ändring påverkar computed property: anropa `OnPropertyChanged(nameof(Prop))`
- Computed properties har inget backingfält

## IsLoading — använd alltid exakt detta mönster
```csharp
private bool _isLoading;
public bool IsLoading
{
    get => _isLoading;
    set
    {
        if (SetProperty(ref _isLoading, value))
        {
            LoadCommand.NotifyCanExecuteChanged();
            DeleteCommand.NotifyCanExecuteChanged();
            SaveCommand.NotifyCanExecuteChanged();
        }
    }
}
```

## Command-regler
- Async operationer: `AsyncRelayCommand` eller `AsyncRelayCommand<T>`
- Sync operationer: `RelayCommand` eller `RelayCommand<T>`
- Delete: alltid `AsyncRelayCommand<T>` — aldrig `RelayCommand`
- Navigering: `RelayCommand<T>` där T är entitetstypen
- canExecute på async commands: alltid `() => !IsLoading`

## Command-syntax — avvik aldrig från detta
```csharp
// Sync
OpenDetailCommand = new RelayCommand<OrderViewModel>(OpenDetail, canExecute: o => o is not null);

// Async utan parameter
LoadCommand = new AsyncRelayCommand(LoadAsync, canExecute: () => !IsLoading);

// Async med parameter
DeleteCommand = new AsyncRelayCommand<OrderViewModel>(DeleteAsync, canExecute: o => o is not null && !IsLoading);
```

## Async-metodmönster — använd alltid exakt detta
```csharp
private async Task LoadAsync(CancellationToken ct)
{
    IsLoading = true;
    try
    {
        var result = await _service.GetAsync(ct);
        Items = new ObservableCollection<ItemViewModel>(result.Select(x => new ItemViewModel(x)));
    }
    catch (OperationCanceledException) { }
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

## Collection-regler
- Alltid `ObservableCollection` — aldrig `List`, `IEnumerable` eller array för UI-bunden data
- Ersätt hela kollektionen — lägg aldrig till i en loop:

```csharp
// FEL
foreach (var item in newItems)
    Orders.Add(new OrderViewModel(item));

// RÄTT
Orders = new ObservableCollection<OrderViewModel>(
    newItems.Select(x => new OrderViewModel(x))
);
```

- Sökning måste alltid anropa service — filtrera aldrig lokalt i ObservableCollection

## Navigering
- Alltid via `INavigationService` — aldrig `Frame.Navigate` eller Window-referenser

```csharp
private void OpenDetail(OrderViewModel? order)
{
    if (order is null) return;
    _navigation.NavigateTo<OrderDetailViewModel>(order.Id);
}
```

## Namngivning
| Typ | Mönster | Exempel |
|-----|---------|---------|
| ViewModel-klass | `[Feature]ViewModel` | `OrderListViewModel` |
| Command | `[Verb][Noun]Command` | `DeleteOrderCommand` |
| Async-metod | `[Verb][Noun]Async` | `LoadOrdersAsync` |
| Backingfält | `_camelCase` | `_isLoading` |
| Tjänstefält | `_[name]Service` | `_orderService` |
| Design-time | `[Feature]ViewModelDesign` | `OrderListViewModelDesign` |

## Design-time data
Varje ViewModel måste ha en design-time-variant:

```csharp
public class OrderListViewModelDesign : OrderListViewModel
{
    public OrderListViewModelDesign() : base(new NavigationServiceDesign(), new OrderServiceDesign())
    {
        Orders = new ObservableCollection<OrderViewModel>
        {
            new() { OrderNumber = "ORD-001", CustomerName = "Testbolaget AB", Total = 1250.00m },
            new() { OrderNumber = "ORD-002", CustomerName = "Demo AB", Total = 750.00m },
        };
    }
}
```

## Komplett ViewModel-exempel — använd som mall
```csharp
public class OrderListViewModel : BaseViewModel
{
    private readonly INavigationService _navigation;
    private readonly IOrderService _orderService;

    public OrderListViewModel(INavigationService navigation, IOrderService orderService)
    {
        _navigation = navigation;
        _orderService = orderService;

        LoadCommand = new AsyncRelayCommand(LoadAsync, canExecute: () => !IsLoading);
        OpenDetailCommand = new RelayCommand<OrderViewModel>(OpenDetail, canExecute: o => o is not null);
        DeleteCommand = new AsyncRelayCommand<OrderViewModel>(DeleteAsync, canExecute: o => o is not null && !IsLoading);
    }

    private ObservableCollection<OrderViewModel> _orders = new();
    public ObservableCollection<OrderViewModel> Orders
    {
        get => _orders;
        private set => SetProperty(ref _orders, value);
    }

    private OrderViewModel? _selectedOrder;
    public OrderViewModel? SelectedOrder
    {
        get => _selectedOrder;
        set => SetProperty(ref _selectedOrder, value);
    }

    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set
        {
            if (SetProperty(ref _isLoading, value))
            {
                LoadCommand.NotifyCanExecuteChanged();
                DeleteCommand.NotifyCanExecuteChanged();
            }
        }
    }

    private string? _errorMessage;
    public string? ErrorMessage
    {
        get => _errorMessage;
        set => SetProperty(ref _errorMessage, value);
    }

    public IAsyncRelayCommand LoadCommand { get; }
    public IRelayCommand<OrderViewModel> OpenDetailCommand { get; }
    public IAsyncRelayCommand<OrderViewModel> DeleteCommand { get; }

    private async Task LoadAsync(CancellationToken ct)
    {
        IsLoading = true;
        try
        {
            var result = await _orderService.GetOrdersAsync(ct);
            Orders = new ObservableCollection<OrderViewModel>(
                result.Select(x => new OrderViewModel(x))
            );
        }
        catch (OperationCanceledException) { }
        catch (Exception ex) { ErrorMessage = ex.Message; }
        finally { IsLoading = false; }
    }

    private void OpenDetail(OrderViewModel? order)
    {
        if (order is null) return;
        _navigation.NavigateTo<OrderDetailViewModel>(order.Id);
    }

    private async Task DeleteAsync(OrderViewModel? order, CancellationToken ct)
    {
        if (order is null) return;
        IsLoading = true;
        try
        {
            await _orderService.DeleteAsync(order.Id, ct);
            await LoadAsync(ct);
        }
        catch (Exception ex) { ErrorMessage = ex.Message; }
        finally { IsLoading = false; }
    }
}
```
