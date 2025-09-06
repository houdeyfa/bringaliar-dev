# bringaliar-dev

# Bring-A-Liar Night — Django 5 (Dev‑first)

Below is a self‑contained Django project you can paste into a new folder. Follow the README at the top for setup. Files are grouped by path.

---

## README.md
```md
# Bring-A-Liar Night (dev-first)

A kind, consent-based riff on *Le Dîner de cons*. Each attendee brings one consenting **Liar (performer)** who delivers obviously-false claims. The room rates **obviousness** and **fun**. Host advances simple phases: **Prep → Perform → Rate → Results**.

## Constraints
- Django 5, server-rendered templates
- SQLite dev DB, single `settings.py` (DEBUG=True default)
- Email backend = console
- "Realtime" = HTMX polling
- No Docker, no Celery, no Redis, no WebSockets
- Auth = built-in (username+email). Password reset prints to console

## One-command dev run
```bash
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

Optional seed demo data (1 host + 3 attendees + liars + fake claims):
```bash
python manage.py seed_demo
```

## Apps
- `core` (base templates, utils, mixins)
- `accounts` (custom user with optional locale + CoC consent timestamp)
- `events` (Event, Attendee, LiarProfile + RSVP + lobby)
- `game` (Round, Claim, Rating, Score + scoring + phases)
- `moderation` (flag/unflag claims; visibility rules)

## Dev defaults
- `DEBUG=True`, `ALLOWED_HOSTS=['*']` (comment contains prod hardening note)
- Timezone: Europe/London (store UTC; display local)
- i18n enabled (en shipped; hooks for fr)

## Tests
Run tiny tests:
```bash
python manage.py test
```
```

---

## requirements.txt
```txt
Django==5.0.7
django-htmx==1.17.3
python-dotenv==1.0.1
```

---

## manage.py
```python
#!/usr/bin/env python
import os
import sys

def main():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'bringaliar.settings')
    from django.core.management import execute_from_command_line
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```

---

## bringaliar/__init__.py
```python
```

---

## bringaliar/settings.py
```python
from pathlib import Path
import os
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-insecure-key-change-me')
DEBUG = os.environ.get('DEBUG', '1') != '0'
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '*').split(',')  # DEV: ['*'] (Harden in prod)

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # third-party
    'django_htmx',
    # local
    'core',
    'accounts',
    'events',
    'game',
    'moderation',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django_htmx.middleware.HtmxMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'bringaliar.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'core.context_processors.settings_context',
            ],
        },
    },
]

WSGI_APPLICATION = 'bringaliar.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

LANGUAGE_CODE = os.environ.get('LANGUAGE_CODE', 'en')
TIME_ZONE = 'Europe/London'
USE_I18N = True
USE_TZ = True

STATIC_URL = 'static/'
STATICFILES_DIRS = [BASE_DIR / 'static']
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# Auth
AUTH_USER_MODEL = 'accounts.User'
LOGIN_REDIRECT_URL = 'home'
LOGOUT_REDIRECT_URL = 'home'
LOGIN_URL = 'login'

# Email -> console backend in dev
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
DEFAULT_FROM_EMAIL = 'no-reply@bringaliar.local'

# i18n hooks
LOCALE_PATHS = [BASE_DIR / 'locale']

# HTMX
HTMX_DEFAULTS = {}
```

---

## bringaliar/urls.py
```python
from django.contrib import admin
from django.urls import path, include
from core.views import home

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('', home, name='home'),
    path('events/', include('events.urls')),
    path('game/', include('game.urls')),
    path('mod/', include('moderation.urls')),
]
```

---

## bringaliar/wsgi.py
```python
import os
from django.core.wsgi import get_wsgi_application
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'bringaliar.settings')
application = get_wsgi_application()
```

---

## core/apps.py
```python
from django.apps import AppConfig

class CoreConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'core'
```

---

## core/context_processors.py
```python
def settings_context(request):
    return {
        'DEBUG': request.settings.DEBUG if hasattr(request, 'settings') else True,
    }
```

---

## core/views.py
```python
from django.shortcuts import render
from events.models import Event

def home(request):
    public_events = Event.objects.filter(visibility='public', status__in=['draft', 'published']).order_by('start_at')[:12]
    return render(request, 'core/home.html', {'events': public_events})
```

---

## core/templates/core/base.html
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{% block title %}Bring‑A‑Liar Night{% endblock %}</title>
  <link rel="stylesheet" href="/static/core.css" />
  <script src="https://unpkg.com/htmx.org@1.9.10" defer></script>
</head>
<body>
  <header class="nav">
    <a href="/" class="brand">🤥 Bring‑A‑Liar Night</a>
    <nav>
      {% if user.is_authenticated %}
        <span>Hello, {{ user.username }}!</span>
        <a href="/events/new">Create Event</a>
        <a href="/accounts/logout/">Logout</a>
      {% else %}
        <a href="/accounts/login/">Login</a>
        <a href="/accounts/register/">Register</a>
      {% endif %}
    </nav>
  </header>
  <main class="container">
    {% for message in messages %}
      <div class="flash">{{ message }}</div>
    {% endfor %}
    {% block content %}{% endblock %}
  </main>
  <footer class="footer">Dev‑first • HTMX polling • Console email • SQLite</footer>
</body>
</html>
```

