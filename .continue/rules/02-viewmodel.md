---
name: ViewModel Rules
description: Use this when the question involves ViewModels, commands, INotifyPropertyChanged, ObservableCollection or MVVM patterns
alwaysApply: false
globs: ["**/*ViewModel.cs", "**/*VM.cs"]
---

# ViewModel-regler

## BaseViewModel
All ViewModels inherit from BaseViewModel:
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

## Properties
- Backing fields: _camelCase
- Public properties: PascalCase
- Always use SetProperty(ref _field, value)
- If change affects canExecute: call NotifyCanExecuteChanged in setter
- If change affects computed property: call OnPropertyChanged(nameof(Prop))

## IsLoading pattern
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

## Command rules
- Async: AsyncRelayCommand or AsyncRelayCommand<T>
- Sync: RelayCommand or RelayCommand<T>
- Delete: always AsyncRelayCommand<T>
- Navigate: RelayCommand<T>
- canExecute on async commands: always () => !IsLoading

## Command syntax — never deviate
LoadCommand = new AsyncRelayCommand(LoadAsync, canExecute: () => !IsLoading);
DeleteCommand = new AsyncRelayCommand<OrderViewModel>(DeleteAsync, canExecute: o => o is not null && !IsLoading);
OpenDetailCommand = new RelayCommand<OrderViewModel>(OpenDetail, canExecute: o => o is not null);

## Async method pattern
private async Task LoadAsync(CancellationToken ct)
{
    IsLoading = true;
    try
    {
        var result = await _service.GetAsync(ct);
        Items = new ObservableCollection<ItemViewModel>(result.Select(x => new ItemViewModel(x)));
    }
    catch (OperationCanceledException) { }
    catch (Exception ex) { ErrorMessage = ex.Message; }
    finally { IsLoading = false; }
}

## Collection rules
- Always ObservableCollection — never List for UI-bound data
- Replace entire collection — never add in loop:
Orders = new ObservableCollection<OrderViewModel>(result.Select(x => new OrderViewModel(x)));
- Search must call service — never filter local ObservableCollection

## Navigation
- Always via INavigationService
- Never Frame.Navigate or Window references
private void OpenDetail(OrderViewModel? order)
{
    if (order is null) return;
    _navigation.NavigateTo<OrderDetailViewModel>(order.Id);
}

## Naming
- ViewModel: [Feature]ViewModel
- Command: [Verb][Noun]Command
- Async method: [Verb][Noun]Async
- Backing field: _camelCase
- Service field: _[name]Service
- Design-time: [Feature]ViewModelDesign

## Design-time data
Every ViewModel must have a design-time variant:
public class OrderListViewModelDesign : OrderListViewModel
{
    public OrderListViewModelDesign() : base(new NavigationServiceDesign(), new OrderServiceDesign())
    {
        Orders = new ObservableCollection<OrderViewModel>
        {
            new() { OrderNumber = "ORD-001", CustomerName = "Testbolaget AB", Total = 1250.00m },
        };
    }
}