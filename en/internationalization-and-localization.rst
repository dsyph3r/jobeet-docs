NOT CONVERTED TO SYMFONY2 YET
=============================

Want to help out?
Fork it on `Github <http://github.com/symfonytuts/jobeet-docs>`_

Day 19: Internationalization and Localization
=============================================

Yesterday, we finished the search engine feature by making it more
fun with the addition of some AJAX goodness. Now, we will talk
about Jobeet
**internationalization** (or
i18n) and
**localization** (or
l10n).

From
`Wikipedia <http://en.wikipedia.org/wiki/Internationalization>`_:

    **Internationalization** is the process of designing a software
    application so that it can be adapted to various
    languages and regions without engineering
    changes.

    **Localization** is the process of adapting software for a specific
    region or language by adding locale-specific
    components and translating text.


As always, the symfony framework has not reinvented the wheel and
its i18n and l10n supports is based on the
`ICU standard <http://www.icu-project.org/>`_.

User
----

No internationalization is possible without a user. When your
website is available in several languages or for different regions
of the world, the user is responsible for choosing the one that
fits him best.

    **NOTE** We have already talked about the symfony User class during
    day 13.


The User Culture
~~~~~~~~~~~~~~~~~~

The i18n and l10n features of symfony are based on the
**user culture**. The culture is the
combination of the language and the country of the user. For
instance, the culture for a user that speaks French is ``fr`` and
the culture for a user from France is ``fr_FR``.

You can manage the user culture by calling the ``setCulture()`` and
``getCulture()`` methods on the ``User`` object:

::

    <?php
    // in an action
    $this->getUser()->setCulture('fr_BE');
    echo $this->getUser()->getCulture();

    **TIP** The language is coded in two lowercase
    characters, according to the
    `ISO 639-1 standard <http://en.wikipedia.org/wiki/ISO_639-1>`_, and
    the country is coded in two uppercase characters, according to the
    `ISO 3166-1 standard <http://en.wikipedia.org/wiki/ISO_3166-1>`_.


The Preferred Culture
~~~~~~~~~~~~~~~~~~~~~

By default, the user culture is the one configured in the
``settings.yml`` configuration file:

::

    [yml]
    # apps/frontend/config/settings.yml
    all:
      .settings:
        default_culture: it_IT

    **TIP** As the culture is managed by the User object, it is stored
    in the user session. During development, if you change
    the default culture, you will have to clear your
    session cookie for the new setting to have any
    effect in your browser.


When a user starts a session on the Jobeet website, we can also
determine the best culture, based on the information provided by
the ``Accept-Language`` HTTP header.

The ``getLanguages()`` method of the request object returns an
array of accepted languages for the current user, sorted by order
of preference:

::

    <?php
    // in an action
    $languages = $request->getLanguages();

But most of the time, your website won't be available in the
world's 136 major languages. The ``getPreferredCulture()`` method
returns the best language by comparing the user preferred languages
and the supported languages of your website:

::

    <?php
    // in an action
    $language = $request->getPreferredCulture(array('en', 'fr'));

In the previous call, the returned language will be English or
French according to the user preferred languages, or English (the
first language in the array) if none match.

Culture in the URL
------------------

The Jobeet website will be available in English and French. As an
URL can only represent a single resource, the culture must be
embedded in the URL. In order to do that, open the
``routing.yml`` file, and add the special
``:sf_culture`` variable for all routes but the ``api_jobs`` and
the ``homepage`` ones. For simple routes, add ``/:sf_culture`` to
the front of the ``url``. For collection routes, add a
``prefix_path`` option that starts with
``/:sf_culture``.

::

    [yml]
    # apps/frontend/config/routing.yml
    affiliate:
      class: sfPropelRouteCollection
      options:
        model:          JobeetAffiliate
        actions:        [new, create]
        object_actions: { wait: get }
        prefix_path:    /:sf_culture/affiliate
    
    category:
      url:     /:sf_culture/category/:slug.:sf_format
      class:   sfPropelRoute
      param:   { module: category, action: show, sf_format: html }
      options: { model: JobeetCategory, type: object }
      requirements:
        sf_format: (?:html|atom)
    
    job_search:
      url:   /:sf_culture/search
      param: { module: job, action: search }
    
    job:
      class: sfPropelRouteCollection
      options:
        model:          JobeetJob
        column:         token
        object_actions: { publish: put, extend: put }
        prefix_path:    /:sf_culture/job
      requirements:
        token: \w+
    
    job_show_user:
      url:     /:sf_culture/job/:company_slug/:location_slug/:id/:position_slug
      class:   sfPropelRoute

options: model: JobeetJob type: object method\_for\_criteria:
doSelectActive options: model: JobeetJob type: object
method\_for\_query: retrieveActiveJob param: { module: job, action:
show } requirements: id: + sf\_method: get

When the ``sf_culture`` variable is used in a route,
symfony will automatically use its value to change the culture of
the user.

As we need as many homepages as languages we support (``/en/``,
``/fr/``, ...), the default homepage (``/``) must redirect to the
appropriate localized one, according to the user culture. But if
the user has no culture yet, because he comes to Jobeet for the
first time, the preferred culture will be chosen for him.

First, add the ``isFirstRequest()`` method to ``myUser``. It
returns ``true`` only for the very first request of a user
session:

::

    <?php
    // apps/frontend/lib/myUser.class.php
    public function isFirstRequest($boolean = null)
    {
      if (is_null($boolean))
      {
        return $this->getAttribute('first_request', true);
      }
    
      $this->setAttribute('first_request', $boolean);
    }

Add a ``localized_homepage`` route:

::

    [yml]
    # apps/frontend/config/routing.yml
    localized_homepage:
      url:   /:sf_culture/
      param: { module: job, action: index }
      requirements:
        sf_culture: (?:fr|en)

Change the ``index`` action of the ``job`` module to implement the
logic to redirect the user to the "best" homepage on the first
request of a session:

::

    <?php
    // apps/frontend/modules/job/actions/actions.class.php
    public function executeIndex(sfWebRequest $request)
    {
      if (!$request->getParameter('sf_culture'))
      {
        if ($this->getUser()->isFirstRequest())
        {
          $culture = $request->getPreferredCulture(array('en', 'fr'));
          $this->getUser()->setCulture($culture);
          $this->getUser()->isFirstRequest(false);
        }
        else
        {
          $culture = $this->getUser()->getCulture();
        }
    
        $this->redirect('localized_homepage');
      }

$this->categories = JobeetCategoryPeer::getWithJobs();
$this->categories =
Doctrine\_Core::getTable('JobeetCategory')->getWithJobs(); }

If the ``sf_culture`` variable is not present in the request, it
means that the user has come to the ``/`` URL. If this is the case
and the session is new, the preferred culture is used as the user
culture. Otherwise the user's current culture is used.

The last step is to redirect the user to the ``localized_homepage``
URL. Notice that the ``sf_culture`` variable has not been passed in
the redirect call as symfony adds it automatically for you.

Now, if you try to go to the ``/it/`` URL, symfony will return a
404 error as we have restricted the ``sf_culture``
variable to ``en``, or ``fr``. Add this requirement to all the
routes that embed the culture:

::

    [yml]
    requirements:
      sf_culture: (?:fr|en)

Culture`\  \ :sub:`Testing
-------------------------------------

It is time to test our implementation. But before adding more
tests, we need to fix the existing ones. As all URLs have changed,
edit all functional test files in ``test/functional/frontend/`` and
add ``/en`` in front of all URLs. Don't forget to also change the
URLs in the ``lib/test/JobeetTestFunctional.class.php`` file.
Launch the test suite to check that you have correctly fixed the
tests:

$ php symfony test:functional frontend

The user tester provides an ``isCulture()`` method that tests the
current user's culture. Open the ``jobActionsTest`` file and add
the following tests:

::

    <?php
    // test/functional/frontend/jobActionsTest.php
    $browser->setHttpHeader('ACCEPT_LANGUAGE', 'fr_FR,fr,en;q=0.7');
    $browser->
      info('6 - User culture')->
    
      restart()->
    
      info('  6.1 - For the first request, symfony guesses the best culture')->
      get('/')->
      with('response')->isRedirected()->
      followRedirect()->
      with('user')->isCulture('fr')->
    
      info('  6.2 - Available cultures are en and fr')->
      get('/it/')->
      with('response')->isStatusCode(404)
    ;
    
    $browser->setHttpHeader('ACCEPT_LANGUAGE', 'en,fr;q=0.7');
    $browser->
      info('  6.3 - The culture guessing is only for the first request')->
    
      get('/')->
      with('response')->isRedirected()->
      followRedirect()->
      with('user')->isCulture('fr')
    ;

Language Switching
------------------

For the user to change the culture, a language
form must be added in the layout. The form
framework does not provide such a form out of the box but as the
need is quite common for internationalized websites, the symfony
core team maintains the
```sfFormExtraPlugin`` <http://www.symfony-project.org/plugins/sfFormExtraPlugin?tab=plugin_readme>`_,
which contains validators,
widgets, and forms which cannot be included
with the main symfony package as they are too specific or have
external dependencies but are nonetheless very useful.

Install the plugin with the ``plugin:install`` task:

::

    $ php symfony plugin:install sfFormExtraPlugin

Or via Subversion with the following command:

::

    $  svn co http://svn.symfony-project.org/plugins/sfFormExtraPlugin/branches/1.3/ plugins/sfFormExtraPlugin

In order for plugin's classes to be loaded, the
``sfFormExtraPlugin`` plugin must be activated in the
``config/ProjectConfiguration.class.php`` file as shown below:

::

    <?php
    // config/ProjectConfiguration.class.php
    public function setup()
    {
      $this->enablePlugins(array(
        'sfDoctrinePlugin', 
        'sfDoctrineGuardPlugin',
        'sfFormExtraPlugin'
      ));
    }

    **NOTE** The ``sfFormExtraPlugin`` contains widgets that require
    external dependencies like JavaScript libraries. You will find a
    widget for rich date selectors, one for a WYSIWYG editor, and much
    more. Take the time to read the documentation as you will find a
    lot of useful stuff.


The ``sfFormExtraPlugin`` plugin provides a ``sfFormLanguage`` form
to manage the language selection. Adding the language form can be
done in the layout like this:

    **NOTE** The code below is not meant to be implemented. It is here
    to show you how you might be tempted to implement something in the
    wrong way. We will go on to show you how to implement it properly
    using symfony.


::

    <?php
    // apps/frontend/templates/layout.php
    <div id="footer">
      <div class="content">
        <!-- footer content -->
    
        <?php $form = new sfFormLanguage(
          $sf_user,
          array('languages' => array('en', 'fr'))
          )
        ?>
        <form action="<?php echo url_for('change_language') ?>">
          <?php echo $form ?><input type="submit" value="ok" />
        </form>
      </div>
    </div>

Do you spot a problem? Right, the form object creation does not
belong to the View layer. It must be created from an action. But as
the code is in the layout, the form must be created for every
action, which is far from practical.

In such cases, you should use a **component**. A
component is like a partial but with some
code attached to it. Consider it as a lightweight action. Including
a component from a template can be done by using the
~``include_component()`` helper~:

::

    <?php
    // apps/frontend/templates/layout.php
    <div id="footer">
      <div class="content">
        <!-- footer content -->
    
        <?php include_component('language', 'language') ?>
      </div>
    </div>

The helper takes the module and the action as arguments. The third
argument can be used to pass parameters to the component.

Create a ``language`` module to host the component and the action
that will actually change the user language:

::

    $ php symfony generate:module frontend language

Components are to be defined in the
``actions/components.class.php`` file.

Create this file now:

::

    <?php
    // apps/frontend/modules/language/actions/components.class.php
    class languageComponents extends sfComponents
    {
      public function executeLanguage(sfWebRequest $request)
      {
        $this->form = new sfFormLanguage(
          $this->getUser(),
          array('languages' => array('en', 'fr'))
        );
      }
    }

As you can see, a components class is quite similar to an actions
class.

The template for a component uses the same naming convention as a
partial would: an underscore (``_``) followed by the component
name:

::

    <?php
    // apps/frontend/modules/language/templates/_language.php
    <form action="<?php echo url_for('change_language') ?>">
      <?php echo $form ?><input type="submit" value="ok" />
    </form>

As the plugin does not provide the action that actually changes the
user culture, edit the ``routing.yml`` file to create the
``change_language`` route:

::

    [yml]
    # apps/frontend/config/routing.yml
    change_language:
      url:   /change_language
      param: { module: language, action: changeLanguage }

And create the corresponding action:

::

    <?php
    // apps/frontend/modules/language/actions/actions.class.php
    class languageActions extends sfActions
    {
      public function executeChangeLanguage(sfWebRequest $request)
      {
        $form = new sfFormLanguage(
          $this->getUser(),
          array('languages' => array('en', 'fr'))
        );
    
        $form->process($request);
    
        return $this->redirect('localized_homepage');
      }
    }

The ``process()`` method of ``sfFormLanguage`` takes care of
changing the user culture, based on the user form submission.

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/19/footer.png
   :alt: Internationalized Footer
   
   Internationalized Footer

Internationalization
--------------------

Languages, Charset`\ , and \ :sub:`Encoding
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Different languages have different character sets. The English
language is the simplest one as it only uses the ASCII
characters, the French language is a bit more complex with
accentuated characters like "é", and languages like Russian,
Chinese, or Arabic are much more complex as all their characters
are outside the ASCII range. Such languages are defined with
totally different character sets.

When dealing with internationalized data, it is better to use the
unicode norm. The idea behind unicode is to
establish a universal set of characters that contains all
characters for all languages. The problem with unicode is that a
single character can be represented with as many as 21 octets.
Therefore, for the web, we use UTF-8, which maps Unicode
code points to variable-length sequences of octets. In UTF-8, most
used languages have their characters coded with less than 3
octets.

UTF-8 is the default encoding used by symfony, and it is defined in
the ``settings.yml`` configuration file:

::

    [yml]
    # apps/frontend/config/settings.yml
    all:
      .settings:
        charset: utf-8

Also, to enable the internationalization layer of symfony, you must
set the ``i18n`` setting to ``true`` in ``settings.yml``:

::

    [yml]
    # apps/frontend/config/settings.yml
    all:
      .settings:
        i18n: true

Templates
~~~~~~~~~

An internationalized website means that the user interface is
translated into several languages.

In a template, all strings that are language dependent must be
wrapped with the ~``__()`` helper~ (notice that there is two
underscores).

The ``__()`` helper is part of the ``I18N`` helper group, which
contains helpers that ease i18n management in templates. As this
helper group is not loaded by default, you need to either manually
add it in each template with ``use_helper('I18N')`` as we already
did for the ``Text`` helper group, or load it globally by adding it
to the ~``standard_helpers`` setting~:

::

    [yml]
    # apps/frontend/config/settings.yml
    all:
      .settings:
        standard_helpers: [Partial, Cache, I18N]

Here is how to use the ``__()`` helper for the Jobeet footer:

::

    <?php
    // apps/frontend/templates/layout.php
    <div id="footer">
      <div class="content">
        <span class="symfony">
          <img src="/images/jobeet-mini.png" />
          powered by <a href="http://www.symfony-project.org/">
          <img src="/images/symfony.gif" alt="symfony framework" /></a>
        </span>
        <ul>
          <li>
            <a href=""><?php echo __('About Jobeet') ?></a>
          </li>
          <li class="feed">
            <?php echo link_to(__('Full feed'), 'job', array('sf_format' => 'atom')) ?>
          </li>
          <li>
            <a href=""><?php echo __('Jobeet API') ?></a>
          </li>
          <li class="last">
            <?php echo link_to(__('Become an affiliate'), 'affiliate_new') ?>
          </li>
        </ul>
        <?php include_component('language', 'language') ?>
      </div>
    </div>

    **NOTE** The ``__()`` helper can take the string for the default
    language or you can also use a unique identifier for each string.
    It is just a matter of taste. For Jobeet, we will use the former
    strategy so templates are more readable.


When symfony renders a template, each time the ``__()`` helper is
called, symfony looks for a translation for the current user's
culture. If a translation is found, it is used, if not, the first
argument is returned as a fallback value.

All translations are stored in a ~catalogue\|Translations
Catalogue~. The i18n framework provides a lot of different
strategies to store the translations. We will use the
`"XLIFF" <http://en.wikipedia.org/wiki/XLIFF>`_ format,
which is a standard and the most flexible one. It is also the store
used by the admin generator and most symfony plugins.

    **NOTE** Other catalogue stores are ``gettext``,
    ``MySQL``, and ``SQLite``. As always, have a look at the
    `i18n API <http://www.symfony-project.org/api/1_4/i18n>`_ for more
    details.


``i18n:extract``
~~~~~~~~~~~~~~~~

Instead of creating the catalogue file by hand, use the built-in
``i18n:extract`` task:

::

    $ php symfony i18n:extract frontend fr --auto-save

The ``i18n:extract`` task finds all strings that need to be
translated in ``fr`` in the ``frontend`` application and creates or
updates the corresponding catalogue. The ``--auto-save`` option
saves the new strings in the catalogue. You can also use the
``--auto-delete`` option to automatically remove strings that do
not exist anymore.

In our case, it populates the file we have created:

::

    [xml]
    <!-- apps/frontend/i18n/fr/messages.xml -->
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE xliff PUBLIC "-//XLIFF//DTD XLIFF//EN"
      "http://www.oasis-open.org/committees/xliff/documents/xliff.dtd">
    <xliff version="1.0">
      <file source-language="EN" target-language="fr" datatype="plaintext"
          original="messages" date="2008-12-14T12:11:22Z"
          product-name="messages">
        <header/>
        <body>
          <trans-unit id="1">
            <source>About Jobeet</source>
            <target/>
          </trans-unit>
          <trans-unit id="2">
            <source>Feed</source>
            <target/>
          </trans-unit>
          <trans-unit id="3">
            <source>Jobeet API</source>
            <target/>
          </trans-unit>
          <trans-unit id="4">
            <source>Become an affiliate</source>
            <target/>
          </trans-unit>
        </body>
      </file>
    </xliff>

Each translation is managed by a ``trans-unit`` tag which has a
unique ``id`` attribute. You can now edit this file and add
translations for the French language:

::

    [xml]
    <!-- apps/frontend/i18n/fr/messages.xml -->
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE xliff PUBLIC "-//XLIFF//DTD XLIFF//EN"
      "http://www.oasis-open.org/committees/xliff/documents/xliff.dtd">
    <xliff version="1.0">
      <file source-language="EN" target-language="fr" datatype="plaintext"
          original="messages" date="2008-12-14T12:11:22Z"
          product-name="messages">
        <header/>
        <body>
          <trans-unit id="1">
            <source>About Jobeet</source>
            <target>A propos de Jobeet</target>
          </trans-unit>
          <trans-unit id="2">
            <source>Feed</source>
            <target>Fil RSS</target>
          </trans-unit>
          <trans-unit id="3">
            <source>Jobeet API</source>
            <target>API Jobeet</target>
          </trans-unit>
          <trans-unit id="4">
            <source>Become an affiliate</source>
            <target>Devenir un affilié</target>
          </trans-unit>
        </body>
      </file>
    </xliff>

    **TIP** As XLIFF is a standard format, a lot of tools exist to ease
    the translation process.
    `Open Language Tools <https://open-language-tools.dev.java.net/>`_
    is an Open-Source Java project with an integrated XLIFF editor.


