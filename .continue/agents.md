# Global regler
- Språk: svenska i kommentarer, engelska i kod
- Namngivning: PascalCase klasser, _camelCase privata fält
- MVVM strikt: inga kodrader bakom View, all logik i ViewModel
- Asynkront: alltid async/await, aldrig .Result eller .Wait()
- DI: konstruktorinjektion, inga statiska klasser