# Django Polls Application

## Часть 1: Запросы и ответы

### Что сделано:
- Создан проект Django `mysite`
- Создано приложение `polls`
- Настроены базовые URL маршруты
- Создано первое представление (view)

### Комментарии:

#### Настройка окружения:
Создал виртуальное окружение для изоляции зависимостей проекта.

python -m venv .venv
.venv\Scripts\activate
pip install django
python -m django --version


#### Структура проекта:
Django/
    .venv/              # Виртуальное окружение
    mysite/             # Главный пакет проекта
        __init__.py
        settings.py     # Настройки (TIME_ZONE, LANGUAGE_CODE, INSTALLED_APPS)
        urls.py         # Главные URL маршруты
        asgi.py
        wsgi.py
    polls/              # Приложение для опросов
        __init__.py
        admin.py        # Регистрация моделей в админке
        apps.py
        models.py       # Модели данных (Question, Choice)
        views.py        # Представления (логика обработки запросов)
        urls.py         # URL маршруты приложения (создал вручную!)
        tests.py        # Тесты
        migrations/     # Миграции БД
    manage.py           # Утилита управления проектом


---

## Часть 2: Модели и база данных

### Что сделано:
- Созданы модели Question и Choice
- Выполнены миграции базы данных
- Настроена админ-панель Django
- Протестирована работа с Django ORM в shell
- Добавлен метод was_published_recently()
- Создан суперпользователь для доступа к админке

### Комментарии:

#### Создание моделей:
Модели - это Python классы, описывающие структуру БД. 

# polls/models.py
import datetime
from django.db import models
from django.utils import timezone

class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField("date published")
    
    def __str__(self):
        return self.question_text  # Читаемое отображение
    
    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days=1) <= self.pub_date <= now

class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
    
    def __str__(self):
        return self.choice_text

#### Типы полей:
- `CharField(max_length=200)` → VARCHAR(200) в БД
- `DateTimeField` → DATETIME
- `IntegerField` → INTEGER
- `ForeignKey` → связь один-ко-многим
- `on_delete=models.CASCADE` → каскадное удаление (удалили вопрос = удалились варианты)

#### Регистрация приложения:
# mysite/settings.py
INSTALLED_APPS = [
    'polls.apps.PollsConfig',  # ← Добавил polls
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]


#### Миграции - система контроля версий для БД:
# 1. Создать файлы миграций (обнаружить изменения)
python manage.py makemigrations polls

# 2. Посмотреть SQL (опционально, для обучения)
python manage.py sqlmigrate polls 0001

# 3. Применить миграции (создать таблицы)
python manage.py migrate


#### Django ORM - работа через Shell:
python manage.py shell

from polls.models import Question, Choice
from django.utils import timezone

# CREATE - создание
q = Question(question_text="What's new?", pub_date=timezone.now())
q.save()

# READ - чтение всех
Question.objects.all()

# READ - фильтрация
Question.objects.filter(id=1)
Question.objects.filter(question_text__startswith="What")
Question.objects.filter(pub_date__year=2024)

# READ - получение одного объекта
q = Question.objects.get(pk=1)

# UPDATE - обновление
q.question_text = "What's up?"
q.save()

# DELETE - удаление
q.delete()

# Работа со связанными объектами
q.choice_set.all()  # Все варианты для вопроса
q.choice_set.create(choice_text='Not much', votes=0)
q.choice_set.count()

# Обратная связь
c = Choice.objects.get(pk=1)
c.question  # Получить связанный Question

#### Django ORM - магия:
- Не пишу SQL вручную - Django генерирует его сам
- Автоматическая защита от SQL-инъекций
- Работаю с объектами Python, а не строками SQL
- Lookup-фильтры: `__startswith`, `__contains`, `__lte`, `__gte`, `__year`
- Lazy evaluation - запрос выполняется только при обращении к данным

#### Создание суперпользователя:
python manage.py createsuperuser
Username: admin
Email: admin@example.com
Password: ********

#### Админ-панель Django:
# polls/admin.py
from django.contrib import admin
from .models import Question

admin.site.register(Question)


#### Метод __str__():
Обязательно добавляю в модели для читаемости:

def __str__(self):
    return self.question_text

Было: `<Question object (1)>`
Стало: `What's new?`

---

## Часть 3: Представления и шаблоны

### Что сделано:
- Созданы представления: index, detail, results
- Настроены HTML шаблоны
- Использованы generic views (ListView, DetailView)
- Добавлено пространство имен для URLs

