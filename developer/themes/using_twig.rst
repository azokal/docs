.. _using-twig:

==========
Using Twig
==========

.. Note::

    Twig is the default rendering engine for *Roadiz* CMS. You’ll find its documentation at http://twig.sensiolabs.org/doc/templates.html

When you use :ref:`Dynamic routing <dynamic-routing>` within your theme, Roadiz will automatically assign some variables for you.

* **request** — [object] Symfony request object which contains useful data such as current URI or GET parameters
* **head**
    * **ajax** — [boolean] Tells if current request is an Ajax one
    * **cmsVersion** — [string]
    * cmsVersionNumber
    * **cmsBuild** — [string]
    * **devMode** — [boolean]
    * **baseUrl** — [string] Server base Url. Basically your domain name, port and folder if you didn’t setup Roadiz at you server root
    * **filesUrl** — [string]
    * **resourcesUrl** — [string] Your theme ``Resources`` url. Useful to reach your assets.
    * **ajaxToken** — [string]
    * **universalAnalyticsId** — [string]
    * **useCdn** - [boolean]
    * **fontToken** — [string]
* **session**
    * **messages** — [array]
    * **id** — [string]
    * **user** — [object]
* **authorizationChecker** — [object]
* **tokenStorage** — [object]

There are some more content only available from *FrontendControllers*.

* **_default_locale** — [string]
* **meta**
    * **siteName** — [string]
    * **siteCopyright** — [string]
    * **siteDescription** — [string]

Then, in each dynamic routing *actions* you will need this line ``$this->storeNodeAndTranslation($node, $translation);``
in order to make page content available from your Twig template.

* **node** — [object]
* **nodeSource** — [object]
* **translation** — [object]
* **pageMeta**
    * **title** — [string]
    * **description** — [string]
    * **keywords** — [string]

All these data will be available in your Twig template using ``{{ }}`` syntax.
For example use ``{{ pageMeta.title }}`` inside your head’s ``<title>`` tag.
You can of course call objects members within Twig using the *dot* separator.

