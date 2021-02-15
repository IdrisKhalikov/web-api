# Web API

## Подготовка

Чтобы установить доверия к сертификатам ASP.NET Core, сделай следующее в зависимости от ОС:
- в Windows выполни из под администратора пакетный файл `register-dev-certs.cmd`.
- в Linux/Mac выполни в терминале в склонированной команду `sudo ./register-dev-certs.sh`


## Задача UsersApi

Требуется реализовать методы API для взаимодействия с коллекцией пользователей в классе `UsersController`
в проекте `WebApi`. Эти методы должны позволять создавать, удалять и обновлять пользователей,
получать информацию о имеющихся пользователях.

Для начала надо научиться запускать `WebApi` под отладкой удобным тебе способом в удобной тебе IDE.
Причем это нормально, что при первом запуске API будет отвечать 404 Not Found.
Убедись, что в браузере открывается адрес `https://localhost:5001`.

На все методы написаны модульные тесты в проекте `Tests`. Они позволят проверить, что API написано правильно.
Модульные тесты ищут API под адресу `https://localhost:5001`.
Это настроено в проекте `Tests` в файле `Configuration.cs` в свойстве `BaseUrl`.
Если API почему-то запусается по другому адресу, то для работы тестов надо изменить `BaseUrl`.

Следующий шаг — запустить тесты (под отладкой или без). Проект `WebApi` при этом должен быть запущен под отладкой.
Тут в зависимости от IDE могут быть нюансы, но как минимум Visual Studio, Visual Studio Code и Rider
поддерживают отладку нескольких процессов одновременно. В частности, запуск тестов под отладкой, когда отлаживается `WebApi`.

Если совсем не понятно как запускать тесты, можно это сделать через .NET CLI:
`dotnet test Tests`


### 1. GET /users/{userId}

Реализуй этот метод API для получения пользователя в методе `GetUserById` контроллера.

Для начала тебе потребуется получить `IUserRepository` через конструктор.
Чтобы DI-контейнер ASP.NET смог создать `IUserRepository`,
привяжи к этому интерфейсу `InMemoryUserRepository` в `Startup.ConfigureServices`:
```cs
services.AddSingleton<IUserRepository, InMemoryUserRepository>();
```

Получи нужного пользователя из репозитория и верни его.
Готовый метод `Ok` позволяет вернуть объект и код ответа 200.
```cs
return Ok(user);
```
Подобные методы есть и для других status code, например, `NotFound()` для кода 404.

Правда вернуть надо не объект из репозитория (фактически, базы данных),
а некое ожидаемое представление ресурса — `UserDto`.
`FullName` в этом представлении должен вычисляться как `$"{src.LastName} {src.FirstName}"`.

Теперь должно проходить часть тестов!


Но в каком представлении API будет возвращать ресурсы? Это зависит от заголовка `Accept`.
Сейчас API возвращать данные только формате `JSON`. Надо добавить `XML`.
Все другие представления результатов можно запретить и возвращать 406 Not Acceptable.
Для этого достаточно немного настроить MVC в `Startup.ConfigureServices`:

```cs
services.AddControllers(options =>
{
    // Этот OutputFormatter позволяет возвращать данные в XML, если требуется.
    options.OutputFormatters.Add(new XmlDataContractSerializerOutputFormatter());
    // Эта настройка позволяет отвечать кодом 406 Not Acceptable на запросы неизвестных форматов.
    options.ReturnHttpNotAcceptable = true;
    // Эта настройка приводит к игнорированию заголовка Accept, когда он содержит */*
    // Здесь она нужна, чтобы в этом случае ответ возвращался в формате JSON
    options.RespectBrowserAcceptHeader = true;
})
.ConfigureApiBehaviorOptions(...);
```

а также пометить метод контроллера атрибутом `[Produces("application/json", "application/xml")]`?
который явно указывает в каких форматах могут возвращаться ответы
и что по умолчанию будет использоваться `application/json`.

Теперь должны проходить все тесты на метод API!


