# Шаг 5: Отображения

Отображения отвечают за бизнес-логику вашего приложения. Как правило, это означает обработку входящих запросов, выполнение некоторых действий и возврат ответа клиенту. Каждое отображение обычно имеет связанный с ним URL-адрес и может обрабатывать один или несколько типов HTTP-запросов (например, запросы GET и/или POST).

Django предоставляет набор [универсальных классов отображений](https://docs.djangoproject.com/en/4.0/topics/class-based-views/generic-display/), которые обрабатывают большую часть шаблонного кода, необходимого для обработки запросов. NetBox также предоставляет набор классов отображений, упрощающих создание отображений для создания, редактирования, удаления и просмотра объектов. Они также предоставляют поддержку специфичных для NetBox функций, таких как настраиваемые поля и регистрация изменений.

На этом этапе мы создадим набор представлений для каждой модели нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step04-forms`.

## Создание отображения

Начните с создания файла view.py в каталоге netbox_access_lists/.

```bash
$ cd netbox_access_lists/
$ edit views.py
```

Нам нужно будет импортировать модули «модели», «таблицы» и «формы» нашего плагина: именно здесь все, что мы создали на данный момент, действительно объединяется! Нам также необходимо импортировать модуль общих представлений NetBox, поскольку он предоставляет базовые классы для наших отображений.

```python
from netbox.views import generic
from . import forms, models, tables
```

:green_circle: **Совет:** Вы заметите, что мы импортируем сюда все модули модели, формы и таблиц. Если вы предпочитаете импортировать каждый из соответствующих классов напрямую, вы, безусловно, можете это сделать; просто не забудьте соответствующим образом изменить определения классов ниже.

Для каждой модели нам необходимо создать четыре представления:

* **Detail view** - Отображение одного объекта
* **List view** - Отображает таблицу всех существующих экземпляров конкретной модели.
* **Edit view** - Управляет добавлением и изменением объектов.
* **Delete view** - Обрабатывает удаление объекта

### AccessList отображения

Общий шаблон, которому мы здесь будем следовать, заключается в создании подкласса общего класса отображения, предоставляемого NetBox, и определении необходимых атрибутов. Нам не нужно будет писать какой-либо существенный код, поскольку отображения, предоставляемые NetBox, позаботятся о логике запроса за нас.

Начнем с детального обзора. Мы создаем подкласс «generic.ObjectView» и определяем набор запросов объектов, которые хотим отображать.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()
```

:green_circle: **Совет:** Отображения требуют от нас определения набора запросов, а не просто модели, поскольку иногда необходимо изменить набор запросов, например. для предварительной выборки связанных объектов или ограничения по определенному атрибуту.

Далее мы добавим представление списка. Для этого представления нам нужно определить как «набор запросов», так и «таблицу».

```python
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.all()
    table = tables.AccessListTable
```

:green_circle: **Совет:** Автору пришло в голову, что выбор названия модели, оканчивающегося на «Список», может немного сбить с толку. Просто помните, что AccessListView — это отображение _detail_ (один объект), а AccessListListView — представление _list_ (несколько объектов).

Прежде чем мы перейдем к следующему отображению, помните ли вы дополнительный столбец, который мы добавили в AccessListTable на третьем шаге? В этом столбце ожидается найти количество правил, назначенных для каждого списка доступа в наборе запросов с именем «rule_count». Давайте добавим это в наш набор запросов сейчас. Мы можем использовать функцию Count() в Django, чтобы расширить SQL-запрос и аннотировать количество связанных правил (не забудьте добавить оператор импорта вверху.)

```python
from django.db.models import Count
# ...
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.annotate(
        rule_count=Count('rules')
    )
    table = tables.AccessListTable
```

Мы закончим отображением редактирования и удаления для AccessList. Обратите внимание, что для отображения редактирования нам также необходимо определить «форму» как класс формы, который мы создали на четвертом шаге.

```python
class AccessListEditView(generic.ObjectEditView):
    queryset = models.AccessList.objects.all()
    form = forms.AccessListForm

class AccessListDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessList.objects.all()
```

Вот и все для первой модели! Мы также создадим еще четыре отображения для AccessListRule.

### AccessListRule отображения

Остальные наши отображения следуют той же схеме, что и первые четыре.

```python
class AccessListRuleView(generic.ObjectView):
    queryset = models.AccessListRule.objects.all()

class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable

class AccessListRuleEditView(generic.ObjectEditView):
    queryset = models.AccessListRule.objects.all()
    form = forms.AccessListRuleForm

class AccessListRuleDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessListRule.objects.all()
```

Теперь, когда наши отображения готовы, нам нужно сделать их доступными, связав каждое из них с URL-адресом.

## Сопоставление отображений с URL-адресами

В каталоге netbox_access_lists/ создайте urls.py. Это определит наши URL-адреса просмотра.

```bash
$ edit urls.py
```

Сопоставление URL-адресов для плагинов NetBox во многом идентично обычным приложениям Django: мы определим `urlpatterns` как итерацию вызовов `path()`, сопоставляющую фрагменты URL-адресов для классов отображений.

Сначала нам нужно импортировать функцию «path» Django из его модуля «urls», а также модули «models» и «views» нашего плагина.

```python
from django.urls import path
from . import models, views
```

У нас есть четыре отображения для каждой модели, но на самом деле нам нужно определить пять путей для каждого. Это связано с тем, что операции добавления и редактирования обрабатываются одним и тем же отображением, но доступ к ним осуществляется через разные URL-адреса. Наряду с URL-адресом и отображением для каждого пути мы также укажем «имя»; это позволяет нам легко ссылаться на URL-адрес в коде.

```python
urlpatterns = (
    path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
    path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
    path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
    path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
    path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),
)
```

We've chosen `access-lists` as the base URL for our `AccessList` model, but you are free to choose something different. However, it is recommended to retain the naming scheme shown, as several NetBox features rely on it. Also note that each of the views must be invoked by its `as_view()` method when passed to `path()`.

:green_circle: **Tip:** The `<int:pk>` string you see in some of the URLs is a [path converter](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters). Specifically, this is an integer (`int`) variable named `pk`. This value is extracted from the request URL and passed to the view when the request is processed, so that the specified object can be located in the database.

Let's add the rest of the paths now. You may find it helpful to separate the paths by model to make the file more readable.

```python
urlpatterns = (

    # Access lists
    path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
    path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
    path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
    path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
    path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),

    # Access list rules
    path('rules/', views.AccessListRuleListView.as_view(), name='accesslistrule_list'),
    path('rules/add/', views.AccessListRuleEditView.as_view(), name='accesslistrule_add'),
    path('rules/<int:pk>/', views.AccessListRuleView.as_view(), name='accesslistrule'),
    path('rules/<int:pk>/edit/', views.AccessListRuleEditView.as_view(), name='accesslistrule_edit'),
    path('rules/<int:pk>/delete/', views.AccessListRuleDeleteView.as_view(), name='accesslistrule_delete'),

)
```
### Добавление отображений журнала изменений

Возможно, вы помните, что одной из функций NetBox является автоматическое [ведение журнала изменений](https://netbox.readthedocs.io/en/stable/additional-features/change-logging/). Вы можете увидеть это в действии, просматривая объект NetBox и выбирая его вкладку «Журнал изменений». Поскольку наши модели наследуют от NetBoxModel, они тоже могут использовать эту функцию.

Мы добавим специальный URL-адрес журнала изменений для каждой из наших моделей. Во-первых, вернувшись в начало `urls.py`, нам нужно импортировать `ObjectChangeLogView` NetBox:

```python
from netbox.views.generic import ObjectChangeLogView
```

Then, we'll add an extra path for each model inside `urlpatterns`:

```python
urlpatterns = (

    # Access lists
    # ...
    path('access-lists/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslist_changelog', kwargs={
        'model': models.AccessList
    }),

    # Access list rules
    # ...
    path('rules/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslistrule_changelog', kwargs={
        'model': models.AccessListRule
    }),

)
```

Обратите внимание, что здесь мы используем ObjectChangeLogView; нам не нужно было создавать для него подклассы для конкретной модели. Кроме того, мы передаем отображению аргумент ключевого слова «модель»: он определяет модель, которая будет использоваться (именно поэтому нам не нужно было создавать подкласс отображения).

## Добавление методов URL-адреса модели

Теперь, когда у нас есть URL-пути, мы можем добавить метод get_absolute_url() к каждой из наших моделей. Этот метод представляет собой [соглашение Django](https://docs.djangoproject.com/en/stable/ref/models/instances/#get-absolute-url); хотя это и не является строго обязательным, он удобно возвращает абсолютный URL-адрес для любого конкретного объекта. Например, вызов `accesslist.get_absolute_url()` вернет `/plugins/access-lists/access-lists/123/` (где 123 — это первичный ключ объекта).

Вернитесь в «models.py», импортируйте функцию «reverse» Django из модуля «urls» в верхней части файла:

```python
from django.urls import reverse
```

Затем добавьте метод get_absolute_url() в класс AccessList после его метода __str__():

```python
class AccessList(NetBoxModel):
    # ...
    def get_absolute_url(self):
        return reverse('plugins:netbox_access_lists:accesslist', args=[self.pk])
```

`reverse()` принимает здесь два аргумента: имя представления и список позиционных аргументов. Имя представления формируется путем объединения трех компонентов:

* The string `'plugins'`
* The name of our plugin
* The name of the desired URL path (defined as `name='accesslist'` in `urls.py`)

Атрибут объекта `pk` также передается и заменяет преобразователь пути `<int:pk>` в URL-адресе.

Мы также добавим метод get_absolute_url() для AccessListRule, соответствующим образом изменив имя представления.

```python
class AccessListRule(NetBoxModel):
    # ...
    def get_absolute_url(self):
        return reverse('plugins:netbox_access_lists:accesslistrule', args=[self.pk])
```

## Тестирование отображений

Теперь момент истины: привела ли вся наша работа к настоящему моменту к функциональным представлениям пользовательского интерфейса? Убедитесь, что сервер разработки запущен, затем откройте браузер и перейдите по адресу <http://localhost:8000/plugins/access-lists/access-lists/>. Вы должны увидеть представление списка списка доступа и (если вы выполнили действия на втором шаге) один список доступа с именем MyACL1.

:blue_square: **Примечание:** В этом руководстве предполагается, что вы используете сервер разработки Django локально на порту 8000. Если у вас другие настройки, вам необходимо соответствующим образом изменить ссылку выше.

![Access lists list view](/images/step05-accesslist-list.png)

Мы видим, что наша таблица успешно отобразила столбцы `name`, `rule_count` и `default_action`, которые мы определили на третьем шаге, а в столбце `rule_count` показаны два правила, назначенные, как и ожидалось.

Если мы нажмем кнопку «Добавить» в правом верхнем углу, мы попадем на форму создания списка доступа. (Создание нового списка доступа пока не работает, но форма должна отображаться, как показано ниже.)

![Access list creation form](/images/step05-accesslist-form.png)

Однако если вы щелкнете ссылку на список доступа в таблице, вы столкнетесь с исключением «TemplateDoesNotExist». Это означает именно то, что здесь написано: мы еще не определили шаблон для этого представления. Не волнуйтесь, это будет дальше!

:blue_square: **Примечание:** Вы могли заметить, что представление «Добавить» для правил по-прежнему не работает, вызывая исключение NoReverseMatch. Это связано с тем, что мы еще не определили серверные части REST API, необходимые для поддержки полей динамической формы. Мы позаботимся об этом, когда на девятом этапе создадим функциональность REST API.

<div align="center">

:arrow_left: [Шаг 4: Формы](/tutorial/step04-forms.md) | [Шаг 6: Шаблоны](/tutorial/step06-templates.md) :arrow_right:

</div>