### Комментарии:

#### Структура шаблонов:
polls/
    templates/
        polls/              
            index.html
            detail.html
            results.html

**Почему вложенная папка polls?**
Django ищет шаблоны в `templates/` всех приложений. Если два приложения имеют `index.html`, возникнет конфликт. Поэтому: `polls/templates/polls/index.html`

#### Представления (views) - функциональные:
# polls/views.py
from django.shortcuts import render, get_object_or_404
from .models import Question

def index(request):
    latest_question_list = Question.objects.order_by("-pub_date")[:5]
    context = {"latest_question_list": latest_question_list}
    return render(request, "polls/index.html", context)

def detail(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, "polls/detail.html", {"question": question})

def results(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, "polls/results.html", {"question": question})


#### Шаблоны Django:
<!-- polls/templates/polls/index.html -->
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}

#### Теги шаблонов:
- `{{ variable }}` - вывод переменной
- `{% for %}...{% endfor %}` - цикл
- `{% if %}...{% endif %}` - условие
- `{% url 'name' %}` - генерация URL по имени
- `{{ question.pub_date|date:"F j, Y" }}` - фильтры

#### URL с параметрами:
# polls/urls.py
from django.urls import path
from . import views

app_name = "polls"  # Пространство имен

urlpatterns = [
    path("", views.index, name="index"),
    path("<int:question_id>/", views.detail, name="detail"),
    path("<int:question_id>/results/", views.results, name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]

`<int:question_id>` - захватывает число из URL и передает в view как параметр

#### Пространство имен (app_name):
- Без него: `{% url 'detail' %}`
- С ним: `{% url 'polls:detail' %}`
- Позволяет избежать конфликтов между приложениями

#### Generic Views - меньше кода:

# polls/views.py
from django.views import generic

class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        return Question.objects.order_by("-pub_date")[:5]

class DetailView(generic.DetailView):
    model = Question
    template_name = "polls/detail.html"

class ResultsView(generic.DetailView):
    model = Question
    template_name = "polls/results.html"

#### URLs для generic views:
# polls/urls.py
urlpatterns = [
    path("", views.IndexView.as_view(), name="index"),
    path("<int:pk>/", views.DetailView.as_view(), name="detail"),
    path("<int:pk>/results/", views.ResultsView.as_view(), name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]

**Изменилось:** `question_id` → `pk` для generic views

---

## Часть 4: Формы и обработка данных

### Что сделано:
- Создана форма для голосования
- Реализована обработка POST-запросов
- Добавлена защита от CSRF
- Использован F() для атомарного обновления
- Реализован паттерн POST/Redirect/GET

### Комментарии:

#### Форма голосования в шаблоне:
<!-- polls/templates/polls/detail.html -->
<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
<fieldset>
    <legend><h1>{{ question.question_text }}</h1></legend>
    {% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}
    {% for choice in question.choice_set.all %}
        <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
        <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
    {% endfor %}
</fieldset>
<input type="submit" value="Vote">
</form>

#### CSRF защита:
`{% csrf_token %}` - обязательный токен для POST-форм в Django
Без него: 403 Forbidden
Django автоматически проверяет токен и защищает от CSRF-атак

#### Обработка голосования:
# polls/views.py
from django.db.models import F
from django.http import HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse

def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST["choice"])
    except (KeyError, Choice.DoesNotExist):
        # Вариант не выбран - показываем форму с ошибкой
        return render(request, "polls/detail.html", {
            "question": question,
            "error_message": "You didn't select a choice.",
        })
    else:
        # F() выражение - атомарное обновление
        selected_choice.votes = F("votes") + 1
        selected_choice.save()
        # POST/Redirect/GET паттерн
        return HttpResponseRedirect(reverse("polls:results", args=(question.id,)))



**3. POST/Redirect/GET паттерн:**
- POST запрос → обработка данных
- Redirect (302) → на страницу результатов
- GET запрос → показать результаты


**4. reverse()** - генерация URL по имени:
reverse("polls:results", args=(question.id,))
# Вернет: /polls/5/results/

**5. HttpResponseRedirect** - редирект после POST:
return HttpResponseRedirect(reverse("polls:results", args=(question.id,)))

#### Шаблон результатов:
<!-- polls/templates/polls/results.html -->
<h1>{{ question.question_text }}</h1>

<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
{% endfor %}
</ul>

<a href="{% url 'polls:detail' question.id %}">Vote again?</a>

`|pluralize` - автоматически добавляет "s" для множественного числа

---

## Часть 5: Автоматизированное тестирование

### Что сделано:
- Написаны тесты для модели Question
- Написаны тесты для представлений
- Исправлена ошибка в was_published_recently()
- Добавлен фильтр будущих вопросов в IndexView
- Добавлен фильтр будущих вопросов в DetailView

### Комментарии:

#### Обнаруженная ошибка:
python manage.py shell

from django.utils import timezone
import datetime
from polls.models import Question

future_question = Question(pub_date=timezone.now() + datetime.timedelta(days=30))
future_question.was_published_recently()
# True - это неправильно! Будущее != недавнее

#### Первый тест для модели:
# polls/tests.py
import datetime
from django.test import TestCase
from django.utils import timezone
from .models import Question

class QuestionModelTests(TestCase):
    def test_was_published_recently_with_future_question(self):
        """
        was_published_recently() должен возвращать False для вопросов 
        с pub_date в будущем
        """
        time = timezone.now() + datetime.timedelta(days=30)
        future_question = Question(pub_date=time)
        self.assertIs(future_question.was_published_recently(), False)

#### Запуск тестов:
python manage.py test polls

Creating test database for alias 'default'...
F
======================================================================
FAIL: test_was_published_recently_with_future_question
----------------------------------------------------------------------
AssertionError: True is not False
======================================================================
Ran 1 test in 0.001s
FAILED (failures=1)

#### Исправление ошибки:
# polls/models.py
def was_published_recently(self):
    now = timezone.now()
    return now - datetime.timedelta(days=1) <= self.pub_date <= now

#### Дополнительные тесты:
def test_was_published_recently_with_old_question(self):
    """
    Для вопросов старше 1 дня должно быть False
    """
    time = timezone.now() - datetime.timedelta(days=1, seconds=1)
    old_question = Question(pub_date=time)
    self.assertIs(old_question.was_published_recently(), False)

def test_was_published_recently_with_recent_question(self):
    """
    Для вопросов в пределах последних 24 часов должно быть True
    """
    time = timezone.now() - datetime.timedelta(hours=23, minutes=59, seconds=59)
    recent_question = Question(pub_date=time)
    self.assertIs(recent_question.was_published_recently(), True)

Теперь все 3 теста проходят

#### Проблема с IndexView:
Вопросы с pub_date в будущем отображаются в списке - это неправильно!

#### Исправление IndexView:
# polls/views.py
from django.utils import timezone

class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        """
        Возвращает последние 5 опубликованных вопросов 
        (без тех, что будут опубликованы в будущем)
        """
        return Question.objects.filter(
            pub_date__lte=timezone.now()  # lte = less than or equal
        ).order_by("-pub_date")[:5]

`pub_date__lte=timezone.now()` - важный фильтр! Показываем только прошлые вопросы.

#### Вспомогательная функция для тестов:
def create_question(question_text, days):
    """
    Создает вопрос с заданным текстом и датой публикации
    days > 0: будущее
    days < 0: прошлое
    days = 0: сейчас
    """
    time = timezone.now() + datetime.timedelta(days=days)
    return Question.objects.create(question_text=question_text, pub_date=time)

#### Тесты для IndexView:
class QuestionIndexViewTests(TestCase):
    def test_no_questions(self):
        """Если нет вопросов, показываем сообщение"""
        response = self.client.get(reverse("polls:index"))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "No polls are available.")
        self.assertQuerySetEqual(response.context["latest_question_list"], [])

    def test_past_question(self):
        """Прошлые вопросы отображаются"""
        question = create_question(question_text="Past question.", days=-30)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question],
        )

    def test_future_question(self):
        """Будущие вопросы НЕ отображаются"""
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse("polls:index"))
        self.assertContains(response, "No polls are available.")
        self.assertQuerySetEqual(response.context["latest_question_list"], [])

    def test_future_question_and_past_question(self):
        """Только прошлые отображаются, даже если есть будущие"""
        question = create_question(question_text="Past question.", days=-30)
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question],
        )

    def test_two_past_questions(self):
        """Отображается несколько вопросов"""
        question1 = create_question(question_text="Past question 1.", days=-30)
        question2 = create_question(question_text="Past question 2.", days=-5)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question2, question1],  # Сортировка по дате (новые первые)
        )

