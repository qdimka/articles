# Как работает конфигурация в .NET Core. Продолжение

Отложим разговоры о производительности и методологиях и вернемся к разговору о конфигурации.
В предыдущей статье мы поговорили о том, что такое конфигурация, как ее использовать, и как определять своих провайдеров конфигурации

В этой статье я хочу затронуть продвинутые сценарии использования, такие как отслеживание изменений, именованную конфигурацию и прочие. Добро пожаловать под кат.

<cut/>

## ```IOptions<T>```

Мы уже познакомились с объектом интерфейса `IConfiguration` и знаем, что конфигурация хранится в виде словаря. 
Как быть, если мы хотим получить параметр из конфигурации в нашем сервисе?

Обычно, решение находится быстро:

```csharp
public class SampleScopedService
{
    private readonly IConfiguration _configuration;
    public SampleScopedService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

	/* ... */
}
```

Что здесь не так?
+ Зависимость от `IConfiguration`
+ `SampleScopedService` должен знать как прочитать параметр из конфигурации (параметром может быть массив, число и т.д)
+ Мы передаем в сервис весь словарь конфигурации, хотя как правило ему нужны только некоторые свойств.

Что можно сделать?

В .NET Core у нас есть возможность использовать строготипизированную объекты конфигурации, которые представлены объектом интерфейса `IOptions<T>`.

```csharp
public interface IOptions<out TOptions>
  where TOptions : class, new()
{
  	TOptions Value { get; }
}
```

Свойство `Value` это класс, описывающий секцию настроек.

Использую данный механизм мы можем изменить код следующим образом:

```csharp
public class SampleOptions
{
    public string Name { get; set; }

    public int Count { get; set; }  
}

public class SampleScopedService
{
    private readonly IOptions<SampleOptions> _options;
    public SampleScopedService(IOptions<SampleOptions> options)
    {
        _options = options;
    }

	/* ... */
}
```

Что изменилось?
+ Зависимость от `IOptions`. Мы перестали зависеть от `IConfiguration`, но получили зависимость от `IOptions`. Я считаю, что это меньшим из двух зол.

+ Так как конфигурация строготипизирована, `SampleScopedService` не нужно знать, как прочитать параметр из конфигурации. Вместо этого мы обращаемся к свойству класса.

+ Мы передаем только те параметры, от которых зависит наш сервис, тем самым соблюдая принцип разделения отвественности.

Все новое, это хорошо ~~забытое~~ переписанное старое.
Подобный функционал можно было использовать и в .Net Framework.

```xml
<?xml version="1.0" encoding="utf-8"?>

<configuration>
   <configSections>
       <section name="customSection"
                type="CustomSection, Samples.CustomSection"/>
   </configSections>
   /* .... */
   <customSection>
       <settings name="Value" />
   </customSection>
</configuration>
```

- Секция `customSection` содержит настройки приложения
- Секция `settings` описывает набор параметров конфигурации




```csharp
public class CustomSection : ConfigurationSection
{
    [ConfigurationProperty("settings", IsRequired = true)]
    public Settings Settings
    {
        get { return (Settings)this["settings"]; }
        set { this["settings"] = value; }
    }
}

public class Settings: ConfigurationElement
{
    [ConfigurationProperty("name", IsRequired = true)]
    public string Name
    {
        get { return (string)this["name"]; }
        set { this["name"] = value; }
    }
}
```
Для того, чтобы получить параметры в виде объекта C#, нам нужно создать наследников для `ConfigurationSection` и `ConfigurationElement`, и реализовать свойства, которые мы сопоставим с секциями xml-файла.

После реализации, мы можем прочитать конфигурацию в объект следующим образом:

```csharp
var section = (CustomSection)ConfigurationManager
 .GetSection( "customSection" );

if ( section != null )
{
   /* .... */
}
```



## ```IOptionsMonitor<T>```

`IOptionsMonitor<T>` позволяет подписаться на изменении конфигурации.

```csharp
public interface IOptionsMonitor<out TOptions>
{
  	TOptions CurrentValue { get; }

  	TOptions Get(string name);

  	IDisposable OnChange(Action<TOptions, string> listener);
}
```
- `CurrentValue` содержит текущее значение
- Метод `Get` позволяет получить экземпля опций, зарегистрированный под определенным именем ( подробнее в конце статьи)
- `OnChange` позволяет подписаться на изменение источника конфигурации.

Объект реализующий этот интерфейс регистрируется как singleton, что дает нам возможность использовать его в своих 



## ```IOptionsSnapshot<T>```

```csharp
public interface IOptionsSnapshot<out TOptions>
  : IOptions<TOptions>
     where TOptions : class, new()
{
  	TOptions Get(string name);
}
```

## ```IConfigure<T>```

```csharp

```

## ```IPostConfigure<T>```

```csharp
public class OptionsConfigurator
    : IPostConfigureOptions<NamedOptions>
{
    public void PostConfigure(string name, NamedOptions options)
    {
        throw new System.NotImplementedException();
    }
}
```

## Валидация параметров конфигурации

```csharp
```