Правда, в бочке меда есть ложка дегтя.
`UserEntity` и `UserDto` отличаются друг от друга минимальным образом.
Было бы удобно, если бы код копирования из `Entity` в `DTO` генерировался автоматически,
и только отличия надо было описать вручную.

И средство для этого есть — `AutoMapper`.

Перед тем как использовать, его нужно сконфигурировать и привязать к DI-контейнеру ASP.NET.
Для этого добавь в `Startup.ConfigureServices` следующий код:
```cs
services.AddAutoMapper(cfg =>
{
    // TODO
}, new System.Reflection.Assembly[0]);
```
После этого ты сможешь в конструкторе UserController запросить IMapper и DI-контейнер его передаст.

Но чтобы понять, что написать вместо TODO и понять как пользоваться автомаппером,
изучи тесты в `WebApi/Samples/AutoMapperTests.cs`. Прежде всего `OneTimeSetUp` и тест `TestCreateFrom`.

Теперь напиши конфигурацию AutoMapper для получения `UserDto` из `UserEntity` вместо TODO
и замени код преобразования UserEntity в UserDto на вызов автомаппера.


### 2. POST /users

Реализуй этот метод API для создания нового пользователя в методе `CreateUser` контроллера.

Раз это создания пользователя, то в теле метода должны передаваться необходиме данные для этого:
`string Login`, `string FirstName` и `string LastName`.
Добавь класс для этого DTO. Назови его на свой вкус.

В аргументе `user` метода `CreateUser` замени тип `object` на созданный тобой класс.

Снова используй `AutoMapper`, чтобы создать `UserEntity` по своему DTO.
Обрати внимание, что тебе НЕ нужно задавать идентификатор,
потому что он будет задан при вызове метода `Insert` в `IUserRepository`.

Правильно при создании нового ресурса возвращать информацию о нем:
1) в заголовке `Location` возвращать адрес, по которому можно получить ресурс
2) в теле ответа можно вернуть идентификатор созданного ресурса
Надо сделать и то, и другое! Но сначала подсказка.

Адрес ресурса — это что-то вроде `/api/users/77777777-7777-7777-7777-777777777777`.
Его можно сгенерировать вручную, но есть способ лучше.

Во-первых, задай в атрибуте `HttpGet` метода `GetUserById` параметр `Name`:
```cs
[HttpGet("{userId}", Name = nameof(GetUserById))]
```

Во-вторых, воспользуйся этим именем при возврате результата из `CreateUser`:
```cs
return CreatedAtRoute(
    nameof(GetUserById),
    new { userId = createdUserEntity.Id },
    value);
```
`CreateAtRoute` вовзращает код 201 Created, в тело передает `value`,
а в `Location` записывает адрес метода, имя которого ему передали первым параметром.
Вторым параметром передаются параметры этого метода.

Теперь основной тест должен проходить!


Значит самое время разобраться с различными видами некорректного ввода.
Тут тебе помогут тесты, но важно разобраться с механизмами валидации, встроенными в MVC.

Поле в классе DTO можно пометить атрибутом `Required`. Тогда MVC будет проверять,
что это поле было задано на клиенте еще до вызова метода контроллера.
```cs
[Required]
public string Login { get; set; }
```
При этом не будет возникать исключений. Все ошибки будут складываться в поле `ModelState` контроллера.

Можно узнать, есть ли ошибки, используя `ModelState.IsValid`.
А что делать с ошибками? Информацию о них надо возвращать с кодом 422 Unprocessable Entity.
И есть встроенный метод, сериализующий `ModelState` и возвращающий код 422.
```cs
return UnprocessableEntity(ModelState);
```

В `ModelState` можно также добавлять информацию о любых других ошибках.
Это позволяет делать произвольные логические проверки и возвращать результат этих проверок клиенту.
```cs
ModelState.AddModelError("Ключ, с которым ассоциируется ошибка", "Сообщение об ошибке");
```

Сделай так, чтобы `Login` был обязательным с помощью атрибута `Required`.
Также добавь проверку, что логин состоит только из цифр и букв прямо в методе контроллера с помощью `AddModelError`.
Проверить, что символ является буквой или цифрой, может метод `char.IsLetterOrDigit`. Используй его.
«Ключ, с которым ассоциируется ошибка» для обеих проверок — `Login`.
При обработке атрибута `Required` MVC самостоятельно будет использовать ключ `Login`.
А вот при добавлении через `AddModelError` ключ `Login` придется прописать явно.

