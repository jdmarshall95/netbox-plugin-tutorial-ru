# Шаг 4: Формы

Классы форм генерируют HTML-элементы формы для пользовательского интерфейса, а также обрабатывают и проверяют вводимые пользователем данные. Они используются в NetBox в основном для создания, изменения и удаления объектов. Мы создадим класс form для каждой из наших моделей плагинов.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step03-таблицы`.

## Создайте формы

Начните с создания файла с именем `forms.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit forms.py
```

В верхней части файла будет импортирован класс NetBox `NetBoxModelForm`, который будет служить базовым классом для наших форм. Мы также импортируем модели нашего плагина.

```python
from netbox.forms import NetBoxModelForm
from .models import AccessList, AccessListRule
```

### Форма списка доступа

Создайте класс с именем `AccessListForm`, подкласс `NetBoxModelForm`. В рамках этого класса определите подкласс `Мета`, определяющий `модель` и `поля` формы. Обратите внимание, что список "поля" также включает `теги`: назначение тегов обрабатывается "NetBoxModel" автоматически, поэтому нам не нужно было добавлять его в нашу модель на втором шаге.

```python
class AccessListForm(NetBoxModelForm):

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

Одного этого достаточно для нашей первой модели, но мы можем внести одну поправку: вместо поля по умолчанию, которое Django сгенерирует для поля модели `комментарии`, мы можем использовать специально созданный класс `CommentField` NetBox. (Это обрабатывает некоторые в основном косметические детали, такие как установка `help_text` и настройка макета поля.) Чтобы сделать это, просто импортируйте класс `CommentField` и переопределите поле формы:

```python
from utilities.forms.fields import CommentField
# ...
class AccessListForm(NetBoxModelForm):
    comments = CommentField()

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

### AccessListRuleForm

Мы создадим форму для `AccessListRule` по тому же шаблону.

```python
class AccessListRuleForm(NetBoxModelForm):

    class Meta:
        model = AccessListRule
        fields = (
            'access_list', 'index', 'description', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'tags',
        )
```

По умолчанию Django создаст "статическое" поле внешнего ключа для связанных объектов. Это отображается в виде выпадающего списка, который предварительно заполнен _всеми_ доступными объектами. Как вы можете себе представить, в экземпляре NetBox со многими тысячами объектов это может стать довольно громоздким.

Чтобы избежать этого, NetBox предоставляет класс `DynamicModelChoiceField`. Это отображает поля внешнего ключа с помощью специального динамического виджета, поддерживаемого REST API NetBox. Это позволяет избежать накладных расходов, связанных со статическим полем, и позволяет пользователю удобно осуществлять поиск нужного объекта.

:green_circle: **Совет:** Класс `DynamicModelMultipleChoiceField` также доступен для полей типа "многие ко многим", которые поддерживают назначение нескольких объектов.

Мы будем использовать `DynamicModelChoiceField` для трех полей внешнего ключа в нашей форме: `access_list`, `source_prefix` и `destination_prefix`. Во-первых, мы должны импортировать класс field, а также модели связанных объектов. `Список доступа` уже импортирован, поэтому нам просто нужно импортировать `Префикс` из приложения NetBox `ipam`. Начало `forms.py` теперь должно выглядеть следующим образом:

```python
from ipam.models import Prefix
from netbox.forms import NetBoxModelForm
from utilities.forms.fields import CommentField, DynamicModelChoiceField
from .models import AccessList, AccessListRule
```

Затем мы переопределяем три соответствующих поля в классе form, создавая экземпляр `DynamicModelChoiceField` с соответствующим значением `queryset` для каждого. (Обязательно сохраните на месте класс `Meta`, который мы уже определили.)

```python
class AccessListRuleForm(NetBoxModelForm):
    access_list = DynamicModelChoiceField(
        queryset=AccessList.objects.all()
    )
    source_prefix = DynamicModelChoiceField(
        queryset=Prefix.objects.all()
    )
    destination_prefix = DynamicModelChoiceField(
        queryset=Prefix.objects.all()
    )
```
Теперь, когда все наши модели, таблицы и формы на месте, мы создадим несколько представлений, чтобы свести все воедино!

<div align="center">

:arrow_left: [Шаг 3: Таблицы](/tutorial/step03-tables.md) | [Шаг 5: Представления](/tutorial/step05-views.md) :arrow_right:

</div>