---

## core/templates/core/home.html
```html
{% extends 'core/base.html' %}
{% block title %}Home · Bring‑A‑Liar{% endblock %}
{% block content %}
<section class="hero">
  <h1>Bring‑A‑Liar Night</h1>
  <p>Each attendee brings one consenting performer who tells obviously false claims.
     Rate <strong>obviousness</strong> and <strong>fun</strong>. Host advances phases. Be kind. 🤝</p>
  {% if user.is_authenticated %}
    <a class="btn" href="/events/new">Create an Event</a>
  {% else %}
    <a class="btn" href="/accounts/register/">Get Started</a>
  {% endif %}
</section>

<h2>Public Events</h2>
<div class="cards">
  {% for e in events %}
    <a class="card" href="/events/{{ e.id }}/">
      <h3>{{ e.title }}</h3>
      <p>{{ e.start_at|date:'D d M, H:i' }} · {{ e.location|default:'Online/To be announced' }}</p>
      <p>Status: {{ e.status }}</p>
    </a>
  {% empty %}
    <p>No public events yet.</p>
  {% endfor %}
</div>
{% endblock %}
```

---

## static/core.css
```css
:root{--bg:#0e0f12;--card:#151822;--muted:#8aa0b4;--txt:#e7ecf2;--acc:#6ee7b7}
*{box-sizing:border-box}body{margin:0;background:var(--bg);color:var(--txt);font:16px/1.5 system-ui,Segoe UI,Roboto}
.header,.nav{display:flex;align-items:center;justify-content:space-between;padding:12px 16px;background:#0b0c10;border-bottom:1px solid #1f2430}
.brand{color:var(--txt);text-decoration:none;font-weight:700}
.container{max-width:980px;margin:24px auto;padding:0 16px}
.hero{background:var(--card);padding:18px;border-radius:14px;margin-bottom:18px}
.btn{display:inline-block;background:var(--acc);color:#0b0c10;padding:10px 14px;border-radius:10px;text-decoration:none;font-weight:700}
.cards{display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:12px}
.card{background:var(--card);padding:14px;border-radius:12px;text-decoration:none;color:var(--txt);border:1px solid #1e2230}
.flash{background:#2c3446;color:#e9f1ff;border-left:4px solid var(--acc);padding:8px 10px;margin:10px 0;border-radius:8px}
.footer{opacity:.6;padding:20px;text-align:center}
.table{width:100%;border-collapse:collapse}
.table th,.table td{padding:8px;border-bottom:1px solid #232838}
.badge{background:#243041;border:1px solid #344055;color:#cfe9ff;border-radius:999px;padding:2px 8px;font-size:12px}
.mini-bar{height:8px;background:#223047;border-radius:8px;overflow:hidden}
.mini-bar > i{display:block;height:100%;background:var(--acc)}
input[type=range]{width:100%}
button, .button{background:#2e374a;color:#eaf0ff;border:none;padding:8px 12px;border-radius:10px;cursor:pointer}
button.primary{background:var(--acc);color:#08281a}
.controls{display:flex;gap:8px;flex-wrap:wrap}
.lock{opacity:.6;pointer-events:none}
```

---

## accounts/models.py
```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from django.utils import timezone

class User(AbstractUser):
    locale = models.CharField(max_length=12, blank=True, default='')
    coc_accepted_at = models.DateTimeField(null=True, blank=True)

    def accept_coc(self):
        if not self.coc_accepted_at:
            self.coc_accepted_at = timezone.now()
            self.save(update_fields=['coc_accepted_at'])
```

---

## accounts/forms.py
```python
from django import forms
from django.contrib.auth.forms import UserCreationForm
from .models import User

class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)
    code_of_conduct = forms.BooleanField(required=True, label="I agree to the Code of Conduct (be kind, obtain consent, no harassment).")
    locale = forms.CharField(required=False)

    class Meta:
        model = User
        fields = ("username", "email", "locale")

    def save(self, commit=True):
        user = super().save(commit)
        user.email = self.cleaned_data['email']
        if self.cleaned_data.get('code_of_conduct'):
            user.accept_coc()
        return user
```

---

## accounts/urls.py
```python
from django.urls import path
from django.contrib.auth import views as auth_views
from .views import register

urlpatterns = [
    path('register/', register, name='register'),
    path('login/', auth_views.LoginView.as_view(template_name='accounts/login.html'), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    path('password_reset/', auth_views.PasswordResetView.as_view(), name='password_reset'),
]
```

