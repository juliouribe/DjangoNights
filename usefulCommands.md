# Create a virtual environment. '.venv' is the folder where it will be created
python3 -m venv .venv

# Activate the environment
source .venv/bin/activate

# Deactivate
deactivate

# Install Django and Black formatter
python -m pip install django~=5.0.0 black

# Save packages to a requirements file
pip freeze > requirements.txt

# Create django project. Here django_project is the name and we're saying to create it in the current directory to avoid double folders.
django-admin startproject django_project .

# Create a django app
python manage.py startapp pages

# Run migrations
python manage.py migrate

# Run server
python manage.py runserver

# Run tests
python manage.py test

