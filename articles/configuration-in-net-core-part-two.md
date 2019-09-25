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

Что не так?
+ Зависимость от `IConfiguration`
+ `SampleScopedService` должен знать как прочитать параметр из конфигурации
+ Мы передаем в сервис весь словарь конфигурации, хотя как правило ему нужны только несколько свойств.

Что можно сделать?

В .NET Core у нас есть возможность использовать строготипизированную конфигурацию, которая представлена объектом интерфейса `IOptions<T>`.

```csharp
public interface IOptions<out TOptions>
  where TOptions : class, new()
{
  	TOptions Value { get; }
}
```

Свойство `Value` это определенный класс, описывающий секцию настроек.
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
1. Зависимость от `IOptions`. Да, мы перестали зависеть от `IConfiguration`, но получили зависимость от `IOptions`. Я считаю, что это меньшее из двух зол.

2. Так как конфигурация строготипизирована, `SampleScopedService` не нужно знать, как прочитать параметр из конфигурации. Вместо этого мы можем обращаться к свойству класса.

3. Мы передаем только те параметры, от которых зависит наш сервис.

На самом деле это не новое поведение. 

Строготипизированную конфигурацию можно было использовать и в .Net Framework.

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
```

```csharp
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

```csharp
var section = (CustomSection)ConfigurationManager
 .GetSection( "customSection" );

if ( section != null )
{
   /* .... */
}
```

## ```IOptionsMonitor<T>```

```csharp
public interface IOptionsMonitor<out TOptions>
{
  	TOptions CurrentValue { get; }

  	TOptions Get(string name);

  	IDisposable OnChange(Action<TOptions, string> listener);
}
```

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