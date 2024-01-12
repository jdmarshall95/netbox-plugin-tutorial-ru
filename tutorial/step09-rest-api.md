# Step 9: REST API

REST API обеспечивает мощную интеграцию с другими системами, которые обмениваются данными с NetBox. Он основан на [Django REST Framework](https://www.django-rest-framework.org/) (DRF), который _не_ является компонентом самого Django. В этом уроке мы увидим, как можно расширить REST API NetBox для обслуживания нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step08-filter-sets`.

Наш код API будет находиться в каталоге `api/` в `netbox_access_lists/`. Давайте продолжим и создадим его, а также файл `__init__.py`:

```bash
$ cd netbox_access_lists/
$ mkdir api
$ touch api/__init__.py
```

## Создание сериализаторов моделей

Сериализаторы в чем-то аналогичны формам: они управляют переводом клиентских данных в объекты Python и обратно, в то время как сам Django обрабатывает абстракцию базы данных. Нам нужно создать сериализатор для каждой из наших моделей. Начните с создания файла `serializers.py` в каталоге `api/`.

```bash
$ edit api/serializers.py
```

В верхней части этого файла нам нужно импортировать модуль сериализаторов из библиотеки rest_framework, а также класс NetBoxModelSerializer NetBox и собственные модели нашего плагина:

```python
from rest_framework import serializers

from netbox.api.serializers import NetBoxModelSerializer
from ..models import AccessList, AccessListRule
```
### Создание сериализатора AccessListSerializer

Сначала мы создадим сериализатор для AccessList, создав подкласс NetBoxModelSerializer. Как и при создании формы модели, мы создадим дочерний класс «Мета» в сериализаторе, указав связанную «модель» и включаемые «поля».

```python
class AccessListSerializer(NetBoxModelSerializer):

    class Meta:
        model = AccessList
        fields = (
            'id', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

Стоит обсудить каждую из областей, которые мы назвали выше. `id` — первичный ключ модели; его всегда следует включать в каждый сериализатор, поскольку он обеспечивает гарантированный метод уникальной идентификации объектов. Поле display встроено в NetBoxModelSerializer: это поле, доступное только для чтения, которое возвращает строковое представление объекта. Это полезно, например, для заполнения раскрывающихся списков полей формы.

Поля name, default_action и comment объявляются в модели AccessList. `tags` обеспечивает доступ к диспетчеру тегов объекта, а `custom_fields` включает данные его настраиваемого поля; оба они предоставляются NetBoxModelSerializer. Наконец, поля «created» и «last_updated» доступны только для чтения и встроены в NetBoxModel.

Наш сериализатор проверит модель и автоматически сгенерирует необходимые поля, однако есть одно поле, которое нам нужно добавить вручную. Каждый сериализатор должен включать поле URL только для чтения, содержащее URL-адрес, по которому можно получить доступ к объекту; думайте об этом как о методе модели `get_absolute_url()`. Чтобы добавить это, мы будем использовать HyperlinkedIdentityField из DRF. Добавьте его над дочерним классом Meta:

```python
class AccessListSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )
```

При вызове класса поля нам необходимо указать соответствующее имя представления. Обратите внимание, что это представление на самом деле еще не существует; мы создадим его чуть позже.

Помните, на третьем этапе мы добавили столбец таблицы, показывающий количество правил, назначенных каждому списку доступа? Это было удобно. Давайте добавим и для него поле сериализатора! Добавьте это прямо под полем URL:

```python
rule_count = serializers.IntegerField(read_only=True)
```

Как и в случае со столбцом таблицы, мы будем полагаться на наше представление (которое будет определено далее) для аннотирования количества правил для каждого списка доступа в базовом наборе запросов.

Наконец, нам нужно добавить URL и Rule_count в Meta.fields:

```python
    class Meta:
        model = AccessList
        fields = (
            'id', 'url', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated', 'rule_count',
        )
```

:green_circle: **Совет:** Порядок перечисления полей определяет порядок их появления в представлении API объекта.

### Создать сериализатор AccessListRuleSerializer

Нам также необходимо создать сериализатор для AccessListRule. Добавьте его в `serializers.py` ниже `AccessListSerializer`. Как и в случае с первым сериализатором, мы добавим класс Meta для определения модели и полей, а также поле URL.

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )

    class Meta:
        model = AccessListRule
        fields = (
            'id', 'url', 'display', 'access_list', 'index', 'protocol', 'source_prefix', 'source_ports',
            'destination_prefix', 'destination_ports', 'action', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

При ссылке на связанные объекты в сериализаторе необходимо учитывать дополнительные моменты. По умолчанию сериализатор вернет только первичный ключ связанного объекта; его цифровой идентификатор. Это требует от клиента выполнения дополнительных запросов API, чтобы определить _любую_ другую информацию о связанном объекте. Удобно автоматически включать в сериализатор некоторую информацию о связанном объекте, например его имя и URL-адрес. Мы можем сделать это, используя _вложенный сериализатор_.

Например, поля «source_prefix» и «destination_prefix» ссылаются на базовую модель NetBox «ipam.Prefix». Мы можем расширить AccessListRuleSerializer, чтобы использовать вложенный сериализатор NetBox для этой модели:

```python
from ipam.api.serializers import NestedPrefixSerializer
# ...
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

Теперь наш сериализатор будет включать сокращенное представление префиксов источника и/или назначения для объекта. Нам следует сделать то же самое и с полем access_list, однако сначала нам нужно создать вложенный сериализатор для модели AccessList.

### Создание вложенных сериализаторов

Начните с импорта класса NetBox `WritableNestedSerializer`. Он будет служить базовым классом для наших вложенных сериализаторов.