Чтобы все работало как надо, поправь настройки `JSONConvert` вот так:
```cs
services.AddControllers(...)
    .ConfigureApiBehaviorOptions(...)
    .AddNewtonsoftJson(options =>
    {
        options.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver();
    });
```

Теперь тесты, связанные с логическими проверками тоже должны проходить!


Осталось сделать так, чтобы если клиент не отправил `FirstName`, то оно задавалось как `"John"`,
а `LastName`, то оно задавалось как `"Doe"`.
Это можно сделать по-разному. Сделай это с помощью атрибута `DefaultValue` так:
```cs
[DefaultValue("John")]
public string FirstName { get; set; }
```

Чтобы это заработало придется еще поднастроить `JSONConvert` вот так:
```cs
services.AddControllers(...)
    .ConfigureApiBehaviorOptions(...)
    .AddNewtonsoftJson(options =>
    {
        options.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver();
        options.SerializerSettings.DefaultValueHandling = DefaultValueHandling.Populate;
    });
```

Теперь должны проходить все тесты на метод API!


### 3. PUT /users/{userId}

Реализуй этот метод API для полного обновления пользователя в методе `UpdateUser` контроллера.


Раз пользователь заменяется полностью, значит можно задавать все те же поля, что и в методе POST.
Вероятно, тебе уже захотелось использовать ранее созданный тобой для POST DTO. Но не торопись.
Часто при обновлении данных предъявляются более жесткие требования. Вот и сейчас так.
Метод должен возвращать код 422, если не заданы `FirstName` или `LastName`. И не должен задавать их по-умолчанию.

Требования к логину не поменялись, поэтому проверки можно скопировать, но не стоит.
Проверить, что логин состоит из букв и цифр, можно с помощью атрибута. Так и сделай:
```cs
[Required]
[RegularExpression("^[0-9\\p{L}]*$", ErrorMessage = "Login should contain only letters or digits")]
public string Login { get; set; }
```

Важно, что метод PUT может работать по-разному.

Первый вариант: если ресурс существует, то он полностью обновляется,
а если ресурса нет, то возвращается код 404 Not Found.
Назовем этот вариант `Update`.

Второй вариант: если ресурс существует, то он полностью обновляется,
а если ресурса нет, то он создается с использованием переданных данных и с переданным id.
Назовем этот вариант `Upsert`.

Вариант `Upsert` интересен тем, что позволяет создавать ресурсы «безопасно».
Например, если из-за проблем сети пришлось повторить запрос на создание некоторого ресурса,
то в случае POST может быть создано два одинаковых ресурса с разными идентификаторами,
а в случае PUT даже при выполнении повторного запроса будет создан один ресурс.


Реализуй PUT в варианте `Upsert` и пройди все тесты!


Подсказка 1: Для вставки новой сущности в репозиторий используй метод `UpdateOrInsert`,
потому что, в отличие от `Insert`, он использует переданный id сущности, а не задает новый.

Подсказка 2: Посмотри тесты `TestFillBy` и `TestFillByReturnSyntax` из `WebApi/Samples/AutoMapperTests.cs`.


### 4. PATCH /users/{userId}

Реализуй этот метод API для частичного обновления пользователя в методе `PartiallyUpdateUser` контроллера.

При частичном обновлении ресурса надо уметь указывать, какие части ресурса обновлять надо, а какие нет.
Таким образом требуется некий формат описания таких частичных изменений.