---

## accounts/views.py
```python
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import RegisterForm

def register(request):
    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            form.save()
            messages.success(request, 'Account created. Please log in.')
            return redirect('login')
    else:
        form = RegisterForm()
    return render(request, 'accounts/register.html', {'form': form})
```

---

## accounts/templates/accounts/login.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>Login</h1>
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button class="primary">Login</button>
</form>
<p>Forgot password? Trigger a reset at /accounts/password_reset/ (console email).</p>
{% endblock %}
```

---

## accounts/templates/accounts/register.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>Register</h1>
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button class="primary">Create Account</button>
</form>
{% endblock %}
```

---

## events/models.py
```python
from django.db import models
from django.conf import settings
from django.core.exceptions import ValidationError
from django.utils import timezone

User = settings.AUTH_USER_MODEL

class Event(models.Model):
    VIS_CHOICES = [('public','public'), ('private','private')]
    STATUS_CHOICES = [('draft','draft'), ('published','published'), ('closed','closed')]

    host = models.ForeignKey(User, on_delete=models.CASCADE, related_name='hosted_events')
    title = models.CharField(max_length=120)
    description = models.TextField(blank=True)
    start_at = models.DateTimeField()
    location = models.CharField(max_length=200, blank=True)
    max_attendees = models.PositiveIntegerField(default=12)
    visibility = models.CharField(max_length=8, choices=VIS_CHOICES, default='public')
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.title} ({self.start_at:%Y-%m-%d})"

class Attendee(models.Model):
    RSVP_CHOICES = [('invited','invited'), ('going','going'), ('maybe','maybe'), ('declined','declined')]
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='attendees')
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='attendances')
    rsvp_status = models.CharField(max_length=8, choices=RSVP_CHOICES, default='invited')
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('event','user')

    def __str__(self):
        return f"{self.user} @ {self.event} ({self.rsvp_status})"

class LiarProfile(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='liars')
    inviter = models.OneToOneField(Attendee, on_delete=models.CASCADE, related_name='liar_profile')
    display_name = models.CharField(max_length=80)
    persona_bio = models.TextField(blank=True)
    consent_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        unique_together = ('event','inviter')

    def clean(self):
        # guarantee the inviter belongs to the same event
        if self.inviter.event_id != self.event_id:
            raise ValidationError('Inviter must belong to the same event.')

    def tick_consent(self):
        self.consent_at = timezone.now()

    def __str__(self):
        return f"{self.display_name} (by {self.inviter.user.username})"
```

---

## events/forms.py
```python
from django import forms
from .models import Event, Attendee, LiarProfile

class EventForm(forms.ModelForm):
    class Meta:
        model = Event
        fields = ('title','description','start_at','location','max_attendees','visibility')
        widgets = {
            'start_at': forms.DateTimeInput(attrs={'type':'datetime-local'}),
        }

class RSVPForm(forms.ModelForm):
    class Meta:
        model = Attendee
        fields = ('rsvp_status',)

class LiarForm(forms.ModelForm):
    consent = forms.BooleanField(required=True, label='My performer consents to participate (record consent timestamp).')

    class Meta:
        model = LiarProfile
        fields = ('display_name','persona_bio')

    def save(self, commit=True):
        inst = super().save(commit=False)
        inst.tick_consent()
        if commit:
            inst.save()
        return inst
```

---

## events/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('new', views.event_new, name='event_new'),
    path('<int:pk>/', views.event_lobby, name='event_lobby'),
    path('<int:pk>/poll', views.lobby_poll, name='lobby_poll'),
    path('<int:pk>/publish', views.event_publish, name='event_publish'),
    path('<int:pk>/rsvp', views.rsvp, name='event_rsvp'),
    path('<int:pk>/liar', views.liar_edit, name='liar_edit'),
]
```

---

## events/views.py
```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from django.http import HttpResponse, HttpResponseForbidden
from django.db.models import Count
from .models import Event, Attendee, LiarProfile
from .forms import EventForm, RSVPForm, LiarForm
from game.models import Round

@login_required
def event_new(request):
    if request.method == 'POST':
        form = EventForm(request.POST)
        if form.is_valid():
            ev = form.save(commit=False)
            ev.host = request.user
            ev.save()
            # auto-create round 1 in prep phase
            Round.objects.create(event=ev, index=1, phase='prep')
            messages.success(request, 'Event created. Share the link with your guests!')
            return redirect('event_lobby', pk=ev.pk)
    else:
        form = EventForm()
    return render(request, 'events/event_new.html', {'form': form})

