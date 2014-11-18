=============================
Simple Seo Backend for Django
=============================

.. image:: https://drone.io/github.com/danigosa/django-simple-seo/status.png
   :target: https://drone.io/github.com/danigosa/django-simple-seo/latest

.. image:: https://drone.io/github.com/danigosa/django-simple-seo/files/tests_html/coverage_status.png
   :target: https://drone.io/github.com/danigosa/django-simple-seo/files/tests_html/index.html

.. image:: https://img.shields.io/pypi/v/django-simple-seo.svg?style=flat
    :target: https://pypi.python.org/pypi/django-simple-seo

.. image:: https://img.shields.io/pypi/dm/django-simple-seo.svg?style=flat
    :target: https://pypi.python.org/pypi/django-simple-seo

Simple seo backend for Django. Inspired by django-seo ( https://github.com/willhardy/django-seo ) but found it quite
complex for the simple functionality it was intended for.

django-simple-seo aims to attach a model to your views with just 4 simple lines of code and everything configured by the admin.

WARNING
*******

This docs refers to version 1.0 or newer. For older versions (<=0.4.1) refer to:

 * https://github.com/danigosa/django-simple-seo/blob/master/README-OLD.rst


What's in django-simple-seo
***************************

 * Python 2.7, 3.2, 3.3, 3.4 and Django 1.4, 1.5 and 1.6
 * View Autodiscovering
 * Registry of models and views in settings
 * Pure Django Models and Django Fields implementation, no metaclasses in action
 * Don't reinvent the wheel: as long as they are django models you can use the goodies out there
 * i18n with django-vinaigrette, django-linguo, django-modeltranslation, etc.
 * Cache can be activated internally to cache raw HTML, but can also be used with johnny-cache, django-cache-machine, etc. for model level caching
 * Single database query onto a single database table (if you have one model, no several seo backends)
 * Support for 'populate_from' attribute (copy values on saving in similar tags)
 * Support for UrlFields and ImageFields in admin (can be override to simple char fields)
 * Support for django-storages for S3 storage(and possibly other backends, but not tested), directly from admin
 * Easily extendible as far as it's all about simple django models and fields
 * Out-of-the-box models for OpenGraph Facebook and Twitter tags
 * Includes Selenium tests for proper HTML generation
 * Test app included (.testapp)

 .. image:: assets/simple_seo_admin.png

 .. image:: assets/simple_seo_admin1.png

What's different on this fork?
*******************************

* Added content type filtering to metadata models. This add an extra liberty to
  modify the metadata served given an instance of a model in addition to the 
  view name. This made more sense than having metadata per-model, since even
  if a model is used in more than one view, the metadata you want to show
  would vary according to the view it's shown on (IMO).
* Allow to use django's template engine on metadata fields. This is related to
  the previous point: when you give an instance of a model, you probably want
  to use that instance to generate specific metadata.
* Added a default metadata instance, which will be used when no other metadata
  is found. It also can be referenced whitin a specific metadata content to
  make for reusable common pieces of metadata (e.g. when you want your title to
  always include some text + specific section text).
* Modified admin to take into account the previous points.
* Modified implementation for models. Instead of having a bunch of concrete
  base models, only make use of a single abstract base model which includes the
  required view_name and content_type and a bunch of mixins which you can
  integrate into your own metadata model.


Installation
------------

Since this is a fork, it isn't on pypi, so you have to use pip with git:

.. code-block:: sh

    $ pip install git+https://github.com/asermax/django-simple-seo.git

Add to your settings:

.. code-block:: python

    INSTALLED_APPS = (
    ...
    'simple_seo',
    ...
    )

Requeriments
------------

 * staticfiles
 * south (optional, if migrations)
 * admin (this includes auth, sessions and contenttypes)
 * django-modeltranslation, django-linguo, django-vinaigrette (**optional**, for i18n)
 * django-storages (**optional**, for cloud storage)
 

1. Create your SEO Model
------------------------

Create a model subclassing BaseMetada, including any other mixin you would
like to use or adding your own metadata fields.

.. code-block:: python

    from simple_seo import models as seo_models


    class Metadata(seo_models.SimpleMetadataMixin, seo_models.BaseMetadata):
        pass


2. Synchronize your DB
----------------------

Synchronize your database with **syncdb**, then your model with **migrate** if you are using migrations:

.. code-block:: sh

    $ ./manage.py syncdb

Or in case of using South:

.. code-block:: sh

    $ ./manage.py schemamigration your_app --auto
    $ ./manage.py migrate your_app


3. Register your model for view managment
-----------------------------------------

Use Django model notation for describing your seo models and the views related to be managed.

The simplest usage is to have just one seo model that manages all views. Do it like this in your **settings.py**:

.. code-block:: python

    SEO_MODEL_REGISTRY = (
        ('testapp.MyMetadata', 'ALL'),
    )

In case you need several seo models and restrict them to certain views, add the following:

.. code-block:: python

    SEO_MODEL_REGISTRY = (
        ('simple_seo.TestMetadata', ('template_test', )),
    )

Please note that simple_seo registry will load views by order and store them in a dictionary. That means:

  * Collisions in model definitions will result in first definition to be *always selected*
  * Defining just one 'ALL' registry will override the rest if it's declared *after* 

Examples of bad configurations:

.. code-block:: python

    SEO_MODEL_REGISTRY = (
        ('testapp.MyMetadata', 'ALL'),
        ('simple_seo.TestMetadata', ('template_test', )),
    )
**PROBLEM**: 'simple_seo.TestMetadata' model won't ever be reached. 'template_test' view will be processed with 'testapp.MyMetadata'

.. code-block:: python

    SEO_MODEL_REGISTRY = (
        ('testapp.MyMetadata', ('template_test', )),
        # ... More and More definitions
        ('testapp.MyMetadata', ('template_test2', 'template_test3')),
    )
**PROBLEM**: 'template_test' view will never be processed as last registry overrides first.

There's no plans to make registry very exotic on this, just following very simple rules it can be as complex as you want, covering vast use cases.


4. Register your model for administration
-----------------------------------------

Add this lines to your admin.py:

.. code-block:: python

    from simple_seo.admin import BaseMetadataAdmin
    from django.contrib import admin
    from .models import MyMetadata


    class MyMetadataAdmin(BaseMetadataAdmin):
        pass

    admin.site.register(MyMetadata, MyMetadataAdmin)


5. Configure URLs for seo autodiscovering
-----------------------------------------

**WARNING:** It's a django related issue but once you call *admin.autodiscover()* the URLConf module remains corrupted forever, that means cannot dive into *urlpatterns*.

To solve that, try to add admin URL and do autodiscovering at the very end of your **urls.py** like this:

.. code-block:: python

    # Put all your URLconfig that should be managed by simple_seo BEFORE admin
    urlpatterns = patterns(
        '',
        url(r'^test/', template_test, name='template_test'),
    )

    # Then add admin configuration AFTER your seo views
    admin.autodiscover()

    urlpatterns += patterns(
        '',
        url(r'^admin/', include(admin.site.urls)),
    )


This configuration will also avoid *autodiscover* admin views to be added to views dropdown in the admin.

6. Add metadata for your views
------------------------------

Your views are autodiscovered for your convenience, create a metadata object for every view you want to administer

 .. image:: assets/simple_seo_admin2.png
    :width: 100%


7. Add metadata to your template
--------------------------------

Just include this template tag in your **<head>** section, no more template code needed, can be on the root *base.html* template and it will autodetect the view and inject appropriate metadata for each.

.. code-block:: html

    {% load simple_seo %}
    <!DOCTYPE html>
    <html>
    <head lang="en">
        <meta charset="UTF-8">
        {% metadata %}
    </head>
    <body>
    TEST
    </body>
    </html>


If you want to use a instance specific metadata, pass it to the templatetag as
a parameter

.. code-block:: html
    
    {% metadata instance %}


7. Extend/Override default behaviour
------------------------------------

*"I prefer to have images as URLs, not static files in my server"*

Just override **og_image** attribute. You can find all base models in **simple_seo.models**, and all tag fields in **simple_seo.fields**:

.. code-block:: python

    from simple_seo import fields as seo_fields, models as seo_models


    class Metadata(seo_models.SimpleMetadataMixin,
                   seo_models.OpenGraphMetadataMixin,
                   seo_models.TwitterMetadataMixin
                   seo_models.BaseMetadata):
        og_image = seo_fields.URLMetaTagField(name="og:image")  # Overrides default og:image field
        another_meta_tag = seo_fields.MetaTagField(name="myvariable", max_length="25")  #  Creates a new custom meta tag for the views

*"I only want Facebook tags, and I prefer to add all fields by hand, no handy population, like a boss"*

.. code-block:: python

    from simple_seo import fields as seo_fields, models as seo_models


    class OpenGraphMetadata(seo_models.OpenGraphMetadataMixin,
                            seo_models.BaseMetadata):
        og_title = seo_fields.MetaTagField(name="og:title", populate_from=None)  # Overrides default og:title field
        og_description = seo_fields.MetaTagField(name="og:description", populate_from=None)  # Overrides default og:description field


8. Cache Settings
-----------------

Some settings are provided to enable caching directly in the app:

.. code-block:: python

    SEO_CACHE_PREFIX = getattr(settings, 'SEO_CACHE_PREFIX', 'simple_seo:')
    SEO_CACHE_TIMEOUT = getattr(settings, 'SEO_CACHE_PREFIX', 60 * 60 * 24)
    SEO_USE_CACHE = getattr(settings, 'SEO_CACHE_PREFIX', False)


9. Sample App
-------------

You have a complete sample app in **testapp** module in this repository.


10. Multilang i18n Support
--------------------------

As said before you can apply any 3rd party app for translating your models to django-simple-seo models.
As an example, this is a complete model translated thanks to django-vinaigrette app: https://github.com/ecometrica/django-vinaigrette

Complete SEO model translated:

.. code-block:: python

    from simple_seo models as seo_models
    import vinaigrette


    class Metadata(seo_models.SimpleMetadataMixin,
                   seo_models.OpenGraphMetadataMixin,
                   seo_models.TwitterMetadataMixin
                   seo_models.BaseMetadata):

        class Meta:
            app_label = 'web'

    vinaigrette.register(
        SiteMetadata,
        [
            'title',
            'description',
            'keywords',
            'author',
            'og:title',
            'og:description',
            'twitter:title',
            'twitter:description',
        ]
    )

After that, just run **./manage.py makemessages** and you're done. See django-vinaigrette for more details.


Contribute!
-----------

Then add to the root of the project your **local_settings.py** for everything your need, for instance adding debug toolbar local setting:

.. code-block:: python

    INTERNAL_IPS = ('10.0.2.2', )
    
To execute the project with **vagrant** and **virtualbox** you can add this Vagrantfile and receipes to the local project and execute **vagrant up**:

https://gist.github.com/danigosa/c2ac2d349c4fcf823cb7

After box is provisioned you'll have an Ubuntu 14.04 with a Python 3.4 virtualenv.

You can find more info of how to develop with remote vagrant servers and the awesome Pycharm IDE here: http://codeispoetry.me/index.php/remote-server-with-pycharm-and-vagrant/



Changelog
=========

**version 1.0**

 * Lots of bugfixing
 * Support for 'populate_from' feature. By default og:title, og:description will populate from title and description. Twitter url, title, image and description will populate from Facebook's
 * Now support for django-modeltranslation and django-linguo (preffixing with '_lang' database fields)
 * Support for django-storages with S3 or other cloud services (tested on S3 only)
 * URLFields don't fail on validation (django defaults patched)
 * Cache working (memcached and django-redis tested)
 * Increased tests for population deep testing (Firefox Selenium testing)
 * New registry by settings, giving control to the developer in a single point (it's backwards incompatible!)

**version 0.4.1**

 * Refactors and bugfixings
 * Support for modeltranslation and linguo i18n backends

**version 0.3.2**

 * Bugfixes in __str__ and __len__ when fields have NULL value in database

**version 0.3.0**

 * Bugfixes
 * Added tests integrated with Selenium for proper HTML generation

**version 0.2.4**

 * Bugfix error in print_tag that did not printed images and urls in templatetag


**version 0.2.3**

 * Bugfix error in caching when i18n activated


**version 0.2.2**

 * Bugfix errors in view  autodiscover in some cases


**Version 0.2.1**

 * Bugfixes
 * Support for South Migrations


**Version 0.2**

 * Added support for namespaced views
 * Added support for ImageField based Image Metatags (og:image, twitter:image)
 * Added support for UrlField based URL Metatags (og:url, twitter:url)
 * Added more base classes to ease setup
