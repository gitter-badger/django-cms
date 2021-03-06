###############
App Integration
###############

It is pretty easy to integrate your own Django applications with django CMS.
You have 7 ways of integrating your app:

1. :ref:`integration_menus`

    Statically extend the menu entries

2. :ref:`integration_attach_menus`

    Attach your menu to a page.

3. :ref:`integration_apphooks`

    Attach whole apps with optional menu to a page.

4. :ref:`integration_modifiers`

    Modify the whole menu tree

5. :ref:`integration_customplugins`

    Display your models / content in cms pages

6. :ref:`integration_toolbar`

    Add Entries to the toolbar to add/edit your models

7. :ref:`integration_templates`

    Templatetags and other CMS-provided utilities

.. _integration_menus:

*****
Menus
*****

Create a ``menu.py`` in your application and write the following inside::

    from menus.base import Menu, NavigationNode
    from menus.menu_pool import menu_pool
    from django.utils.translation import ugettext_lazy as _

    class TestMenu(Menu):

        def get_nodes(self, request):
            nodes = []
            n = NavigationNode(_('sample root page'), "/", 1)
            n2 = NavigationNode(_('sample settings page'), "/bye/", 2)
            n3 = NavigationNode(_('sample account page'), "/hello/", 3)
            n4 = NavigationNode(_('sample my profile page'), "/hello/world/", 4, 3)
            nodes.append(n)
            nodes.append(n2)
            nodes.append(n3)
            nodes.append(n4)
            return nodes

    menu_pool.register_menu(TestMenu)

If you refresh a page you should now see the menu entries from above.
The get_nodes function should return a list of
:class:`NavigationNode <menus.base.NavigationNode>` instances. A
:class:`NavigationNode` takes the following arguments:

- ``title``

  What the menu entry should read as

- ``url``,

  Link if menu entry is clicked.

- ``id``

  A unique id for this menu.

- ``parent_id=None``

  If this is a child of another node supply the id of the parent here.

- ``parent_namespace=None``

  If the parent node is not from this menu you can give it the parent
  namespace. The namespace is the name of the class. In the above example that
  would be: "TestMenu"

- ``attr=None``

  A dictionary of additional attributes you may want to use in a modifier or
  in the template.

- ``visible=True``

  Whether or not this menu item should be visible.

Additionally, each :class:`NavigationNode` provides a number of methods which are
detailed in the :class:`NavigationNode <menus.base.NavigationNode>` API references.

Customize menus at runtime
--------------------------

To adapt your menus according to request dependent conditions (say: anonymous /
logged in user), you can use `Navigation Modifiers`_  or you can leverage existing
ones.

For example it's possible to add ``{'visible_for_anonymous': False}`` /
``{'visible_for_authenticated': False}`` attributes recognized by the
django CMS core ``AuthVisibility`` modifier.

Complete example::

    class UserMenu(Menu):
        def get_nodes(self, request):
                return [
                    NavigationNode(_("Profile"), reverse(profile), 1, attr={'visible_for_anonymous': False}),
                    NavigationNode(_("Log in"), reverse(login), 3, attr={'visible_for_authenticated': False}),
                    NavigationNode(_("Sign up"), reverse(logout), 4, attr={'visible_for_authenticated': False}),
                    NavigationNode(_("Log out"), reverse(logout), 2, attr={'visible_for_anonymous': False}),
                ]

.. _integration_attach_menus:

************
Attach Menus
************

Classes that extend from :class:`menus.base.Menu` always get attached to the
root. But if you want the menu to be attached to a CMS Page you can do that as
well.

Instead of extending from :class:`~menus.base.Menu` you need to extend from
:class:`cms.menu_bases.CMSAttachMenu` and you need to define a name. We will do
that with the example from above::


    from menus.base import NavigationNode
    from menus.menu_pool import menu_pool
    from django.utils.translation import ugettext_lazy as _
    from cms.menu_bases import CMSAttachMenu

    class TestMenu(CMSAttachMenu):

        name = _("test menu")

        def get_nodes(self, request):
            nodes = []
            n = NavigationNode(_('sample root page'), "/", 1)
            n2 = NavigationNode(_('sample settings page'), "/bye/", 2)
            n3 = NavigationNode(_('sample account page'), "/hello/", 3)
            n4 = NavigationNode(_('sample my profile page'), "/hello/world/", 4, 3)
            nodes.append(n)
            nodes.append(n2)
            nodes.append(n3)
            nodes.append(n4)
            return nodes

    menu_pool.register_menu(TestMenu)


Now you can link this Menu to a page in the 'Advanced' tab of the page
settings under attached menu.

