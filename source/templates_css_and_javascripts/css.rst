====
CSS
====

.. admonition:: Description

    Creating and registering CSS files for Plone and Plone add-on products.
    CSS related Python functionality.

.. contents::

Introduction
==============

This page has Plone-specific CSS instructions.

In Plone, most of CSS files are managed by ``portal_css`` tool via the Zope
Management Interface.  Page templates can still import CSS files directly,
but ``portal_css`` does CSS file compression and merging automatically if
used.

Registering a new CSS file
==========================

You can register stylesheets to be included in Plone's various CSS bundles
using GenericSetup XML.

Example *profiles/default/cssregistry.xml*

.. code-block:: xml

    <?xml version="1.0"?>
    <!-- This file holds the setup configuration for the portal_css tool. -->

    <object name="portal_css">

     <!-- Stylesheets that will be registered with the portal_css tool are defined
          here. You can also specify values for existing resources if you need to
          modify some of their properties.
          Stylesheet elements accept these parameters:
          - 'id' (required): it must respect the name of the css or DTML file
            (case sensitive). '.dtml' suffixes must be ignored.
          - 'expression' (optional - default: ''): a tal condition.
          - 'media' (optional - default: ''): possible values: 'screen', 'print',
            'projection', 'handheld'...
          - 'rel' (optional - default: 'stylesheet')
          - 'rendering' (optional - default: 'import'): 'import', 'link' or
            'inline'.
          - 'enabled' (optional - default: True): boolean
          - 'cookable' (optional - default: True): boolean (aka 'merging allowed')

          See registerStylesheet() arguments in
          ResourceRegistries/tools/CSSRegistry.py for the latest list of all
          available keys and default values.
          -->


         <stylesheet
            id="++resource++yourproduct.something/yourstylesheet.css"
            media="" rel="stylesheet" rendering="import"
            cacheable="True" compression="safe" cookable="True"
            enabled="1" expression="" insert-after="ploneKss.css"/>

    </object>

Expressions
-----------

The ``expression`` attribute of ``portal_css`` defines when your CSS file is
included on an HTML page.  For more information see
:doc:`expressions documentation </functionality/expressions>`.

Inserting CSS as last into anonomyous bundles
---------------------------------------------

Plone compresses and merges CSS files to *bundles*.

For Plone 3.x optimal place to put CSS file available to all users is after
``ploneKss.css`` as in the example above.

CSS files for logged-in members only
--------------------------------------

Add the following expression to your CSS file::

    not: portal/portal_membership/isAnonymousUser

If you want to load the CSS in the bundle with Plone's default
``member.css``, use ``insert-after="member.css"``. In this case, however,
the file will be one of the first CSS files to be loaded and cannot override
values from other files unless CSS directive !import is used.

Conditional comments (IE)
==============================

* http://plone.org/products/plone/roadmap/232a

``cssregistry.xml`` example:

.. code-block :: xml

 <!-- Load IE6 - IE8 only stylsheet to fix layout problems -->
 <stylesheet title="" applyPrefix="False" authenticated="False"
    cacheable="True" compression="safe" conditionalcomment="lt IE 9"
    cookable="True" enabled="1" expression="" id="++resource++plonetheme.xxx.stylesheets/ie.css"
    media="screen" rel="stylesheet" rendering="link" insert-before="ploneCustom.css" />


Generating CSS classes programmatically in templates
====================================================

# Try to put string generation code in your view/viewlet if you have one.

# If you do not have a view (``main_template``) you can create a view and
  call it as in the following example.

View class generating CSS class spans::

    from Products.Five.browser import BrowserView
    from Products.CMFCore.utils  import getToolByName

    class CSSHelperView(BrowserView):
        """ Used by main_template <body> to set CSS classes """

        def __init__(self, context, request):
            self.context = context
            self.requet = request

        def logged_in_class(self):
            """ Get CSS class telling whether the user is logged in or not

            This allows us to fine-tune layout when edit frame et. al.
            are on the screen.
            """
            mt = getToolByName(self.context, 'portal_membership')
            if mt.isAnonymousUser(): # the user has not logged in
                return "member-anonymous"
            else:
                return "member-logged-in"

Registering the view in ZCML::

    <browser:view
            for="*"
            name="css_class_helper"
            class=".views.CSSHelperView"
            permission="zope.Public"
            allowed_attributes="logged_in_class"
            />

