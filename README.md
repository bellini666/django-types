# django-types [![PyPI](https://img.shields.io/pypi/v/django-types.svg)](https://pypi.org/project/django-types/)

Type stubs for [Django](https://www.djangoproject.com).

> Note: this project was forked from
> <https://github.com/typeddjango/django-stubs> with the goal of removing the
> [`mypy`](https://github.com/python/mypy) plugin dependency so that `mypy`
> can't [crash due to Django
> config](https://github.com/typeddjango/django-stubs/issues/318), and that
> non-`mypy` type checkers like
> [`pyright`](https://github.com/microsoft/pyright) will work better with
> Django.

## install

```bash
pip install django-types
```

If you're on a Django version < 3.1, you'll need to monkey patch Django's
`QuerySet` and `Manager` classes so we can index into them with a generic
argument. You can either use [`django-stubs-ext`](https://pypi.org/project/django-stubs-ext/`) or do this yourself manually:

```python
import types
from django.db.models.manager import BaseManager
from django.db.models.query import QuerySet

def no_op(self, x):
    return self

QuerySet.__class_getitem__ = types.MethodType(no_op, QuerySet)
BaseManager.__class_getitem__ = types.MethodType(no_op, BaseManager)
```

## usage

### getting `objects` to work

By default the base `Model` class doesn't have `objects` defined, so you'll
have to explicitly type the property.

```python
from django.db import connection, models
from django.db.models.manager import Manager

class User(models.Model):
    title = models.CharField(max_length=255)

    objects = Manager["User"]()

reveal_type(User.objects.all().first())
# note: Revealed type is 'Optional[User]'
```

### `HttpRequest`'s `user` property

The `HttpRequest`'s `user` property has a type of `Union[AbstractBaseUser, AnonymousUser]`,
but for most of your views you'll probably want either an authed user or an
`AnonymousUser`.

So we can define a subclass for each case:

```python
class AuthedHttpRequest(HttpRequest):
    user: User  # type: ignore [assignment]
```

And then you can use it in your views:

```python
@auth.login_required
def activity(request: AuthedHttpRequest, team_id: str) -> HttpResponse:
    ...
```

You can also get more strict with your `login_required` decorator so that the
first argument of the fuction it is decorating is `AuthedHttpRequest`:

```python
from typing import Any, Union, TypeVar, cast
from django.http import HttpRequest, HttpResponse
from typing_extensions import Protocol
from functools import wraps

class RequestHandler1(Protocol):
    def __call__(self, request: AuthedHttpRequest) -> HttpResponse:
        ...


class RequestHandler2(Protocol):
    def __call__(self, request: AuthedHttpRequest, __arg1: Any) -> HttpResponse:
        ...


RequestHandler = Union[RequestHandler1, RequestHandler2]


# Verbose bound arg due to limitations of Python typing.
# see: https://github.com/python/mypy/issues/5876
_F = TypeVar("_F", bound=RequestHandler)


def login_required(view_func: _F) -> _F:
    @wraps(view_func)
    def wrapped_view(
        request: AuthedHttpRequest, *args: object, **kwargs: object
    ) -> HttpResponse:
        if request.user.is_authenticated:
            return view_func(request, *args, **kwargs)  # type: ignore [call-arg]
        raise AuthenticationRequired

    return cast(_F, wrapped_view)
```

Then the following will type error:

```python
@auth.login_required
def activity(request: HttpRequest, team_id: str) -> HttpResponse:
    ...
```