#### Django Test Client:
self.client.get(url)  # Имитирует GET запрос
self.client.post(url, data)  # Имитирует POST запрос

response.status_code  # HTTP код (200, 404, etc.)
response.context["variable"]  # Данные из контекста шаблона
response.content  # HTML содержимое

#### Методы проверки (assertions):
self.assertEqual(a, b)  # a == b
self.assertIs(a, b)  # a is b
self.assertContains(response, text)  # text есть в HTML
self.assertQuerySetEqual(qs1, qs2)  # QuerySet равны

#### Исправление DetailView:
class DetailView(generic.DetailView):
    model = Question
    template_name = "polls/detail.html"

    def get_queryset(self):
        """
        Исключает вопросы, которые еще не опубликованы
        """
        return Question.objects.filter(pub_date__lte=timezone.now())

Теперь нельзя открыть будущий вопрос по прямой ссылке - вернется 404!

#### Тесты для DetailView:
class QuestionDetailViewTests(TestCase):
    def test_future_question(self):
        """Будущий вопрос возвращает 404"""
        future_question = create_question(question_text="Future question.", days=5)
        url = reverse("polls:detail", args=(future_question.id,))
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    def test_past_question(self):
        """Прошлый вопрос отображается"""
        past_question = create_question(question_text="Past Question.", days=-5)
        url = reverse("polls:detail", args=(past_question.id,))
        response = self.client.get(url)
        self.assertContains(response, past_question.question_text)

