# Step 10: GraphQL API

В дополнение к REST API NetBox также имеет API [GraphQL](https://graphql.org/). Это можно использовать для удобного запроса произвольных коллекций данных об объектах NetBox. API GraphQL NetBox создан с использованием [Graphene](https://graphene-python.org/) и [`graphene-django`](https://docs.graphene-python.org/projects/django/en/latest/) библиотек.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step09-rest-api`.

Начните с создания `graphql.py`. Здесь будут храниться наш тип объекта и классы запросов.

```bash
$ cd netbox_access_lists/
$ edit graphql.py
```
Нам нужно будет импортировать несколько ресурсов. Сначала нам понадобится класс ObjectType в Graphene, а также собственный NetBoxObjectType в NetBox, который наследуется от него. (Последний будет использоваться для наших типов моделей.) Нам также понадобятся классы ObjectField и ObjectListField, предоставляемые NetBox для нашего запроса. И, наконец, импортируйте модули «модели» и «наборы фильтров» нашего плагина.

```python
from graphene import ObjectType
from netbox.graphql.types import NetBoxObjectType
from netbox.graphql.fields import ObjectField, ObjectListField
from . import filtersets, models
```

## Создание типов объектов

Подкласс NetBoxObjectType для создания двух классов типов объектов, по одному для каждой из наших моделей. Как и в случае с серилизаторами REST API, создайте дочерний класс Meta для каждого определения его модели и полей. Однако вместо явного перечисления каждого поля по имени в нашем случае мы можем использовать специальное значение `__all__`, чтобы указать, что мы хотим включить все доступные поля модели. Кроме того, объявите filterset_class в AccessListRuleType, чтобы прикрепить набор фильтров.

```python
class AccessListType(NetBoxObjectType):

    class Meta:
        model = models.AccessList
        fields = '__all__'


class AccessListRuleType(NetBoxObjectType):

    class Meta:
        model = models.AccessListRule
        fields = '__all__'
        filterset_class = filtersets.AccessListRuleFilterSet
```

## Создание запроса

Затем нам нужно создать наш класс запроса. Подкласс Graphene's ObjectType и определите два поля для каждой модели: поле объекта и поле списка.

```python
class Query(ObjectType):
    access_list = ObjectField(AccessListType)
    access_list_list = ObjectListField(AccessListType)

    access_list_rule = ObjectField(AccessListRuleType)
    access_list_rule_list = ObjectListField(AccessListRuleType)
```

Затем нам просто нужно предоставить наш класс запроса платформе плагинов как «схему»:

```python
schema = Query
```

:green_circle: **Совет:** Путь к классу запроса можно изменить, установив `graphql_schema` в классе конфигурации плагина.

Чтобы опробовать API GraphQL, откройте `<http://netbox:8000/graphql/>` в браузере и введите следующий запрос:

```
query {
  access_list_list {
    id
    name
      rules {
      index
      action
      description
    }
  }
}
```

Вы должны получить ответ с указанием идентификатора, имени и правил для каждого списка доступа в NetBox. Для каждого правила будет указан его индекс, действие и описание. Поэкспериментируйте с различными запросами, чтобы увидеть, какие еще данные вы можете запросить. (Для вдохновения обратитесь к определениям моделей.)

![GraphiQL interface](/images/step10-graphiql.png)

На этом руководство по разработке плагина завершено. Отличная работа! Теперь все готово для создания собственного плагина!

<div align="center">

:arrow_left: [Step 9: REST API](/tutorial/step09-rest-api.md) | [Step 11: Поиск](/tutorial/step11-search.md) :arrow_right:

</div>