Один из возможных форматов — [JSON Patch](https://tools.ietf.org/html/rfc6902).
Вот пример использования:
```json
[
    { "op": "test", "path": "/a/b/c", "value": "foo" },
    { "op": "remove", "path": "/a/b/c" },
    { "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
    { "op": "replace", "path": "/a/b/c", "value": 42 },
    { "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
    { "op": "copy", "from": "/a/b/d", "path": "/a/b/e" }
]
```

Поддержать такой формат в ASP.NET Core довольно просто, потому что есть встроенный класс `JsonPatchDocument<T>`.

Если полный DTO для обновления (как в PUT) называется `UpdateDto`, то из тела запроса можно достать данные так:
```cs
[FromBody] JsonPatchDocument<UpdateDto> patchDoc
```

А затем применить изменения, описанные JSON-Patch так:
```cs
patchDoc.ApplyTo(updateDto, ModelState);
```

К входным параметрам есть требования: обязательные поля, формат значения.
Значит надо провалидировать получившийся объект:
```cs
// Валидация по атрибутам
TryValidateModel(user);
// Другие валидации...
```

В остальном PATCH похож на PUT.
Реализуй его в варианте `Update`, то есть метод не должен создавать ресурсы, только обновлять.
Хотя, в зависимости от задумки автора API, метод PATCH может быть реализован в варианте `Upsert`.


### 5. DELETE /users/{userId}

Реализуй этот метод API для удаления пользователя в методе `DeleteUser` контроллера.

Метод DELETE в случае успеха обычно возвращает 204 No Content, потому что возвращать нечего.

В остальном тут все должно быть интуитивно понятно. Тем более, что у тебя есть тесты.


### 6. HEAD /users/{userId}

HEAD — это то же самое, что GET, только без тела.
Он нужен, чтобы проверять наличие объекта, но не сам объект.

И его очень легко реализовать в ASP.NET Core.

Просто добавь атрибут `[HttpHead("{userId}")]` к `GetUserById` и проверь, что тесты проходят.


### 7. GET /users

Реализуй этот метод API для получения всех пользователей в методе `GetUsers` контроллера.


Получить из репозитория и вернуть всех пользователей несложно:
```cs
var users = Mapper.Map<IEnumerable<UserDto>>(userEntities);
return Ok(users);
```
Даже не придется дополнительно конфигурировать `AutoMapper`.


Только так делать нельзя: пользователей обычно много, а значит результат надо возвращать постранично.

Репозиторий как раз возвращает пользователей постранично, поэтому:
```cs
var pageList = userRepository.GetPage(pageNumber, pageSize);
var users = Mapper.Map<IEnumerable<UserDto>>(pageList);
return Ok(users);
```

Правда потребуются дополнительные параметры `pageNumber` и `pageSize`.
Query string подходит для передачи таких параметров.

Пусть параметры будут ограничены так:

| Параметр   	| min 	| max 	| default 	|
|------------	|-----	|-----	|---------	|
| pageNumber 	| 1   	| ∞   	| 1       	|
| pageSize   	| 1   	| 20  	| 10      	|


Уже можно написать реализацию метода.


Но хороший метод с постраничным разделением должен предоставлять информацию о количестве страниц,
чтобы клиент знал, все ли объекты получены, надо ли запрашивать что-то еще.

Можно это сделать с помощью заголовка. Например, `X-Pagination`. И передавать в нем JSON с информацией о странице.
Добавить заголовок можно так:
```cs
var paginationHeader = new
{
    previousPageLink = ...
    nextPageLink = ...
    totalCount = ...,
    pageSize = ...,
    currentPage = ...,
    totalPages = ...,
};
Response.Headers.Add("X-Pagination", JsonConvert.SerializeObject(paginationHeader));
```

Чтобы построить `previousPageLink` и `nextPageLink` можно воспользоваться `LinkGenerator`.
Его можно получить через конструктор (он сконфигурирован в MVC).
И использовать так:
```cs
linkGenerator.GetUriByRouteValues(HttpContext, "Имя метода из атрибута", new {a: 5, b: 2});
```


Допиши реализацию метода с заголовком `X-Pagination`.


### 8. OPTIONS /users

Реализуй этот метод API для получения списка доступных методов в контексте всех пользователей.

Этот метод должен возвращать 200 OK с пустым телом (но не 204 No Content).
Опций может быть много. Требуется добавить заголовок `Allow` и перечислить доступные методы через запятую.

Добавить заголовок можно так:
```cs
Response.Headers.Add("HeaderName", "HeaderValue");
```