#  AttemptController.cs 

# Контроллер - это мост. Его задача - это получить данные от пользователя и передать в сервис. сам он лишь посредник. АНалогия - менеджер на рэссепшн. Встретил клиента - отвел к специалисту

### Dependency Injection

```csharp
private readonly IAttemptsService _attemptsService;

public AttemptController(IAttemptsService attemptsService)
{
    _attemptsService = attemptsService;
}
```

# Контроллер сам не создает сервис, а получает его "снаружи" через конструктор. Это DI (внедрение зависимостей. Это надо для того, чтоб не привязываться к конкретной реализации и иметь возможность подмен на пробных тестах.
# private readonly - поле нельзя уже перезаписать после иницилизации 

## Ссылки

- [Interfaces в C#](https://www.geeksforgeeks.org/c-sharp/c-sharp-interface/)
- [MVC архитектура](https://www.geeksforgeeks.org/system-design/mvc-architecture-system-design/)
- [Dependency Injection](https://stackify.com/dependency-injection-c-sharp/)
- [Атрибуты и фильтры ASP.NET](https://scholarhat.com/tutorial/mvc/understanding-aspnet-mvc-filters-and-attributes)
- [DTO в C#](https://medium.com/@20011002nimeth/understanding-data-transfer-objects-dtos-in-c-net-best-practices-examples-fe3e90238359)
