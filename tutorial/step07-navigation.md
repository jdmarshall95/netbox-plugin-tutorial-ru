# Шаг 7: Навигация

До сих пор мы вручную вводили URL-адреса для доступа к представлениям нашего плагина. Очевидно, что для регулярного использования этого будет недостаточно, поэтому давайте посмотрим, как добавить несколько ссылок в навигационное меню NetBox.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step06-templates`.

## Добавление пунктов меню навигации

Начните с создания файла navigation.py в каталоге netbox_access_lists/.

```bash
$ cd netbox_access_lists/
$ edit navigation.py
```

Нам нужно будет импортировать класс PluginMenuItem, предоставленный NetBox, чтобы добавить новые пункты меню; сделайте это в верхней части файла.

```python
from extras.plugins import PluginMenuItem
```

Далее мы создадим кортеж с именем «menu_items». Здесь будут храниться наши настроенные экземпляры PluginMenuItem.

```python
menu_items = ()
```

Давайте добавим ссылку на представление списка для каждой из наших моделей. Это делается путем создания экземпляра PluginMenuItem с (минимум) двумя аргументами:

* `link` — имя URL-пути, на который мы ссылаемся.
* `link_text` — Текст ссылки.

Создайте два экземпляра PluginMenuItem внутри Menu_items:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists'
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules'
    ),
)
```

После перезагрузки страницы вы должны увидеть новый раздел «Плагины» в конце меню навигации, а под ним — раздел «Списки доступа NetBox» с двумя нашими ссылками. При переходе по любой из этих ссылок будет выделен соответствующий пункт меню.

:blue_square: **Примечание:** Если пункты меню не отображаются, попробуйте перезапустить сервер разработки (`manage.py runserver`).

![Navigation menu items](/images/step07-menu-items1.png)

Это гораздо удобнее!

### Добавление кнопок меню

Пока мы этим занимаемся, мы можем добавить прямые ссылки на представления «добавить» для списков доступа и правил в виде кнопок. Нам нужно будет импортировать два дополнительных класса в начало файла navigation.py: PluginMenuButton и ButtonColorChoices.

```python
from extras.plugins import PluginMenuButton, PluginMenuItem
from utilities.choices import ButtonColorChoices
```

`PluginMenuButton` используется аналогично `PluginMenuItem`: создайте его экземпляр с необходимыми аргументами ключевого слова, чтобы вызвать кнопку меню. Эти аргументы таковы:

* `link` — имя URL-пути, на который ссылается кнопка.
* `title` — текст, отображаемый при наведении курсора мыши на кнопку.
* `icon_class` — имя(а) класса CSS, указывающее отображаемый значок.
* `color` — цвет кнопки (выбор предоставляется с помощью `ButtonColorChoices`)

Создайте эти экземпляры в `navigation.py` _над_ `menu_items`. Поскольку каждый элемент меню ожидает получения итерации экземпляров кнопок, мы создадим каждый из них внутри списка.

```python
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]

accesslistrule_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]
```

Затем кнопки можно передать пунктам меню через аргумент ключевого слова «buttons»:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists',
        buttons=accesslist_buttons
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules',
        buttons=accesslistrule_buttons
    ),
)
```

Теперь мы должны увидеть зеленые кнопки «Добавить» рядом со ссылками нашего меню.

![Navigation menu items with buttons](/images/step07-menu-items2.png)

<div align="center">

:arrow_left: [Step 6: Шаблоны](/tutorial/step06-templates.md) | [Step 8: Наборы фильтров](/tutorial/step08-filter-sets.md) :arrow_right:

</div>