@login_required
def event_lobby(request, pk):
    ev = get_object_or_404(Event, pk=pk)
    # Ensure membership stub
    Attendee.objects.get_or_create(event=ev, user=request.user)
    round1 = Round.objects.filter(event=ev, index=1).first()
    attendees = ev.attendees.select_related('user').annotate(has_liar=Count('liar_profile'))
    return render(request, 'events/lobby.html', {
        'event': ev,
        'round': round1,
        'attendees': attendees,
    })

@login_required
def lobby_poll(request, pk):
    ev = get_object_or_404(Event, pk=pk)
    attendees = ev.attendees.select_related('user').annotate(has_liar=Count('liar_profile'))
    round1 = Round.objects.filter(event=ev, index=1).first()
    return render(request, 'events/_lobby_panel.html', {
        'event': ev, 'round': round1, 'attendees': attendees
    })

@login_required
def event_publish(request, pk):
    ev = get_object_or_404(Event, pk=pk)
    if request.user != ev.host:
        return HttpResponseForbidden()
    ev.status = 'published'
    ev.save(update_fields=['status'])
    messages.success(request, 'Event published.')
    return redirect('event_lobby', pk=pk)

@login_required
def rsvp(request, pk):
    ev = get_object_or_404(Event, pk=pk)
    att, _ = Attendee.objects.get_or_create(event=ev, user=request.user)
    if request.method == 'POST':
        form = RSVPForm(request.POST, instance=att)
        if form.is_valid():
            form.save()
            messages.success(request, 'RSVP updated.')
    return redirect('event_lobby', pk=pk)

@login_required
def liar_edit(request, pk):
    ev = get_object_or_404(Event, pk=pk)
    att = get_object_or_404(Attendee, event=ev, user=request.user)
    instance = getattr(att, 'liar_profile', None)
    if request.method == 'POST':
        form = LiarForm(request.POST, instance=instance)
        if form.is_valid():
            liar = form.save(commit=False)
            liar.event = ev
            liar.inviter = att
            liar.save()
            messages.success(request, 'Liar profile saved (consent recorded). Now add claims in Round 1 (Prep).')
            return redirect('event_lobby', pk=pk)
    else:
        form = LiarForm(instance=instance)
    return render(request, 'events/liar_edit.html', {'event': ev, 'form': form})
```

---

## events/templates/events/event_new.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>New Event</h1>
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button class="primary">Create</button>
</form>
{% endblock %}
```

---

## events/templates/events/lobby.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>{{ event.title }}</h1>
<p>{{ event.description }}</p>
<div class="controls">
  <form method="post" action="/events/{{ event.id }}/rsvp">{% csrf_token %}
    <select name="rsvp_status">
      {% for val,label in attendee.RSVP_CHOICES %}{% endfor %}
    </select>
  </form>
  {% if user == event.host %}
    <form method="post" action="/events/{{ event.id }}/publish">{% csrf_token %}
      <button class="primary">Publish</button>
    </form>
    <a class="button" href="/game/{{ event.id }}/rounds/1/">Open Round 1</a>
  {% endif %}
  <a class="button" href="/events/{{ event.id }}/liar">Your Liar</a>
</div>

<div id="lobby-panel" hx-get="/events/{{ event.id }}/poll" hx-trigger="load, every 4s" hx-swap="outerHTML">
  {% include 'events/_lobby_panel.html' %}
</div>
{% endblock %}
```

---

## events/templates/events/_lobby_panel.html
```html
<div id="lobby-panel">
  <h2>Roster & Phase</h2>
  <p>Status: <span class="badge">{{ event.status }}</span>
     · Round 1 phase: <span class="badge">{{ round.phase }}</span></p>
  <table class="table">
    <thead><tr><th>User</th><th>RSVP</th><th>Liar</th></tr></thead>
    <tbody>
      {% for a in attendees %}
        <tr>
          <td>{{ a.user.username }}</td>
          <td>{{ a.rsvp_status }}</td>
          <td>{% if a.has_liar %}✅{% else %}—{% endif %}</td>
        </tr>
      {% endfor %}
    </tbody>
  </table>
</div>
```

---

## events/templates/events/liar_edit.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>Your Liar</h1>
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button class="primary">Save Liar</button>
</form>
{% endblock %}
```

---

