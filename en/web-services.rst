NOT CONVERTED TO SYMFONY2 YET
=============================

Want to help out?
Fork it on `Github <http://github.com/symfonytuts/jobeet-docs>`_

Day 15: Web Services
====================

With the addition of feeds on Jobeet, job seekers can now be
informed of new jobs in real-time.

On the other side of the fence, when you post a job, you will want
to have the greatest exposure possible. If your job is syndicated
on a lot of small websites, you will have a better chance to find
the right person. That's the power of the
`long tail <http://en.wikipedia.org/wiki/The_Long_Tail>`_.
Affiliates will be able to publish the latest posted jobs on their
websites thanks to the web services we will develop
along this day.

Affiliates
---------------------

As per day 2 requirements:

"Story F7: An affiliate retrieves the current active job list"

The Fixtures
~~~~~~~~~~~~

Let's create a new fixture file for the
affiliates:

::

    [yml]

# data/fixtures/030\_affiliates.yml # data/fixtures/affiliates.yml
JobeetAffiliate: sensio\_labs: url: http://www.sensio-labs.com/
email: fabien.potencier@example.com is\_active: true token:
sensio\_labs jobeet\_category\_affiliates: [programming]
JobeetCategories: [programming]

::

      symfony:
        url:       http://www.symfony-project.org/
        email:     fabien.potencier@example.org
        is_active: false
        token:     symfony

jobeet\_category\_affiliates: [design, programming]
JobeetCategories: [design, programming]

Creating records for the middle table of a many-to-many
relationship is as simple as defining an array with a key of the
middle table name plus an ``s``. Creating records for many-to-many
relationships is as simple as defining an array with the key which
is the name of the relationship. The content of the array is the
object names as defined in the fixture files. You can link objects
from different files, but the names must be defined first.

In the fixtures file, tokens are hardcoded to simplify the testing,
but when an actual user applies for an account, the
token will need to be generated:


.. raw:: html

   <?php
       // lib/model/JobeetAffiliate.php
       class JobeetAffiliate extends BaseJobeetAffiliate
       {
         public function save(PropelPDO $con = null)
         {
           if (!$this->
   
getToken()) {
:math:`$this->setToken(sha1($`this->getEmail().rand(11111,
99999))); }

::

        return parent::save($con);
      }
    
      // ...
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetAffiliate.class.php
       class JobeetAffiliate extends BaseJobeetAffiliate
       {
         public function save(Doctrine_Connection $conn = null)
         {
           if (!$this->
   
getToken()) {
:math:`$this->setToken(sha1($`this->getEmail().rand(11111,
99999))); }

::

        return parent::save($conn);
      }
    
      // ...
    }

You can now reload the data:

::

    $ php symfony propel:data-load

The Job Web Service
~~~~~~~~~~~~~~~~~~~

As always, when you create a new resource, it's a good habit to
define the URL first:

::

    [yml]
    # apps/frontend/config/routing.yml
    api_jobs:
      url:     /api/:token/jobs.:sf_format
      class:   sfPropelRoute
      param:   { module: api, action: list }
      options: { model: JobeetJob, type: list, method: getForToken }
      requirements:
        sf_format: (?:xml|json|yaml)

For this route, the special ``sf_format`` variable ends
the URL and the valid values are ``xml``, ``json``, or ``yaml``.

The ``getForToken()`` method is called when the action retrieves
the collection of objects related to the route. As we need to check
that the affiliate is activated, we need to override the default
behavior of the route:


.. raw:: html

   <?php
       // lib/model/JobeetJobPeer.php
       class JobeetJobPeer extends BaseJobeetJobPeer
       {
         static public function getForToken(array $parameters)
         {
           $affiliate = JobeetAffiliatePeer::getByToken($parameters['token']);
           if (!$affiliate || !$affiliate->
   
getIsActive()) { throw new sfError404Exception(sprintf('Affiliate
with token "%s" does not exist or is not activated.',
$parameters['token'])); }

::

        return $affiliate->getActiveJobs();
      }
    
      // ...
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJobTable.class.php
       class JobeetJobTable extends Doctrine_Table
       {
         public function getForToken(array $parameters)
         {
           $affiliate = Doctrine_Core::getTable('JobeetAffiliate')
             ➥ ->
   
findOneByToken(:math:`$parameters['token']); if (!$`affiliate \|\|
!$affiliate->getIsActive()) { throw new
sfError404Exception(sprintf('Affiliate with token "%s" does not
exist or is not activated.', $parameters['token'])); }

::

        return $affiliate->getActiveJobs();
      }
    
      // ...
    }

If the token does not exist in the database, we throw an
``sfError404Exception`` exception. This exception class is then
automatically converted to a ``404|404 Error`` response.
This is the simplest way to generate a ``404`` page from a model
class.

The ``getForToken()`` method uses two new methods we will create
now.

First, the ``getByToken()`` method must be created to get an
affiliate given its token:

::

    <?php
    // lib/model/JobeetAffiliatePeer.php
    class JobeetAffiliatePeer extends BaseJobeetAffiliatePeer
    {
      static public function getByToken($token)
      {
        $criteria = new Criteria();
        $criteria->add(self::TOKEN, $token);
    
        return self::doSelectOne($criteria);
      }
    }

Then, the ``getActiveJobs()`` method returns the list of currently
active jobs for the categories selected by the affiliate: The
``getForToken()`` method uses one new method named
``getActiveJobs()`` and returns the list of currently active jobs:


.. raw:: html

   <?php
       // lib/model/JobeetAffiliate.php
       class JobeetAffiliate extends BaseJobeetAffiliate
       {
         public function getActiveJobs()
         {
           $cas = $this->
   
getJobeetCategoryAffiliates();
:math:`$categories = array(); foreach ($`cas as $ca) {
$categories[] = $ca->getCategoryId(); }

::

        $criteria = new Criteria();
        $criteria->add(JobeetJobPeer::CATEGORY_ID, $categories, Criteria::IN);
        JobeetJobPeer::addActiveJobsCriteria($criteria);
    
        return JobeetJobPeer::doSelect($criteria);
      }
    
      // ...
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetAffiliate.class.php
       class JobeetAffiliate extends BaseJobeetAffiliate
       {
         public function getActiveJobs()
         {
           $q = Doctrine_Query::create()
             ->
   
select('j.\*') ->from('JobeetJob j') ->leftJoin('j.JobeetCategory
c') ->leftJoin('c.JobeetAffiliates a') ->where('a.id = ?',
$this->getId());

::

        $q = Doctrine_Core::getTable('JobeetJob')->addActiveJobsQuery($q);
    
        return $q->execute();
      }
    
      // ...
    }

The last step is to create the ``api`` action and templates.
Bootstrap the module with the ``generate:module`` task:

::

    $ php symfony generate:module frontend api

    **NOTE** As we won't use the default ``index`` action, you can
    remove it from the action class, and remove the associated template
    ``indexSucess.php``.


The Action
~~~~~~~~~~

All formats share the same ``list`` action:

::

    <?php
    // apps/frontend/modules/api/actions/actions.class.php
    public function executeList(sfWebRequest $request)
    {
      $this->jobs = array();
      foreach ($this->getRoute()->getObjects() as $job)
      {
        $this->jobs[$this->generateUrl('job_show_user', $job, true)] =
         ➥ $job->asArray($request->getHost());
      }
    }

Instead of passing an array of ``JobeetJob`` objects to the
templates, we pass an array of strings. As we have three different
templates for the same action, the logic to process the values has
been factored out in the ``JobeetJob::asArray()`` method:

::

    <?php

// lib/model/JobeetJob.php //
lib/model/doctrine/JobeetJob.class.php class JobeetJob extends
BaseJobeetJob { public function asArray($host) { return array(
'category' => $this->getJobeetCategory()->getName(), 'type' =>
$this->getType(), 'company' => $this->getCompany(), 'logo' =>
:math:`$this->getLogo() ? 'http://'.$`host.'/uploads/jobs/'.$this->getLogo()
: null, 'url' => $this->getUrl(), 'position' =>
$this->getPosition(), 'location' => $this->getLocation(),
'description' => $this->getDescription(), 'how\_to\_apply' =>
$this->getHowToApply(), 'expires\_at' => $this->getCreatedAt('c'),
'expires\_at' => $this->getCreatedAt(), ); }

::

      // ...
    }

The ``xml`` Format
~~~~~~~~~~~~~~~~~~

Supporting the ``xml`` format is as simple as creating a template:

::

    <?php
    <!-- apps/frontend/modules/api/templates/listSuccess.xml.php -->
    <?xml version="1.0" encoding="utf-8"?>
    <jobs>
    <?php foreach ($jobs as $url => $job): ?>
      <job url="<?php echo $url ?>">
    <?php foreach ($job as $key => $value): ?>
        <<?php echo $key ?>><?php echo $value ?></<?php echo $key ?>>
    <?php endforeach ?>
      </job>
    <?php endforeach ?>
    </jobs>

The ``json`` Format
~~~~~~~~~~~~~~~~~~~

Support the `JSON format <http://json.org/>`_ is similar:

::

    <?php
    <!-- apps/frontend/modules/api/templates/listSuccess.json.php -->
    [
    <?php $nb = count($jobs); $i = 0; foreach ($jobs as $url => $job): ++$i ?>
    {
      "url": "<?php echo $url ?>",
    <?php $nb1 = count($job); $j = 0; foreach ($job as $key => $value): ++$j ?>
      "<?php echo $key ?>": <?php echo json_encode($value).($nb1 == $j ? '' : ',') ?>
    
    <?php endforeach ?>
    }<?php echo $nb == $i ? '' : ',' ?>
    
    <?php endforeach ?>
    ]

The ``yaml`` Format
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For built-in formats, symfony does some configuration in the
background, like changing the content type, or disabling the
layout.

As the YAML format is not in the list of the built-in request
formats, the response content type can be changed and the layout
disabled in the action:

::

    <?php
    class apiActions extends sfActions
    {
      public function executeList(sfWebRequest $request)
      {
        $this->jobs = array();
        foreach ($this->getRoute()->getObjects() as $job)
        {
          $this->jobs[$this->generateUrl('job_show_user', $job, true)] =
           ➥ $job->asArray($request->getHost());
        }
    
        switch ($request->getRequestFormat())
        {
          case 'yaml':
            $this->setLayout(false);
            $this->getResponse()->setContentType('text/yaml');
            break;
        }
      }
    }

In an action, the ``setLayout()`` method changes the default
layout or disables it when set to
``false``.

The template for YAML reads as follows:

::

    <?php
    <!-- apps/frontend/modules/api/templates/listSuccess.yaml.php -->
    <?php foreach ($jobs as $url => $job): ?>
    -
      url: <?php echo $url ?>
    
    <?php foreach ($job as $key => $value): ?>
      <?php echo $key ?>: <?php echo sfYaml::dump($value) ?>
    
    <?php endforeach ?>
    <?php endforeach ?>

If you try to call the web service with a non-valid token, you will
have a 404 XML page for the XML format, and a 404 JSON page for the
JSON format. But for the YAML format, symfony does not know what to
render.

Whenever you create a format, a ~custom error template\|Custom
Error Templates~ must be created. The template will be used for 404
pages, and all other exceptions.

As the exception should be different in the
production and development environment, two files are needed
(``config/error/exception.yaml.php`` for debugging, and
``config/error/error.yaml.php`` for production):

::

    <?php
    // config/error/exception.yaml.php
    <?php echo sfYaml::dump(array(
      'error'       => array(
        'code'      => $code,
        'message'   => $message,
        'debug'     => array(
          'name'    => $name,
          'message' => $message,
          'traces'  => $traces,
        ),
    )), 4) ?>
    
    // config/error/error.yaml.php
    <?php echo sfYaml::dump(array(
      'error'       => array(
        'code'      => $code,
        'message'   => $message,
    ))) ?>

Before trying it, you must create a layout for YAML format:

::

    <?php
    // apps/frontend/templates/layout.yaml.php
    <?php echo $sf_content ?>

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/16/404.png
   :alt: 404
   
   404

    **TIP** Overriding the 404 error and ~exception\|Exception
    Handling~ templates for built-in templates is as simple as creating
    a file in the ``config/error/`` directory.


Web Service Tests
-------------------------------------------

To test the web service, copy the affiliate fixtures from
``data/fixtures/`` to the ``test/fixtures/`` directory and replace
the content of the auto-generated ``apiActionsTest.php`` file with
the following content:

::

    <?php
    // test/functional/frontend/apiActionsTest.php
    include(dirname(__FILE__).'/../../bootstrap/functional.php');
    
    $browser = new JobeetTestFunctional(new sfBrowser());
    $browser->loadData();
    
    $browser->
      info('1 - Web service security')->
    
      info('  1.1 - A token is needed to access the service')->
      get('/api/foo/jobs.xml')->
      with('response')->isStatusCode(404)->
    
      info('  1.2 - An inactive account cannot access the web service')->
      get('/api/symfony/jobs.xml')->
      with('response')->isStatusCode(404)->
    
      info('2 - The jobs returned are limited to the categories configured for the affiliate')->
      get('/api/sensio_labs/jobs.xml')->
      with('request')->isFormat('xml')->
      with('response')->begin()->
        isValid()->
        checkElement('job', 32)->
      end()->
    
      info('3 - The web service supports the JSON format')->
      get('/api/sensio_labs/jobs.json')->
      with('request')->isFormat('json')->
      with('response')->matches('/"category"\: "Programming"/')->
    
      info('4 - The web service supports the YAML format')->
      get('/api/sensio_labs/jobs.yaml')->
      with('response')->begin()->
        isHeader('content-type', 'text/yaml; charset=utf-8')->
        matches('/category\: Programming/')->
      end()
    ;

In this test, you will notice three new methods:


-  ``isValid()``: Checks whether or not the XML response is well
   formed
-  ``isFormat()``: It tests the format of a request
-  ``matches()``: For non-HTML format, if checks that the response
   verifies the regex passed as an argument

    **TIP** The ``isValid()`` method accepts a boolean as first
    parameter that allows to validates the XML response against its
    XSD.

    $browser->with('response')->isValid(true);

    It also accepts the path to a special XSD file against to which the
    response has to be validated.

    $browser->with('response')->isValid('/path/to/schema/xsd');


The Affiliate Application Form
------------------------------

Now that the web service is ready to be used, let's create the
account creation form for affiliates. We will yet again describe
the classic process of adding a new feature to an application.

Routing
~~~~~~~

You guess it. The route is the first thing we
create:

::

    [yml]
    # apps/frontend/config/routing.yml
    affiliate:
      class:   sfPropelRouteCollection
      options:
        model: JobeetAffiliate
        actions: [new, create]
        object_actions: { wait: get }

It is a classic ##ORM## collection route with a new configuration
option: ``actions``. As we don't need all the seven default actions
defined by the route, the ``actions`` option instructs the route to
only match for the ``new`` and ``create`` actions. The additional
``wait`` route will be used to give the soon-to-be affiliate some
feedback about his account.

Bootstrapping
~~~~~~~~~~~~~

The classic second step is to generate a module:

::

    $ php symfony propel:generate-module frontend affiliate JobeetAffiliate --non-verbose-templates

Templates
~~~~~~~~~

The ``propel:generate-module`` task generate the classic seven
actions and their corresponding templates. In
the ``templates/`` directory, remove all the files but the
``_form.php`` and ``newSuccess.php`` ones. And for the files we
keep, replace their content with the following:

::

    <?php
    <!-- apps/frontend/modules/affiliate/templates/newSuccess.php -->
    <?php use_stylesheet('job.css') ?>
    
    <h1>Become an Affiliate</h1>
    
    <?php include_partial('form', array('form' => $form)) ?>
    
    <!-- apps/frontend/modules/affiliate/templates/_form.php -->
    <?php include_stylesheets_for_form($form) ?>
    <?php include_javascripts_for_form($form) ?>
    
    <?php echo form_tag_for($form, 'affiliate') ?>
      <table id="job_form">
        <tfoot>
          <tr>
            <td colspan="2">
              <input type="submit" value="Submit" />
            </td>
          </tr>
        </tfoot>
        <tbody>
          <?php echo $form ?>
        </tbody>
      </table>
    </form>

Create the ``waitSuccess.php`` template:

::

    <?php
    <!-- apps/frontend/modules/affiliate/templates/waitSuccess.php -->
    <h1>Your affiliate account has been created</h1>
    
    <div style="padding: 20px">
      Thank you!
      You will receive an email with your affiliate token
      as soon as your account will be activated.
    </div>

Last, change the link in the footer to point to the ``affiliate``
module:

::

    <?php
    // apps/frontend/templates/layout.php
    <li class="last">
      <a href="<?php echo url_for('affiliate_new') ?>">Become an affiliate</a>
    </li>

Actions
~~~~~~~

Here again, as we will only use the creation form, open the
``actions.class.php`` file and remove all methods but
``executeNew()``, ``executeCreate()``, and ``processForm()``.

For the ``processForm()`` action, change the redirect URL to the
``wait`` action:

::

    <?php
    // apps/frontend/modules/affiliate/actions/actions.class.php
    $this->redirect($this->generateUrl('affiliate_wait', $jobeet_affiliate));

The ``wait`` action is simple as we don't need to pass anything to
the template:

::

    <?php
    // apps/frontend/modules/affiliate/actions/actions.class.php
    public function executeWait(sfWebRequest $request)
    {
    }

The affiliate cannot choose its token, nor can he activates his
account right away. Open the ``JobeetAffiliateForm`` file to
customize the form:

::

    <?php

// lib/form/JobeetAffiliateForm.class.php //
lib/form/doctrine/JobeetAffiliateForm.class.php class
JobeetAffiliateForm extends BaseJobeetAffiliateForm { public
function configure() { $this->useFields(array( 'url', 'email',
'jobeet\_categories\_list' ));
$this->widgetSchema['jobeet\_category\_affiliate\_list']->setOption('expanded',
true);
$this->widgetSchema['jobeet\_category\_affiliate\_list']->setLabel('Categories');

::

        $this->validatorSchema['jobeet_category_affiliate_list']->setOption('required', true);

$this->widgetSchema['jobeet\_categories\_list']->setOption('expanded',
true);
$this->widgetSchema['jobeet\_categories\_list']->setLabel('Categories');

::

        $this->validatorSchema['jobeet_categories_list']->setOption('required', true);

$this->widgetSchema['url']->setLabel('Your website URL');
$this->widgetSchema['url']->setAttribute('size', 50);

::

        $this->widgetSchema['email']->setAttribute('size', 50);
    
        $this->validatorSchema['email'] = new sfValidatorEmail(array('required' => true));
      }
    }

The new ``sfForm::useFields()`` method allows to specify the white
list of fields to keep. All non mentionned fields will be removed
from the form.

The form framework supports ~many-to-many relationship\|Many to
Many Relationships (Forms)~ like any other column. By default, such
a relation is rendered as a drop-down box thanks to the
``sfWidgetFormPropelChoice`` widget. As seen during day 10, we have
changed the rendered tag by using the ``expanded`` option.

As emails and URLs tend to be quite longer than the default size of
an input tag, default HTML attributes can be set by using the
``setAttribute()`` method.

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/16/affiliate_form.png
   :alt: Affiliate form
   
   Affiliate form

Tests
~~~~~

The last step is to write some functional tests
for the new feature.

Replace the generated tests for the ``affiliate`` module by the
following code:

::

    <?php
    // test/functional/frontend/affiliateActionsTest.php
    include(dirname(__FILE__).'/../../bootstrap/functional.php');
    
    $browser = new JobeetTestFunctional(new sfBrowser());
    $browser->loadData();
    
    $browser->
      info('1 - An affiliate can create an account')->
    
      get('/affiliate/new')->
      click('Submit', array('jobeet_affiliate' => array(
        'url'                            => 'http://www.example.com/',
        'email'                          => 'foo@example.com',

'jobeet\_category\_affiliate\_list' =>
array($browser->getProgrammingCategory()->getId()),
'jobeet\_categories\_list' =>
array(Doctrine\_Core::getTable('JobeetCategory')->findOneBySlug('programming')->getId()),
)))-> with('response')->isRedirected()-> followRedirect()->
with('response')->checkElement('#content h1', 'Your affiliate
account has been created')->

::

      info('2 - An affiliate must at least select one category')->
    
      get('/affiliate/new')->
      click('Submit', array('jobeet_affiliate' => array(
        'url'   => 'http://www.example.com/',
        'email' => 'foo@example.com',
      )))->

with('form')->isError('jobeet\_category\_affiliate\_list')
with('form')->isError('jobeet\_categories\_list') ;

To simulate selecting checkboxes, pass an array of identifiers to
check. To simplify the task, a new ``getProgrammingCategory()``
method has been created in the ``JobeetTestFunctional`` class:

::

    <?php
    // lib/test/JobeetTestFunctional.class.php
    class JobeetTestFunctional extends sfTestFunctional
    {
      public function getProgrammingCategory()
      {
        $criteria = new Criteria();
        $criteria->add(JobeetCategoryPeer::SLUG, 'programming');
    
        return JobeetCategoryPeer::doSelectOne($criteria);
      }
    
      // ...
    }

But as we already have this code in the
``getMostRecentProgrammingJob()`` method, it is time to
refactor the code and create a
``getForSlug()`` method in ``JobeetCategoryPeer``:

::

    <?php
    // lib/model/JobeetCategoryPeer.php
    static public function getForSlug($slug)
    {
      $criteria = new Criteria();
      $criteria->add(self::SLUG, $slug);
    
      return self::doSelectOne($criteria);
    }

Then, replace the two occurrences of this code in
``JobeetTestFunctional``.

The Affiliate Backend
---------------------

For the backend, an ``affiliate`` module must
be created for affiliates to be activated by the administrator:

::

    $ php symfony propel:generate-admin backend JobeetAffiliate --module=affiliate

To access the newly created module, add a link in the main menu
with the number of affiliate that need to be activated:

::

    <?php
    <!-- apps/backend/templates/layout.php -->
    <li>

Affiliates -

.. raw:: html

   <?php echo JobeetAffiliatePeer::countToBeActivated() ?>
   
Affiliates -

.. raw:: html

   <?php echo Doctrine_Core::getTable('JobeetAffiliate')->
   
countToBeActivated() ?>

.. raw:: html

   </li>
   
// lib/model/JobeetAffiliatePeer.php class JobeetAffiliatePeer
extends BaseJobeetAffiliatePeer { static public function
countToBeActivated() { $criteria = new Criteria();
$criteria->add(self::IS\_ACTIVE, 0);

::

        return self::doCount($criteria);
      }

// lib/model/doctrine/JobeetAffiliateTable.class.php class
JobeetAffiliateTable extends Doctrine\_Table { public function
countToBeActivated() { $q = $this->createQuery('a')
->where('a.is\_active = ?', 0);

::

        return $q->count();
      }

// ...

::

    }

As the only action needed in the backend is to activate or
deactivate accounts, change the default generator ``config``
section to simplify the interface a bit and add a link to activate
accounts directly from the list view:

::

    [yml]
    # apps/backend/modules/affiliate/config/generator.yml
    config:
      fields:
        is_active: { label: Active? }
      list:
        title:   Affiliate Management
        display: [is_active, url, email, token]
        sort:    [is_active]
        object_actions:
          activate:   ~
          deactivate: ~
        batch_actions:
          activate:   ~
          deactivate: ~
        actions: {}
      filter:
        display: [url, email, is_active]

To make administrators more productive, change the default filters
to only show affiliates to be activated:

::

    <?php
    // apps/backend/modules/affiliate/lib/affiliateGeneratorConfiguration.class.php
    class affiliateGeneratorConfiguration extends BaseAffiliateGeneratorConfiguration
    {
      public function getFilterDefaults()
      {
        return array('is_active' => '0');
      }
    }

The only other code to write is for the ``activate``,
``deactivate`` actions:

::

    <?php
    // apps/backend/modules/affiliate/actions/actions.class.php
    class affiliateActions extends autoAffiliateActions
    {
      public function executeListActivate()
      {
        $this->getRoute()->getObject()->activate();
    
        $this->redirect('jobeet_affiliate');
      }
    
      public function executeListDeactivate()
      {
        $this->getRoute()->getObject()->deactivate();
    
        $this->redirect('jobeet_affiliate');
      }
    
      public function executeBatchActivate(sfWebRequest $request)
      {

:math:`$affiliates = JobeetAffiliatePeer::retrieveByPks($`request->getParameter('ids'));
$q = Doctrine\_Query::create() ->from('JobeetAffiliate a')
->whereIn('a.id', $request->getParameter('ids'));

::

        $affiliates = $q->execute();

foreach ($affiliates as $affiliate) { $affiliate->activate(); }

::

        $this->redirect('jobeet_affiliate');
      }
    
      public function executeBatchDeactivate(sfWebRequest $request)
      {

:math:`$affiliates = JobeetAffiliatePeer::retrieveByPks($`request->getParameter('ids'));
$q = Doctrine\_Query::create() ->from('JobeetAffiliate a')
->whereIn('a.id', $request->getParameter('ids'));

::

        $affiliates = $q->execute();

foreach ($affiliates as $affiliate) { $affiliate->deactivate(); }

::

        $this->redirect('jobeet_affiliate');
      }
    }

// lib/model/JobeetAffiliate.php //
lib/model/doctrine/JobeetAffiliate.class.php class JobeetAffiliate
extends BaseJobeetAffiliate { public function activate() {
$this->setIsActive(true);

::

        return $this->save();
      }
    
      public function deactivate()
      {
        $this->setIsActive(false);
    
        return $this->save();
      }
    
      // ...
    }

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/16/backend.png
   :alt: Affiliate backend
   
   Affiliate backend

Final Thoughts
--------------

Thanks to the REST architecture of symfony, it is quite
easy to implement web services for your projects. Although, we
wrote code for a read-only web service today, you have enough
symfony knowledge to implement a read-write web service.

The implementation of the affiliate account creation form in the
frontend and its backend counterpart was really easy as you are now
familiar with the process of adding new features to your project.

If you remember requirements from day 2:

"The affiliate can also limit the number of jobs to be returned,
and refine his query by specifying a category."

The implementation of this feature is so easy that we will let you
do it tonight.

Whenever an affiliate account is activated by the administrator, an
email should be sent to the affiliate to confirm his subscription
and give him his token. Sending emails is the topic we will talk
about tomorrow.

**ORM**