-

    **TIP** As XLIFF is a file-based format, the same precedence and
    merging rules that exist for other symfony configuration files are
    also applicable. I18n files can exist in a project, an application,
    or a module, and the most specific file overrides translations
    found in the more global ones.


Translations with Arguments
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The main principle behind internationalization is to translate
whole sentences. But some sentences embed dynamic values. In
Jobeet, this is the case on the homepage for the "more..." link:

::

    <?php
    <!-- apps/frontend/modules/job/templates/indexSuccess.php -->
    <div class="more_jobs">
      and <?php echo link_to($count, 'category', $category) ?> more...
    </div>

The number of jobs is a variable that must be replaced by a
placeholder for translation:

::

    <?php
    <!-- apps/frontend/modules/job/templates/indexSuccess.php -->
    <div class="more_jobs">
      <?php echo __('and %count% more...', array('%count%' => link_to($count, 'category', $category))) ?>
    </div>

The string to be translated is now "and %count% more...", and the
``%count%`` placeholder will be replaced by the real number at
runtime, thanks to the value given as the second argument to the
``__()`` helper.

Add the new string manually by inserting a ``trans-unit`` tag in
the ``messages.xml`` file, or use the ``i18n:extract`` task to
update the file automatically:

::

    $ php symfony i18n:extract frontend fr --auto-save

After running the task, open the XLIFF file to add the French
translation:

::

    [xml]
    <trans-unit id="6">
      <source>and %count% more...</source>
      <target>et %count% autres...</target>
    </trans-unit>

The only requirement in the translated string is to use the
``%count%`` placeholder somewhere.

Some other strings are even more complex as they involve
plurals. According to some numbers, the
sentence changes, but not necessarily the same way for all
languages. Some languages have very complex grammar rules for
plurals, like Polish or Russian.

On the category page, the number of jobs in the current category is
displayed:

::

    <?php
    <!-- apps/frontend/modules/category/templates/showSuccess.php -->
    <strong><?php echo count($pager) ?></strong> jobs in this category

When a sentence has different translations according to a number,
the ``format_number_choice()`` helper should be used:

::

    <?php
    <?php echo format_number_choice(
        '[0]No job in this category|[1]One job in this category|(1,+Inf]%count% jobs in this category',
        array('%count%' => '<strong>'.count($pager).'</strong>'),
        count($pager)
      )
    ?>

The ~``format_number_choice()`` helper~ takes three arguments:


-  The string to use depending on the number
-  An array of placeholders
-  The number to use to determine which text to use

The string that describes the different translations according to
the number is formatted as follow:


-  Each possibility is separated by a pipe character (``|``)
-  Each string is composed of a range followed by the translation

The range can describe any range of numbers:


-  ``[1,2]``: Accepts values between 1 and 2, inclusive
-  ``(1,2)``: Accepts values between 1 and 2, excluding 1 and 2
-  ``{1,2,3,4}``: Only values defined in the set are accepted
-  ``[-Inf,0)``: Accepts values greater or equal to negative
   infinity and strictly less than 0
-  ``{n: n % 10 > 1 && n % 10 < 5}``: Matches numbers like 2, 3, 4,
   22, 23, 24

Translating the string is similar to other message strings:

::

    [xml]
    <trans-unit id="7">
      <source>[0]No job in this category|[1]One job in this category|(1,+Inf]%count% jobs in this category</source>
      <target>[0]Aucune annonce dans cette catégorie|[1]Une annonce dans cette catégorie|(1,+Inf]%count% annonces dans cette catégorie</target>
    </trans-unit>

Now that you know how to internationalize all kind of strings, take
the time to add ``__()`` calls for all templates of the frontend
application. We won'tt internationalize the backend application.

Forms
~~~~~~~~~~~~~~~~~~~~~

The form classes contain many strings that need to be translated,
like labels, error messages, and help messages. All these strings
are automatically internationalized by symfony, so you only need to
provide translations in the XLIFF files.

    **NOTE** Unfortunately, the ``i18n:extract`` task does not yet
    parse form classes for untranslated strings.


##ORM## Objects
~~~~~~~~~~~~~~~

For the Jobeet website, we won't ~internationalize all
tables\|Model Internationalization~ as it does not make sense to
ask the job posters to translate their job posts in
all available languages. But the category table definitely needs to
be translated.

The ##ORM## plugin supports i18n tables out of the box. For each
table that contains localized data, two tables need to be created:
one for columns that are i18n-independent, and the other one with
columns that need to be internationalized. The two tables are
linked by a one-to-many relationship.

Update the ``schema.yml`` accordingly:

[yml] # config/schema.yml jobeet\_category: \_attributes: { isI18N:
true, i18nTable: jobeet\_category\_i18n } id: ~

::

    jobeet_category_i18n:
      id:           { type: integer, required: true, primaryKey: true,
       ➥ foreignTable: jobeet_category, foreignReference: id }
      culture:      { isCulture: true, type: varchar, size: 7,
       ➥ required: true, primaryKey: true }
      name:         { type: varchar(255), required: true }
      slug:         { type: varchar(255), required: true }