## game/models.py
```python
from django.db import models
from django.utils import timezone
from django.core.exceptions import ValidationError
from events.models import Event, LiarProfile, Attendee

PHASES = [('prep','prep'), ('perform','perform'), ('rate','rate'), ('closed','closed')]

class Round(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='rounds')
    index = models.PositiveIntegerField()
    phase = models.CharField(max_length=8, choices=PHASES, default='prep')

    class Meta:
        unique_together = ('event','index')

    def __str__(self):
        return f"Round {self.index} ({self.phase})"

class Claim(models.Model):
    round = models.ForeignKey(Round, on_delete=models.CASCADE, related_name='claims')
    liar = models.ForeignKey(LiarProfile, on_delete=models.CASCADE, related_name='claims')
    content = models.CharField(max_length=280)
    category = models.CharField(max_length=40, blank=True)
    performed_at = models.DateTimeField(null=True, blank=True)
    is_flagged = models.BooleanField(default=False)
    flag_reason = models.TextField(blank=True, null=True)

    def clean(self):
        if self.liar.event_id != self.round.event_id:
            raise ValidationError('Liar and Round must belong to the same event.')

    def __str__(self):
        return f"{self.liar.display_name}: {self.content[:40]}"

class Rating(models.Model):
    claim = models.ForeignKey(Claim, on_delete=models.CASCADE, related_name='ratings')
    rater = models.ForeignKey(Attendee, on_delete=models.CASCADE, related_name='ratings')
    obviousness = models.PositiveSmallIntegerField()
    fun = models.PositiveSmallIntegerField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ('claim','rater')

class Score(models.Model):
    event = models.ForeignKey(Event, on_delete=models.CASCADE, related_name='scores')
    attendee = models.ForeignKey(Attendee, on_delete=models.CASCADE, related_name='scores')
    points = models.IntegerField(default=0)

    class Meta:
        unique_together = ('event','attendee')
```

---

## game/forms.py
```python
from django import forms
from .models import Claim, Rating

class ClaimForm(forms.ModelForm):
    class Meta:
        model = Claim
        fields = ('content','category')

class RatingForm(forms.ModelForm):
    obviousness = forms.IntegerField(min_value=1, max_value=5, widget=forms.NumberInput(attrs={'type':'range','step':1}))
    fun = forms.IntegerField(min_value=1, max_value=5, widget=forms.NumberInput(attrs={'type':'range','step':1}))
    class Meta:
        model = Rating
        fields = ('obviousness','fun')
```

---

## game/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('<int:event_id>/rounds/<int:index>/', views.round_hub, name='round_hub'),
    path('<int:event_id>/rounds/<int:index>/advance', views.advance_phase, name='advance_phase'),
    path('<int:event_id>/rounds/<int:index>/add_claim', views.add_claim, name='add_claim'),
    path('<int:event_id>/rounds/<int:index>/perform_next', views.perform_next, name='perform_next'),
    path('claim/<int:claim_id>/rate', views.rate_claim, name='rate_claim'),
    path('<int:event_id>/leaderboard', views.leaderboard_partial, name='leaderboard_partial'),
]
```

---

## game/utils.py
```python
import math
from statistics import median

def compute_indices(obvious_list, fun_list):
    if not obvious_list:
        return 0,0,0
    ob = round(100 * median(obvious_list) / 5)
    fu = round(100 * median(fun_list) / 5)
    facepalm = round(0.7*ob + 0.3*fu)
    return ob, fu, facepalm

def inviter_points(facepalm, obvious_list):
    pts = math.ceil(facepalm/10)
    if obvious_list:
        fives = sum(1 for v in obvious_list if v == 5)
        if fives / len(obvious_list) >= 0.7:
            pts += 2
    return pts
```

---

## game/views.py
```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.http import HttpResponseForbidden
from django.contrib import messages
from django.utils import timezone
from django.db.models import Count, Avg

from events.models import Event, Attendee, LiarProfile
from .models import Round, Claim, Rating, Score
from .forms import ClaimForm, RatingForm
from .utils import compute_indices, inviter_points


def ensure_member(request, event):
    Attendee.objects.get_or_create(event=event, user=request.user)

@login_required
def round_hub(request, event_id, index):
    event = get_object_or_404(Event, pk=event_id)
    ensure_member(request, event)
    rnd = get_object_or_404(Round, event=event, index=index)

    # Phase gate conditions
    attendees = event.attendees.filter(rsvp_status__in=['going','maybe'])
    liars = LiarProfile.objects.filter(event=event)

    # Prep: users add claims (3-6) for their own liar
    my_att = Attendee.objects.get(event=event, user=request.user)
    my_liar = getattr(my_att, 'liar_profile', None)
    my_claims = Claim.objects.filter(round=rnd, liar=my_liar) if my_liar else []
    claim_form = ClaimForm()

    # Stats / leaderboard
    return render(request, 'game/round_hub.html', {
        'event': event, 'round': rnd,
        'attendees': attendees, 'my_liar': my_liar, 'my_claims': my_claims,
        'claim_form': claim_form,
    })

