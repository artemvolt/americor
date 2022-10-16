### Введение
Выполняя тестовое задание я исходил из того, что нам нравится yii2 и у нас нет потребности в полной изоляции компонентов. \
Мы хотим использовать под разные потребности свои инструменты. \
Исходя из своего опыта, я перечислил замечания по коду, которые можно улучшить, что в долгосрочной перспективе позволит нам проще управлять сложностью кода.

### 1. Тестирование
Я бы добавил тест на текущий функционал, хотя бы общий тест, когда мы кидаем запрос на урл и получаем файл или смотрим грид (codeception, suite functional).\
Это позволило бы нам провести рефакторинг, например, по комментариям ниже и убедиться, что система работает. \
После, добавить unit тесты и, в зависимости, от критичности функционала - добавить приемочные.

### 2. Модель HistorySearch
Если отталкиваться от того, что модель History - это модель данных бд, то HistorySearch - это коллекция этих моделей. \
По моему мнению, я бы не наследовал HistorySearch от History, наследовал от \yii\base\Model. \
Это будет отдельная форма для поиска по истории и она будет делать свой поиск на отлично, а модель History будет отображать строку из бд и ее связи. \
Так же уйдя от наследования, мы разгрузим форму поиска от ненужных наследуемых методов самой модели. Мы снижаем вероятность создания жёсткосвязанной системы.\ С ростом системы будет тяжело понять какие методы унаследовались и перекрыли друг друга. 
С этим подходом есть проблемы. \
Давайте представим, что нам нужно кешировать запрос, как мы могли бы это сделать:

```php
use app\models\History;
use yii\helpers\ArrayHelper;

class HistorySearch  extends \yii\base\Model {
    public function search($params)
    { 
        // Добавим кеширование в простом варианте через параметр
        $query = History::find();
        $cache = ArrayHelper::getValue($params, 'cache');
        if ([] !== $cache) {
            $query->cache($cache['duration'], $cache['dependency'] ?? null)
        }
        //..
    }
}
```

Предположим, теперь мы получаем историю из другого бд-сервера, как мы могли бы заменить коннект?\
Например, так же добавить в params:

```php
use yii\data\ActiveDataProvider;

class HistorySearch  extends \yii\base\Model {
    public function search($params)
    { 
        //...
        $dataProvider = new ActiveDataProvider([
            'query' => $query
        ]);
        $db = $params['db'] ?? null;
        if ($db) {
            $dataProvider->db = $db;
        }
        //..
    }
}
```

А если, мы, например, данные для разработчиков выводим из бд, а для высоконагруженной части через elastic? Как бы поменялся наш код? Скорей всего, мы могли бы добавить if на условие либо параметр в метод search.

```php
use yii\data\ActiveDataProvider;
use \Webmozart\Assert\Assert;

class HistorySearch  extends \yii\base\Model {
    public function search($params)
    { 
        //...
        $place = $params['place'] ?? null;
        Assert::oneOf($place, ['web', 'admin']);
        if ($place === 'web') {
            //...случай c data provider
        } elseif($place === 'admin') {
            //...случай с elastic search
        }
        //...
    }
}
```

Довольно тяжеловато получается для одного метода, мы могли бы внутри как-нибудь разнести поведение через интерфейс, разбить на классы, но, исходя из моего опыта, я бы предложил посмотреть на это с другой стороны.

Я использовал для работы с выборками репозитории (хранилища). Пусть будет у нас такие классы:
- stores 
  - HistoryStore
  - db
    - HistoryStore
  - cache
    - HistoryStore

```php
namespace stores;

use yii\data\DataProviderInterface;

interface HistoryStore {
    /**
      * Грид поиска истории  
      * 
      * @param array $params
      * @return DataProviderInterface
      */
    public function search(array $params):DataProviderInterface
}
```

