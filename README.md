![redtea](https://img-fotki.yandex.ru/get/15516/43145129.2/0_ca88d_f2845ae2_orig)
coffeescript библиотека

>Не надо превращать сложное в архисложное.  
>*Владимир Алекно*

- [Почему наши web-интерфейсы такие медленные и сложные?](#)
- [Простой и понятный путь](#)
- **redtea**
- [Основные преимущества](#)
	- [1. Взаимодействие с DOM без посредников](#)
	- [2. Привязка функционала к определенному блоку элементов – виджеты.](#)
	- [3. Привязка данных - автоструктуры.](#)
	- [4. Управление данными и глобальное взаимодействие – менеджеры](#)

## Почему наши web-интерфейсы такие медленные и сложные?
>echo '<ВR>';

Если вы помните как писались древние пхп сайты (еще до того как туда притащили rails-way фрэймворки). То это была жуткая мешанина html, sql, php и т.д... Но кто объяснит чем это
```html
<li ng-repeat="todo in todos | filter:statusFilter track by $index" ng-class="{completed: todo.completed, editing: todo == editedTodo}">
  <div class="view">
    <input class="toggle" type="checkbox" ng-model="todo.completed" ng-change="toggleCompleted(todo)">
    <label ng-dblclick="editTodo(todo)">{{todo.title}}</label>
    <button class="destroy" ng-click="removeTodo(todo)"></button>
  </div>
  <form ng-submit="saveEdits(todo, 'submit')">
    <input class="edit" ng-trim="false" ng-model="todo.title" todo-escape="revertEdits(todo)" ng-blur="saveEdits(todo, 'blur')" todo-focus="todo == editedTodo">
  </form>
</li>
```
лучше? Кто вообще объяснит, почему управление содержимым страницы **на клиенте** должно осуществляться через xml?

И это только первая часть проблемы, другая ее часть менее очевидна. Это то, что я называю *основным потоком инициализации*. Т.е. существует основной поток инициализации, в рамках которого вы можете применять все эти чудесные шаблоны, привязки и практики. Но после его прохождения (т.е. после инициализации компонента), все это применять уже невозможно совсем.

Предположим, у нас есть простая html табличка
```html
<table>
  <tr>
    <td></td>
    <td></td>
  </tr>
<table>
```
Можете ли вы динамически добавлять/удалять ее строки - да. Легко ли это - да. В любой момент времени - да. Есть ли у вас ограничения - нет.  
Теперь рассмотрим какой-нибудь extjs grid. Легко ли туда добавлять строки? - невозможно. Работа допустима только в рамках [специализированных интерфейсов](https://pp.vk.me/c631219/v631219094/3286/SIPG87EEjnQ.jpg). Extjs grid - это комбайн, который выплевывает текст.  
Шаблонизатор, после того как выдал текст уже не может его изменить. Вся работа по изменению внутренних частей работающих компонентов всегда будет происходить через костыли, т.е. через некие *локальные потоки инициализации*, которые попытаются представить, будто что-то проинициализировано в основном потоке инициализации. С этим сопряжен ряд трудностей, наиболее очевидная из них, что шаблоны и код пишутся на разных языках, происходят вкрапления одного кода в другой.
```js
globalId += 1;
$popover = $el('<div id="nspopover-' + globalId +'"></div>')
  .addClass('ns-popover-' + placement_ + '-placement')
  .addClass('ns-popover-' + align_ + '-align')
  .css('position', 'absolute')
  .css('display', 'none')
;
```
Другое следствие - callback-hell. И не говоря о том, что все это крайне неочевидно и трудно для понимания.

### Корень проблемы
Почему мы выбираем столь неудобные пути? Дело в том, что популярные ныне идеологии разработки веб-интерфейсов происходят из других сред. И если говорить о монстрах вроде extjs - то это настольное программирование, если говорить о решениях вроде backbone и angular - это бэкендные mvc фрэймворки типа rails. И те и другие рассматривают DOM-дерево как некую свалку, или, в лучшем случае, базу данных - что не может благоприятно сказываться на производительности.

##Простой и понятный путь
DOM-дерево - это объекты. К генерации объектов следует относиться бережно. Инициализацию объектов необходимо контролировать. Всегда нужно знать сколько у тебя объектов и зачем они нужны.

Именно эту задачу решает библиотека redtea. И прежде я предлагаю провести небольшой тест производительности.  
Существует сайт который демонстрирует основные возможности веб-фрэймворков на примере простого приложения http://todomvc.com  
Сравним производительность типового приложения на angularjs http://todomvc.com/examples/angularjs-perf/  
и redtea http://special-k.github.io/todolist/

Для этого придется добавить побольше строк. Это легко делается через консоль (F12):  
для angularjs-perf
```js
var t = [], count = 1000;
for(var i = 0; i < count; i++ ) {t.push({title: 'task ' + i, completed: false})}
localStorage.setItem('todos-angularjs-perf', JSON.stringify(t));
```
для redtea
```js
var t = [], count = 1000;
for(var i = 0; i < count; i++ ) {t.push({task: 'задача ' + i, finished: false})}
localStorage.setItem('todos-redtea', JSON.stringify(t));
```
К примеру, на моем ПК в firefox достаточно 1000 *(всего лишь 1000!)* строк, чтобы увидеть существенное торможение angularjs на массовых операция (отметить все/снять все, посмотреть все/активные/сделанные задачи).

Отмечу, что код приложения на redtea не оптимален, особенно функция [scope()](https://github.com/special-k/todolist/blob/master/lib/coffeescripts/todolist.js.coffee#L30) - она просто чудовищно не оптимальна и чрезвычайно часто используется. И даже с этим огромным огрехом приложение на redtea гораздо быстрее angularjs - полагаю, это говорит о верности концепции.

## Основные преимущества

### 1. Взаимодействие с DOM без посредников

Для примера создадим небольшой фрагмент HTML: блок, содержащий цену, с отдельным отображением целой части и “копеек”.
Пара слов о cofeescript.

>Если вы никогда не видели coffescipt, то нужно знать примерно следующие: cofeescript мало чем отличает от js, в основном изменения носят косметический характер. Так в cofeescript “this.” заменен символом “@”, “function” заменен “->”, а вместо фигурных скобок используются питоноподобные отступы. Кроме того скобки часто можно опускать.

```coffee
document.body.append ->
  @div class: 'price', ->
    @span class: 'mainSum', ->
      @tn '10'
    @span class: 'additionSum', ->
      @tn '00'
```
Это добавляет на страницу примерно следующий HTML-код
```html
<div class="price">
  <span class="mainSum">10</span>
  <span class="additionSum">00</span>
</div>
```
Для формирования узлов DOM-дерева используется haml/jade подобный DSL. Однако в отличии от первых, это не шаблонизатор. Каждый вызов функции здесь непосредственно создает узел (по сути вызывается createElement).

В последствии, вы легко можете обращаться к этим узлам.
Приведем пример такого обращения, для этого немного перепишем код.
```coffee
mainSum = null
additionSum = null
document.body.append ->
  @div class: 'price', ->
    @span class: 'mainSum', ->
      mainSum = @tn '10'
    @span class: 'additionSum', ->
      additionSum = @tn '00'
```
Предположим, вы хотите изменить цену. Для этого нужно просто установить новые значения в текстовые узлы (которые уже записаны в локальные переменные).
```coffee
mainSum.nodeValue = '20'
additionSum.nodeValue = '50'
```
Все!

Используя jquery/angular и т.д. вы никогда не работали с тектовыми узлами. Но чтобы изменить текст на странице, вообще-то, достаточно установить значение атрибута текстовому узлу.
Давайте перечислии чего вы при этом НЕ сделали:
 1. вы НЕ сканировали DOM-дерево в поисках узла, который небходимо заменить (т.е. $('#elemId_123 .aaa .bbb') и т.д.)
 2. Вы НЕ передавали в шаблонизатор параметры и не формировали строку.
 3. Вы НЕ парсили результат шаблонизатора в DOM-элементы.
 4. Вы НЕ заменяли блок на странице вновь созданным.

Вы хотели поменять текст и вы сделали именно это. Я думаю, очевидно, что даже одна из НЕ сделанных операций гораздо тяжеловеснее работы с текстовыми узлами (а после все удивляются, отчего сайты ТАК тормозят!)

### 2. Привязка функционала к определенному блоку элементов – виджеты.
Мы продолжим работать с этим небольшим блоком цены: сделаем работу с ним более удобной.
```coffee
class Price extends RT.Widget

  @register 'price'

  collectionName: 'prices'

  createDom: (self)->
    @div class: 'price', ->
      @span class: 'mainSum', ->
        self.mainSum = @tn '0'
      @span class: 'additionSum', ->
        self.additionSum = @tn '00'

  init: (params)->
    @setValue params.value

  setValue: (v)->
    t = v.toFixed(2).split('.')
    @mainSum.nodeValue = t[0]
    @additionSum.nodeValue = t[1]
```
Создав класс виджета, можно работать следующим образом:
```coffee
price = null
document.body.append ->
  price = @price value: 10

price.setValue 20.5
```
Все выглядит так, будто в HTML есть элемент "price".
Это одна из основных фишек виджетов – бесшовное встраивание в DSL.

### 3. Привязка данных - автоструктуры.
Несколько усложним задачу - сделаем список цен.

Вначале создадим сам виджет списка.
```coffee
class List extends RT.Widget

  @register 'list'

  createDom: (self)->
    @div class: 'list'
```
И добавим на страницу:
```coffee
list = null
document.body.append ->
  list = @list ->
    @price value: 10
    @price value: 20
    @price value: 30
```
При формировании списка автоматически создалась структура данных.
```coffee
list.storageItem.get('widgets')[0].get('priceValue') //null – пока что она пустая
```
В каждом виджете есть объект storageItem, при этом storageItem дочерних компонентов попадает в родительский.
storageItem представляет собой простое key-value хранилище, с событиями на изменение полей.
Существует возможность указать конкретное поле родительского компонента для добавления дочерних storageItem (по-умолчанию это поле называется "widgets").
Теперь необходимо задействовать данный механизм в виджетах:
```coffee
class List extends RT.Widget

  @register 'list'

  createDom: (self)->
    @div class: 'list'

  init: ->
    @storageItem.getCollectionField('widgets').bi 'onItemAdded', 'onItemAdded', context: @

  onItemAdded: (eventObject, item)->
    @dom.append ->
      @price storageItem: item


class Price extends RT.Widget

  @register 'price'

  createDom: (self)->
    @div class: 'price', ->
      @span class: 'mainSum', ->
        self.mainSum = @tn '0'
      @span class: 'additionSum', ->
        self.additionSum = @tn '00'

  init: ->
    @setValue @storageItem.get('priceValue') || 0
    @storageItem.bi 'onFieldChanged', 'onFieldChanged', context: @

  setValue: (v)->
    t = v.toFixed(2).split('.')
    @mainSum.nodeValue = t[0]
    @additionSum.nodeValue = t[1]

  onFieldChanged: (eventObject, field, value)->
    if field == 'priceValue'
      @setValue value
```
Создание списка станет выглядить несколько иначе:
```coffee
list = null
document.body.append ->
  list = @list ->
    @price storageData: 
      priceValue: 0
```
Обновление значения:
```coffee
list.storageItem.get('widgets')[0].setValue 'priceValue', 10
```
Другой способ добавления строк:
```coffee
list.storageItem.get('widgets').push new RT.StorageItem priceValue: 20
list.storageItem.get('widgets').push new RT.StorageItem priceValue: 30
```
Т.о. в концепции создания виджетов нет различия между добавлением данных и добавлением виджетов через DSL: как бы вы не создавали виджеты, вы будете получать одинаковую структуру DOM, и одинаковую структуру данных.

Через структуры данных можно легко взаимодействовать с сервером.
Например с помощью Backbone.
```coffee
BackboneModel.on 'sync', ->
  #//функция, которая сверяет данные в list.storageItem.get('widgets') и модели bacbone

list.storageItem.get('widgets').bi 'onItemAdded', (eventObject, item)->
  backboneItem = new BackboneModel item.serialize()
  backboneItem.save()
```
### 4. Управление данными и глобальное взаимодействие – менеджеры
Сделаем механизм управления данными более удобным.
```coffee
class DataManager extends RT.BaseManager
  constructor: ->

  storageItem: (item)->
    BackboneModel.on 'sync', ->
      #//функция, которая сверяет данные в list.storageItem.get('widgets') и модели bacbone

    item.bi 'onItemAdded', (eventObject, item)->
      backboneItem = new BackboneModel item.serialize()
      backboneItem.save()
```
Привязка менеджеров осуществляется через объект RT.Stratum
```coffee
class App extends RT.Stratum
  constructor: ->
    super
    @addManager 'dataManager', new  DataManager

class List extends RT.Widget

  @register 'list'

  managers: ['dataManager'] # подключаем менеджер

  dataManagerLoaded: -> # инициируем работы менеджера
    @dataManager.addStorageItem @storageItem.getCollectionField('widgets')
```
Менеджеры могут иметь совершенно разное назначение, задействоваться в нескольких виджетах одновременно, а также в других менеджерах.
