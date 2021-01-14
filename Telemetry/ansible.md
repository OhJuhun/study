# Jinja Template
```jinja
{% if value is defined %}
    myValue     {{ value }}
{% else %}
    myValue     default
{% endif %}

# and 연산 대소문자 유의/ or도 같은 방식
{% if value is defined and value|bool == true %} # boolean
    myValue     {{ value }}
{% elif value is defined and value == "A" %}
    myValue     B
{% else %}
    myValue     C
{% endif %}

# for
{% for value in value_list %}
...
{% endfor %}

# for with if
{% for user in users if users %}
...
{% endfor %}
```