```php
namespace stores\db;

use app\models\History;
use yii\data\DataProviderInterface;
use stores\HistoryStore as IHistoryStore;
use yii\db\Connection;

class HistoryStore implements IHistoryStore {
    /**
     * @var Connection
     */
    private  $connection;
 
    public function __construct(Connection $connection) {
        $this->connection = $connection;
    }
    
    /**
      * @inheritDoc
      */
    public function search(array $params):DataProviderInterface {
        //... Наш предыдущий код
        return new ActiveDataProvider([
            'query' => History::find(),
            'db' => $this->connection
        ]);
    }
}
```

Возвращаясь к предыдущему предположению про смену бд, мы это легко сможем сделать, но здесь мы видим минусы ActiveRecord. \
Несмотря на то, что запрос будет выполняться на втором сервере, relations модели будут тянуться с основной бд, надо переопределять метод в модели getDb и в репозитории еще подменять модели в зависимости от коннекта - с этим можно жить (можно добавить фабрику для определения нужной модели).

```php
// Пример из моего проекта, когда тяжелые расчеты мы переносили на другой бд-сервер. Переключить код получилось относительно быстро.

$mainStore = new HistoryStore(Yii::$app->db);
$reportStore = new HistoryStore(Yii::$app->report_db);

````

Сделаем кеширование:

```php
namespace stores\cache;

use app\models\History;
use yii\caching\Cache;
use yii\data\DataProviderInterface;
use stores\HistoryStore as IHistoryStore;
use yii\caching\TagDependency;

class HistoryStore implements IHistoryStore {
    /**
     * @var Cache
     */
    private $cache;
    
    /**
     * @var IHistoryStore
     */
    private $store;
 
    public function __construct(IHistoryStore $store, Cache $cache) {
        $this->cache = $cache;
        $this->store = $store;
    }
    
    /**
      * @inheritDoc
      */
    public function search(array $params):DataProviderInterface {
        $provider = $this->store->search($params);
        
        $page = $provider->getPagination()->getPage();
        $pageSize = $provider->getPagination()->getPageSize();
        $key = "history_search_" . $page . "_" . $pageSize;
        $this->cache->getOrSet($key, function () use ($provider) {
            $provider->prepare();
        }, 60 * 60, new TagDependency(['tags' => 'history_search']))
        return $provider;
    }
}
```

Параметры кеширования мы можем вынести в класс, например, CachePolicy.
Если нам надо применить кеширование, мы легко это сможем сделать без изменения основного кода для работы с бд:

```php
use stores\db\HistoryStore as HistoryDbStore;
use stores\cache\HistoryStore as HistoryCacheStore;

$store = new HistoryCacheStore(
    new HistoryDbStore(Yii::$app->db)
    Yii::$app->cache
);

```
Если нам резко надо будет перейти с файлов на redis, мы просто меняем зависимость.
Отвечая на прошлый вопрос про эластик, мы можем сделать еще одну реализацию:

```php

namespace stores\engines;

use app\models\History;
use yii\caching\Cache;
use yii\data\ArrayDataProvider;
use yii\data\DataProviderInterface;
use stores\HistoryStore as IHistoryStore;
use yii\caching\TagDependency;
use Elasticsearch\Client;

class HistoryStore implements IHistoryStore {
    /**
     * @var Client
     */
    private Client $client;
 
    public function __construct(Client $client) {
        $this->client = $client;
    }
    
    /**
      * @inheritDoc
      */
    public function search(array $params):DataProviderInterface {
        $pagination = new Pagination([
            'pageSizeLimit' => [15, 100],
            'validatePage' => false,
        ]);

        $sort = new Sort([
            'defaultOrder' => ['id' => SORT_DESC],
            //...
        ]);
        
        $response = $this->client->search([
            // параметры для запроса
        ])
        
        return new ArrayDataProvider([
           // ...
        ])
    }
}

```

И так же мы ее можем оборачивать в кеш. \
Если нам нужно учитывать доступность кеша, например, если файловый недоступен, переключиться на redis, мы так же можем подменить реализацию CacheInterface.

Если мы решили кеширование включить во всем проекте, то мы можем создать bootstrap файл:

```php