.. code-block:: html+jinja

    <article>
        <h1><a href="{{ path(nodeSource) }}">{{ nodeSource.title }}</a></h1>
        <div>{{ nodeSource.content|markdown }}</div>

        {# Use complex syntax to grab documents #}
        {% set images = nodeSource.handler.documentsFromFieldName('images') %}
        {# or Shortcut syntax #}
        {% set images = nodeSource.images %}

        {% for image in images %}
            {% set imageMetas = image.documentTranslations.first %}
            <figure>
                {{ image|display({'width':200 }) }}
                <figcaption>{{ imageMetas.name }} — {{ imageMetas.copyright }}</figcaption>
            </figure>
        {% endfor %}
    </article>

Generating paths and url
------------------------

Standard Twig ``path`` and ``url`` methods are both working for *static* and *dynamic* routing. In Roadiz, these methods
can take either a ``string`` identifier or a ``NodesSources`` instance. Of course optional parameters are available for
both, they will automatically create an *http query string* when using a node-source.

.. code-block:: html+jinja

    {# Path generation with a Symfony route  #}
    {# Eg. /fr  #}
    {{ path('homePageLocale', {_locale: 'fr'}) }}

    {# Path generation with a node-source  #}
    {# Eg. /en/about-us  #}
    {{ path(nodeSource) }}

    {# Url generation with a node-source  #}
    {# Eg. http://localhost:8080/en/about-us  #}
    {{ url(nodeSource) }}

    {# Path generation with a node-source and parameters  #}
    {# Eg. /en/about-us?page=2  #}
    {{ path(nodeSource, {'page': 2}) }}



Handling node-sources with Twig
-------------------------------

Most of yout front-end work will consist in editing *Twig* templating, *Twig* assignations and… *Twig* filters. Roadiz core entities are already linked together so you don’t have to prepare your data before rendering it. Basically, you can access *nodes* or *node-sources* data directly in *Twig* using the “dot” seperator.

There is even some magic about *Twig* when accessing private or protected fields:
just write the fieldname and it will use the getter method instead: ``{{ nodeSource.content|markdown }}`` will be interpreted as
``{{ nodeSource.getContent|markdown }}`` by *Twig*.

.. note::
    Roadiz will transform your node-type fields names to *camel-case* to create getters and setters into you NS class.
    So if you created a ``header_image`` field, getter will be named ``getHeaderImage()``.
    However, if you called it ``headerimage``, getter will be ``getHeaderimage()``

You can access methods too! You will certainly need to get node-sources’ documents to display them. Instead of declaring each document
in your PHP controller before, you can directly use them in *Twig*:

.. code-block:: html+jinja

    {% set images = nodeSource.images %}
    {% for image in images %}
        {% set imageMetas = image.documentTranslations.first %}
        <figure>
            {{ image|display({ 'width':200 }) }}
            <figcaption>{{ imageMetas.name }} — {{ imageMetas.copyright }}</figcaption>
        </figure>
    {% endfor %}

Loop over node-source children
------------------------------

With Roadiz you will be able to grab each node-source children using custom ``children`` Twig filter.
This filter is a shortcut for ``childBlock->getHandler()->getChildren(null, null, $authorizationChecker)``.

.. code-block:: html+jinja

    {% set childrenBlocks = nodeSource|children %}
    {% for childBlock in childrenBlocks %}
    <div class="block">
        <h2>{{ childBlock.title }}</h2>
        <div>{{ childBlock.content|markdown }}</div>
    </div>
    {% endfor %}

`getChildren method <http://api.roadiz.io/RZ/Roadiz/Core/Handlers/NodesSourcesHandler.html#method_getChildren>`_ must be called with a valid ``AuthorizationChecker`` instance if you **don’t want anonymous visitors to see unpublished contents**. Its first parameters can be set to filter over children and override default ordering. If your are using ``|children`` filter, *authorization-checker* is automatically passed to ``getChildren`` method.

.. code-block:: html+jinja

    {#
     # This statement will only grab *visible* children node-sources and
     # will order them ascendently according to their *title*.
     #}
    {% set childrenBlocks = nodeSource|children(
        {'node.visible': true},
        {'title': 'ASC'}
    ) %}

.. note::
    Calling ``getChildren()`` from a node-source *handler* or ``|children`` filter will **always** return ``NodesSources`` objects from the same translation as their parent.


Add previous and next links
---------------------------

In this example, we want to create links to jump to *next* and *previous* pages. We will use node-source handler methods
``getPrevious()`` and ``getNext()`` which work the same as ``getChildren()`` method.
``|previous`` and ``|next`` Twig filters are also available.

.. code-block:: html+jinja

    {% set prev = nodeSource|previous %}
    {% set next = nodeSource|next %}

    {% if (prev or next) %}
    <nav class="contextual-menu">
        {% if prev %}
        <a class="previous" href="{{ path(prev) }}"><i class="uk-icon-arrow-left"></i> {{ prev.title }}</a>
        {% endif %}
        {% if next %}
        <a class="next" href="{{ path(next) }}">{{ next.title }} <i class="uk-icon-arrow-right"></i></a>
        {% endif %}
    </nav>
    {% endif %}

.. note::
    Calling ``getPrevious`` and ``getNext`` from a node-source *handler* will **always** return ``NodesSources`` objects from
    the same translation as their sibling.


Additional filters
------------------

Roadiz’s Twig environment implements some useful filters, such as:

* ``markdown``: Convert a markdown text to HTML
* ``inlineMarkdown``: Convert a markdown text to HTML without parsing *block* elements (useful for just italics and bolds)
* ``markdownExtra``: Convert a markdown-extra text to HTML (footnotes, simpler tables, abbreviations)
* ``centralTruncate(length, offset, ellipsis)``: Generate an ellipsis at the middle of your text (useful for filenames). You can decenter the ellipsis position using ``offset`` parameter, and even change your ellipsis character with ``ellipsis`` parameter.

NodesSources filters
^^^^^^^^^^^^^^^^^^^^

These following Twig filters will only work with ``NodesSources`` entities… not ``Nodes``.
Use them with the *pipe* syntax, eg. ``nodeSource|next``.

* ``children``: shortcut for ``$source->getHandler()->getChildren()``
* ``next``: shortcut for ``$source->getHandler()->getNext()``
* ``previous``: shortcut for ``$source->getHandler()->getPrevious()``
* ``firstSibling``: shortcut for ``$source->getHandler()->getFirstSibling()``
* ``lastSibling``: shortcut for ``$source->getHandler()->getLastSibling()``
* ``parent``: shortcut for ``$source->getHandler()->getParent()``
* ``parents``: shortcut for ``$source->getHandler()->getParents()``
* ``tags``: shortcut for ``$source->getHandler()->getTags()``
* ``render(themeName)``: initiate a sub-request for rendering a given block *NodesSources*

Documents filters
^^^^^^^^^^^^^^^^^

These following Twig filters will only work with ``Document`` entities.
Use them with the *pipe* syntax, eg. ``document|display``.

* ``url``: returns document public URL as *string*. See :ref:`document URL options <display-documents>`.
* ``display``: generates an HTML tag to display your document. See :ref:`document display options <display-documents>`.
* ``imageRatio``: return image size ratio as *float*.
* ``imageSize``: returns image size as *array* with ``width`` and ``height``.
* ``imageOrientation``: get image orientation as *string*, returns ``landscape`` or ``portrait``.
* ``path``: shortcut for document real path on server.
* ``exists``: shortcut to test if document file exists on server. Returns ``boolean``.

Translations filters
^^^^^^^^^^^^^^^^^^^^

These following Twig filters will only work with ``Translation`` entities.
Use them with the *pipe* syntax, eg. ``translation|menu``.

* ``menu``: shortcut for ``$translation->getViewer()->getTranslationMenuAssignation()``.

This filter returns some useful informations about current page available languages and their
urls. See `getTranslationMenuAssignation method definition <http://api.roadiz.io/RZ/Roadiz/Core/Viewers/TranslationViewer.html#method_getTranslationMenuAssignation>`_.
You do not have to pass it the current request object as the filter will grab it
for you. But you can specify if you want *absolute* urls or not.


Standard filters and extensions are also available:

* ``{{ path('myRoute') }}``: for generating static routes Url.
* ``truncate`` and ``wordwrap`` which are parts of the `Text Extension <http://twig.sensiolabs.org/doc/extensions/text.html>`_ .


Create your own Twig filters
----------------------------

Imagine now that your are rendering some dynamic CSS stylesheets with Twig.
Your are listing your website projects which all have a distinct color. So you’ve created a
CSS route and a ``dynamic-colors.css.twig``.

.. code-block:: html+jinja

    {% for project in projects %}
    .{{ project.node.nodeName }} h1 {
        color: {{ project.color }};
    }
    {% endfor %}

This code should output a CSS like that:

.. code-block:: css

    .my-super-project h1 {
        color: #FF0000;
    }
    .my-second-project h1 {
        color: #00FF00;
    }

Then you should see your “super project” title in red on your website. OK, that’s great.
But what should I do if I need to use a RGBA color to control the Alpha channel value?
For example, I want to set project color to a ``<div class="date">`` background like this:

.. code-block:: css

    .my-super-project .date {
        background-color: rgba(255, 0, 0, 0.5);
    }
    .my-second-project .date {
        background-color: rgba(0, 255, 0, 0.5);
    }

*Great… I already see coming guys complaining that “rgba” is only supported since IE9… We don’t give a shit!…*

Hum, hum. So you need a super filter to extract decimal values from our backoffice stored hexadecimal color.
Roadiz enables us to extend Twig environment filters thanks to *dependency injection!*

You just have to extend ``setupDependencyInjection`` static method in your main
theme class. Create it if it does not exist yet.

.. code-block:: php

    // In your SuperThemeApp.php
    public static function setupDependencyInjection(\Pimple\Container $container)
    {
        parent::setupDependencyInjection($container);

        // We extend twig filters
        $container->extend('twig.filters', function ($filters, $c) {

            // The first filter will extract red value
            $red = new \Twig_SimpleFilter('red', function ($hex) {
                if ($hex[0] == '#' && strlen($hex) == 7) {
                    return hexdec(substr($hex, 1, 2));
                } else {
                    return 0;
                }
            });
            $filters->add($red);

            // The second filter will extract green value
            $green = new \Twig_SimpleFilter('green', function ($hex) {
                if ($hex[0] == '#' && strlen($hex) == 7) {
                    return hexdec(substr($hex, 3, 2));
                } else {
                    return 0;
                }
            });
            $filters->add($green);

            // The third filter will extract blue value
            $blue = new \Twig_SimpleFilter('blue', function ($hex) {
                if ($hex[0] == '#' && strlen($hex) == 7) {
                    return hexdec(substr($hex, 5, 2));
                } else {
                    return 0;
                }
            });
            $filters->add($blue);

            // Then we return our extended filters collection
            return $filters;
        });
    }

And… Voilà! You can use ``red``, ``green`` and ``blue`` filters in your Twig template.

.. code-block:: html+jinja

    {% for project in projects %}
    .{{ project.node.nodeName }} .date {
        background-color: rgba({{ project.color|red }}, {{ project.color|green }}, {{ project.color|blue }}, 0.5);
    }
    {% endfor %}

Use custom Twig extensions
--------------------------

Just like you did to add your own *Twig* filters, you can add your own *Twig* extensions.
Instead of extending ``twig.filters`` service, just extend ``twig.extensions`` service.

.. code-block:: php

    // In your SuperThemeApp.php
    public static function setupDependencyInjection(\Pimple\Container $container)
    {
        parent::setupDependencyInjection($container);

        // We extend twig extensions
        $container->extend('twig.extensions', function ($extensions, $c) {
            $extensions->add(new MySuperThemeTwigExtension());
            return $extensions;
        });
    }
