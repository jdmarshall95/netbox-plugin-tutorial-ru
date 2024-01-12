# Шаг 6: Шаблоны

Шаблоны отвечают за отображение HTML-содержимого для представлений NetBox. Каждый шаблон существует в виде файла, содержащего смесь HTML и кода шаблона. Вообще говоря, каждая модель в плагине NetBox должна иметь свой собственный шаблон. Шаблоны также можно создавать или настраивать для других представлений, но шаблоны по умолчанию, предоставляемые NetBox, подходят в большинстве случаев.

Серверная часть рендеринга NetBox использует [язык шаблонов Django](https://docs.djangoproject.com/en/stable/topics/templates/) (DTL). Если вы использовали [Jinja2](https://jinja2docs.readthedocs.io/en/stable/), он сразу покажется вам очень знакомым, но имейте в виду, что между ними есть некоторые важные различия. Как правило, DTL гораздо более ограничен в типах логики, которую он может выполнять: например, прямое выполнение кода Python невозможно. Обязательно изучите документацию Django, прежде чем пытаться создавать какие-либо сложные шаблоны.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step05-views`.

## Структура файла шаблона

NetBox ищет шаблоны в каталоге `templates/` (если он существует) в корне плагина. В этом каталоге создайте подкаталог с именем плагина:

```bash
$ cd netbox_access_lists/
$ mkdir -p templates/netbox_access_lists/
```

Файлы шаблонов будут находиться в этом каталоге. Шаблоны по умолчанию предоставляются для всех общих представлений, кроме ObjectView, поэтому нам нужно будет создать шаблоны для наших представлений AccessListView и AccessListRuleView.

По умолчанию каждый подкласс ObjectView будет искать шаблон, содержащий имя связанной с ним модели. Например, AccessListView будет искать accesslist.html. Это можно переопределить, установив в представлении `template_name`, но такое поведение подходит для наших целей.

## Создание шаблона AccessList

Начните с создания файла accesslist.html в каталоге шаблонов плагина.

```bash
$ edit templates/netbox_access_lists/accesslist.html
```

Хотя нам нужно создать собственный шаблон, NetBox проделал за нас большую часть работы и предоставляет универсальный шаблон, который мы можем легко расширить. В верхней части файла добавьте тег «extends»:

```
{% extends 'generic/object.html' %}
```

Это указывает механизму рендеринга сначала загрузить шаблон NetBox в `generic/object.html` и заполнить только тот контент, который мы предоставляем в тегах `block`.

Давайте расширим блок содержимого общего шаблона некоторой информацией о списке доступа.

```
{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Name</th>
              <td>{{ object.name }}</td>
            </tr>
            <tr>
              <th scope="row">Default Action</th>
              <td>{{ object.get_default_action_display }}</td>
            </tr>
            <tr>
              <th scope="row">Rules</th>
              <td>{{ object.rules.count }}</td>
            </tr>
          </table>
        </div>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
    </div>
    <div class="col col-md-6">
      {% include 'inc/panels/tags.html' %}
      {% include 'inc/panels/comments.html' %}
    </div>
  </div>
{% endblock content %}
```

Здесь мы создали строку Boostrap 5 и два элемента столбца. В первом столбце у нас есть простая карточка для отображения названия списка доступа и действия по умолчанию, а также количества назначенных ему правил. А под ним вы увидите тег include, который подключает дополнительный шаблон для отображения любых настраиваемых полей, связанных с моделью. Во втором столбце мы добавили еще два шаблона для отображения тегов и комментариев.

:green_circle: **Совет:** Если вы не уверены, как лучше всего построить макет страницы, в основных шаблонах NetBox можно найти множество примеров.

Давайте посмотрим на наш новый шаблон! Снова перейдите к представлению списка (по адресу <http://localhost:8000/plugins/access-lists/access-lists/>) и перейдите по ссылке к определенному списку доступа. Вы должны увидеть что-то вроде изображения ниже.

:blue_square: **Примечание:** Если NetBox сообщает, что шаблон все еще не существует, возможно, вам придется вручную перезапустить сервер разработки (`manage.py runserver`).

![Access list view](/images/step06-accesslist1.png)

This is nice, but it would be handy to include the access list's assigned rules on the page as well.

### Добавление таблицы правил

Чтобы включить правила списка доступа, нам нужно будет предоставить дополнительные _контекстные данные_ под представлением. Откройте «views.py» и найдите класс «AccessListView». (Это должен быть первый определенный класс.) Добавьте к этому классу метод get_extra_context() согласно приведенному ниже коду.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()

    def get_extra_context(self, request, instance):
        table = tables.AccessListRuleTable(instance.rules.all())
        table.configure(request)

        return {
            'rules_table': table,
        }
```

Этот метод делает три вещи:

1. Создайет экземпляр AccessListRuleTable с набором запросов, соответствующим всем правилам, назначенным этому списку доступа.
2. Настраивает экземпляр таблицы в соответствии с текущим запросом (с учетом предпочтений пользователя).
3. Возвращает словарь контекстных данных, ссылающийся на экземпляр таблицы.

Это делает таблицу доступной для нашего шаблона как контекстную переменную Rules_table. Добавим его в наш шаблон.

Во-первых, нам нужно импортировать тег render_table из библиотеки django-tables2, чтобы мы могли отображать таблицу в формате HTML. Добавьте это вверху шаблона, сразу под строкой `{% extends 'generic/object.html' %}`:

```
{% load render_table from django_tables2 %}
```

Затем сразу над строкой `{% endblock content %}` в конце файла вставьте следующий код шаблона:

```
  <div class="row">
    <div class="col col-md-12">
      <div class="card">
        <h5 class="card-header">Rules</h5>
        <div class="card-body table-responsive">
          {% render_table rules_table %}
        </div>
      </div>
    </div>
  </div>
```

После обновления списка доступа в браузере вы должны увидеть таблицу правил внизу страницы.

![Access list view with rules table](/images/step06-accesslist2.png)

## Создание шаблона AccessListRule

Говоря о правилах, давайте не будем забывать о нашей модели AccessListRule: ей тоже нужен шаблон. Создайте `accesslistrule.html` рядом с нашим первым шаблоном:

```bash
$ edit templates/netbox_access_lists/accesslistrule.html
```

И скопируйте содержимое ниже:

```
{% extends 'generic/object.html' %}

{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List Rule</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Access List</th>
              <td>
                <a href="{{ object.access_list.get_absolute_url }}">{{ object.access_list }}</a>
              </td>
            </tr>
            <tr>
              <th scope="row">Index</th>
              <td>{{ object.index }}</td>
            </tr>
            <tr>
              <th scope="row">Description</th>
              <td>{{ object.description|placeholder }}</td>
            </tr>
          </table>
        </div>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
      {% include 'inc/panels/tags.html' %}
    </div>
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Details</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Protocol</th>
              <td>{{ object.get_protocol_display }}</td>
            </tr>
            <tr>
              <th scope="row">Source Prefix</th>
              <td>
                {% if object.source_prefix %}
                  <a href="{{ object.source_prefix.get_absolute_url }}">{{ object.source_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Source Ports</th>
              <td>{{ object.source_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Destination Prefix</th>
              <td>
                {% if object.destination_prefix %}
                  <a href="{{ object.destination_prefix.get_absolute_url }}">{{ object.destination_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Destination Ports</th>
              <td>{{ object.destination_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Action</th>
              <td>{{ object.get_action_display }}</td>
            </tr>
          </table>
        </div>
      </div>
    </div>
  </div>
{% endblock content %}
```

На этом этапе вы, вероятно, сможете сказать, что делает большая часть приведенного выше кода шаблона, но вот несколько деталей, которые стоит упомянуть:

* URL-адрес родительского списка доступа правила получается путем вызова `object.access_list.get_absolute_url()` (метод, который мы добавили на пятом шаге), _без_ круглых скобок (отличие от DTL). Этот метод также используется для связанных префиксов.
* К описанию правила применяется фильтр «заполнитель» NetBox. (Это отображает &mdash; для пустых полей.)
* Атрибуты `protocol` и `action` отображаются, например, путем вызова. `object.get_protocol_display()` (опять без скобок). Это [соглашение Django](https://docs.djangoproject.com/en/stable/ref/models/instances/#extra-instance-methods) для полей статического выбора, чтобы возвращать удобную для пользователя метку, а не необработанный код.

![Access list rule view](/images/step06-accesslistrule.png)

Не стесняйтесь экспериментировать с различными макетами и контентом, прежде чем переходить к следующему шагу.

<div align="center">

:arrow_left: [Step 5: Отображения](/tutorial/step05-views.md) | [Step 7: Навигация](/tutorial/step07-navigation.md) :arrow_right:

</div>