@login_required
def advance_phase(request, event_id, index):
    event = get_object_or_404(Event, pk=event_id)
    if request.user != event.host:
        return HttpResponseForbidden()
    rnd = get_object_or_404(Round, event=event, index=index)

    # Validate rules on transitions
    if rnd.phase == 'prep':
        # Ensure each attendee (going/maybe) has exactly one liar with 3-6 claims
        active_atts = event.attendees.filter(rsvp_status__in=['going','maybe'])
        for att in active_atts:
            if not hasattr(att,'liar_profile'):
                messages.error(request, f"{att.user.username} is missing a Liar.")
                return redirect('round_hub', event_id=event.id, index=rnd.index)
            ccount = Claim.objects.filter(round=rnd, liar=att.liar_profile).count()
            if ccount < 3 or ccount > 6:
                messages.error(request, f"{att.user.username}'s Liar must have 3–6 claims (has {ccount}).")
                return redirect('round_hub', event_id=event.id, index=rnd.index)
        rnd.phase = 'perform'
        rnd.save(update_fields=['phase'])
        messages.success(request, 'Round moved to PERFORM. Host can click Next to reveal claims.')
    elif rnd.phase == 'perform':
        # move to rate when all claims performed
        if rnd.claims.filter(performed_at__isnull=True).exists():
            messages.error(request, 'There are still unperformed claims.')
            return redirect('round_hub', event_id=event.id, index=rnd.index)
        rnd.phase = 'rate'
        rnd.save(update_fields=['phase'])
        messages.success(request, 'Round moved to RATE.')
    elif rnd.phase == 'rate':
        rnd.phase = 'closed'
        rnd.save(update_fields=['phase'])
        messages.success(request, 'Round closed. See Results / Leaderboard.')
    return redirect('round_hub', event_id=event.id, index=rnd.index)

@login_required
def add_claim(request, event_id, index):
    event = get_object_or_404(Event, pk=event_id)
    rnd = get_object_or_404(Round, event=event, index=index)
    my_att = get_object_or_404(Attendee, event=event, user=request.user)
    if not hasattr(my_att, 'liar_profile'):
        messages.error(request, 'Create your Liar first in the event lobby.')
        return redirect('round_hub', event_id=event.id, index=index)
    if request.method == 'POST' and rnd.phase == 'prep':
        form = ClaimForm(request.POST)
        if form.is_valid():
            c = form.save(commit=False)
            c.round = rnd
            c.liar = my_att.liar_profile
            c.save()
            messages.success(request, 'Claim added.')
    return redirect('round_hub', event_id=event.id, index=index)

@login_required
def perform_next(request, event_id, index):
    event = get_object_or_404(Event, pk=event_id)
    rnd = get_object_or_404(Round, event=event, index=index)
    if request.user != event.host or rnd.phase != 'perform':
        return HttpResponseForbidden()
    next_claim = rnd.claims.filter(performed_at__isnull=True).order_by('id').first()
    if next_claim:
        next_claim.performed_at = timezone.now()
        next_claim.save(update_fields=['performed_at'])
        messages.success(request, 'Revealed next claim.')
    else:
        messages.info(request, 'No more claims. Advance phase to RATE.')
    return redirect('round_hub', event_id=event.id, index=index)

@login_required
def rate_claim(request, claim_id):
    claim = get_object_or_404(Claim, pk=claim_id)
    event = claim.round.event
    att = get_object_or_404(Attendee, event=event, user=request.user)
    if att == claim.liar.inviter:
        messages.error(request, "Inviter cannot rate their own Liar's claim.")
        return redirect('round_hub', event_id=event.id, index=claim.round.index)
    if claim.round.phase != 'rate':
        messages.error(request, 'Not in rating phase.')
        return redirect('round_hub', event_id=event.id, index=claim.round.index)

    rating, created = Rating.objects.get_or_create(claim=claim, rater=att)
    if request.method == 'POST':
        form = RatingForm(request.POST, instance=rating)
        if form.is_valid():
            form.save()
            messages.success(request, 'Rating saved.')
    return redirect('round_hub', event_id=event.id, index=claim.round.index)

@login_required
def leaderboard_partial(request, event_id):
    event = get_object_or_404(Event, pk=event_id)
    # Compute points on the fly per rule; store Score cache for table simplicity
    from collections import defaultdict
    totals = defaultdict(int)
    for claim in Claim.objects.filter(round__event=event):
        ratings = list(claim.ratings.values_list('obviousness','fun'))
        if not ratings:
            continue
        ob_list = [r[0] for r in ratings]
        fu_list = [r[1] for r in ratings]
        ob, fu, face = compute_indices(ob_list, fu_list)
        pts = inviter_points(face, ob_list)
        inviter_att = claim.liar.inviter
        totals[inviter_att.id] += pts
    # Update or create Score rows
    for att_id, pts in totals.items():
        Score.objects.update_or_create(event=event, attendee_id=att_id, defaults={'points': pts})
    scores = Score.objects.filter(event=event).select_related('attendee__user').order_by('-points')
    return render(request, 'game/_leaderboard.html', {'scores': scores})