use yii\base\BootstrapInterface;
use stores\HistoryStore as IHistoryStore;
use stores\db\HistoryStore as HistoryDbStore;
use stores\cache\HistoryStore as HistoryCacheStore;

class SetUp implements BootstrapInterface
{
    public function bootstrap($app): void
    {
        Yii::$container->set(IHistoryStore::class, function () use ($app) {
            return new HistoryCacheStore(
                new HistoryDbStore($app->db)
                $app->cache
            )
        })
    }
}
```

Таким образом, если в каком-то месте нам надо будет изменить реализацию, выключить кеш или поменять на эластик, мы можем это сделать не переписывая существующий код, а лишь его дополнив.\
Так же это дает гибкость при unit-тестировании. Мы можем подменять зависимости на заглушки, например, внешние http-клиенты заменять стабами.\
Или, например, если используем PasswordHasher при создании хеша пароля пользователя, мы можем его мокать, например, проверять что метод вызвался один раз, потому что алгоритм хеширования у нас долгий.

### 4. Использование HistorySearch в нескольких местах или useCases

Поиск используется у нас как в контроллере, так и в виджете. \
Предположим, что его еще мы будем использовать в апи. \
Мы бы так же вызвали метод search и отдали провайдер в нужном формате. Проблема возникнет, если мы захотим добавить еще что-нибудь к нашим бизнес-требованиям.
Например, нам нужно считать кол-во запросов, когда используется поиск по истории и кинуть событие, что поиск был использован.
Из моей практики я вводил понятие useCases.

- assets
- config
- ...
- useCases
  - HistoryUseCase

```php

use stores\HistoryStore;

class HistoryUseCase {
    /**
     * @var HistoryStore
     */
    private  $store;
    /**
     * @var EventDispatcher
     */
    private  $dispatcher;
    
    public function __construct(HistoryStore $store, EventDispatcher $dispatcher) {
        $this->store = $store;
        $this->dispatcher = $dispatcher;
    }
    
    public function search(Request $request): Response {
        if (false === $request->searchForm->validate()) {
            return Response::validateErrors($request->searchForm->getErrors());
        }
        $provider = $this->store->search($request->searchForm->getAttributes());
        $this->store->useSearch(); // увеличиваем счетчик
        $this->dispatcher->dispatch(new UseHistorySearch($request->searchForm)); // кидаем событие
        
        $response = Response::success();
        $response->provider = $provider;
        return $response;
    }   
}
```

Классы Request, Response в рамках useCase'ов - dto для удобства работы.
В зависимости от ответа, мы можем по-разному отдавать ошибки - для апи json, для web кидать через flash.
Дальше по всему проекту я работал только с этим классом, это позволяет добавлять поведение не изменяя код по всему проекту. \
Так же используется Yii::$container:

```php

use yii\base\BootstrapInterface;
use stores\HistoryStore as IHistoryStore;
use stores\db\HistoryStore as HistoryDbStore;
use stores\cache\HistoryStore as HistoryCacheStore;