```python
from netbox.api.serializers import NetBoxModelSerializer, WritableNestedSerializer
```

Затем создайте два вложенных класса сериализатора, по одному для каждой модели нашего плагина. Каждый из них будет иметь поле `url` и дочерний класс `Meta`, как и обычные сериализаторы, однако атрибут `Meta.fields` для каждого из них ограничен минимальным количеством полей: `id`, `url`, `display `и дополнительный удобный идентификатор. Добавьте их в `serializers.py` _над_ обычными сериализаторами (потому что нам нужно определить `NestedAccessListSerializer`, прежде чем мы сможем ссылаться на него).

```python
class NestedAccessListSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )

    class Meta:
        model = AccessList
        fields = ('id', 'url', 'display', 'name')

class NestedAccessListRuleSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )

    class Meta:
        model = AccessListRule
        fields = ('id', 'url', 'display', 'index')
```

Теперь мы можем переопределить поле access_list в AccessListRuleSerializer, чтобы использовать вложенный сериализатор:

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )
    access_list = NestedAccessListSerializer()
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

## Создание отображений

Далее нам нужно создать представления для обработки логики API. Подобно тому, как сериализаторы примерно аналогичны формам, представления API работают аналогично представлениям пользовательского интерфейса, которые мы создали на пятом шаге. Однако, поскольку функциональность API высоко стандартизирована, создание представлений существенно упрощается: обычно нам нужно создать только один _набор_ представлений_ для каждой модели. Набор представлений — это единый класс, который может обрабатывать операции просмотра, добавления, изменения и удаления, каждая из которых требует выделенных представлений в пользовательском интерфейсе.

Начните с создания `api/views.py` и импорта класса `NetBoxModelViewSet` NetBox, а также модулей `models` и `filtersets` нашего плагина и наших сериализаторов.

```python
from netbox.api.viewsets import NetBoxModelViewSet

from .. import filtersets, models
from .serializers import AccessListSerializer, AccessListRuleSerializer
```

Сначала мы создадим набор представлений для списков доступа, унаследовав его от `NetBoxModelViewSet` и определив его атрибуты `queryset` и`serializer_class`. (Обратите внимание, что мы предварительно извлекаем назначенные теги для набора запросов.)

```python
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags')
    serializer_class = AccessListSerializer
```

Вспомним, что мы добавили поле rule_count в AccessListSerializer; давайте соответствующим образом аннотируем набор запросов, чтобы гарантировать, что поле будет заполнено (так же, как мы это сделали для столбца таблицы на пятом шаге). Не забудьте импортировать служебный класс `Count` Django.

```python
from django.db.models import Count
# ...
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags').annotate(
        rule_count=Count('rules')
    )
    serializer_class = AccessListSerializer
```

Далее мы добавим набор представлений для правил. В дополнение к `queryset` и `serializer_class` мы прикрепим набор фильтров для этой модели как `filterset_class`. Обратите внимание, что мы также предварительно загружаем все связанные поля объекта в дополнение к тегам, чтобы повысить производительность при перечислении большого количества объектов.

```python
class AccessListRuleViewSet(NetBoxModelViewSet):
    queryset = models.AccessListRule.objects.prefetch_related(
        'access_list', 'source_prefix', 'destination_prefix', 'tags'
    )
    serializer_class = AccessListRuleSerializer
    filterset_class = filtersets.AccessListRuleFilterSet
```

## Создание URL-адресов конечных точек

Наконец, мы создадим URL-адреса конечных точек API. Это работает немного иначе, чем представления пользовательского интерфейса: вместо определения ряда путей мы создаем экземпляр _router_ и регистрируем для него каждое установленное представление.

Создайте `api/urls.py` и импортируйте `NetBoxRouter` NetBox и наши представления API:

```python
from netbox.api.routers import NetBoxRouter
from . import views
```

Далее мы определим `app_name`. Это будет использоваться для разрешения имен представлений API для нашего плагина.

```python
app_name = 'netbox_access_list'
```

Затем мы создаем экземпляр NetBoxRouter и регистрируем в нем каждое представление, используя желаемый URL-адрес. Это конечные точки, которые будут доступны в каталоге `/api/plugins/access-lists/`.

```python
router = NetBoxRouter()
router.register('access-lists', views.AccessListViewSet)
router.register('access-list-rules', views.AccessListRuleViewSet)
```

Наконец, мы раскрываем атрибут `urls` маршрутизатора как `urlpatterns`, чтобы он был обнаружен платформой плагинов.

```python
urlpatterns = router.urls
```

:green Circle: **Совет:** Базовый URL-адрес для конечных точек REST API вашего плагина определяется атрибутом `base_url` класса конфигурации плагина, который мы создали на первом этапе.

Теперь, когда все наши компоненты REST API готовы, мы сможем отправлять запросы API. (Обратите внимание, что сначала вам может потребоваться предоставить токен для аутентификации.) Вы можете быстро убедиться, что наши конечные точки работают правильно, перейдя по адресу <http://localhost:8000/api/plugins/access-lists/> в своем браузере, пока залогинился в NetBox. Вы должны увидеть две доступные конечные точки; нажатие на любой из них вернет список объектов.

:blue_square: **Примечание:** Если конечные точки REST API не загружаются, попробуйте перезапустить сервер разработки (`manage.py runserver`).

![REST API - Root view](/images/step09-rest-api1.png)

![REST API - Access list rules](/images/step09-rest-api2.png)

<div align="center">

:arrow_left: [Step 8: Наборы фильтров](/tutorial/step08-filter-sets.md) | [Step 10: GraphQL API](/tutorial/step10-graphql-api.md) :arrow_right:

</div>

