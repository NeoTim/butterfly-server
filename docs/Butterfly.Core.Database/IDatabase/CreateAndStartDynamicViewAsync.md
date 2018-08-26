# IDatabase.CreateAndStartDynamicViewAsync method (1 of 2)

```csharp
public Task<DynamicViewSet> CreateAndStartDynamicViewAsync(string sql, 
    Action<DataEventTransaction> listener, object values = null, string name = null, 
    string[] keyFieldNames = null)
```

## See Also

* class [DynamicViewSet](../../Butterfly.Core.Database.Dynamic/DynamicViewSet.md)
* class [DataEventTransaction](../../Butterfly.Core.Database.Event/DataEventTransaction.md)
* interface [IDatabase](../IDatabase.md)
* namespace [Butterfly.Core.Database](../../Butterfly.Core.md)

---

# IDatabase.CreateAndStartDynamicViewAsync method (2 of 2)

```csharp
public Task<DynamicViewSet> CreateAndStartDynamicViewAsync(string sql, 
    Func<DataEventTransaction, Task> listener, object values = null, string name = null, 
    string[] keyFieldNames = null)
```

## See Also

* class [DynamicViewSet](../../Butterfly.Core.Database.Dynamic/DynamicViewSet.md)
* class [DataEventTransaction](../../Butterfly.Core.Database.Event/DataEventTransaction.md)
* interface [IDatabase](../IDatabase.md)
* namespace [Butterfly.Core.Database](../../Butterfly.Core.md)

<!-- DO NOT EDIT: generated by xmldocmd for Butterfly.Core.dll -->