.. _integration_apphooks:

*********
App-Hooks
*********

With App-Hooks you can attach whole Django applications to pages. For example
you have a news app and you want it attached to your news page.

To create an apphook create a ``cms_app.py`` in your application. And in it
write the following::

    from cms.app_base import CMSApp
    from cms.apphook_pool import apphook_pool
    from django.utils.translation import ugettext_lazy as _

    class MyApphook(CMSApp):
        name = _("My Apphook")
        urls = ["myapp.urls"]

    apphook_pool.register(MyApphook)

Replace ``myapp.urls`` with the path to your applications ``urls.py``.

Now edit a page and open the advanced settings tab. Select your new apphook
under "Application". Save the page.

.. warning::

    Whenever you add or remove an apphook, change the slug of a page containing
    an apphook or the slug if a page which has a descendant with an apphook,
    you have to restart your server to re-load the URL caches.
    
.. note::

    If at some point you want to remove this apphook after deleting the cms_app.py
    there is a cms management command called uninstall apphooks
    that removes the specified apphook(s) from all pages by name.
    eg. ``manage.py cms uninstall apphooks MyApphook``.
    To find all names for uninstallable apphooks there is a command for this as well
    ``manage.py cms list apphooks``.

If you attached the app to a page with the url ``/hello/world/`` and the app has
a urls.py that looks like this::

    from django.conf.urls import *

    urlpatterns = patterns('sampleapp.views',
        url(r'^$', 'main_view', name='app_main'),
        url(r'^sublevel/$', 'sample_view', name='app_sublevel'),
    )

The ``main_view`` should now be available at ``/hello/world/`` and the
``sample_view`` has the url ``/hello/world/sublevel/``.


.. note::

    CMS pages **below** the page to which the apphook is attached to, **can** be visible,
    provided that the apphook urlconf regexps are not too greedy. From a URL resolution
    perspective, attaching an apphook works in same way than inserting the apphook urlconf
    in the root urlconf at the same path as the page is attached to.

.. note::

    All views that are attached like this must return a
    :class:`~django.template.RequestContext` instance instead of the
    default :class:`~django.template.Context` instance.


Apphook Menus
-------------

If you want to add a menu to that page as well that may represent some views
in your app add it to your apphook like this::

    from myapp.menu import MyAppMenu

    class MyApphook(CMSApp):
        name = _("My Apphook")
        urls = ["myapp.urls"]
        menus = [MyAppMenu]

    apphook_pool.register(MyApphook)


For an example if your app has a :class:`Category` model and you want this
category model to be displayed in the menu when you attach the app to a page.
We assume the following model::

    from django.db import models
    from django.core.urlresolvers import reverse
    import mptt

    class Category(models.Model):
        parent = models.ForeignKey('self', blank=True, null=True)
        name = models.CharField(max_length=20)

        def __unicode__(self):
            return self.name

        def get_absolute_url(self):
            return reverse('category_view', args=[self.pk])

    try:
        mptt.register(Category)
    except mptt.AlreadyRegistered:
        pass

We would now create a menu out of these categories::

    from menus.base import NavigationNode
    from menus.menu_pool import menu_pool
    from django.utils.translation import ugettext_lazy as _
    from cms.menu_bases import CMSAttachMenu
    from myapp.models import Category

    class CategoryMenu(CMSAttachMenu):

        name = _("test menu")

        def get_nodes(self, request):
            nodes = []
            for category in Category.objects.all().order_by("tree_id", "lft"):
                node = NavigationNode(
                    category.name,
                    category.get_absolute_url(),
                    category.pk,
                    category.parent_id
                )
                nodes.append(node)
            return nodes

    menu_pool.register_menu(CategoryMenu)

If you add this menu now to your app-hook::

    from myapp.menus import CategoryMenu

    class MyApphook(CMSApp):
        name = _("My Apphook")
        urls = ["myapp.urls"]
        menus = [MyAppMenu, CategoryMenu]

You get the static entries of :class:`MyAppMenu` and the dynamic entries of
:class:`CategoryMenu` both attached to the same page.

.. _multi_apphook:

Attaching an Application multiple times
---------------------------------------

If you want to attach an application multiple times to different pages you have 2 possibilities.

1. Give every application its own namespace in the advanced settings of a page.
2. Define an ``app_name`` attribute on the CMSApp class.

The problem is that if you only define a namespace you need to have multiple templates per attached app.

For example::

    {% url 'my_view' %}

Will not work anymore when you namespace an app. You will need to do something like::

    {% url 'my_namespace:my_view' %}

