# Шаг 3: Таблицы

Вы, вероятно, знакомы со списками объектов в NetBox. Именно так мы отображаем все экземпляры объектов определенного типа, таких как сайты или устройства, в пользовательском интерфейсе. Эти списки генерируются классами таблиц, определенными для каждой модели, с использованием библиотеки [django-tables2](https://django-tables2.readthedocs.io/).

Хотя было бы возможно сгенерировать необработанный HTML-код для элементов `<таблица>` непосредственно в шаблоне, это было бы громоздко и сложно поддерживать. Кроме того, эти классы динамических таблиц обеспечивают удобную функциональность, такую как сортировка и разбивка на страницы.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step02-models`.

## Создайте таблицы

Мы создадим две таблицы, по одной для каждой из наших моделей. Начните с создания `tables.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit tables.py
```

В верхней части этого файла импортируйте библиотеку `django-tables2`. Это предоставит классы столбцов для полей, которые мы хотим настроить. Мы также импортируем класс NetBox `NetBoxTable`, который будет служить базовым классом для наших таблиц, и `ChoiceFieldColumn`. Наконец, мы импортируем модели нашего плагина из `models.py`.

```python
import django_tables2 as tables

from netbox.tables import NetBoxTable, ChoiceFieldColumn
from .models import AccessList, AccessListRule
```

### AccessListTable

Создайте класс с именем `AccessListTable` в качестве подкласса `NetBoxTable`. Внутри этого класса создайте дочерний класс "Meta", наследующий от "NetBoxTable".Meta`; это определит модель таблицы, поля и столбцы по умолчанию.

```python
class AccessListTable(NetBoxTable):

    class Meta(NetBoxTable.Meta):
        model = AccessList
        fields = ('pk', 'id', 'name', 'default_action', 'comments', 'actions')
        default_columns = ('name', 'default_action')
```

Атрибут `model` указывает `django-tables2`, какую модель использовать при построении таблицы, а атрибут `fields` определяет, какие поля модели будут добавлены в таблицу. `default_columns` определяет, какие из доступных столбцов отображаются по умолчанию.

Столбцы `pk` и `actions` отображают селекторы флажков и выпадающие меню соответственно для каждой строки таблицы; они предоставляются классом NetBoxTable. В столбце "id" будет отображаться числовой первичный ключ объекта, который включен почти в каждую таблицу в NetBox, но обычно отключен по умолчанию. Остальные три столбца являются производными от полей, которые мы определили в модели `AccessList`.

Того, что у нас есть на данный момент, достаточно для отображения таблицы, но мы можем внести некоторые небольшие улучшения. Во-первых, давайте сделаем столбец `имя` ссылкой на каждый объект. Чтобы сделать это, мы переопределим столбец по умолчанию, определив `name` в классе и передав `linkify=True`.

```python
class AccessListTable(NetBoxTable):
    name = tables.Column(
        linkify=True
    )
```

Также напомним, что поле `default_action` в модели `AccessList` является полем выбора, каждому выбору присваивается цвет. Чтобы отобразить эти значения, мы будем использовать класс NetBox `ChoiceFieldColumn`.

```python
    default_action = ChoiceFieldColumn()
```

Также было бы неплохо включить счетчик, показывающий количество правил, назначенных ему каждым списком доступа. Мы можем добавить пользовательский столбец с именем `rule_count`, чтобы показать это. (Данные для этого столбца будут аннотированы представлением; подробнее об этом на пятом шаге.) Нам также нужно будет добавить этот столбец в наши `поля` и (необязательно) `default_columns` в подклассе `Meta`. Наша готовая таблица должна выглядеть примерно так:

```python
class AccessListTable(NetBoxTable):
    name = tables.Column(
        linkify=True
    )
    default_action = ChoiceFieldColumn()
    rule_count = tables.Column()

    class Meta(NetBoxTable.Meta):
        model = AccessList
        fields = ('pk', 'id', 'name', 'rule_count', 'default_action', 'comments', 'actions')
        default_columns = ('name', 'rule_count', 'default_action')
```

### AccessListRuleTable

Мы также создадим таблицу для нашей модели `AccessListRule`, используя тот же подход, что и выше. Начните с привязки столбцов `access_list` и `index`. Первый будет связан со списком родительского доступа, а второй - с отдельным правилом. Мы также хотим объявить `protocol` и `action` экземплярами `ChoiceFieldColumn`.

```python
class AccessListRuleTable(NetBoxTable):
    access_list = tables.Column(
        linkify=True
    )
    index = tables.Column(
        linkify=True
    )
    protocol = ChoiceFieldColumn()
    action = ChoiceFieldColumn()

    class Meta(NetBoxTable.Meta):
        model = AccessListRule
        fields = (
            'pk', 'id', 'access_list', 'index', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'description', 'actions',
        )
        default_columns = (
            'access_list', 'index', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'actions',
        )
```

Это должно быть все, что нам нужно, чтобы перечислить эти объекты в пользовательском интерфейсе. Далее мы определим некоторые формы, позволяющие создавать и изменять объекты.

<div align="center">

:arrow_left: [Шаг 2: Модели](/tutorial/step02-models.md) | [Шаг 4: Формы](/tutorial/step04-forms.md) :arrow_right:

</div>