class SetUp implements BootstrapInterface
{
    public function bootstrap($app): void
    {
        $container = Yii::$container;
        $container->set(IHistoryStore::class, function () use ($app) {
            return new HistoryCacheStore(
                new HistoryDbStore($app->db)
                $app->cache
            )
        });
        $container->set(HistoryService::class, function () use ($container) {
            return new HistoryService($container->get(IHistoryStore::class));
        })
    }
}
```

### 5. Структура проекта
Экспериментируя со структурой проекта и исходя из того, что фреймворк является инфраструктурой для нашего ядра, я предложил, что наши сервисные классы, хранилища, диспатчеры, сущности, модели данных лучше выносить в какую-нибудь отдельную папку, например:

- assets
- config
- ....
- widgets
- core или americor
  - dispatchers
  - listeners
  - entities
  - models
  - useCases
  - ...

Это в долгосрочной перспективе может нам облегчить переход, например, на другой фреймворк: мы сможем выделить интерфейсы и подменить реализации.

### 6. Абстракция над временем
Тестируя функционал, который зависит от времени, мы сталкиваемся с тем, что нам нужно прокрутить время, но как это сделать, если мы используем date(), DateTime? \
Мы, конечно, можем в системе менять через какую-нибудь команду системное время. Как можно проще? \
Я использовал библиотеку Carbon и в тестах мы можем использовать Carbon::setNow(), то есть промотать время вперед или назад.
К тому же, абстракции над стандартными типами нам помогают сделать наш код более понятным и удобным.

Пример из текущего кода (models/Task.php):
```php
class Task {
     /**
     * @return bool
     */
    public function getIsOverdue()
    {
        return $this->status !== self::STATUS_DONE && strtotime($this->due_date) < time();
    }
}
```
Можно сделать следующим образом:
```php
class Task {
     /**
     * @return bool
     */
    public function getIsOverdue()
    {
        $dueDate = Carbon::make($this->due_date);
        return $this->status !== self::STATUS_DONE && $dueDate < Carbon::now();
    }
}
```
Тогда в unit-тестах мы сможем протестировать этот метод не изменяя системные настройки языка или ОС.
```php
$task = new Task(['due_date' => '2023-02-02 10:00:00']);
$this->assertFalse($task->isOverdue());
Carbon::setNow('2023-02-02 12:00:00');
$this->assertTrue($task->isOverdue());
```
Конечно, мы должны не забывать, что в тестах нам нужно сбрасывать время после выполнения теста.

### 7. Представления
Для моделей мы можем создать слой представления, например:

- presenters
  - history
    - HistoryPresenter

views/site/export.php
```php
[
    'label' => Yii::t('app', 'User'),
    'value' => function (History $model) {
      return isset($model->user) ? $model->user->username : Yii::t('app', 'System');
    }
],
```
можно заменить на
```php
[
   'label' => Yii::t('app', 'User'),
   'value' => function (History $model) {
      return new HistoryPresenter($model)->username();
      // или
      return $model->present()->user()->username();
      // или
      // Если User не существует, можно использовать NullObject, чтобы цепочка вызовов всегда работала,
      // просто если пользователь не существует, вернется Yii::t('app', 'System')
      return new HistoryPresenter($model)->user()->username;
   }
],
```

### 8. Коллекции
Если нам перестать работать с нативными массивами, а перейти на абстракцию, мы могли более гибко управлять кодом.\
Примером абстракции может служить, например, коллекции в Laravel (https://laravel.com/docs/9.x/collections).\
Здесь стоит отменить тот момент, что yii2 DataProvider надо будет переделать, чтобы работать с коллекциями, тогда нам было бы проще включать представления:
```php
(new HistoryCollection($models))->present(HistoryPresenter::class)
(new HistoryCollection($models))->present(ApiHistoryPresenter::class)
```
и код бы выглядел более лаконично:
```php
[
    'label' => Yii::t('app', 'User'),
    'value' => function (HistoryPresenter $model) {
      return $model->user()->username();
    }
],
```
Так же, мы могли бы лучше управлять реализацией, в одном случае, мы используем классические функции для работы с массивами, в другом случае переходим на итераторы.\
В рамках этого проекта я не увидел прямой работы с массивами, но посчитал, что стоит об этом упомянуть.

### 9. Глобальные переменные
widgets/Export/Export.php
```php
$_POST[Yii::$app->request->methodParam] = 'POST';
$_POST[$this->exportRequestParam] = true;
$_POST[$this->exportTypeParam] = $this->exportType;
$_POST[$this->colSelFlagParam] = false;
```
В виджете происходит обращение к глобальным переменным - так делать не стоит, потому что метод Yii::$app->request не узнает об этом изменении:
vendor/yiisoft/yii2/web/Request.php
```php
 public function getBodyParams()
 {
    if ($this->_bodyParams === null) {
        //...
    }
    return $this->_bodyParams;
 }