---


## Часть 6: Статические файлы (CSS)

### Что сделано:
- Создана структура для статических файлов
- Добавлены стили CSS
- Добавлено фоновое изображение
- Настроена загрузка статики через {% static %}

### Комментарии:

#### Структура статических файлов:

polls/
    static/
        polls/              
            style.css
            images/
                background.png



#### Создание CSS файла:

#### Подключение статики в шаблоне:

<!-- polls/templates/polls/index.html -->
{% load static %}

<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="{% static 'polls/style.css' %}">
    <title>Polls</title>
</head>
<body>
    {% if latest_question_list %}
        <ul>
        {% for question in latest_question_list %}
            <li>
                <a href="{% url 'polls:detail' question.id %}">
                    {{ question.question_text }}
                </a>
            </li>
        {% endfor %}
        </ul>
    {% else %}
        <p>No polls are available.</p>
    {% endif %}
</body>
</html>

#### Ключевые моменты:

1. {% load static %} - загружает шаблонный тег static
Обязательно добавлять в начало каждого шаблона, где используется статика!

2. {% static 'polls/style.css' %} - генерирует правильный URL
Django автоматически добавит /static/ префикс:

{% static 'polls/style.css' %} → /static/polls/style.css

3. Путь в CSS к изображению:

url("images/background.png")  /* Относительный путь от style.css */

Полный путь: polls/static/polls/images/background.png
В CSS: images/background.png (относительно расположения CSS файла)

4. Настройки в settings.py:

# mysite/settings.py
STATIC_URL = '/static/'  # URL префикс для статики (уже есть по умолчанию)

# Для production нужно добавить:
STATIC_ROOT = BASE_DIR / 'staticfiles'

#### Как Django находит статику:

В режиме разработки (DEBUG=True):
1. Django автоматически отдает файлы из app/static/
2. Использует django.contrib.staticfiles (должно быть в INSTALLED_APPS)
3. Не требует дополнительной настройки

Порядок поиска:
1. polls/static/polls/style.css
2. другое_приложение/static/polls/style.css
3. Папки из STATICFILES_DIRS (если настроены)

#### Сбор статики для production:

python manage.py collectstatic

Эта команда:
- Соберет всю статику из всех приложений
- Скопирует в папку STATIC_ROOT
- Используется перед деплоем



---

## Часть 7: Настройка админ-панели

### Что сделано:
- Настроена кастомная форма редактирования Question
- Добавлено inline редактирование Choice
- Настроен список вопросов (list_display)
- Добавлены фильтры и поиск
- Добавлен декоратор @admin.display
- Кастомизирован заголовок админки
- Создан шаблон templates/admin/base_site.html

### Комментарии:

#### Базовая настройка admin:

# polls/admin.py

Что изменилось:
- По умолчанию: question_text, pub_date
- После настройки: pub_date, question_text

Сначала дата, потом текст - более логично при создании опроса.

#### Группировка полей (fieldsets):

