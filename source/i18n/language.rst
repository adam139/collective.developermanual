--------------------
Language functions
--------------------

.. admonition:: Description

    Acessing and changing the language state of Plone programmatically. 

.. contents:: :local:

Introduction
============

Each session has a language associated with it.

The active language is negotiated by the ``plone.i18n.negotiator`` module.
Several factors may involve determining what the language should be:

* Cookies (setting from the language selector)

* Domain name (like .fi for Finnish, .se for Swedish)

* Context (current content) language

* Browser language headers

Language is negotiated at the beginning of the page view.

Getting the current language
============================

Example view/viewlet method of getting the current language.

.. code-block:: python

    from Acquisition import aq_inner
    from zope.component import getMultiAdapter
    
    class MyView(BrowserView):

        ...

        def language(self):
            """
            @return: Two-letter string, the active language code
            """
            context = aq_inner(self.context)
            portal_state = getMultiAdapter((context, self.request), name=u'plone_portal_state')
            current_language = portal_state.language()
            return current_language


Getting language of content item
================================

All content objects don't necessarily support the ``Language()`` look-up
defined by the ``IDublinCore`` interface. Below is the safe way to extract
the served language on the content.

Example BrowserView method::

    from Acquisition import aq_inner

    def language(self):
        """ Get the language of the context.

        Useful in producing <html> tag.
        You need to output language for every HTML page, see http://www.w3.org/TR/xhtml1/#strict


        @return: The two letter language code of the current content.
        """
        portal_state = self.context.unrestrictedTraverse("@@plone_portal_state")

        return aq_inner(self.context).Language() or portal_state.default_language()

Simple language conditions in page templates
===============================================

You can do this if full translation strings are not worth the trouble::

   <div class="main-text">
     <a tal:condition="python:context.restrictedTraverse('@@plone_portal_state').language() == 'fi'" href="http://www.saariselka.fi/sisalto?force-web">Siirry täydelle web-sivustolle</a>
     <a tal:condition="python:context.restrictedTraverse('@@plone_portal_state').language() != 'fi'" href="http://www.saariselka.fi/sisalto?force-web">Go to full website</a>
   </div>


Set site language settings
==========================

Manually::

    # Setup site language settings
    portal = context.getSite()
    ltool = portal.portal_languages
    defaultLanguage = 'en'
    supportedLanguages = ['en','es']
    ltool.manage_setLanguageSettings(defaultLanguage, supportedLanguages,
                                          setUseCombinedLanguageCodes=False)

For unit testing, you need to run this in ``afterSetUp()`` after setting up
the languages::

    # THIS IS FOR UNIT TESTING ONLY
    # Normally called by pretraverse hook,
    # but must be called manually for the unit tests
    # Goes only for the current request
    ltool.setLanguageBindings()

Using ``GenericSetup`` and ``propertiestool.xml``

.. code-block:: xml

    <object name="portal_properties" meta_type="Plone Properties Tool">
       <object name="site_properties" meta_type="Plone Property Sheet">
          <property name="default_language" type="string">en</property>
       </object>
    </object>

On ``LinguaPlone``-enabled sites, using GenericSetup XML
``portal_languages.xml``

.. code-block:: xml

    <?xml version="1.0"?>
    <object>
     <default_language value="fi"/>
     <use_path_negotiation value="False"/>
     <use_cookie_negotiation value="True"/>
     <use_request_negotiation value="False"/>
     <use_cctld_negotiation value="False"/>
     <use_combined_language_codes value="False"/>
     <display_flags value="True"/>
     <start_neutral value="False"/>
     <supported_langs>
      <element value="en"/>
      <element value="fi"/>
     </supported_langs>
    </object>


Customizing language selector
=============================

Multilingual Plone has two kinds of language selector viewlets:

* Plone vanilla

* LinguaPlone -  LinguaPlone has its own language selector which replaces
  the default Plone selector if the add on product is installed.


More information

* https://svn.plone.org/svn/plone/plone.app.i18n/trunk/plone/app/i18n/locales/browser/selector.py

* https://svn.plone.org/svn/plone/plone.app.i18n/trunk/plone/app/i18n/locales/browser/languageselector.pt

* http://svn.plone.org/svn/plone/Products.LinguaPlone/tags/2.4/Products/LinguaPlone/browser/selector.py

Making language flags point to different top level domains
----------------------------------------------------------

If you use multiple domain names for different languages it is often
desirable to make the language selector point to a different domain. Search
engines do not really like the dynamic language switchers and will index
switching links, messing up your site search results.

Example

.. code-block:: html

    <tal:language
        tal:define="available view/available;
                    languages view/languages;
                    showFlags view/showFlags;">


        <ul id="portal-languageselector"
            tal:condition="python:available and len(languages)>=2">
            <tal:language repeat="lang languages">
            <li tal:define="code lang/code;
                            selected lang/selected"
                tal:attributes="class python: selected and 'currentLanguage' or '';">

                    <a href=""
                       tal:condition="python:code =='fi'"
                       tal:define="flag lang/flag|nothing;
                                   name lang/name"
                       tal:attributes="href string:http://www.twinapex.fi;
                                       title name">
                        <tal:flag condition="python:showFlags and flag">
                            <img
                                 width="14"
                                 height="11"
                                 alt=""
                                 tal:attributes="src string:${view/portal_url}${flag};
                                                 title python: name;
                                                 class python: selected and 'currentItem' or '';" />
                        </tal:flag>
                        <tal:nonflag condition="python:not showFlags or not flag"
                                     replace="name">language name</tal:nonflag>
                    </a>

                    <a href=""
                       tal:condition="python:code =='en'"
                       tal:define="flag lang/flag|nothing;
                                   name lang/name"
                       tal:attributes="href string:http://www.twinapex.com;
                                       title name">
                        <tal:flag condition="python:showFlags and flag">
                            <img
                                 width="14"
                                 height="11"
                                 alt=""
                                 tal:attributes="src string:${view/portal_url}${flag};
                                                 title python: name;
                                                 class python: selected and 'currentItem' or '';" />
                        </tal:flag>
                        <tal:nonflag condition="python:not showFlags or not flag"
                                     replace="name">language name</tal:nonflag>
                    </a>&nbsp;

            </li>
            </tal:language>
        </ul>
    </tal:language>


