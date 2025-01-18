# Пример написания плагинов для TestY

В данном примере написан плагин для импорта тест-сьютов и тест-кейсов из .xlsx файла формата:

|   |        A         |        B         |           C           | 
|:-:|:----------------:|:----------------:|:---------------------:|
| 1 | Имя Тест-Сьюта 1 | Имя Тест-Кейса 1 | Сценарий Тест-Кейса 1 |
| 2 |                  | Имя Тест-Кейса 2 | Сценарий Тест-Кейса 2 | 
| 3 |                  | Имя Тест-Кейса 3 | Сценарий Тест-Кейса 3 |
| 4 | Имя Тест-Сьюта 2 | Имя Тест-Кейса 4 | Сценарий Тест-Кейса 4 |
| 5 |                  | Имя Тест-Кейса 5 | Сценарий Тест-Кейса 5 |

В результате создастся два тест-сьюта, первый с тест-кейсами 1-3, второй c тест-кейсами 4-5

# Перед началом разработки
Установите TestY в ваше виртуальное окружение python, чтобы ide давала подсказки к
импортам сервисов/моделей/хуков из ядра TestY.   
`pip install git+https://gitlab-pub.yadro.com/testy/testy.git`  
Если все сделано правильно в вашем коде не возникнет никаких ошибок импорта в следующем примере
```python
from testy.plugins.hooks import TestyPluginConfig, hookimpl
```
Для того чтобы проверить локально что ваш плагин работает и для debug целей вы можете установить его  
с помощью ```pip install -e path/to/plugin/root```
# Содеримое плагина

## Файл setup.py

setup.py — это основной скрипт в Python, который используется для распространения и установки модулей. Этот файл
позволяет корректно установить Python пакет на другой компьютер или в другую среду разработки.

В функцию setup передаются следующие параметры:

1. package_name: Имя основного модуля плагина
2. version: версия плагина
3. description: описание плагина
4. packages: список путей ко всем вашим файлам python (используется функция find_packages)
5. install_requires: список имен и версий пакетов (точно как в файле requirements.txt)
6. include_package_data: указывает, включать ли данные пакета при установке плагина.
7. zip_safe: указывает, может ли плагин быть установлен и запущен в виде архива ZIP.
8. entry_points: Указывает точки входа для плагина, нужно указывать обязательно, используется pluggy для нахождения
   имплементаций необходимых хуков  
   Пример использования ```entry_points={'testy': ['allure-uploader-v2=allure_uploader_v2']}``` ключом в entry_points
   для
   плагинов testy будет `testy`

## Файл MANIFEST.in

Если в setup.py параметр include_package_data = True, то в данном файле следует указать какие файлы следует
включить в сборку, по умолчанию из плагина тянутся python модули, если нужны файлы с раширением .html
путь до файлов должен быть указан в `MANIFEST.in`

## Директория plugin_example

### Директория templates

В данной директории находятся .html файлы, которые используются для серверного рендеринга (используя django templates)

Статические файлы (css, js, etc) следует помещать в директорию static

### Файл \_\_init\_\_.py

[/URLS_SENTINEL/]: # ()
В данном файле находится код, который нужно выполнить при импортировании модулей из пакета, в нем следует импортировать
класс, который задает конфигурацию плагина:

```python
# Импортировать хуки из TestY
from testy.plugins.hooks import TestyPluginConfig, hookimpl


# Сконфигурировать свой плагин
class AllureUploaderConfig(TestyPluginConfig):
    package_name = 'allure_uploader_v2'
    verbose_name = 'Allure uploader v2'
    description = 'Upload your allure report into testy'
    version = '2.0.1'
    plugin_base_url = 'allure-uploader-v2'
    author = 'Roman Kabaev'
    index_reverse_name = 'config-list'
    # urls_module должен быть валидным путем импорта url-ов плагина
    # если плагин не подразумевает наличие url-ов в качестве аргумента передается URLS_SENTINEL
    urls_module = 'allure_uploader_v2.urls'


# Имплементация pluggy хука как точки входа в тести для вашей конфигурации плагина
@hookimpl
def config():
    return AllureUploaderConfig
```

Основные моменты при создании класса конфигурации:

1. package_name: имя плагина, **должно совпадать с названием директории**
2. plugin_base_url: задает эндпоинт в TestY, по которому будет доступен плагин (
   _https://testy.host/plugins/{plugin_base_url}/_)
3. index_reverse_name: имя по которому будет django reverse будет искать index страницу плагина,

## Директория xlsx_parser_lib

Здесь находятся основные файлы для обработки данных.

**ВАЖНО:** Делайте все ваши импорты от src директории вашего плагина
```
plugin_root
│
├── src
│   ├── lib
│   │    ├── serializers.py
│   │    └── utils.py
│   ├── views.py
│   ├── model.py
│   └── urls.py
├── setup.cfg
├── setup.py
└── .gitignore
```
```python
# Пример того как должны выглядеть импорты в коде
# views.py
from src.lib.utils import some_util
```
```python
# serializers.py
from src.lib.utils import some_serialization_util
```
```python
# module as a whole
import src.lib.utils as utils
```
## Файл views.py

Содержит view классы в которых содержатся методы плагина

**ВАЖНО:** revers-names это имена которые помогают разрешать url-ы по читаемым именам например если вы используете
серверный
рендеринг django, он использует jinja темплейты и ваша ссылка может быть разрешена следующим образом:  
```<a class="nav-link" href="{% url 'plugins:allure_uploader_v2:config-list' %}">Configs</a>```
Формат reverse имени по которому будет разрешаться url следующий:  
```plugins:<your_app_label>:<your view name>```  
Пример:

```python
# Пример разрешения url-ов
# urls.py
urlpatterns = [path('', views.ProjectListView.as_view(), name='index')]
```

```python
# views.py
class UploadFileApiView(CreateAPIView):
    serializer_class = Serializer

    def create(self, request, *args, **kwargs):
        ...
        return redirect(reverse('plugins:plugin_example_v2:index'))
```

## Файл urls

# Установка плагина
1. ```shell
   pip install git + https://gitlab-pub.yadro.com/testy/allure-uploader-v2.git
   ```
    Если плагин опубликован на pypi его можно установить по имени пакета
   ```shell
    pip install plugin-example
   ```
2. Для добавления вашего плагина в список зависимостей 
   1. добавьте зависимость в файл `testy/requirements/requirements.in`
   2. Выполните следующие команды
   ```shell
      cd testy/requirements
      pip-compile dev-requirements.in
      pip-compile requirements.in
      ```
3. Запустите TestY любым удобным вам способом 