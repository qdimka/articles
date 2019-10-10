# Как работает конфигурация в .NET Core. Продолжение

Отложим разговоры о производительности и методологиях и вернемся к разговору о конфигурации.
В предыдущей статье мы поговорили о том, что такое конфигурация, как ее использовать, и как определять провайдеров конфигурации.


В этой статье я хочу затронуть продвинутые сценарии использования, такие как отслеживание изменений, именованные параметры конфигурации и другие возможности. Добро пожаловать под кат.


<cut/>


## ```IOptions<T>```


Мы уже знакомы с объектом интерфейса `IConfiguration` и знаем, что конфигурация хранится в виде словаря. 
Как быть, если мы хотим получить параметр из конфигурации в нашем сервисе?


Вот так выглядит решение на скорую руку:


```csharp
public class SampleScopedService
{
    private readonly IConfiguration _configuration;
    public SampleScopedService(IConfiguration configuration)
    {
        _configuration = configuration;
    }


    /* ... */
}
```

+ Зависимость от `IConfiguration`
+ `SampleScopedService` должен знать как прочитать параметр из конфигурации (параметром может быть массив, число и т.д)
+ В сервис передается полный словарь конфигурации, хотя как правило используются только некоторые свойств.


Что можно сделать?


В .NET Core у нас есть возможность использовать строготипизированные объекты конфигурации, которые представлены объектом интерфейса `IOptions<T>`.

```csharp
public interface IOptions<out TOptions>
  where TOptions : class, new()
{
    TOptions Value { get; }
}
```

Интерфейс содержит единственное свойство `Value` - класс, описывающий секцию настроек.

Использую данный механизм можно изменить код следующим образом.

Добавим объект в котором будут храниться настройки сервиса.


```csharp
public class SampleOptions
{
    public string Name { get; set; }


    public int Count { get; set; }  
}
```


Зарегистрируем опции в `Startup`.


```csharp
public void ConfigureServices(IServiceCollection services)
{
     /*...*/

    services.Configure<SampleOptions>("SampleOptionsInstance", 
        Configuration.GetSection("SampleOptions"));
    /*...*/
}
```


Выбранная перегрузка метода `Configure<SampleOptions>` принимает название экземпляра опций и объект интерфейса `IConfigurationSection`.


Первый аргумент необязателен и необходим нам в том случае, если мы хотим зарегистрировать несколько экземпляров `SampleOptions`, которые будут хранить в себе разные параметры конфигурации. Указание имени позволит запрашивать конкретный экземпляр объекта.


Второй аргумент это секция конфигурации. Например, если мы читаем конфигурацию из json-файла, то `SampleOptions` будет выглядеть следующим образом:


```json
{


  "SampleOptions": {
    "Name": "I am appsettings",
    "Count": 1
  }
}
```

Затем мы изменяем `SampleScopedService`, конструктор которого теперь принимает `IOptions<SampleOptions>`.


```csharp
public class SampleScopedService
{
    private readonly IOptions<SampleOptions> _options;
    public SampleScopedService(IOptions<SampleOptions> options)
    {
        _options = options;
    }

    /* ... */
}
```

+ Мы перестали зависеть от `IConfiguration`, но получили зависимость от `IOptions`. Я считаю, что это меньшим из двух зол.


+ Так как конфигурация строготипизирована, `SampleScopedService` не нужно знать, как прочитать параметр из конфигурации. Вместо этого мы обращаемся к свойству класса.


+ Мы передаем только те параметры, от которых зависит наш сервис, тем самым соблюдая принцип разделения отвественности.


Все новое, это хорошо ~~забытое~~ пересмотренное старое.
Подобный функционал можно было использовать в .Net Framework.

`IOptions<SampleOptions>` регистрируется как `Singleton`.
<spoiler>
```xml
<?xml version="1.0" encoding="utf-8"?>

<configuration>
   <configSections>
       <section name="customSection"
                type="CustomSection, Samples.CustomSection"/>
   </configSections>
   /* .... */
   <customSection>
       <settings name="Value" />
   </customSection>
</configuration>
```
В элементе `configSections` мы объявляем секцию в которой будут храниться параметры конфигурации (аттрибут `name`), а также указываем с каким типом она будет сопоставлена (аттрибут `type`).


Затем объявляем секцию `customSection` и помещаем в нее элемент `settings`, который описывает набор параметров конфигурации

Затем нам необходимо настроить чтение этого элемента в класс C#.

```csharp
public class CustomSection : ConfigurationSection
{
    [ConfigurationProperty("settings", IsRequired = true)]
    public Settings Settings
    {
        get { return (Settings)this["settings"]; }
        set { this["settings"] = value; }
    }
}


public class Settings: ConfigurationElement
{
    [ConfigurationProperty("name", IsRequired = true)]
    public string Name
    {
        get { return (string)this["name"]; }
        set { this["name"] = value; }
    }
}
```
Для того, чтобы получить параметры в виде объекта, нам нужно создать наследников для `ConfigurationSection` и `ConfigurationElement`, и реализовать свойства, которые мы сопоставим с секциями xml-файла.


После реализации, мы можем прочитать конфигурацию в объект следующим образом:


```csharp
var section = (CustomSection)ConfigurationManager
 .GetSection( "customSection" );


if ( section != null )
{
   /* .... */
}
```
</spoiler>

## ```IOptionsMonitor<T>```

```csharp
public interface IOptionsMonitor<out TOptions>
{
    TOptions CurrentValue { get; }


    TOptions Get(string name);


    IDisposable OnChange(Action<TOptions, string> listener);
}
```
+ Свойство `CurrentValue` содержит текущее значение экземпляра опций
+ Метод `Get` позволяет получить именованный экземпляр опций
+ `OnChange` отвечает за подписку на изменение источника конфигурации

`IOptionsMonitor<T>` стоит использовать в случаях, когда мы хотим реагировать на изменение источника конфигурации или использовать именованные опции.
Использование данного сервиса, позволяет нам изменять конфигурацию во время работы приложения, без перезагрузки приложения.

Для регистрации, нам не нужно выполнять дополнительных действий, достаточно регистрации опций из примера выше.

```csharp
public void ConfigureServices(IServiceCollection services)
{
     /*...*/

    services.Configure<SampleOptions>("SampleOptionsInstance", 
        Configuration.GetSection("SampleOptions"));
    /*...*/
}
```

Затем передаем `IOptionsMonitor<SettingsOptions>` в конструктор `SampleSingletonService`.

```csharp
public class SampleSingletonService
{
    public SampleSingletonService(
        IOptionsMonitor<SettingsOptions> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
    }

    /*...*/
}
```


Объект реализующий этот интерфейс регистрируется как `Singleton`.




## ```IOptionsSnapshot<T>```


```csharp
public interface IOptionsSnapshot<out TOptions>
  : IOptions<TOptions>
     where TOptions : class, new()
{
    TOptions Get(string name);
}
```


## ```IConfigure<T>```


```csharp


```


## ```IPostConfigure<T>```


```csharp
public class OptionsConfigurator
    : IPostConfigureOptions<NamedOptions>
{
    public void PostConfigure(string name, NamedOptions options)
    {
        throw new System.NotImplementedException();
    }
}
```


## Валидация параметров конфигурации


```csharp
```