```

---

## game/templates/game/round_hub.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>{{ event.title }} — Round {{ round.index }} ({{ round.phase }})</h1>

{% if user == event.host %}
  <div class="controls">
    <form method="post" action="/game/{{ event.id }}/rounds/{{ round.index }}/advance">{% csrf_token %}
      <button class="primary">Advance Phase</button>
    </form>
    {% if round.phase == 'perform' %}
      <form method="post" action="/game/{{ event.id }}/rounds/{{ round.index }}/perform_next">{% csrf_token %}
        <button>Reveal Next Claim</button>
      </form>
    {% endif %}
  </div>
{% endif %}

{% if round.phase == 'prep' %}
  <section class="hero">
    <h2>Your Liar's Claims (3–6)</h2>
    {% if my_liar %}
      <form method="post" action="/game/{{ event.id }}/rounds/{{ round.index }}/add_claim">{% csrf_token %}
        {{ claim_form.as_p }}
        <button>Add Claim</button>
      </form>
      <ul>
        {% for c in my_claims %}
          <li>{{ c.content }} <span class="badge">{{ c.category|default:'misc' }}</span></li>
        {% empty %}<li>No claims yet.</li>{% endfor %}
      </ul>
    {% else %}
      <p>Create your Liar in the event lobby first.</p>
    {% endif %}
  </section>
{% endif %}

{% if round.phase == 'perform' %}
  <section>
    <h2>Performed Claims</h2>
    <ul>
      {% for c in round.claims.all %}
        <li>
          {% if c.performed_at %}
            <strong>{{ c.liar.display_name }}</strong>: {{ c.content }}
          {% else %}
            <em>Pending…</em>
          {% endif %}
        </li>
      {% endfor %}
    </ul>
  </section>
{% endif %}

{% if round.phase == 'rate' %}
  <section>
    <h2>Rate Claims</h2>
    <div class="cards">
      {% for c in round.claims.all %}
        {% if c.liar.inviter.user_id != user.id %}
        <div class="card">
          <p><strong>{{ c.liar.display_name }}</strong>: {{ c.content }}</p>
          <form method="post" action="/game/claim/{{ c.id }}/rate">{% csrf_token %}
            <label>Obviousness (1–5)</label>
            <input type="range" name="obviousness" min="1" max="5" step="1" required>
            <label>Fun (1–5)</label>
            <input type="range" name="fun" min="1" max="5" step="1" required>
            <button>Submit</button>
          </form>
        </div>
        {% endif %}
      {% endfor %}
    </div>
  </section>
{% endif %}

<section>
  <h2>Leaderboard</h2>
  <div id="leaderboard" hx-get="/game/{{ event.id }}/leaderboard" hx-trigger="load, every 5s" hx-swap="outerHTML">
    {% include 'game/_leaderboard.html' %}
  </div>
</section>

{% if round.phase == 'closed' %}
  <section class="hero">
    <h2>Results & Awards</h2>
    <p>See leaderboard above. (Awards: Most Outrageous, Deadpan Disaster, Crowd Pleaser — pick from highest median fun/obviousness blends.)</p>
  </section>
{% endif %}
{% endblock %}
```

---

## game/templates/game/_leaderboard.html
```html
<div id="leaderboard">
  <table class="table">
    <thead><tr><th>Attendee</th><th>Points</th></tr></thead>
    <tbody>
      {% for s in scores %}
        <tr><td>{{ s.attendee.user.username }}</td><td>{{ s.points }}</td></tr>
      {% empty %}
        <tr><td colspan="2">No scores yet.</td></tr>
      {% endfor %}
    </tbody>
  </table>
</div>
```

---

## moderation/apps.py
```python
from django.apps import AppConfig
class ModerationConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'moderation'
```

---

## moderation/urls.py
```python
from django.urls import path
from .views import flag_claim, unflag_claim

urlpatterns = [
    path('flag/<int:claim_id>/', flag_claim, name='flag_claim'),
    path('unflag/<int:claim_id>/', unflag_claim, name='unflag_claim'),
]
```

---

## moderation/views.py
```python
from django.shortcuts import redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from django.http import HttpResponseForbidden
from game.models import Claim

@login_required
def flag_claim(request, claim_id):
    c = get_object_or_404(Claim, pk=claim_id)
    c.is_flagged = True
    c.flag_reason = 'flagged by user'
    c.save(update_fields=['is_flagged','flag_reason'])
    messages.info(request, 'Claim flagged. Hidden until host unflags.')
    return redirect('round_hub', event_id=c.round.event.id, index=c.round.index)

@login_required
def unflag_claim(request, claim_id):
    c = get_object_or_404(Claim, pk=claim_id)
    if request.user != c.round.event.host:
        return HttpResponseForbidden()
    c.is_flagged = False
    c.flag_reason = ''
    c.save(update_fields=['is_flagged','flag_reason'])
    messages.success(request, 'Claim unflagged.')
    return redirect('round_hub', event_id=c.round.event.id, index=c.round.index)
```

---

