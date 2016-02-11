# Библиотека RedTea

## Основные преимущества

### 1. Взаимодействие с DOM без посредников

Для примера создадим небольшой фрагмент HTML: блок, содержащий цену, с отдельным отображением целой части и “копеек”.
Пара слов о cofeescript. Если вы никогда не видели coffescipt, то нужно знать примерно следующие: cofeescript мало чем отличает от js, в основном изменения носят косметический характер. Так в cofeescript “this.” заменен символом “@”, “function” заменен “->”, а вместо фигурных скобок используются питоноподобные отступы. Кроме того скобки часто можно опускать.
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
Для формирования узлов DOM-дерева использщуется haml/jade подобный DSL. Однако в отличии от первых, это не шаблонизатор. Каждый вызов функции здесь непосредственно создает узел (по сути вызывается createElement).

В последствии, вы легко можете обращаться к этим узлам.
Приведем пример такого обращения, для этого немного перепишем код.
```coffee
mainSum = null
additionSum = null
document.body.append ->
  @div class: 'price', ->
    mainSum = @span class: 'mainSum', ->
      @tn '10'
    @span class: 'additionSum', ->
      additionSum = @tn '00'
```
Теперь, предположим вы хотите изменить цену. Для этого нужно просто установить новые значения в текстовые узлы, которые мы теперь записали в локальные переменные.
```coffee
mainSum.nodeValue = '20'
additionSum.nodeValue = '50'
```
Все!
Используя jquery/angular и т.д. вы никогда не работали с тектовыми узлами. Но чтобы изменить текст на странице, вообще-то, достаточно установить значение атрибута текстовому узлу.
Давайте перечисли чего вы при этом НЕ сделали:
1. вы НЕ сканировали DOM-дерево в поисках узла, который небходимо заменить (т.е. $('#elemId_123') и т.д.)
2. Вы НЕ передавали в шаблонизатор параметры и не формировали строку.
3. Вы НЕ парсили эту строку в DOM-элементы
4. Вы НЕ заменяли блок на странице вновь созданным.

Вы хотели поменять текст и вы сделали именно это. Я думаю, очевидно, что даже одна из НЕ сделанных операций гораздо тяжеловеснее работы с текстовыми узлами (а после все удивляются, отчего сайты ТАК тормозят!)

### 2. Привязка функционала к определенному блоку элементов – виджеты.
Мы продолжим работать с этим небольшим блоком цены. Сделаем работу с ним более удобной.
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
Создав такой класс, теперь мы можем работать следующим образом:
```coffee
price = null
document.body.append ->
  price = @price value: 10

price.setValue 20.5
```
Все выглядит так, будто в HTML есть такой элемент – price.
Это одна из основных фишек виджетов – бесшовно встраиваться в DSL верски.

### 3. Автоструктуры данных.
Теперь попробуем решить проблему привязки данных. Для этого несколько усложним задачу.
Теперь у нас будет список цен.

Сначала создадим сам виджет списка.
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
Получился список цен, но это еще не все: для виджетов автоматически создалась структура данных.
```coffee
list.storageItem.get('widgets')[0].get('priceValue') //null – пока что она пустая
```
Теперь необходимо задействовать данный механизм в наших виджетах:
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
Теперь создание списка будет выглядить несколько иначе:
```coffee
list = null
document.body.append ->
  list = @list ->
    @price storageData: 
      priceValue: 0
```
И отредактируем значение:
```coffee
list.storageItem.get('widgets')[0].setValue 'priceValue', 10
```
Но это не единственный способ добавить новую строку:
```coffee
list.storageItem.get('widgets').push new RT.StorageItem priceValue: 20
list.storageItem.get('widgets').push new RT.StorageItem priceValue: 30
```
Т.о. в концепции создания виджетов нет различия между добавлением данных и добавлением виджетов: как бы вы не создавали виджеты, вы будете получать одинаковую структуру DOM, и одинаковую структуру данных.

Через структуры данных можно легко взаимодействовать с сервером.
Например с помощью bacbone.
```coffee
BacboneModel.on 'sync', ->
  #//функция, которая сверяет данные в list.storageItem.get('widgets') и модели bacbone

list.storageItem.get('widgets').bi 'onItemAdded', (eventObject, item)->
  backboneItem = new BacboneModel item.serialize()
  backboneItem.save()
```
### 4. Управление данными и глобальное взаимодействие – менеджеры
Сделаем механизм управления данными более удобным.
```coffee
class DataManager extends RT.BaseManager
  constructor: ->

  storageItem: (item)->
    BacboneModel.on 'sync', ->
      #//функция, которая сверяет данные в list.storageItem.get('widgets') и модели bacbone

    item.bi 'onItemAdded', (eventObject, item)->
      backboneItem = new BacboneModel item.serialize()
      backboneItem.save()
```
Привязка виджетов осуществляется через объект RT.Stratum
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
