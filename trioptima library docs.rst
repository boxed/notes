tri.declarative
===============

tri.declarative contains tools to make it easy to create declarative constructs in your code. 

:code:`@declarative`
---------------------

Easily write libraries with APIs like: 

.. code:: python

    class FooTable(Table):
        foo = Column()
        bar = Column()

    f = FooTable() # equivalent to `Table([Column(name='foo'), Column('bar')])`


Write the implementation of your API like this:

.. code:: python

    @creation_ordered
    class Column(object):
        def __init__(self, name):
            # ...
    
    @declarative('columns')
    class Table(object):
        def __init__(self, columns):
            self.columns = columns
            # ...

This makes it super easy to create declarative style APIs for all your code.
        

:code:`evaluate_recursive` + :code:`filter_show`
------------------------------------------------

Define some immutable structure that you evaluate at some later time. We've used this for 
example to define the behavior of menus. Instead of this:

.. code:: python
    
    def menu_view_func(request):
        menu_items = [
            MenuItem('foo'), 
            MenuItem('bar'),
        ]
        if request.user.is_staff:
            menu_items.append(MenuItem('baz'))
        menu_items.append(MenuItem('quuz'))
        return render_menu(menu_items)
    
write this:

.. code:: python

    menu_items = [
        MenuItem('foo'), 
        MenuItem('bar'),
        MenuItem('baz', show=lambda request: request.user.is_staff),
        MenuItem('quuz'),
    ]
    
    def menu_view_func(request):
        return render_menu(filter_show(evaluate_recursive(menu_items, request=request)))
        
:code:`evaluate_recursive` will return a new list with new instances of :code:`MenuItem` where the 
member :code:`show` on the :code:`MenuItem` baz has been evaluated to True or False. Then :code:`filter_show` will return a new list without all MenuItems where show is False.


tri.* design philosophy
=======================

Usages of :code:`__` syntax
---------------------

We really like the syntax in django queries where you can do :code:`Q(foo__bar=1)` to traverse attributes or table relations. We think this syntax can be used much more generally. For example in tri.form you can use the :code:`__` syntax to easily create forms that can edit fields in other tables. Say you have a :code:`User` model but you store other data in another model called :code:`UserDetails` you can do :code:`user_details__eye_color = Field()` in your form to access that property. 

But we also use it to expose configurability of underlying layers from the layers above. An example of this is in tri.query you can declare a :code:`Variable` which is a thing you can search for. This can have an HTML GUI implemented by a tri.form :code:`Field`. Now, say you want to change something in the display of this GUI. Normally in an OOP world you have to subclass :code:`Field` to create your changed behavior, then subclass :code:`Variable` to use your new class as the GUI. This becomes a lot of code, especially if this configuration is a one-off! 

In tri.query instead this is accomplished by having kwargs with a prefix followed by :code:`__` dispatched to the underlying library. Example: :code:`foo = Variable(form_field__required=True)`. The :code:`Variable` class knows nothing about the :code:`required` parameters. It only needs to know that kwargs starting with :code:`form_field__` are dispatched to :code:`Field`. This can be done in many layers.

This design philosophy creates layers that compose cleanly without losing any of the flexibility of the layers below.

To support this style of working we provide the functions :code:`extract_subkeys`, :code:`setattr_path` and :code:`getattr_path`.

Strongly opinionated defaults that are easy to override
-------------------------------------------------------

Opinionated defaults means that the Right Thing is also the Easy Thing. Having defaults for URL patterns for editing of objects for example means that your URLs will be consistent because it's just simpler to go with the flow.

Sometimes though you need to do something special. Maybe you need to preserve some old URL pattern that was put in place a long time ago. For cases like this it is important that strongly opinionated defaults can be overridden.

And in order to make overriding defaults more flexible we support callables everywhere we can. More on that:

Callables everywhere
--------------------

In order to create nice declarative code that still works for dynamic situations some things needs to be specified as behaviors, not as static data. To make this easy we aim to make all configuration parameters support both a value directly but also accept callables. The evaluation from a callable to the concrete value is performed as late as possible to enable maximum amount of dynamic behavior.

Layered à la carte customization
--------------------------------

A common problem with many library designs is that they fall into two categories:

- Lots and lots of boilerplate to get going (e.g. Win32 API)
- Super easy to get going, but if you want to customize something you have to rewrite huge chunks or even the entire thing (.NET GUI tables, djangos ModelForm)

We believe it doesn't have to be like this. An example of a design that handles this nicely is tri.form. You can create a form easily like this:

.. code:: python

    form = Form.from_model(data=request.GET, model=Foo)
    
but at the same time you can go in and set some specific option on a field that was auto generated:

.. code:: python

    form = Form.from_model(data=request.GET, model=Foo, some_field_name__help_text='Help text')
    
Another example is in tri.table where you can configure the rendering of an HTML table on these levels (and you can use none to all of them at the same time!):

- cell_value: how to extract the value to be rendered from a row
- cell_format: how to format the extracted value into a string
- cell_template: an HTML template to render the contents of the :code:`td` freely
- row_template: template for the :code:`tr`
- table_template: template for the entire :code:`table` tag

(There are other configuration options, but these are enough to demonstrate the point)

This type of API design creates a smooth gradient from just-give-me-something to I'll-configure-it-fully-myself. 

Immutability
------------

When writing declarative definitions for what you want with lambdas and then evaluating them, it's very important that nothing of the evaluated state persists until the next run through of the code. Having all the APIs for declarative structures being immutable makes sure that mistakes are caught easily.

Things that relate to each other should be close together
---------------------------------------------------------

A good example for code where this rule is not applied is django forms:

.. code:: python
    
    class FooForm(Form):
        bar = CharField()
        
        [... lots and lots of field definitions ...]
        
        def clean_bar(self):
            # SURPRISE! Here we totally change the behavior of bar!
            
In tri.form we make sure that the behavior that relates to a field is declared on the field:

.. code:: python
    
    class FooForm(Form):
        bar = CharField(parse=lambda ...)  # or you can create a staticmethod on FooForm and reference it here

