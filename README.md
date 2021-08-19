# Non-user authentication in Django and Django Rest Framework (DRF)

Let's imagine that our application needs to be used by some particular users, who
are not real users on our database backend.
For example, let's say we have a medical app used by doctors and patients. And
then we need to provide certain access to Hospital administrators to get an
overview of their doctors' and patients' activities. These hospital administrators
are not and cannot become system users.

The problem becomes complex since Django's authentication flow revolves around the concept of a user, and plenty of assumptions rely on the user being available.

How would you approach authentication and authorization in that case? How would
this non-system users access the app, and how would the app handle them?

## Authentication and authorization

But first let's see an overview of authentication and authorization in Django/DRF. It's important to
understand the request flow in order to come up with a good solution. The
overview here mentions those Django components that take place during
authentication.

Authentication consists of the synergy between middleware and view
functionality.

### Django's SessionMiddleware
Django's `SessionMiddleware` reads the session cookie for all incoming
requests and sets it for all outgoing requests that don't have it.
When reading the session cookie on an incoming request, the middleware sets
the `request.session` parameter to refer to a `SessionStore` object which wraps
the session. 

The `SessionMiddleware` runs for all incoming requests. Even anonymous requests get a session that refers to `AnonymousUser`. Even requests to an API that uses Token-based authentication, and are therefore anonymous from a session perspective, do get the `request.session` object as well as the response cookie.

### Django's AuthenticationMiddleware
Django's `AuthenticationMiddleware` sets the `request.user` parameter to refer
to the `User` instance that the `SessionStore` object pointed to by
`request.session`, refers to.

### DRF View
#### Request class
One of the very first things a DRF view does is [wrap the request
object](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L385) around its very own `Request` class. Among others, [this overrides the `request.user` object](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/request.py#L220), essentially cancelling out the `AuthenticationMiddleware`'s earlier action, and rendering that middleware useless.

Therefore, as far as API requests go, the `AuthenticationMiddleware` is
entirely redundant.

#### Authentication
The view [performs
authentication](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L414) by simply invoking [`request.user`](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L324). This, essentially [calls the `authenticate` method](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/request.py#L227) on the necessary authentication classes and sets the value of `request.user` to the `User` instance returned by the authenticator class.

In other words, one authentication class sets the `request.user` object to the `User` instance that made the request. The user instance could, for example, be the one that the Token corresponds to. 

It's worth mentioning that authentication classes are either defined project-wide using the `REST_FRAMEWORK['DEFAULT_AUTHENTICATION_CLASSES']` parameter, or on class level, using the `authentication_classes` parameter.

#### Permissions
The view [checks permissions](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L415), ensuring that the `request.user` has the necessary access permissions or role to proceed.

## Solution
Now that we have a solid understanding of authentication and authorization flow
in Django/DRF, let's figure out a solution to the problem mentioned earlier.

It should be obvious by now, that most of our logic should live in the
authentication class, whose goal in general is to pair the incoming request
with a user. In our case, of course, there is no such thing as a user, but
rather a hospital. 

Let's assume that the `Hospital` is indeed one of the data models in our
application. The solution that I've come up with, involves an extension of the
`AnonymousUser` class that can refer to a `Hospital` instance. This way we get
an in-memory `User` instance without littering the database, and the instance
still gets to refer to the `Hospital` that the request is made on behalf of.
```python
# auth.py
# ---------
from django.contrib.auth.models import AnonymousUser
from django.contrib.auth.backends import ModelBackend


class HospitalAdministrator(AnonymousUser):
    """
    Extension of `AnonymousUser` class:
    - Adds the `hospital` parameter to the `AnonymousUser` instance.
    - Ensures that its `is_authenticated` property always returns `True`.
    """

    def __init__(self, hospital):
        super().init()
        self.hospital = hospital

    @property
    def is_authenticated(self):
        return True


class HospitalAdministratorAuthentication(ModelBackend):

    def authenticate(self, request):
        """
        Authenticates the incoming request using the requests' API key and creates an
        `AnonymousUser` that is associated with the `Hospital` instance
        that corresponds to the API key or Token.
        """
        # Authentication logic goes here. It pairs the API key or Token with
        # the `Hospital` instance `hospital` that it belongs to.
        # ...
        # ...

        user = HospitalAdministrator(hospital)
        return user, None
```

This way, any time the `authenticate` method runs successfully, the incoming request
will get the `request.user` object that contains the `request.user.hospital`
parameter which indicates the `Hospital` instance that talks to our backend.

From that point on, `request.user.hospital` may be used throughout the request
life cycle.

```python
# permissions.py
# ---------
from rest_framework.permissions import BasePermission


class HospitalAdministratorPermission(BasePermission):

    def has_permission(self, request, view):
        return bool(request.user and request.user.hospital)
```

```python
# views.py
# ---------
from rest_framework.views import GenericAPIView
from .auth import HospitalAdministratorAuthentication
from .permissions import HospitalAdministratorPermission


class HospitalPatientsView(GenericAPIView):

    # Any view can make use of this authentication class
    authentication_classes = (HospitalAdministratorAuthentication, )
    permission_classes = (HospitalAdministratorPermission, )

    def get_queryset(self):
        return self.request.user.hospital.patients()
```

## Conclusions

As you can see the view's querying logic is based on the concept of `Hospital` and
we've accomplished this with quite a few benefits:
- Without littering the database with dummy users.
- Without changing our data model to accommodate new types of users (hospital
administrators).
- Using the regular Django/DRF authentication flow.
- We are able to plug in our authentication and permission classes into any
    view and work as we always do.
