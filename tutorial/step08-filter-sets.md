# Шаг 8: Наборы фильтров

Фильтры позволяют пользователям запрашивать только определенное подмножество объектов, соответствующих запросу; например, при фильтрации списка сайтов по статусу или региону. NetBox использует библиотеку [`django-filters`](https://django-filter.readthedocs.io/en/stable/) для создания и применения наборов фильтров для моделей. Мы можем создавать наборы фильтров, чтобы включить ту же функциональность и в наш плагин.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout Step07-navigation`.

## Создание набора фильтров

Начните с создания файла filtersets.py в каталоге netbox_access_lists/.

```bash
$ cd netbox_access_lists/
$ edit filtersets.py
```

В верхней части этого файла мы импортируем класс `NetBoxModelFilterSet` NetBox, который будет служить базовым классом для нашего набора фильтров, а также нашу модель `AccessListRule`. (Для краткости мы собираемся создать набор фильтров только для одной модели, но должно быть ясно, как повторить этот подход и для модели AccessList.)

```python
from netbox.filtersets import NetBoxModelFilterSet
from .models import AccessListRule
```

Затем создайте класс с именем AccessListRuleFilterSet, являющийся подклассом NetBoxModelFilterSet. Внутри этого класса создайте дочерний класс «Мета» и определите атрибуты «модель» и «поля» набора фильтров. (Вы можете заметить, что это выглядит знакомо; это очень похоже на процесс создания формы модели.) Параметр «fields» должен перечислить все поля модели, по которым мы можем захотеть фильтровать.

```python
class AccessListRuleFilterSet(NetBoxModelFilterSet):

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')
```

`NetBoxModelFilterSet` выполняет некоторые важные для нас функции, включая поддержку фильтрации по значениям пользовательских полей и тегам. Он также создает фильтр q общего назначения, который вызывает метод search(). (По умолчанию это ничего не дает.) Мы можем переопределить этот метод, чтобы определить нашу логику поиска общего назначения. Давайте добавим метод search после дочернего класса Meta, чтобы переопределить поведение по умолчанию.

```python
    def search(self, queryset, name, value):
        return queryset.filter(description__icontains=value)
```

Это вернет все правила, описание которых содержит запрошенную строку. Конечно, вы можете расширить это значение, чтобы оно соответствовало и другим полям, но для наших целей этого должно быть достаточно.

## Создание формы фильтра

Набор фильтров обрабатывает «закулисный» процесс фильтрации запросов, но нам также необходимо создать класс формы для отображения полей фильтра в пользовательском интерфейсе. Мы добавим это в «forms.py». Сначала импортируйте модуль «forms» Django (который предоставит нужные нам классы полей) и добавьте «NetBoxModelFilterSetForm» к существующему оператору импорта для «netbox.forms»:

```python
from django import forms
# ...
from netbox.forms import NetBoxModelForm, NetBoxModelFilterSetForm
```

Затем создайте класс формы с именем AccessListRuleFilterForm, создающий подкласс NetBoxModelFilterSetForm, и объявите атрибут с именем model, ссылающийся на AccessListRule (который уже был импортирован для одной из существующих форм).

```python
class AccessListRuleFilterForm(NetBoxModelFilterSetForm):
    model = AccessListRule
```

:blue_square: **Примечание:** Обратите внимание, что атрибут `model` объявлен непосредственно в классе: нам не нужен дочерний класс `Meta`.

Далее нам нужно определить поле формы для каждого фильтра, который мы хотим отображать в пользовательском интерфейсе. Начнем с фильтра access_list: он ссылается на связанный объект, поэтому нам нужно использовать ModelMultipleChoiceField (чтобы позволить пользователям фильтровать по нескольким объектам). Добавьте поле формы с тем же именем, что и его одноранговый фильтр, указав набор запросов, который будет использоваться при выборке связанных объектов.

```python
    access_list = forms.ModelMultipleChoiceField(
        queryset=AccessList.objects.all(),
        required=False
    )
```

Обратите внимание, что мы также установили `required=False`: так должно быть для _всех_ полей в форме фильтра, поскольку фильтры никогда не являются обязательными.

:blue_square: **Примечание:** Для этого поля мы используем класс `ModelMultipleChoiceField` Django вместо `DynamicModelChoiceField` NetBox, поскольку последний требует функциональной конечной точки REST API для модели. Как только мы реализуем REST API на девятом этапе, вы можете вернуться к этой форме и изменить `access_list` на `DynamicModelChoiceField`.

Далее мы добавим поле для фильтра «позиция». Это целочисленное поле, поэтому «IntegerField» должен работать хорошо:

```python
    index = forms.IntegerField(
        required=False
    )
```

Наконец, мы добавим поля для фильтров на основе выбора протокола и действия. «MultipleChoiceField» следует использовать, чтобы позволить пользователям выбирать один или несколько вариантов. При объявлении этих полей мы должны передать набор допустимых вариантов выбора, поэтому сначала расширьте соответствующий оператор импорта в верхней части `forms.py`:

```python
from .models import AccessList, AccessListRule, ActionChoices, ProtocolChoices
```

Затем добавьте поля формы в AccessListRuleFilterForm:

```python
    protocol = forms.MultipleChoiceField(
        choices=ProtocolChoices,
        required=False
    )
    action = forms.MultipleChoiceField(
        choices=ActionChoices,
        required=False
    )
```

## Обновление отображения

Последний шаг, прежде чем мы сможем использовать наш новый набор фильтров и форму, — это включить их в виде списка модели. Откройте «views.py» и расширьте последний оператор импорта, включив в него модуль «filtersets»:

```python
from . import filtersets, forms, models, tables
```

Затем добавьте атрибуты filterset и filterset_form в AccessListRuleListView:

```python
class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable
    filterset = filtersets.AccessListRuleFilterSet
    filterset_form = forms.AccessListRuleFilterForm
```

Убедившись, что сервер разработки перезапущен, перейдите к представлению списка правил в браузере. Теперь вы должны увидеть вкладку «Фильтры» рядом с вкладкой «Результаты». Под ним мы найдем четыре поля, которые мы создали в AccessListRuleFilterForm, а также встроенное поле «поиск»..

![Access list rules filter form](/images/step08-filter-form.png)

Если вы еще этого не сделали, создайте еще несколько списков доступа и правил и поэкспериментируйте с фильтрами. Подумайте, как можно фильтровать данные по дополнительным полям или добавить более сложную логику в набор фильтров.

:green_circle: **Совет:** Вы можете заметить, что мы не добавили поле формы для фильтра `id` модели: это связано с тем, что оно вряд ли будет полезно для человека, использующего пользовательский интерфейс. Однако мы по-прежнему хотим поддерживать фильтрацию объектов по их первичным ключам, поскольку это _очень_ полезно для потребителей REST API NetBox, о котором мы поговорим далее.

<div align="center">

:arrow_left: [Step 7: Навигация](/tutorial/step07-navigation.md) | [Step 9: REST API](/tutorial/step09-rest-api.md) :arrow_right:

</div>