## admin registration (optional but handy)
### events/admin.py
```python
from django.contrib import admin
from .models import Event, Attendee, LiarProfile
admin.site.register(Event)
admin.site.register(Attendee)
admin.site.register(LiarProfile)
```

### game/admin.py
```python
from django.contrib import admin
from .models import Round, Claim, Rating, Score
admin.site.register(Round)
admin.site.register(Claim)
admin.site.register(Rating)
admin.site.register(Score)
```

---

## management command (seed)
### events/management/__init__.py
```python
```

### events/management/commands/__init__.py
```python
```

### events/management/commands/seed_demo.py
```python
from django.core.management.base import BaseCommand
from django.contrib.auth import get_user_model
from django.utils import timezone
from events.models import Event, Attendee, LiarProfile
from game.models import Round, Claim

User = get_user_model()

class Command(BaseCommand):
    help = 'Seed demo data: 1 event with 3 attendees, liars, and claims.'

    def handle(self, *args, **kwargs):
        host, _ = User.objects.get_or_create(username='host', defaults={'email':'host@example.com'})
        host.set_password('host'); host.save()
        ev = Event.objects.create(host=host, title='Demo Night', description='Sample Bring‑A‑Liar', start_at=timezone.now())
        rnd = Round.objects.create(event=ev, index=1, phase='prep')
        users = []
        for i in range(1,4):
            u, _ = User.objects.get_or_create(username=f'guest{i}', defaults={'email':f'g{i}@x.com'})
            u.set_password(f'guest{i}'); u.save(); users.append(u)
        for u in users:
            att = Attendee.objects.create(event=ev, user=u, rsvp_status='going')
            liar = LiarProfile.objects.create(event=ev, inviter=att, display_name=f"{u.username}'s Liar", persona_bio='Deadpan expert', consent_at=timezone.now())
            Claim.objects.create(round=rnd, liar=liar, content='I invented the USB toaster.', category='tech')
            Claim.objects.create(round=rnd, liar=liar, content='I ran a marathon in flip-flops.', category='sports')
            Claim.objects.create(round=rnd, liar=liar, content='I trained a pigeon to code.', category='wild')
        self.stdout.write(self.style.SUCCESS('Seeded demo data. Login as host/host or guest1..3/guest1..3'))
```

---

## tests
### game/tests.py
```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from django.utils import timezone
from events.models import Event, Attendee, LiarProfile
from game.models import Round, Claim, Rating, Score
from game.utils import compute_indices, inviter_points

User = get_user_model()

class RuleTests(TestCase):
    def setUp(self):
        self.host = User.objects.create_user('h','h@x.com','h')
        self.ev = Event.objects.create(host=self.host, title='T', start_at=timezone.now())
        self.rnd = Round.objects.create(event=self.ev, index=1)
        self.a1 = Attendee.objects.create(event=self.ev, user=User.objects.create_user('a1'))
        self.a2 = Attendee.objects.create(event=self.ev, user=User.objects.create_user('a2'))
        self.l1 = LiarProfile.objects.create(event=self.ev, inviter=self.a1, display_name='L1', consent_at=timezone.now())
        self.l2 = LiarProfile.objects.create(event=self.ev, inviter=self.a2, display_name='L2', consent_at=timezone.now())

    def test_unique_liar_per_attendee(self):
        with self.assertRaises(Exception):
            LiarProfile.objects.create(event=self.ev, inviter=self.a1, display_name='dup')

    def test_rating_uniqueness(self):
        c = Claim.objects.create(round=self.rnd, liar=self.l1, content='X')
        rater = self.a2
        Rating.objects.create(claim=c, rater=rater, obviousness=5, fun=4)
        with self.assertRaises(Exception):
            Rating.objects.create(claim=c, rater=rater, obviousness=3, fun=3)

    def test_scoring(self):
        ob, fu, face = compute_indices([5,4,5], [3,5,4])
        self.assertTrue(0 <= ob <= 100 and 0 <= fu <= 100 and 0 <= face <= 100)
        pts = inviter_points(face, [5,5,5])
        self.assertGreaterEqual(pts, 2)
```

---

## templates: small auth overrides (optional)
### templates/registration/password_reset_form.html
```html
{% extends 'core/base.html' %}
{% block content %}
<h1>Password Reset</h1>
<p>In dev, this prints a console email with reset link.</p>
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button class="primary">Send</button>
</form>
{% endblock %}
```

---

## migrations
Create with:
```bash
python manage.py makemigrations accounts events game moderation
```
(Already covered by `migrate` if you run after generating models.)

---

## Notes
- Visibility rules: access is limited by membership (attendee row auto-created on lobby/round visit). For private events, avoid sharing the link broadly.
- Flagged claims are hidden socially (UI hint) and can be unflagged by the host.
- Awards page (`/events/{id}/recap`) can be derived from scores and medians; left as an exercise to extend.