The problem is now if you attach apps to multiple pages your namespace will change.
The solution for this problem are application namespaces.

If you'd like to use application namespaces to reverse the URLs related to
your app, you can assign a value to the `app_name` attribute of your app
hook like this::

    class MyNamespacedApphook(CMSApp):
        name = _("My Namespaced Apphook")
        urls = ["myapp.urls"]
        app_name = "myapp_namespace"

    apphook_pool.register(MyNamespacedApphook)


.. note::
    If you do provide an ``app_label``, then you will need to also give the app
    a unique namespace in the advanced settings of the page. If you do not, and
    no other instance of the app exists using it, then the 'default instance
    namespace' will be automatically set for you. You can then either reverse
    for the namespace(to target different apps) or the app_name (to target
    links inside the same app).

If you use app namespace you will need to give all your view ``context`` a ``current_app``::

  def my_view(request):
      current_app = resolve(request.path_info).namespace
      context = RequestContext(request, current_app=current_app)
      return render_to_response("my_templace.html", context_instance=context)

.. note::
    You need to set the current_app explicitly in all your view contexts as django does not allow an other way of doing
    this.

You can reverse namespaced apps similarly and it "knows" in which app instance it is:

.. code-block:: html+django

    {% url myapp_namespace:app_main %}

If you want to access the same url but in a different language use the language
template tag:

.. code-block:: html+django

    {% load i18n %}
    {% language "de" %}
        {% url myapp_namespace:app_main %}
    {% endlanguage %}


.. note::

    The official Django documentation has more details about application and
    instance namespaces, the `current_app` scope and the reversing of such
    URLs. You can look it up at https://docs.djangoproject.com/en/dev/topics/http/urls/#url-namespaces

When using the `reverse` function, the `current_app` has to be explicitly passed
as an argument. You can do so by looking up the `current_app` attribute of
the request instance::

    def myviews(request):
        current_app = resolve(request.path_info).namespace

        reversed_url = reverse('myapp_namespace:app_main',
                current_app=current_app)
        ...

Or, if you are rendering a plugin, of the context instance::

    class MyPlugin(CMSPluginBase):
        def render(self, context, instance, placeholder):
            # ...
            current_app = resolve(request.path_info).namespace
            reversed_url = reverse('myapp_namespace:app_main',
                    current_app=current_app)
            # ...

.. _apphook_permissions:

Apphook Permissions
-------------------

By default all apphooks have the same permissions set as the page they are assigned to.
So if you set login required on page the attached apphook and all it's urls have the same
requirements.

To disable this behavior set ``permissions = False`` on your apphook::

    class SampleApp(CMSApp):
        name = _("Sample App")
        urls = ["project.sampleapp.urls"]
        permissions = False



If you still want some of your views to have permission checks you can enable them via a decorator:

``cms.utils.decorators.cms_perms``

Here is a simple example::

    from cms.utils.decorators import cms_perms

    @cms_perms
    def my_view(request, **kw):
        ...


If you have your own permission check in your app, or just don't want to wrap some nested apps
with CMS permission decorator, then use ``exclude_permissions`` property of apphook::

    class SampleApp(CMSApp):
        name = _("Sample App")
        urls = ["project.sampleapp.urls"]
        permissions = True
        exclude_permissions = ["some_nested_app"]


For example, django-oscar_ apphook integration needs to be used with exclude permissions of dashboard app,
because it use `customizable access function`__. So, your apphook in this case will looks like this::

    class OscarApp(CMSApp):
        name = _("Oscar")
        urls = [
            patterns('', *application.urls[0])
        ]
        exclude_permissions = ['dashboard']

.. _django-oscar: https://github.com/tangentlabs/django-oscar
.. __: https://github.com/tangentlabs/django-oscar/blob/0.7.2/oscar/apps/dashboard/nav.py#L57

Automatically restart server on apphook changes
-----------------------------------------------

As mentioned above, whenever you add or remove an apphook, change the slug of a
page containing an apphook or the slug if a page which has a descendant with an
apphook, you have to restart your server to re-load the URL caches. To allow
you to automate this process, the django CMS provides a signal
:obj:`cms.signals.urls_need_reloading` which you can listen on to detect when
your server needs restarting. When you run ``manage.py runserver`` a restart
should not be needed.

.. warning::

    This signal does not actually do anything. To get automated server
    restarting you need to implement logic in your project that gets
    executed whenever this signal is fired. Because there are many ways of
    deploying Django applications, there is no way we can provide a generic
    solution for this problem that will always work.

.. warning::

    The signal is fired **after** a request. If you change something via API
    you need a request for the signal to fire.


.. _integration_modifiers:

********************
Navigation Modifiers
********************

Navigation Modifiers give your application access to navigation menus.

A modifier can change the properties of existing nodes or rearrange entire
menus.


An example use-case
-------------------

A simple example: you have a news application that publishes pages
independently of django CMS. However, you would like to integrate the
application into the menu structure of your site, so that at appropriate 
places a *News* node appears in the navigation menu.

In such a case, a Navigation Modifier is the solution.


How it works
------------

Normally, you'd want to place modifiers in your application's 
``menu.py``.

To make your modifier available, it then needs to be registered with 
``menus.menu_pool.menu_pool``.

Now, when a page is loaded and the menu generated, your modifier will
be able to inspect and modify its nodes.

A simple modifier looks something like this::

    from menus.base import Modifier
    from menus.menu_pool import menu_pool

    class MyMode(Modifier):
        """

        """
        def modify(self, request, nodes, namespace, root_id, post_cut, breadcrumb):
            if post_cut:
                return nodes
            count = 0
            for node in nodes:
                node.counter = count
                count += 1
            return nodes
    
    menu_pool.register_modifier(MyMode)

It has a method :meth:`~menus.base.Modifier.modify` that should return a list
of :class:`~menus.base.NavigationNode` instances.
:meth:`~menus.base.Modifier.modify` should take the following arguments:

- request

  A Django request instance. You want to modify based on sessions, or
  user or permissions?

- nodes

  All the nodes. Normally you want to return them again.

- namespace

  A Menu Namespace. Only given if somebody requested a menu with only nodes
  from this namespace.

- root_id

  Was a menu request based on an ID?

- post_cut

  Every modifier is called two times. First on the whole tree. After that the
  tree gets cut to only show the nodes that are shown in the current menu.
  After the cut the modifiers are called again with the final tree. If this is
  the case ``post_cut`` is ``True``.

- breadcrumb

  Is this not a menu call but a breadcrumb call?


Here is an example of a built-in modifier that marks all node levels::


    class Level(Modifier):
        """
        marks all node levels
        """
        post_cut = True

        def modify(self, request, nodes, namespace, root_id, post_cut, breadcrumb):
            if breadcrumb:
                return nodes
            for node in nodes:
                if not node.parent:
                    if post_cut:
                        node.menu_level = 0
                    else:
                        node.level = 0
                    self.mark_levels(node, post_cut)
            return nodes

        def mark_levels(self, node, post_cut):
            for child in node.children:
                if post_cut:
                    child.menu_level = node.menu_level + 1
                else:
                    child.level = node.level + 1
                self.mark_levels(child, post_cut)
    
    menu_pool.register_modifier(Level)


.. _integration_customplugins:

**************
Custom Plugins
**************

If you want to display content of your apps on other pages custom plugins are
a great way to accomplish that. For example, if you have a news app and you
want to display the top 10 news entries on your homepage, a custom plugin is
the way to go.

For a detailed explanation on how to write custom plugins please head over to
the :doc:`custom_plugins` section.


.. _integration_toolbar:

*******
Toolbar
*******

Your app might also want to integrate in the :doc:`toolbar` to
provide a more streamlined user experience for your admins.


.. _integration_templates:

**********************
Working with templates
**********************

Application can reuse cms templates by mixing cms templatetags and normal django
templating language.


static_placeholder
------------------

Plain :ttag:`placeholder` cannot be used in templates used by external applications,
use :ttag:`static_placeholder` instead.

.. _page_template:

CMS_TEMPLATE
------------
.. versionadded:: 3.0

``CMS_TEMPLATE`` is a context variable available in the context; it contains
the template path for CMS pages and application using apphooks, and the default
template (i.e.: the first template in :setting:`CMS_TEMPLATES`) for non-CMS
managed urls.

This is mostly useful to use it in the ``extends`` templatetag in the application
templates to get the current page template.

Example: cms template

.. code-block:: html+django

    {% load cms_tags %}
    <html>
        <body>
        {% cms_toolbar %}
        {% block main %}
        {% placeholder "main" %}
        {% endblock main %}
        </body>
    </html>


Example: application template

.. code-block:: html+django

    {% extends CMS_TEMPLATE %}
    {% load cms_tags %}
    {% block main %}
    {% for item in object_list %}
        {{ item }}
    {% endfor %}
    {% static_placeholder "sidebar" %}
    {% endblock main %}

``CMS_TEMPLATE`` memorizes the path of the cms template so the application
template can dynamically import it.


render_model
------------
.. versionadded:: 3.0

:ttag:`render_model` allows to edit the django models from the frontend by
reusing the django CMS frontend editor.