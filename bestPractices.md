# Namespace your templates
When creating a templates folder, if it's inside the app use namespacing to make it easier to find.
/pages/templates/pages.

# Use names in URLs
Pass in a name arg when defining a url and then use that in templates so you only need to update the url in one place
...
path("", home_page_view, name="home"),
...
 <a href="{% url 'home' %}">Home</a>

