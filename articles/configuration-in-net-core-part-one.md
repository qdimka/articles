# Как работает конфигурация в .NET Core

Давайте отложим разговоры о DDD и рефлексии на время. Предлагаю поговорить о простом, об организации настроек приложения.

После того как мы с коллегами решили перейти на .NET Core, возник вопрос, как организовать файлы конфигурации, как выполнять трансформации и пр. в новой среде. Во многих примерах встречается следующий код, и многие его успешно используют.

```cs
public IConfiguration Configuration { get; set; }
public IHostingEnvironment Environment { get; set; }

public Startup(IConfiguration configuration, IHostingEnvironment environment)
{
   Environment = environment;
   Configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .AddJsonFile($"appsettings.{Environment.EnvironmentName}.json")
            .Build();
}
```

Но давайте разберемся, как работает конфигурация, и в каких случаях использовать данный подход, а в каких довериться разработчикам .NET Core. Прошу под кат.
<cut/>

## Как было раньше

Как и у любой истории, у этой статьи есть начало. Одним из первых вопросов после перехода на ASP.NET Core были трансформации конфигурационных файлов.
<spoiler title="Вспомним как это было ранее c web.config">
![](https://habrastorage.org/webt/28/rh/tx/28rhtxxm6uw-67utmgjas0oh0ii.png)

Конфигурация состояла из нескольких файлов. Основным был файл <i>web.config</i>, и к нему уже применялись трансформации (*web.Development.config* и др.) в зависимости от конфигурации сборки. При этом активно использовались *xml*-атрибуты для поиска и трансформации секции *xml*-документа.
</spoiler>

Но как мы знаем в ASP.NET Core файл *web.config* заменен на *appsettings.json* и привычного механизма трансформаций больше нет.

<spoiler title="Что нам говорит google?">

Результатом поиска " Трансформации в ASP.NET Core " в *google* стал следующий код:
```cs
public IConfiguration Configuration { get; set; }
public IHostingEnvironment Environment { get; set; }

public Startup(IConfiguration configuration, IHostingEnvironment environment)
{
   Environment = environment;
   Configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .AddJsonFile($"appsettings.{Environment.EnvironmentName}.json")
            .Build();
}
```
В конструкторе класса *Startup* мы создаем объект конфигурации с помощью *ConfigurationBuilder*. При этом мы явно указываем какие источники конфигурации мы хотим использовать.

И такой:
```cs
public IConfiguration Configuration { get; set; }
public IHostingEnvironment Environment { get; set; }

public Startup(IConfiguration configuration, IHostingEnvironment environment)
{
   Environment = environment;
   Configuration = new ConfigurationBuilder()
            .AddJsonFile($"appsettings.{Environment.EnvironmentName}.json")
            .Build();
}
```
В зависимости от переменной окружения выбирается тот или иной источник конфигурации.
</spoiler>
Данные ответы часто встречаются на <abbr title="StackOverflow">SO</abbr> и других менее популярных ресурсах. Но не покидало ощущение. что мы идем не туда. Как быть если я хочу использовать переменные окружения или аргументы командной строки в конфигурации? Почему мне нужно писать этот код в каждом проекте?

В поисках истины пришлось забраться вглубь документации и исходного кода. И я хочу поделиться полученным знанием в данной статье.

Давайте разберемся, как работает конфигурация в .NET Core.

## Конфигурация

Конфигурация в .NET Core представлена объектом интерфейса *IConfiguration*.

```cs
public interface IConfiguration
{
   string this[string key] { get; set; }

   IConfigurationSection GetSection(string key);

   IEnumerable<IConfigurationSection> GetChildren();

   IChangeToken GetReloadToken();
}
```

- **[string key]** индексатор, который позволяет по ключу получить значение параметра конфигурации
- **GetSection(string key)** возвращает секцию конфигурации, которая соответствует ключу *key*
- **GetChildren()** возвращает набор подсекций текущей секции конфигурации
- **GetReloadToken()** возвращает экземпляр *IChangeToken*, который можно использовать для получения уведомлений при изменении конфигурации

Конфигурация представляет собой набор пар "ключ-значение". При чтении из источника конфигурации (файл, переменные окружения) иерархические данные приводятся к плоской структуре. Например *json*-объект вида
```js
{
 "Settings": {
  "Key": "I am options"
 }
}
```
будет приведен к плоскому виду:
```
Settings:Key = I am options
```
Здесь ключом является *Settings:Key*, а значением *I am options*.
Для наполнения конфигурации используются провайдеры конфигурации.

## Провайдеры конфигурации
За чтение данных из источника конфигурации отвечает объект интерфейса
*IConfigurationProvider*:
```cs
public interface IConfigurationProvider
{
   bool TryGet(string key, out string value);

   void Set(string key, string value);

   IChangeToken GetReloadToken();

   void Load();

   IEnumerable<string> GetChildKeys(IEnumerable<string> earlierKeys, string parentPath);
}
```
- **TryGet(string key, out string value)** позволяет по ключу получить значение параметра конфигурации
- **Set(string key, string value)** используется для установки значения параметра конфигурации
- **GetReloadToken()** возвращает экземпляр *IChangeToken*, который можно использовать для получения уведомлений при изменении источника конфигурации
- **Load()** метод который отвечает за чтение источника конфигурации
- **GetChildKeys(IEnumerable&lt;string&gt; earlierKeys, string parentPath)** позволяет получить список всех ключей, которые предоставляет данный поставщик конфигурации

Из коробки доступны следующие провайдеры:
- Json
- Ini
- Xml
- Environment Variables
- InMemory
- Azure
- Кастомный провайдер конфигурации

Приняты следующие соглашения использования провайдеров конфигурации.
1. Источники конфигурации считываются в том порядке, в котором они были указаны
2. Если в разных источниках конфигурации присутствуют одинаковые ключи (сравнение идет без учета регистра), то используется значение, которое было добавлено последним.

Если мы создаем экземпляр *web*-сервера используя *CreateDefaultBuilder*, то по умолчанию подключаются следующие провайдеры конфигурации:

![](https://habrastorage.org/webt/pq/0l/5q/pq0l5qp3qultg9xhtg1kvpimj_y.png)

- **ChainedConfigurationProvider** через этот провайдер можно получать значения и ключи конфигурации, которые были добавлены другими провайдерами конфигурации
- **JsonConfigurationProvider** использует в качестве источника конфигурации *json*-файлы. Как можно заметить, в список провайдеров добавлены три провайдера данного типа. Первый использует в качестве источника *appsettings.json*, второй *appsettings.{environment}.json*. Третий считывает данные из *secrets.json*. Если выполнить сборку приложения в конфигурации *Release*, третий провайдер не будет подключен, потому что не рекомендуется использовать секреты в *Production*-среде
- **EnvironmentVariablesConfigurationProvider** получает параметры конфигурации из переменных окружения
- **CommandLineConfigurationProvider** позволяет добавлять аргументы командой строки в конфигурацию

Так как конфигурация хранится как словарь, то необходимо обеспечить уникальность ключей. По умолчанию это работает так.

Если в провайдере *CommandLineConfigurationProvider* имеется элемент с ключом key и в провайдере *JsonConfigurationProvider* имеется элемент с ключом key, элемент из *JsonConfigurationProvider* будет заменен элементом из *CommandLineConfigurationProvider* так как он регистрируется последним и имеет больший приоритет.

<spoiler title="Вспомним пример из начала статьи">

```cs
public IConfiguration Configuration { get; set; }
public IHostingEnvironment Environment { get; set; }

public Startup(IConfiguration configuration, IHostingEnvironment environment)
{
   Environment = environment;
   Configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .AddJsonFile($"appsettings.{Environment.EnvironmentName}.json")
            .Build();
}
```

Нам не нужно самим создавать *IConfiguration*, чтобы выполнить трансформацию файлов конфигурации, так как это включено по умолчанию. Данный подход необходим в том случае, когда мы хотим ограничить количество источников конфигурации. 

</spoiler>
 
## Кастомный провайдер конфигурации

Для того, чтобы написать свой поставщик конфигурации необходимо реализовать интерфейсы *IConfigurationProvider* и *IConfigurationSource*. *IConfigurationSource* новый интерфейс, который мы еще не рассматривали в данной статье.

```cs

public interface IConfigurationSource
{
    IConfigurationProvider Build(IConfigurationBuilder builder);
}

```

Интерфейс состоит из единственного метода *Build*, который принимает в качестве параметра *IConfigurationBuilder* и возвращает новый экземпляр *IConfigurationProvider*.

Для реализации своих поставщиков конфигурации нам доступны абстрактные классы *ConfigurationProvider* и *FileConfigurationProvider*. В этих классах уже реализована логика методов *TryGet*, *Set*, *GetReloadToken*, *GetChildKeys* и остается реализовать только метод *Load*.

Рассмотрим на примере. Необходимо реализовать чтение конфигурации из *yaml*-файла, при этом также необходимо чтобы мы могли изменять конфигурацию без перезагрузки нашего приложения.

Создадим класс *YamlConfigurationProvider* и сделаем его наследником *FileConfigurationProvider*.

```cs
public class YamlConfigurationProvider : FileConfigurationProvider
{
    private readonly string _filePath;

    public YamlConfigurationProvider(FileConfigurationSource source) 
        : base(source)
    {
    }

    public override void Load(Stream stream)
    {
        throw new NotImplementedException();
    }
}
```

В приведенном фрагменте кода можно заметить некоторые особенности класса *FileConfigurationProvider*. Конструктор принимает экземпляр *FileConfigurationSource*, который содержит в себе *IFileProvider*. *IFileProvider* используется для чтения файла, и для подписки на событие изменения файла. Также можно заметить, что метод *Load* принимает *Stream* в котором открыт для чтения файл конфигурации. Это метод класса *FileConfigurationProvider* и его нет в интерфейсе *IConfigurationProvider*.

Добавим простую реализацию, которая позволит считать *yaml*-файл. Для чтения файла я воспользуюсь пакетом *YamlDotNet*.

<spoiler title="Реализация YamlConfigurationProvider">

```cs
 public class YamlConfigurationProvider : FileConfigurationProvider
{
    private readonly string _filePath;

    public YamlConfigurationProvider(FileConfigurationSource source) 
        : base(source)
    {
    }

    public override void Load(Stream stream)
    {
        if (stream.CanSeek)
        {
            stream.Seek(0L, SeekOrigin.Begin);
            using (StreamReader streamReader = new StreamReader(stream))
            {
                var fileContent = streamReader.ReadToEnd();
                var yamlObject = new DeserializerBuilder()
                    .Build()
                    .Deserialize(new StringReader(fileContent)) as IDictionary<object, object>;

                Data = new Dictionary<string, string>();

                foreach (var pair in yamlObject)
                {
                    FillData(String.Empty, pair);
                }
            }
        }
    }


    private void FillData(string prefix, KeyValuePair<object, object> pair)
    {
        var key = String.IsNullOrEmpty(prefix)
            ? pair.Key.ToString() 
            : $"{prefix}:{pair.Key}";

        switch (pair.Value)
        {
            case string value:
                Data.Add(key, value);
                break;

            case IDictionary<object, object> section:
            {
                foreach (var sectionPair in section)
                    FillData(pair.Key.ToString(), sectionPair);

                break;
            }
        }
    }
}
```

</spoiler>

Для создания экземпляра нашего провайдера конфигурации необходимо реализовать *FileConfigurationSource*.

<spoiler title="Реализация YamlConfigurationSource ">

```cs
public class YamlConfigurationSource : FileConfigurationSource
{
    public YamlConfigurationSource(string fileName)
    {
        Path = fileName;
        ReloadOnChange = true;
    }

    public override IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        this.EnsureDefaults(builder);
        return new YamlConfigurationProvider(this);
    }
}
```

Тут важно отметить, что для инициализации свойств базового класса необходимо вызвать метод *this.EnsureDefaults(builder)*.
</spoiler>

Для регистрации кастомного провайдера конфигурации в приложении необходимо добавить экземпляр провайдера в *IConfigurationBuilder*. Можно вызвать метод *Add* из *IConfigurationBuilder*, но я сразу вынесу логику инициализации *YamlConfigurationProvider* в *extension*-метод.
<spoiler title="Реализация YamlConfigurationExtensions">

```cs
public static class YamlConfigurationExtensions
{
    public static IConfigurationBuilder AddYaml(
        this IConfigurationBuilder builder, string filePath)
    {
        if (builder == null)
            throw new ArgumentNullException(nameof(builder));

        if (string.IsNullOrEmpty(filePath))
            throw new ArgumentNullException(nameof(filePath));

        return builder
            .Add(new YamlConfigurationSource(filePath));
    }
}
```

</spoiler>

<spoiler title="Вызов метода AddYaml">

```cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, builder) =>
            {
                builder.AddYaml("appsettings.yaml");
            })
            .UseStartup<Startup>();
}
```

</spoiler>

## Отслеживание изменений

В новом *api*-конфигурации появилась возможность перечитывать источник конфигурации при его изменении. При этом не происходит перезапуска приложения.
Как это работает:
- Поставщик конфигурации отслеживает изменение источника конфигурации
- Если произошло изменение конфигурации, создается новый *IChangeToken*
- При изменении *IChangeToken* вызывается перезагрузка конфигурации

Посмотрим как реализовано отслеживание изменений в *FileConfigurationProvider*.

```cs
ChangeToken.OnChange(
    //producer
    () => Source.FileProvider.Watch(Source.Path), 
    //consumer
    () => {                         
        Thread.Sleep(Source.ReloadDelay);
        Load(reload: true);
    });
```

В метод *OnChange* статического класса *ChangeToken* передается два параметра. Первый параметр это функция которая возвращает новый *IChangeToken* при изменении источника конфигурации (в данном случае файла), это т.н *producer*. Вторым параметром идет функция-*callback* (или *consumer*), которая будет вызвана при изменении источника конфигурации.
Подробнее о классе [ChangeToken](https://docs.microsoft.com/ru-ru/aspnet/core/fundamentals/change-tokens?view=aspnetcore-2.2).

Не все провайдеры конфигурации реализуют отслеживание изменений. Этот механизм доступен для потомков *FileConfigurationProvider* и *AzureKeyVaultConfigurationProvider*.

## Заключение

В .NET Core у нас появился легкий, удобный механизм для управления настройками приложения. Много надстроек доступно из коробки, многие вещи используются по умолчанию. 
Конечно каждый сам решает, какой способ ему использовать, но я за то, чтобы люди знали свои инструменты. 

Данная статья затрагивает лишь основы. Помимо основ нам доступны IOptions, сценарии пост-конфигурации, валидация настроек и многое другое. Но это уже другая история. 

Проект приложения с примерами из данной статьи вы можете найти в репозитории на [Github](https://github.com/qdimka/netcore-configuration-sample).
Делитесь в комментариях, кто какие подходы по организации конфигурации использует?
Спасибо за внимание.

**upd**.: Как верно подсказал @AdAbsurdum, в случае работы с массивами не всегда будет происходит замена элементов при слиянии конфигурации из двух источников.
Рассмотрим пример. При чтении массива из *appsettings.json* получим такой плоский вид:
```
array:0=valueA
```
При чтении из *appsettings.Development.json*:
```
array:0=valueB
array:1=value
```
В итоге в конфигурации будет:
```
array:0=valueB
array:1=value
```
Все элементы с уникальными индексами (*array:1* в примере) будут добавлены в итоговый массив. Элементы из разных источников конфигурации, но имеющие одинаковый индекс (*array:0* в примере) подвергнутся слиянию, и будет использован элемент, который был добавлен последним.