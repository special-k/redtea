![redtea](https://img-fotki.yandex.ru/get/15516/43145129.2/0_ca88d_f2845ae2_orig)
coffeescript библиотека

>Не превращайте сложное в архисложное.  
>*Владимир Алекно*

#### [Почему наши web-интерфейсы такие медленные и сложные?](#Почему-наши-web-интерфейсы-такие-медленные-и-сложные-1)  
#### [Redtea](#redtea-1)  
##### [Виджеты](#Виджеты-1)  
###### [StorageItem](#storageitem-1)  
##### [Менеджеры](#Менеджеры-1)  
##### [CoffeeScript и DSL](#coffeescript-и-dsl-1)
##### [Почему redtea настолько быстр](#Почему-redtea-настолько-быстр-1)

# Почему наши web-интерфейсы такие медленные и сложные?
>echo '<ВR>';

Если вы помните как писались древние пхп сайты (еще до того как туда притащили rails-way фрэймворки), то это была жуткая мешанина html, sql, php и т.д... Но кто объяснит чем это
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

>Писать код бесконечно лучше, чем метаданные, будь то шаблон или сложный язык запросов.  
>*Жан-Жак Дюбре*

И это только первая часть проблемы, другая ее часть менее очевидна. Это то, что я называю *основным потоком инициализации*. Т.е. существует основной поток инициализации, в рамках которого вы можете применять все эти чудесные шаблоны, привязки и практики. Но после его прохождения (т.е. после инициализации компонента), все это применять уже невозможно совсем.

Предположим, есть простая html табличка.
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
Дело в том, что популярные ныне идеологии разработки веб-интерфейсов происходят из других сред. И если говорить о монстрах вроде extjs - то это настольное программирование, если говорить о решениях вроде backbone и angular - это бэкендные MVC фрэймворки типа rails. И те и другие рассматривают DOM-дерево как некую свалку, или, в лучшем случае, базу данных - что не может благоприятно сказываться на производительности.

### Простой и понятный путь
DOM-дерево - это объекты. К генерации объектов следует относиться бережно. Инициализацию объектов необходимо контролировать. Всегда нужно знать сколько у тебя объектов и зачем они нужны.

Именно эту задачу решает библиотека redtea. И прежде я предлагаю провести небольшой тест производительности.  
Существует сайт который демонстрирует основные возможности веб-фрэймворков на примере простого приложения http://todomvc.com  
Сравним производительность типового приложения на React http://todomvc.com/examples/react/  
и redtea http://special-k.github.io/todolist/

Для этого добавим немного строк. Это легко сделать через консоль (F12):  
для reactjs
```js
var t = [], count = 1000;
for(var i = 0; i < count; i++ ) {t.push({title: 'task ' + i, completed: false, id: 'id_' + i})}
localStorage.setItem('react-todos', JSON.stringify(t));
```
для redtea
```js
var t = [], count = 1000;
for(var i = 0; i < count; i++ ) {t.push({task: 'задача ' + i, finished: false})}
localStorage.setItem('todos-redtea', JSON.stringify(t));
```
К примеру, на моем ПК в firefox достаточно 1000 *(всего лишь 1000!)* строк, чтобы увидеть существенное торможение reactjs на массовых операция (отметить все/снять все, посмотреть все/активные/сделанные задачи).
Еще один тест на производительность - это [dbmonster](http://special-k.github.io/repaint) (и прохождение этого теста другими библиотеками http://mathieuancelin.github.io/js-repaint-perfs/). 

# Redtea

Основные компоненты redtea это виджеты и менеджеры.

## Виджеты
![Виджет](https://img-fotki.yandex.ru/get/67504/43145129.3/0_da043_85a2efc7_orig)

В самом первом приближении, виджет - это обертка вокруг конкретной DOM-структуры, но не только. Философия redtea говорит, что виджеты - это объекты. Объекты должны взаимодействовать. Функционал redtea во многом направлен на реализацию данного возаимодействия, контроль взаимодействия и удобство взаимодействия.

Итак, создадим простейший виджет.
```coffee
class SomeWidget extends RT.Widget
  createDom: (self)->
    @div()
```
Это минимальный набор операций, для создания виджета.  
Redtea позволяет регистрировать виджеты для упрощения использования в дальнейшем. Сейчас разберем что это значит.
```coffee
class SomeWidget extends RT.Widget

  @register 'someWidget'  # <-- регистрация виджета

  createDom: (self)->
    @div()
```
И теперь пример для чего это нужно.
```coffee
class SomeAnotherWidget extends RT.Widget

  createDom: (self)->
    @div ->
      @someWidget() # <-- redtea позволяет легко задействовать зарегистрированные виджеты в DOM-структурах 
```
Далее рассмотрим работу с DOM структурой виджета. Для этого следует сохранить узлы в объектные переменные.
```coffee
class SomeWidget extends RT.Widget
  createDom: (self)->
    @div ->
      self.span1 = @span()
      @span().setas('span2', self) # <-- более удобный способ
  
  #для примера создадим метод, чтобы работать с узлами
  updateAllSpan: (className)->
    @span1.classList.add className
    @span2.classList.add className
```
В redtea не принято сканировать DOM-структуры в поисках нужного узла. Ссылку на любой узел/виджет можно получить в момент его создания - это является краеугольным камнем производительности redtea.

### StorageItem

Другим важным функционалом виджетов является специальный механизм биндинга данных (на картинке выше он обозначен буквой S), который встроен в каждый виджет. Это, своего рода, персональная модель виджета.
```coffee
class SomeWidget extends RT.Widget

  @register 'someWidget'
  
  createDom: (self)->
    @div ->
      @span().setas 'span1', self
      @span().setas 'span2', self
      
  storageInit: ->
    @storageItem.bi 'onFieldChanged', @onFieldChanged, context: @ # <-- подписка на изменение данных
  
  # реализация обновления данных
  onFieldChanged: (eventObject, field, value)->
    switch field
      when 'field1' then @span1.className = value
      when 'field2' then @span2.className = value
```
Использование механизма привязки данных.
```coffee
class SomeAnotherWidget extends RT.Widget

  createDom: (self)->
    @div ->
      @someWidget()
      
  init: ->
    @storageItem().get('widgets').first().set('field1', 'class1') # <--
```
Т.о. не требуется непосредственное обращения к методам виджета, вместо этого происходит обращение к storageItem виджета. Важно, что поле 'widgets' было сгенерировано автоматически, и все storageItem дочерних компонентов (первого порядка) попадют в специальные поля-коллекции.

![Взаимодействие StorageItem](https://img-fotki.yandex.ru/get/27964/43145129.3/0_da04a_3b39449f_orig)

При этом существует возможность устанавливать имя колекции, в которую будет попадать storageItem дочернего элемента.
```coffee
class SomeWidget extends RT.Widget

  collectionName: 'someName'
```
Существует возможность подписаться сразу на изменения всех storageItem коллекции. Т.о. коллекция виджета определяется исходя из его функционального назначения.

## Менеджеры

![Менеджер подключенный к виджету](https://img-fotki.yandex.ru/get/70180/43145129.3/0_da04b_96d8d340_orig)  

Другая основная сущность в redtea - это менеджер. Менеджеры - это объекты, доступ к которым можно получить из виджетов и других менеджеров, для чего необходимо выполнять процедуру подключения.
```coffee
class SomeWidget extends RT.Widget

  managers: ['someManager'] # <-- подключение менеджера
  
  someManagerAdded: ->
    @someManager # <-- обращение к подключенному менеджеру
```
Очевидно, что прежде менеджер необходимо регистрировать, это, как правило, делается в основном классе приложения.
```coffee
class glob.Repaint extends RT.Stratum

  constructor: ->
    super
    @addManager 'someManager', new SomeManager # <-- добавление менеджера в приложение
```
После этого менеджер можно подключать в виджеты и другие менеджеры.  
Назначение менеджеров, как уже сказано, может быть совершенно разным. Менеджер, по сути, просто способ подключения функционала.  
Наиболее очевидная задача менеджера - адаптация данных модели к структуре storageItem виджетов. Пример в приложении [repaint](https://github.com/special-k/repaint/blob/master/lib/coffeescripts/repaint.js.coffee#L15).

При подключении одного менеджера в нескольких можно реализовать связку, напоминающую MVC.

![Менеджер подключенный к виджету](https://img-fotki.yandex.ru/get/66316/43145129.3/0_da086_60b3e4b4_orig)

Говоря об MVC следует отметить, что реализации модели в redtea нет, нужно будет использовать модели из других библиотек. Это происходит потому, что реализация модели сильно связана с конкретной реализацией бэкенда, и redtea стремиться быть агностичным по отношении к бэкенду. В целом, через подписку на изменеие storageItem взаимодействие с моделью устанавливается достаточно легко.

## Coffeescript и DSL

Язык, на котором вы пишете, должен соответствовать задачам. Наилучший вариант - взять язык общего назначения и реализовать внутри него DSL (можно, конечно, написать язык с "0", но это трудоемко и рисковано). Javascript плохо подходит для этой цели, все-таки, это язык с весьма скудными синтаксическими возможностями. Совсем другое дело coffeescript: благодаря лаконичности языковых конструкций, он хорошо подходит для DSL. К слову, reactjs и angularjs2 так же используют специфичные языки.

>Если вы никогда не видели coffeescript, то нужно знать примерно следующие: coffeescript мало чем отличает от js, в основном изменения носят косметический характер. Так в coffeescript “this.” заменен символом “@”, “function” заменен “->”, а вместо фигурных скобок используются питоноподобные отступы. Кроме того скобки часто можно опускать.

## Почему redtea настолько быстр

### 1. Взаимодействие с DOM без посредников

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
Перечислим чего вы при этом НЕ сделали:
 1. вы НЕ сканировали DOM-дерево в поисках узла, который небходимо заменить (т.е. $('#elemId_123 .aaa .bbb') и т.д.)
 2. Вы НЕ передавали в шаблонизатор параметры и не формировали строку.
 3. Вы НЕ парсили результат шаблонизатора в DOM-элементы.
 4. Вы НЕ заменяли блок на странице вновь созданным.

Вы хотели поменять текст и вы сделали именно это. Очевидно, что даже одна из НЕ сделанных операций гораздо тяжеловеснее работы с текстовыми узлами (а после все удивляются, отчего сайты ТАК тормозят!)

### 2. Пакетное обновление DOM
События StorageItem распространяются пакетно, тем самым обеспечивая пакетное редактирование DOM и высочайшую производительность [dbmonster](http://special-k.github.io/repaint).
