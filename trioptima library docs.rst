tri.declarative
===============

tri.declarative contains tools to make it easy to create declarative constructs in your code. 

@declarative
------------

Write the implementation of your API like this:

.. code::python

    class Column(object):
        def __init__(self, name):
            # ...
    
    class Table(object):
        def __init__(self, columns):
            self.columns = columns
            # ...
            
    foo_table = Table(columns=[Column('foo'), Column('bar')])

and just by adding the decorator `@declarative_member` on `Column` and `@declarative('columns')` on `Table` 
you can use the API as before AND like this:

.. code::python

    class FooTable(Table):
        foo = Column()
        bar = Column()
        

evaluate_recursive + filter_show
--------------------------------

Define some immutable structure that you evaluate at some later time. We've used this for 
example to define the behavior of menus. Instead of this:

.. code::python
    
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

.. code::python

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