Custom language negotiator
==========================

Below some example code.

``languages.py``::

        """ Custom language negotiator based on hostname.  
        """

        from Products.PloneLanguageTool import LanguageTool
        
        # These are default languages available when hostname cannot be solved
        all_languages = [ "fi", "en" ]
        
        def get_host_name(request):
            """ Extract host name in virtual host safe manner 
            
            @param request: HTTPRequest object, assumed contains environ dictionary
            
            @return: Host DNS name, as requested by client. Lowercased, no port part.
            """
            
            if "HTTP_X_FORWARDED_HOST" in request.environ:
                # Virtual host
                host = request.environ["HTTP_X_FORWARDED_HOST"]
            elif "HTTP_HOST" in request.environ:
                # Direct client request
                host = request.environ["HTTP_HOST"]
            else:
                host = None
                return host
                
            # separate to domain name and port sections
            host=host.split(":")[0].lower()
                
            return host 
        
        
        def get_language(domain_name):
            """    
            @param domain_name: Full qualified domain name of HTTP request
            """
            
            if domain_name.endswith(".mobi") or domain_name.endswith(".com"):
                return "en"
            elif domain_name.endswith(".fi"):
                return "fi"
            else:
                return "en"         
        
        def getCcTLDLanguages(self):
            """
            Monkey-patched top level domain language negotiator.
            
            This will be installed by collective.monkeypatcher.
            """        
            
            if not hasattr(self, 'REQUEST'):
                return None
            
            request = self.REQUEST
            
            # Could not extract hostname
            hostname = get_host_name(request)
                
            if not hostname:
                return all_languages
                             
            # Limit available languages based on hostname
            langs = [ get_language(hostname) ]
            
            return langs
            
        # Also we need to fix a bug present in Plone 3.3.5
        # 
        #    @memoize
        #    def language(self):
        #        # TODO Looking for lower-case language is wrong, the negotiator
        #        # machinery uses uppercase LANGUAGE. We cannot change this as long
        #        # as we don't ship with a newer PloneLanguageTool which respects
        #        # the content language, though.
        #        return self.request.get('language', None) or \
        #                aq_inner(self.context).Language() or self.default_language()
        
        from plone.memoize.view import memoize, memoize_contextless
        
        def working_portal_state_language(self):
                return self.request.get('LANGUAGE', None) or \
                        self.request.get('language', None) or \
                        aq_inner(self.context).Language() or \
                        self.default_language()
        
        working_portal_state_language = memoize(working_portal_state_language)

``configure.zcml``

.. code-block:: xml

  <!-- Use collective.monkeypatcher to introduce our custom language negotiation phase -->  
  <monkey:patch
        description="Add custom TLD language resolution"
        class="Products.PloneLanguageTool.LanguageTool"
        original="getCcTLDLanguages"
        replacement=".languages.getCcTLDLanguages"
        />
  
  <monkey:patch
        description="Fix Plone 3.3.5 bug"
        class="plone.app.layout.globals.portal.PortalState"
        original="language"
        replacement=".languages.working_portal_state_language"
        />
  
Login aware language negotiation
==========================================

Because language negotiation happens before the authentication by default
and if you wish to use authenticated credentials in the negotiation you 
can do the following.

This can be done by hooking to after traversal event.

Example event registration

.. code-block:: xml

    <configure
        xmlns="http://namespaces.zope.org/zope"
        xmlns:browser="http://namespaces.zope.org/browser"
        xmlns:zcml="http://namespaces.zope.org/zcml"
        >
        <subscriber handler=".language_negotiation.Negotiator"/>
    </configure>

Related event handler::

    ################################################################
    # weishaupt.policy
    # (C) 2012, ZOPYX Ltd.
    ################################################################
    
    from zope.interface import Interface
    from zope.component import adapter
    from ZPublisher.interfaces import IPubEvent,IPubAfterTraversal
    from Products.CMFCore.utils import getToolByName
    from AccessControl import getSecurityManager
    from zope.app.component.hooks import getSite
    
    @adapter(IPubAfterTraversal)
    def Negotiator(event):
    
        # Keep the current request language (negotiated on portal_languages)
        # untouched
    
        site = getSite()
        ms = getToolByName(site, 'portal_membership')
        member = ms.getAuthenticatedMember()
        if member.getUserName() == 'Anonymous User':
            return
    
        language = member.language
        if language:
            # Fake new language for all authenticated users
            event.request['LANGUAGE'] = language
        else:
            lt = getToolByName(site, 'portal_languages')
            event.request['LANGUAGE'] = lt.getDefaultLanguage()

Other
=====

* http://reinout.vanrees.org/weblog/2007/12/14/translating-schemata-names.html

* http://maurits.vanrees.org/weblog/archive/2007/09/i18n-locales-and-plone-3.0

* http://blogs.ingeniweb.com/blogs/user/7/tag/i18ndude/

* http://plone.org/products/archgenxml/documentation/how-to/handling-i18n-translation-files-with-archgenxml-and-i18ndude/view?searchterm=



