#############################################################
Django Sorcery - Django Framework integration with SQLAlchemy
#############################################################

|Build Status| |Read The Docs| |PyPI version| |Coveralls Status|

* Free software: MIT license
* GitHub: https://github.com/shosca/django-sorcery

SQLAlchemy is an excellent orm. And Django is a great framework, until you decide not to use Django ORM. This library
provides utilities, helpers and configurations to ease the pain of using SQLAlchemy with Django. It aims to provide
a similar development experience to building a Django application with Django ORM, except with SQLAlchemy.

Installation
============

::

    pip install django-sorcery

Quick Start
===========

Lets start by creating a site:

.. code:: console

    $ django-admin startproject mysite

And lets create an app:

.. code:: console

    $ cd mysite
    $ python manage.py startapp polls

This will create a polls app with standard django app layout:

.. code:: console

    $ tree
    .
    ├── manage.py
    ├── polls
    │   ├── admin.py
    │   ├── apps.py
    │   ├── __init__.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    └── mysite
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py

    3 directories, 12 files

And lets add our ``polls`` app in ``INSTALLED_APPS`` in ``mysite/settings.py``:

.. code:: python

    INSTALLED_APPS = [
        'polls.apps.PollsConfig',
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
    ]

Now we're going to make a twist and start building our app with ``sqlalchemy``. Lets define our models in
``polls/models.py``:

.. code:: python

    from django_sorcery.db import databases


    db = databases.get("default")


    class Question(db.Model):
        pk = db.Column(db.Integer(), autoincrement=True, primary_key=True)
        question_text = db.Column(db.String(length=200))
        pub_date = db.Column(db.DateTime())


    class Choice(db.Model):
        pk = db.Column(db.Integer(), autoincrement=True, primary_key=True)
        choice_text = db.Column(db.String(length=200))
        votes = db.Column(db.Integer(), default=0)

        question = db.ManyToOne(Question, backref=db.backref("choices", cascade="all, delete-orphan"))


    db.configure_mappers()
    db.create_all()

Right now, we have enough to hop in django shell:

.. code:: console

    $ python manage.py shell

    >>> from polls.models import Choice, Question, db  # Import the model classes and the db

    # we have no choices or questions in db yet
    >>> Choice.query.all()
    []
    >>> Question.query.all()
    []

    # Lets create a new question
    >>> from django.utils import timezone
    >>> q = Question(question_text="What's new?", pub_date=timezone.now())
    >>> q
    Question(pk=None, pub_date=datetime.datetime(2018, 5, 19, 0, 54, 20, 778186, tzinfo=<UTC>), question_text="What's new?")

    # lets save our question, we need to add our question to the db
    >>> db.add(q)

    # at this point the question is in pending state
    >>> db.new
    IdentitySet([Question(pk=None, pub_date=datetime.datetime(2018, 5, 19, 0, 54, 20, 778186, tzinfo=<UTC>), question_text="What's new?")])

    # lets flush to the database
    >>> db.flush()

    # at this point our question is in persistent state and will receive a primary key
    >>> q.pk
    1

    # lets change the question text
    >>> q.question_text = "What's up?"
    >>> db.flush()

    # Question.objects and Question.query are both query properties that return a query object bound to db
    >>> Question.objects
    <django_sorcery.db.query.Query at 0x7feb1c7899e8>
    >>> Question.query
    <django_sorcery.db.query.Query at 0x7feb1c9377f0>

    # and lets see all the questions
    >>> Question.objects.all()
    [Question(pk=1, pub_date=datetime.datetime(2018, 5, 19, 0, 54, 20, 778186, tzinfo=<UTC>), question_text="What's up?")]

    >>> exit()

Let's add a couple of views in ``polls/views.py``, starting with a list view:

.. code:: python

    from django.shortcuts import render
    from django.template import loader
    from django.http import HttpResponseRedirect
    from django.urls import reverse

    from django_sorcery.shortcuts import get_object_or_404

    from .models import Question, Choice, db

    def index(request):
        latest_question_list = Question.objects.order_by(Question.pub_date.desc())[:5]
        context = {'latest_question_list': latest_question_list}
        return render(request, 'polls/index.html', context)


    def detail(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/detail.html', {'question': question})


    def results(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/results.html', {'question': question})


    def vote(request, question_id):
        question = get_object_or_404(Question, pk=question_id)

        selected_choice = Choice.query.filter(
            Choice.question == question,
            Choice.pk == request.POST['choice'],
        ).one_or_none()

        if not selected_choice:
            return render(request, 'polls/detail.html', {
                'question': question,
                'error_message': "You didn't select a choice.",
            })

        selected_choice.votes += 1
        db.flush()
        return HttpResponseRedirect(reverse('polls:results', args=(question.pk,)))

and register the view in ``polls/urls.py``:

.. code:: python

    from django.urls import path

    from . import views


    app_name = 'polls'
    urlpatterns = [
        path('', views.index, name='index'),
        path('<int:question_id>/', views.detail, name='detail'),
        path('<int:question_id>/results', views.results, name='results'),
        path('<int:question_id>/vote', views.vote, name='vote'),
    ]

and register the ``SQLAlchemyMiddleware`` to provide unit-of-work per request pattern:

.. code:: python

    MIDDLEWARE = [
        'django_sorcery.db.middleware.SQLAlchemyMiddleware',
        # ...
    ]

and add some templates:

``polls/templates/polls/index.html``:

.. code:: html

    {% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
    <li><a href="{% url 'polls:detail' question.pk %}">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
    {% else %}
    <p>No polls are available.</p>
    {% endif %}

``polls/templates/polls/detail.html``:

.. code:: html

    <h1>{{ question.question_text }}</h1>

    {% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

    <form action="{% url 'polls:vote' question.pk %}" method="post">
    {% csrf_token %}
    {% for choice in question.choices %}
        <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.pk }}" />
        <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br />
    {% endfor %}
    <input type="submit" value="Vote" />
    </form>


``polls/templates/polls/results.html``:

.. code:: html

    <h1>{{ question.question_text }}</h1>

    <ul>
    {% for choice in question.choices %}
        <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
    {% endfor %}
    </ul>

    <a href="{% url 'polls:detail' question.pk %}">Vote again?</a>

This is all fine but we can do one better using generic views. Lets adjust our views in ``polls/views.py``:

.. code:: python

    from django.shortcuts import render
    from django.http import HttpResponseRedirect
    from django.urls import reverse

    from django_sorcery.shortcuts import get_object_or_404
    from django_sorcery import views

    from .models import Question, Choice, db


    class IndexView(views.ListView):
        template_name = 'polls/index.html'
        context_object_name = 'latest_question_list'

        def get_queryset(self):
            return Question.objects.order_by(Question.pub_date.desc())[:5]


    class DetailView(views.DetailView):
        model = Question
        session = db
        template_name = 'polls/detail.html'


    class ResultsView(DetailView):
        template_name = 'polls/results.html'


    def vote(request, question_id):
        question = get_object_or_404(Question, pk=question_id)

        selected_choice = Choice.query.filter(
            Choice.question == question,
            Choice.pk == request.POST['choice'],
        ).one_or_none()

        if not selected_choice:
            return render(request, 'polls/detail.html', {
                'question': question,
                'error_message': "You didn't select a choice.",
            })

        selected_choice.votes += 1
        db.flush()
        return HttpResponseRedirect(reverse('polls:results', args=(question.pk,)))

and adjust the ``polls/urls.py`` like:

.. code:: python

    from django.urls import path

    from . import views


    app_name = 'polls'
    urlpatterns = [
        path('', views.IndexView.as_view(), name='index'),
        path('<int:pk>/', views.DetailView.as_view(), name='detail'),
        path('<int:pk>/results', views.ResultsView.as_view(), name='results'),
        path('<int:question_id>/vote', views.vote, name='vote'),
    ]

The default values for ``template_name`` and ``context_object_name`` are similar to django's generic views. If we
handn't defined those the default for template names would've been ``polls/question_detail.html`` and
``polls/question_list.html`` for the detail and list template names, and ``question`` and ``question_list`` for context
names for detail and list views.

This is all fine but we can even do one better using a viewset. Lets adjust our views in ``polls/views.py``:

.. code:: python

    from django.http import HttpResponseRedirect
    from django.urls import reverse, reverse_lazy

    from django_sorcery.routers import action
    from django_sorcery.viewsets import ModelViewSet

    from .models import Question, Choice, db


    class PollsViewSet(ModelViewSet):
        model = Question
        fields = "__all__"
        destroy_success_url = reverse_lazy("polls:question-list")

        def get_success_url(self):
            return reverse("polls:question-detail", kwargs={"pk": self.object.pk})

        @action(detail=True)
        def results(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        @action(detail=True, methods=["POST"])
        def vote(self, request, *args, **kwargs):
            self.object = self.get_object()

            selected_choice = Choice.query.filter(
                Choice.question == self.object, Choice.pk == request.POST.get("choice")
            ).one_or_none()

            if not selected_choice:
                context = self.get_detail_context_data(object=self.object)
                context["error_message"] = "You didn't select a choice."
                self.action = "retrieve"
                return self.render_to_response(context)

            selected_choice.votes += 1
            db.flush()
            return HttpResponseRedirect(reverse("polls:question-results", args=(self.object.pk,)))

And adjusting our ``polls/urls.py`` like:

.. code:: python

    from django.urls import path, include

    from django_sorcery.routers import SimpleRouter

    from . import views

    router = SimpleRouter()
    router.register("", views.PollsViewSet)

    app_name = "polls"
    urlpatterns = [path("", include(router.urls))]

With these changes we'll have the following urls:

.. code:: console

    $ ./manage.py run show_urls
    /polls/	polls.views.PollsViewSet	polls:question-list
    /polls/<pk>/	polls.views.PollsViewSet	polls:question-detail
    /polls/<pk>/delete/	polls.views.PollsViewSet	polls:question-destroy
    /polls/<pk>/edit/	polls.views.PollsViewSet	polls:question-edit
    /polls/<pk>/results/	polls.views.PollsViewSet	polls:question-results
    /polls/<pk>/vote/	polls.views.PollsViewSet	polls:question-vote
    /polls/new/	polls.views.PollsViewSet	polls:question-new

This will map the following operations to following actions on the viewset:

====== ======================== =============== ===============
Method Path                     Action          Route Name
====== ======================== =============== ===============
GET    /polls/                  list            question-list
POST   /polls/                  create          question-list
GET    /polls/new/              new             question-new
GET    /polls/1/                retrieve        question-detail
POST   /polls/1/                update          question-detail
PUT    /polls/1/                update          question-detail
PATCH  /polls/1/                update          question-detail
DELETE /polls/1/                destroy         question-detail
GET    /polls/1/edit/           edit            question-edit
GET    /polls/1/delete/         confirm_destoy  question-delete
POST   /polls/1/delete/         destroy         question-delete
====== ======================== =============== ===============

Now, lets add an inline formset to be able to add choices to questions, adjust ``polls/views.py``:

.. code:: python

    from django.http import HttpResponseRedirect
    from django.urls import reverse, reverse_lazy

    from django_sorcery.routers import action
    from django_sorcery.viewsets import ModelViewSet
    from django_sorcery.formsets import inlineformset_factory

    from .models import Question, Choice, db


    ChoiceFormSet = inlineformset_factory(relation=Question.choices, fields=(Choice.choice_text.key,), session=db)


    class PollsViewSet(ModelViewSet):
        model = Question
        fields = (Question.question_text.key, Question.pub_date.key)
        destroy_success_url = reverse_lazy("polls:question-list")

        def get_success_url(self):
            return reverse("polls:question-detail", kwargs={"pk": self.object.pk})

        def get_form_context_data(self, **kwargs):
            kwargs["choice_formset"] = self.get_choice_formset()
            return super(PollsViewSet, self).get_form_context_data(**kwargs)

        def get_choice_formset(self, instance=None):
            if not hasattr(self, "_choice_formset"):
                instance = instance or self.object
                self._choice_formset = ChoiceFormSet(
                    instance=instance, data=self.request.POST if self.request.POST else None
                )

            return self._choice_formset

        def process_form(self, form):
            if form.is_valid() and self.get_choice_formset(instance=form.instance).is_valid():
                return self.form_valid(form)

            return form.invalid(self, form)

        def form_valid(self, form):
            self.object = form.save()
            self.object.choices = self.get_choice_formset().save()
            db.flush()
            return HttpResponseRedirect(self.get_success_url())

        @action(detail=True)
        def results(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        @action(detail=True, methods=["POST"])
        def vote(self, request, *args, **kwargs):
            self.object = self.get_object()

            selected_choice = Choice.query.filter(
                Choice.question == self.object, Choice.pk == request.POST.get("choice")
            ).one_or_none()

            if not selected_choice:
                context = self.get_detail_context_data(object=self.object)
                context["error_message"] = "You didn't select a choice."
                self.action = "retrieve"
                return self.render_to_response(context)

            selected_choice.votes += 1
            db.flush()
            return HttpResponseRedirect(reverse("polls:question-results", args=(self.object.pk,)))

And add ``choice_formset`` in the ``polls/templates/question_edit.html`` and ``polls/templates/question_edit.html``

.. code:: html

    <form ... >
        ...
        {{ choice_formset }}
        ...
    </form >


.. |Build Status| image:: https://travis-ci.org/shosca/django-sorcery.svg?branch=master
   :target: https://travis-ci.org/shosca/django-sorcery
.. |Read The Docs| image:: https://readthedocs.org/projects/django-sorcery/badge/?version=latest
   :target: http://django-sorcery.readthedocs.io/en/latest/?badge=latest
.. |PyPI version| image:: https://badge.fury.io/py/django-sorcery.svg
   :target: https://badge.fury.io/py/django-sorcery
.. |Coveralls Status| image:: https://coveralls.io/repos/github/shosca/django-sorcery/badge.svg?branch=master
   :target: https://coveralls.io/github/shosca/django-sorcery?branch=master