class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None, {"fields": ["question_text"]}),  # Секция без заголовка
        ("Date information", {
            "fields": ["pub_date"],
            "classes": ["collapse"]  # Свернутая секция
        }),
    ]

Структура fieldsets:
- Кортеж: (Заголовок, {настройки})
- None - секция без заголовка
- "classes": ["collapse"] - секция свернута по умолчанию
- "classes": ["wide"] - широкая секция
- "classes": ["collapse", "wide"] - можно комбинировать

Что понял:
- fieldsets улучшает UX для больших форм
- Логическая группировка полей
- Визуальное разделение секций

#### Inline редактирование Choice:

class ChoiceInline(admin.TabularInline):
    model = Choice
    extra = 3  # 3 пустые формы для новых вариантов

class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None, {"fields": ["question_text"]}),
        ("Date information", {
            "fields": ["pub_date"],
            "classes": ["collapse"]
        }),
    ]
    inlines = [ChoiceInline]  # Добавили inline

admin.site.register(Question, QuestionAdmin)


Типы Inline:
- StackedInline - вертикальное расположение (занимает много места)
- TabularInline - табличное (компактно) 

Параметры:
- extra = 3 - количество пустых форм
- max_num = 10 - максимум вариантов
- min_num = 2 - минимум вариантов
- can_delete = True - можно удалять (по умолчанию)

#### Настройка списка вопросов:

class QuestionAdmin(admin.ModelAdmin):
    # ... fieldsets и inlines ...
    
    list_display = [
        "question_text",
        "pub_date",
        "was_published_recently"  # Кастомный метод модели
    ]
    
    list_filter = ["pub_date"]  # Фильтр по дате
    
    search_fields = ["question_text"]  # Поиск по тексту

list_display - столбцы в таблице:
- Список полей для отображения
- Можно использовать методы модели
- Можно кликать на заголовки для сортировки
- Порядок важен - так они и отобразятся

list_filter - фильтры на боковой панели:
- Django автоматически определяет тип фильтра по типу поля
- DateTimeField → "Any date", "Today", "Past 7 days", "This month", "This year"
- BooleanField → "All", "Yes", "No"
- ForeignKey → Список связанных объектов

search_fields - поле поиска вверху:
- Использует SQL LIKE запрос
- Можно указать несколько полей: ["question_text", "choice__choice_text"]
- Поддерживает поиск по связанным моделям через __
- Регистронезависимый поиск


#### Кастомизация заголовка админки:

Шаг 1: Настройка TEMPLATES в settings.py

# mysite/settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],  # ← Добавил эту строку
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

Что сделано:
- Добавил "DIRS": [BASE_DIR / "templates"]
- Теперь Django будет искать шаблоны в корневой папке templates/
- Приоритет: сначала templates/, потом app/templates/

Шаг 2: Создание структуры папок

В корне проекта (там где manage.py):

mkdir templates
mkdir templates\admin

Структура:

Django/
    templates/
        admin/
            base_site.html  # Кастомный шаблон
    mysite/
    polls/
    manage.py

Шаг 3: Найти исходный шаблон Django

python -c "import django; print(django.__path__)"

Шаг 4: Создать templates/admin/base_site.html

{% extends "admin/base.html" %}

{% block title %}
{% if subtitle %}{{ subtitle }} | {% endif %}{{ title }} | {{ site_title|default:_('Django site admin') }}
{% endblock %}

{% block branding %}
<div id="site-name">
    <a href="{% url 'admin:index' %}">
        Polls Administration
    </a>
</div>
{% if user.is_anonymous %}
  {% include "admin/color_theme_toggle.html" %}
{% endif %}
{% endblock %}

{% block nav-global %}{% endblock %}

- Было: <h1 id="site-name"><a href="{% url 'admin:index' %}">Django administration</a></h1>
- Стало: <div id="site-name"><a href="{% url 'admin:index' %}">Polls Administration</a></div>


#### Как работает переопределение шаблонов:

1. Django ищет шаблон admin/base_site.html
2. Сначала проверяет DIRS: templates/admin/base_site.html
3. Использует наш кастомный шаблон
4. Если бы не нашел - использовал бы стандартный из django/contrib/admin/

Это работает для любого шаблона админки:
- admin/base.html - базовый шаблон
- admin/change_form.html - форма редактирования
- admin/change_list.html - список объектов
- admin/delete_confirmation.html - подтверждение удаления


---

## Часть 6:
