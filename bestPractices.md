# Namespace your templates
When creating a templates folder, if it's inside the app use namespacing to make it easier to find.
/pages/templates/pages.

# Use names in URLs
Pass in a name arg when defining a url and then use that in templates so you only need to update the url in one place
...
path("", home_page_view, name="home"),
...
 <a href="{% url 'home' %}">Home</a>

# Add __str__ to models to improve readibility
Adding def __str__(self): will update the admin page so we should always include this in models even if it's quick and simple.

# Use absolute URL with models to better link to detail pages
In your models file add something like this.
def get_absolute_url(self):
    return reverse("post_detail", kwargs={"pk": self.pk})
Then where you want to link in a template you can call
href="{% post.get_absolute_url %}"