The ``_attributes`` entry defines options for the table.

And update the fixtures for categories:

::

    [yml]
    # data/fixtures/010_categories.yml
    JobeetCategory:
      design:        { }
      programming:   { }
      manager:       { }
      administrator: { }
    
    JobeetCategoryI18n:
      design_en:        { id: design, culture: en, name: Design }
      programming_en:   { id: programming, culture: en, name: Programming }
      manager_en:       { id: manager, culture: en, name: Manager }
      administrator_en: { id: administrator, culture: en,
       ➥ name: Administrator }
    
      design_fr:        { id: design, culture: fr, name: Design }
      programming_fr:   { id: programming, culture: fr,
       ➥ name: Programmation }
      manager_fr:       { id: manager, culture: fr, name: Manager }
      administrator_fr: { id: administrator, culture: fr,
       ➥ name: Administrateur }

Rebuild the model to create the ``i18n`` stub classes:

::

    $ php symfony propel:build --all --no-confirmation
    $ php symfony cc

As the ``name`` and ``slug`` columns have been moved to the i18n
table, move the ``setName()`` method from ``JobeetCategory`` to
``JobeetCategoryI18n``:

::

    <?php
    // lib/model/JobeetCategoryI18n.php
    public function setName($name)
    {
      parent::setName($name);
    
      $this->setSlug(Jobeet::slugify($name));
    }

We also need to fix the ``getForSlug()`` method in
``JobeetCategoryPeer``:

::

    <?php
    // lib/model/JobeetCategoryPeer.php
    static public function getForSlug($slug)
    {
      $criteria = new Criteria();
      $criteria->addJoin(JobeetCategoryI18nPeer::ID, self::ID);
      $criteria->add(JobeetCategoryI18nPeer::CULTURE, 'en');
      $criteria->add(JobeetCategoryI18nPeer::SLUG, $slug);
    
      return self::doSelectOne($criteria);
    }

[yml] # config/doctrine/schema.yml JobeetCategory: actAs:
Timestampable: ~ I18n: fields: [name] actAs: Sluggable: { fields:
[name], uniqueBy: [lang, name] } columns: name: { type:
string(255), notnull: true }

By turning on the ``I18n`` behavior, a model named
``JobeetCategoryTranslation`` will be automatically created and the
specified ``fields`` are moved to that model.

Notice we simply turn on the ``I18n`` behavior and move the
``Sluggable`` behavior to be attached to the
``JobeetCategoryTranslation`` model which is automatically created.
The ``uniqueBy`` option tells the ``Sluggable`` behavior which
fields determine whether a slug is unique or not. In this case each
slug must be unique for each ``lang`` and ``name`` pair.

And update the fixtures for categories:

::

    [yml]
    # data/fixtures/categories.yml
    JobeetCategory:
      design:
        Translation:
          en:
            name: Design
          fr:
            name: design
      programming:
        Translation:
          en:
            name: Programming
          fr:
            name: Programmation
      manager:
        Translation:
          en:
            name: Manager
          fr:
            name: Manager
      administrator:
        Translation:
          en:
            name: Administrator
          fr:
            name: Administrateur

We also need to override the ``findOneBySlug()`` method in
``JobeetCategoryTable``. Since Doctrine provides some magic finders
for all columns in a model, we need to simply create the
``findOneBySlug()`` method so that we override the default magic
functionality Doctrine provides.

We need to make a few changes so that the category is retrieved
based on the english slug in the ``JobeetCategoryTranslation``
table.

::

    <?php
    // lib/model/doctrine/JobeetCategoryTable.cass.php
    public function findOneBySlug($slug)
    {
      $q = $this->createQuery('a')
        ->leftJoin('a.Translation t')
        ->andWhere('t.lang = ?', 'en')
        ->andWhere('t.slug = ?', $slug);
      return $q->fetchOne();
    }

Rebuild the model:

::

    $ php symfony doctrine:build --all --and-load --no-confirmation
    $ php symfony cc

    **TIP** As the ``propel:build --all --and-load`` removes all tables
    and data from the database, don't forget to re-create a user to
    access the Jobeet backend with the ``guard:create-user`` task.
    Alternatively, you can add a fixture file to add it automatically
    for you.


When building the model, symfony creates proxy methods in the main
``JobeetCategory`` object to conveniently access the i18n columns
defined in ``JobeetCategoryI18n``:

::

    <?php
    $category = new JobeetCategory();
    
    $category->setName('foo');       // sets the name for the current culture
    $category->setName('foo', 'fr'); // sets the name for French
    
    echo $category->getName();     // gets the name for the current culture
    echo $category->getName('fr'); // gets the name for French

When using the ``I18n`` behavior, proxies are created between the
``JobeetCategory`` object and the ``JobeetCategoryTranslation``
object so all the old functions for retrieving the category name
will still work and retrieve the value for the current culture.

