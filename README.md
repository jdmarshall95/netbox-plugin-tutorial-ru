# Руководство по разработке плагинов для NetBox

Это руководство призвано продемонстрировать процесс разработки пользовательского плагина для NetBox версии 3.2 или более поздней. Следуя каждому из предписанных шагов, читатель создаст с нуля простой плагин для управления списками доступа в NetBox, используя все основные компоненты фреймворка плагинов NetBox .

Полная копия демонстрационного плагина, созданного в этом руководстве, доступна в [`netbox-plugin-demo`].(https://github.com/netbox-community/netbox-plugin-demo ) хранилище для справки. Для вашего удобства завершенный код, соответствующий каждому шагу в руководстве, существует в виде именованной ветви в демонстрационном репозитории. Например, если вы хотите начать с шага 5, просто переключитесь на ветку `step04-forms`.

### Подготовка

Прежде чем пытаться создать плагин, пожалуйста, оцените свои личные способности. Авторы плагинов должны обладать достаточными знаниями в следующих областях:

* Программирование на Python
* Фреймворк [Django](https://www.djangoproject.com/)
* Основы REST API (где применимо)
* Установка, настройка и использование NetBox

### Оглавление

* [Шаг 1: Initial Setup](/tutorial/step01-initial-setup.md) :arrow_left: Начать здесь!
* [Шаг 2: Models](/tutorial/step02-models.md)
* [Шаг 3: Tables](/tutorial/step03-tables.md)
* [Шаг 4: Forms](/tutorial/step04-forms.md)
* [Шаг 5: Views](/tutorial/step05-views.md)
* [Шаг 6: Templates](/tutorial/step06-templates.md)
* [Шаг 7: Navigation](/tutorial/step07-navigation.md)
* [Шаг 8: Filter Sets](/tutorial/step08-filter-sets.md)
* [Шаг 9: REST API](/tutorial/step09-rest-api.md)
* [Шаг 10: GraphQL API](/tutorial/step10-graphql-api.md)
* [Шаг 11: Search](/tutorial/step11-search.md)

### Документация

* [NetBox Plugin Development Documentation](https://netbox.readthedocs.io/en/stable/plugins/development/)

### Getting Help

If you run into any snags working through the tutorial, please join us in the **#netbox** channel on the [NetDev Community Slack](https://netdev.chat/) for help.

### Feedback and Issues

If you happen to uncover an error or discrepancy in the tutorial, please be sure to [open an issue](https://github.com/netbox-community/netbox-plugin-tutorial/issues/new/choose) so that it can be documented and fixed.