```
Лучше избегать работу с глобальными переменными напрямую и продолжать с ними работать через компонент:
```php
Yii::$app->request->setBodyParams([
    Yii::$app->request->methodParam => 'POST'
    $this->exportRequestParam => true,
    $this->exportTypeParam => $this->exportType,
    $this->colSelFlagParam => false
]);
```
Состояние объекта Request будет корректным, потому что мы работаем с его доступными методами, тем самым корректно изменяем его состояние. \
Так же можно добавить, что у нас не будет проблем при запуске тестов, потому что некоторые типы тестов работают с функционалом yii2. Меняя глобально массив $_POST, высока вероятность, что мы не сможем воспользоваться инструментами для тестирования.

### 10. HistoryListHelper

Если исходить из того, что данными оперирует тот, кто ими владеет, получается нам HistoryListHelper и не нужен. \
Метод HistoryListHelper::getBodyByModel($model) мы можем перенести в саму модель History, но так как мы затрагивали презентеров, то можно это перенести в них.

Так же продолжая мысль, данный блок:
widgets/HistoryList/views/_item.php
```php
switch ($model->event) {
    case History::EVENT_CREATED_TASK:
    case History::EVENT_COMPLETED_TASK:
    case History::EVENT_UPDATED_TASK:
        $task = $model->task;

        echo $this->render('_item_common', [
            'user' => $model->user,
            'body' => HistoryListHelper::getBodyByModel($model),
            //...
        ]);
        break;
    case History::EVENT_INCOMING_SMS:
    case History::EVENT_OUTGOING_SMS:
        echo $this->render('_item_common', [
            'user' => $model->user,
            //...
```
можно заменить на:
```php
$event = $model->event;
switch(true) {
    case $event->isCreatedTask():
    case $event->isCompetedTask():
    case $event->isUpdatedTask():
        //...
}
```
То есть мы скрываем логику определения состояния в модель, которая владеет этой информацией.
Так же строки вида:
```php
$call->status == Call::STATUS_ANSWERED
$call->direction == Call::DIRECTION_INCOMING
```
Должны заменяться на:
```php
$call->isStatusAnswered()
$call->isDirectionIncoming()
```
или
```php
$call->status()->isAnswered()
$call->direction()->isIncoming()
```
В любом случае клиент не должен управлять внутренним устройством, потому что со временем логика определения статуса может поменяться и тогда нам придется это менять по всему проекту, а так поменяем только в модели.

Можно было бы упростить структуру switch case, например, добавив шаблоны для каждого типа event'а:
```php
$event = $model->event;
switch(true) {
    case $event->isTask():
        return $this->render('items/task', [...]);
    case $event->isSms():
        return $this->render('items/sms', [...]);
    case $event->isFax():
    //..
    case $event->isCall():
    //..
}
```

Если данный шаблон строки в гриде будет использоваться где-то еще, в рамках другого грида, мы можем уже подумать над абстракциями для UI-компонентов, которая будет иметь метод display и ренденрить нужный шаблон в зависимости от контекста использования.

### 11. Компонент переведов 

Я рекомендовал бы донастроить компонент и сделать алиасы для длинных предложений, вынести их в файлы переводов.

### 12. Экспорт больших данных

Я бы наблюдал по метрикам приложения, как ведет себя экспорт со временем. \
Если увеличивается потребление памяти, сконфигурировал бы опцию batchSize в ExportMenu. \
Скажу честно, виджет этот не использовал. \
Дальше можно оптимизировать саму логику экспорта, можно оптимизировать запрос к бд. \
В запросе можно добавить селект только нужных полей. \
Потом можно смотреть профиль запроса и точечно его улучшать. \
Так же мы на работе применяли таблицы-срезы, чтобы уходить от джойнов - таблица создавалась в фоне. \
Потом мы пересмотрели систему отчетов и сделали ее на очередях и по готовности отправляли уведомление пользователю, что отчет готов.\
Настройки бы перенес уже в контроллер, либо в виджет, не оставлял бы их в отображении:
```php
ini_set('max_execution_time', 0);
ini_set('memory_limit', '2048M');
```

### Заключение
Я хотел бы предложить продолжить обсуждение, уточнение деталей на встрече : )
Если нужно будет выполнить рефакторинг кода - я готов это сделать. 

### Спасибо за уделенное время