::

    <?php
    $category = new JobeetCategory();
    $category->setName('foo'); // sets the name for the current culture
    $category->getName(); // gets the name for the current culture
    
    $this->getUser()->setCulture('fr'); // from your actions class
    
    $category->setName('foo'); // sets the name for French
    echo $category->getName(); // gets the name for French

>**TIP** >To reduce the number of ~database
requests\|Performances~, use the >``doSelectWithI18n()`` method
instead of the regular ``doSelect()`` one. It will >retrieve the
main object and the i18n one in one query. > >

.. raw:: html

   <?php
   >     
   
:math:`$categories = JobeetCategoryPeer::doSelectWithI18n($`c,
$culture); >**TIP** >To reduce the number of ~database
requests\|Performances~, join the >``JobeetCategoryTranslation`` in
your queries. It will retrieve the main object >and the i18n one in
one query. > >

.. raw:: html

   <?php
   >     
   
$categories = Doctrine\_Query::create() > ->from('JobeetCategory
c') > ->leftJoin('c.Translation t WITH t.lang = ?', $culture) >
->execute(); > >The ``WITH`` keyword above will append a condition
to the automatically added >``ON`` condition of the query. So, the
``ON`` condition of the join will end up >being. > > [sql] > LEFT
JOIN c.Translation t ON c.id = t.id AND t.lang = ?

As the ``category`` route is tied to the ``JobeetCategory`` model
class and because the ``slug`` is now part of
``JobeetCategoryI18n``, the route is not able because the ``slug``
is now part of the ``JobeetCategoryTranslation``, the route is not
able to retrieve the ``Category`` object automatically. To help the
routing system, let's create a method that will take care of object
retrieval:


.. raw:: html

   <?php
       // lib/model/JobeetCategoryPeer.php
       class JobeetCategoryPeer extends BaseJobeetCategoryPeer
       {
         static public function doSelectForSlug($parameters)
         {
           $criteria = new Criteria();
           $criteria->
   
addJoin(JobeetCategoryI18nPeer::ID, JobeetCategoryPeer::ID);
$criteria->add(JobeetCategoryI18nPeer::CULTURE,
$parameters['sf\_culture']);
$criteria->add(JobeetCategoryI18nPeer::SLUG, $parameters['slug']);

::

        return self::doSelectOne($criteria);
      }
    }

Since we already overrode the ``findOneBySlug()`` let's refactor a
little bit more so these methods can be shared. We'll create a new
``findOneBySlugAndCulture()`` and ``doSelectForSlug()`` methods and
change the ``findOneBySlug()`` method to simply use the
``findOneBySlugAndCulture()`` method.

::

    <?php
    // lib/model/doctrine/JobeetCategoryTable.class.php
    public function doSelectForSlug($parameters)
    {
      return $this->findOneBySlugAndCulture($parameters['slug'], $parameters['sf_culture']);
    }
    
    public function findOneBySlugAndCulture($slug, $culture = 'en')
    {
      $q = $this->createQuery('a')
        ->leftJoin('a.Translation t')
        ->andWhere('t.lang = ?', $culture)
        ->andWhere('t.slug = ?', $slug);
      return $q->fetchOne();
    }
    
    public function findOneBySlug($slug)
    {
      return $this->findOneBySlugAndCulture($slug, 'en');
    }

Then, use the ``method`` option to
tell the ``category`` route to use the ``doSelectForSlug()`` method
to retrieve the object:

::

    [yml]
    # apps/frontend/config/routing.yml
    category:
      url:     /:sf_culture/category/:slug.:sf_format
      class:   sfPropelRoute
      param:   { module: category, action: show, sf_format: html }
      options: { model: JobeetCategory, type: object, method: doSelectForSlug }
      requirements:
        sf_format: (?:html|atom)

We need to reload the fixtures to regenerate the proper slugs for
the categories:

::

    $ php symfony propel:data-load

Now the ``category`` route is internationalized and the URL for a
category embeds the translated category slug:

::

    /frontend_dev.php/fr/category/programmation
    /frontend_dev.php/en/category/programming

Admin Generator
~~~~~~~~~~~~~~~

For the backend, we want the French and the English translations to
be edited in the same form:

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/19/backend_categories.png
   :alt: Backend categories
   
   Backend categories

Embedding an i18n form can be done by using
the ``embedI18N()`` method:

::

    <?php
    // lib/form/JobeetCategoryForm.class.php
    class JobeetCategoryForm extends BaseJobeetCategoryForm
    {
      public function configure()
      {

unset($this['jobeet\_category\_affiliate\_list']); unset(
$this['jobeet\_affiliates\_list'], $this['created\_at'],
$this['updated\_at'] );

::

        $this->embedI18n(array('en', 'fr'));
        $this->widgetSchema->setLabel('en', 'English');
        $this->widgetSchema->setLabel('fr', 'French');
      }
    }

The admin generator interface supports internationalization out of
the box. It comes with translations for more than 20 languages, and
it is quite easy to add a new one, or to customize an existing one.
Copy the file for the language you want to customize from symfony
(admin translations are to be found in
``lib/vendor/symfony/lib/plugins/sfPropelPlugin/i18n/``) in the
application
``lib/vendor/symfony/lib/plugins/sfDoctrinePlugin/i18n/``) in the
application ``i18n`` directory. As the file in your application
will be merged with the symfony one, only keep the modified strings
in the application file.

