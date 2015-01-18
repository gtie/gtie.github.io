---
layout: post
title: What I don't like about django-tables2 
---

You might already know, but in case you don't - [django-tables2](http://django-tables2.readthedocs.org) is a great library for easy rendering of HTML tables. It allows the form to be declared as a Python object, and then rendered with one swift template tag invocation. It's for tables what [django-crispy-forms](http://django-crispy-forms.readthedocs.org/en/latest/index.html) is for forms.

As someone, who doesn't like to fiddle with the frontend at all, apps like the above two are pretty great, especially in the prototyping phase. Plus, they have a some pretty significant benefits:
 * Generated HTML is clean and consistent across all pages that use those libraries
 * Reusing code thorugh Python inheritance (as opposed to the weaker reusal options givn by templates)
 * More compact template code (at the expense of extra Python code though)

This being said, there is one thing you need to consider when using that approach: you're breaking the MVC model. No excuses there.

Now, if you are thinking "To hell with your theoretical paradigms, I have a site to build!", I'll give you an example of how that can bite your ass in practice.

Let's say we have a django-tables2 object defined as:

{% highlight python %}
import django_tables2 as tables

class BooksTable (tables.Table):
    title = tables.Column()
    autor = tables.Column()
    genre = tables.Column()
{% endhighlight %}

And then you render it to a nice table looking like:

Title         | Author       | Genre   |
------------- | -------------|---------|
...           | ...          | ...     |
...           | ...          | ...     |


Compared to the standard rendering approach, we've moved some code from the template to the BooksTable class in Python, but it's all fine and dady at this point. No serious drawbacks to speak of. 

Imagine, however that you now want to add a 4th column with a "Delete" buton on it, and the functionality to go with it. Since it is a data-altering action, you will need a POST request there + CSRF protection (you never allow user actions to alter data without POST + CSRF, right??). 

In the regular template-rendered world, you would simply add a new cell there with a content like:

{% highlight html %}
      <td>
        <form action="{% raw %}{% url 'book_delete' book.pk %}{% endraw %}">
             {% raw %}{% csrf_token %}{% endraw %}
            <button class="btn-default btn-xs" type="submit">Delete</button>
        </form>
      </td>
{% endhighlight %}

If there is a dedicated frontend developer on the project, it's clear that applying this modification is his job, and it is clear where to add the new button and what the result would look like.

Can we do that with django-tables2? Yes, we can, but there are a number of obstacles, since we are buidling the form in the Python-code world, instead of the teamplate world. There are two major annoyances when dealing with this problem using django-tables2:
 * We have to put the entire form code inside a Python module (the alternative of rendering a template for each row a good solution is way too wasteful). This means that whenever the looks need to be adjusted, you (or even worse - your frontend dev) will need to edit the Python module to make their changes
 * The csrf token input is hard to get inside the Python code in an efficient manner. Furthermore, it makes the whole rendering process dependent on an extra variable, outside of the pure data rows.

The implementation that I ended up with is the following:

{% highlight python %}
from django.utils.safestring import mark_safe

import django_tables2 as tables
from django.template import RequestContext, Template

class BooksTable (tables.Table):
    title = tables.Column()
    autor = tables.Column()
    genre = tables.Column()
    delete = tables.Column(empty_values=(), sortable=False)
    
    def __init__ (self, request, *args, **kwargs):
        self.csrf_string = Template('{% raw %}{% csrf_token %}{% endraw %}').render(RequestContext(request))
        super(BooksTable, self).__init__(*args, **kwargs)
        
    def render_delete(self, record):
        output = '''
        <form action="{form_action}">
            {csrf_str}
            <button class="btn-default btn-xs" type="submit">Delete</button>
        </form>
        '''
        context = {'form_action': reverse('book_delete', args =[record.pk]),
                   'csrf_str' : self.csrf_string}
        return mark_safe(output.format(**context ))
{% endhighlight %}

It does render the __csrf_token__ tag just once, rather than on every row, so functionally it's OK. The Python/HTML separation is gone however, and now the frontend devs have to fiddle with the code inside Python modules to make their changes. Furthermore, the code starts looking like poorly implemented PHP (like there is any other kind:P). That's the price you're paying for messing up the MVC.