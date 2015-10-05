tri.declarative
===============

tri.declarative contains tools to make it easy to create declarative constructs in your code. 

@declarative
------------

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
        

evaluate_recursive + filter_show
--------------------------------

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
        
`evaluate_recursive` will return a new list with new instances of MenuItem where the 
member `show` on the MenuItem baz has been evaluated to True or False. Then `filter_show` will 
return a new list without all MenuItems where show is False.


tri.* design philosophy
=======================

Usages of `__` syntax
---------------------

We really like the syntax in django queries where you can do `Q(foo__bar=1)` to traverse attributes or table relations. We think this syntax can be used much more generally. For example in tri.form you can use the `__` syntax to easily create forms that can edit fields in other tables. Say you have a `User` model but you store other data in another model called `UserDetails` you can do `user_details__eye_color = Field()` in your form to access that property. 

But we also use it to expose configurability of underlying layers from the layers above. An example of this is in tri.query you can declare a `Variable` which is a thing you can search for. This can have an HTML GUI implemented by a tri.form `Field`. Now, say you want to change something in the display of this GUI. Normally in an OOP world you have to subclass `Field` to create your changed behavior, then subclass `Variable` to use your new class as the GUI. This becomes a lot of code, especially if this configuration is a one-off! 

In tri.query instead this is accomplished by having kwargs with a prefix followed by `__` dispatched to the underlying library. Example: 'foo = Variable(gui__required=True)`. The `Variable` class knows nothing about the `required` parameters. It only needs to know that kwargs starting with `gui__` are dispatched to `Field`. This can be done in many layers.

This design philosophy creates layers that compose cleanly without losing any of the flexibility of the layers below.

Callables everywhere
--------------------

In order to create nice declarative code that still works for dynamic situations some things needs to be specified as behaviors, not as static data. To make this easy we aim to make all configuration parameters support both a value directly but also accept callables. The evaluation from a callable to the concrete value is performed as late as possible to enable maximum amount of dynamic behavior.

Things that relate to eachother should be close together
--------------------------------------------------------

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