You will notice that the admin generator translation files are
named like ``sf_admin.fr.xml``, instead of ``fr/messages.xml``. As
a matter of fact, ``messages`` is the name of the default catalogue
used by symfony, and can be changed to allow a better separation
between different parts of your application. Using a catalogue
other than the default one requires that you specify it when using
the ``__()`` helper:

::

    <?php
    <?php echo __('About Jobeet', array(), 'jobeet') ?>

In the above ``__()`` call, symfony will look for the "About
Jobeet" string in the ``jobeet`` catalogue.

Tests
~~~~~

Fixing tests is an integral part of the
internationalization migration. First, update the test fixtures for
categories by copying the fixtures we have defined above in
``test/fixtures/010_categories.yml``. define above in
``test/fixtures/categories.yml``.

Don't forget to update methods in the
``lib/test/JobeetTestFunctional.class.php`` file in order to care
of our modifications concerning the ``JobeetCategory``'s
internationalization.

::

    <?php
    public function getMostRecentProgrammingJob()
    {
      $q = Doctrine_Query::create()
        ->select('j.*')
        ->from('JobeetJob j')
        ->leftJoin('j.JobeetCategory c')
        ->leftJoin('c.Translation t')
        ->where('t.slug = ?', 'programming');
    
      $q = Doctrine_Core::getTable('JobeetJob')->addActiveJobsQuery($q);
    
      return $q->fetchOne();
    }

Rebuild the model for the ``test`` environment:

::

    $ php symfony propel:build --all --and-load --no-confirmation --env=test

You can now launch all tests to check that they are running fine:

::

    $ php symfony test:all

    **NOTE** When we have developed the backend interface for Jobeet,
    we have not written functional tests. But whenever you create a
    module with the symfony command line, symfony also generate test
    stubs. These stubs are safe to remove.


Localization
------------

Templates
~~~~~~~~~~~~~~~~~~~~

Supporting different cultures also means supporting different way
to format dates and numbers. In a template, several helpers are at
your disposal to help take all these differences into account,
based on the current user culture:

In the
```Date`` <http://www.symfony-project.org/api/1_4/DateHelper>`_
helper group:

\| Helper \| Description \| \| ------------------------------ \|
---------------------------------------------------------- \| \|
``format_date()`` \| Formats a date \| \| ``format_datetime()`` \|
Formats a date with a time (hours, minutes, seconds) \| \|
``time_ago_in_words()`` \| Displays the elapsed time between a date
and now in words \| \| ``distance_of_time_in_words()`` \| Displays
the elapsed time between two dates in words \| \|
``format_daterange()`` \| Formats a range of dates \|

In the
```Number`` <http://www.symfony-project.org/api/1_4/NumberHelper>`_
helper group:

\| Helper \| Description \| \| ------------------- \|
------------------ \| \| ``format_number()`` \| Formats a number \|
\| ``format_currency()`` \| Formats a currency \|

In the
```I18N`` <http://www.symfony-project.org/api/1_4/I18NHelper>`_
helper group:

\| Helper \| Description \| \| ------------------- \|
------------------------------- \| \| ``format_country()`` \|
Displays the name of a country \| \| ``format_language()`` \|
Displays the name of a language \|

~Forms (I18n)~
~~~~~~~~~~~~~~

The form framework provides several widgets and
validators for localized data:


-  ```sfWidgetFormI18nDate`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nDate>`_
-  ```sfWidgetFormI18nDateTime`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nDateTime>`_
-  ```sfWidgetFormI18nTime`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nTime>`_

-  ```sfWidgetFormI18nChoiceCountry`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nChoiceCountry>`_
-  ```sfWidgetFormI18nChoiceCurrency`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nChoiceCurrency>`_
-  ```sfWidgetFormI18nChoiceLanguage`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nChoiceLanguage>`_
-  ```sfWidgetFormI18nChoiceTimezone`` <http://www.symfony-project.org/api/1_4/sfWidgetFormI18nChoiceTimezone>`_

-  ```sfValidatorI18nChoiceCountry`` <http://www.symfony-project.org/api/1_4/sfValidatorI18nChoiceCountry>`_
-  ```sfValidatorI18nChoiceLanguage`` <http://www.symfony-project.org/api/1_4/sfValidatorI18nChoiceLanguage>`_
-  ```sfValidatorI18nChoiceTimezone`` <http://www.symfony-project.org/api/1_4/sfValidatorI18nChoiceTimezone>`_


Final Thoughts
--------------

Internationalization and localization are first-class citizens in
symfony. Providing a localized website to your users is very easy
as symfony provides all the basic tools and even gives you command
line tasks to make it fast.

Be prepared for a very special day as we will be moving a lot of
files around and exploring a different approach to organizing a
symfony project.

**ORM**