Calling the view in main_template.pt::

    <body tal:define="css_class_helper nocall:here/@@css_class_helper" tal:attributes="class string:${here/getSectionFromURL} template-${template/id} ${css_class_helper/logged_in_class};
                          dir python:test(isRTL, 'rtl', 'ltr')">

Defining CSS styles reaction to the presence of the class::

    #region-content { padding: 0 0 0 0px !important;}
    .member-logged-in #region-content { padding: 0 0 0 4px !important;}

Per-folder CSS theme overrides
=================================

* http://pypi.python.org/pypi/Products.CustomOverrides

Striping listing colours
==========================

In your template you can define classes for 1) the item itself 2) extra odd
and even class

.. code-block:: html

     <div tal:attributes="class python:'feed-folder-item feed-folder-item-' + (repeat['child'].even() and 'even' or 'odd')">

And you can colorize this with CSS

.. code-block:: css

    .feed-folder-item {
            padding: 0.5em;
    }

    /* Make sure that all items have same amount of padding at the bottom,
    whether they have last paragraph with margin or not.*/
    #content .feed-folder-item p:last-child {
        margin-bottom: 0;
    }

    .feed-folder-item-odd {
        background: #ddd;
    }

    .feed-folder-item-even {
        background: white;
    }


``plone.css``
=============

``plone.css`` is automagically generated dynamically based on the full
``portal_css`` registry configuration.  It is used in e.g. TinyMCE to load
all CSS styles into TinyMCE ``<iframe>`` in a single pass. It is not
used on the normal Plone pages.

``plone.css`` generation:

* https://github.com/plone/Products.CMFPlone/blob/master/Products/CMFPlone/skins/plone_scripts/plone.css.py

CSS reset
===========

If you are building a custom theme and you want to do cross-browser CSS
reset, the following snippet is recommended

.. code-block:: css

    /* @group CSS Reset .*/

    /* Remove implicit browser styles to have a neutral starting point:
       - No elements should have implicit margin/padding
       - No underline by default on links (we add it explicitly in the body text)
       - When we want markers on lists, we will be explicit about it, and they render inline by default
       - Browsers are inconsistent about hX/pre/code, reset
       - Linked images should not have borders
       .*/

    * { margin: 0; padding: 0; }
    * :link,:visited { text-decoration:none }
    * ul,ol { list-style:none; }
    * li { display: inline; }
    * h1,h2,h3,h4,h5,h6,pre,code { font-size:1em; }
    * a img,:link img,:visited img { border:none }
    a { outline: none; }
    table { border-spacing: 0; }
    img { vertical-align: middle; }

Adding new CSS body classes
=============================

Plone themes provide ``<body>`` CSS classes to identify view, template, site
section, etc. for theming.

The default body CSS classes look like this

.. code-block:: html

  <body class="template-subjectgroup portaltype-xxx-app-subjectgroup site-LS section-courses icons-on" dir="ltr">

But you can include your own CSS classes as well.
This can be done by overriding ``plone.app.layout.globals.LayoutPolicy``
class which is registerd as ``plone_layout`` view.

``layout.py``

.. code-block:: python

    """ Override the default Plone layout utility.
    """

    from zope.component import queryUtility
    from zope.component import queryMultiAdapter

    from plone.i18n.normalizer.interfaces import IIDNormalizer
    from plone.app.layout.globals import layout as base

    class LayoutPolicy(base.LayoutPolicy):
        """
        Enhanched layout policy helper.

        Extend the Plone standard class to have some more <body> CSS classes
        based on the current context.
        """

        def bodyClass(self, template, view):
            """Returns the CSS class to be used on the body tag.
            """

            # Call parent
            body_class = base.LayoutPolicy.bodyClass(self, template, view)

            # Include context and parent classes
            normalizer = queryUtility(IIDNormalizer)

            body_class += " context-" + normalizer.normalize(self.context.getId())

            parent = self.context.aq_parent

            # Check that we have a valid parent
            if hasattr(parent, "getId"):
                body_class += " parent-" + normalizer.normalize(parent.getId())

            return body_class

Related ZCML registration

.. code-block:: xml

    <browser:page
        name="plone_layout"
        for="*"
        permission="zope.Public"
        class=".layout.LayoutPolicy"
        allowed_interface="plone.app.layout.globals.interfaces.ILayoutPolicy"
        />
