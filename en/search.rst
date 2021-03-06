NOT CONVERTED TO SYMFONY2 YET
=============================

Want to help out?
Fork it on `Github <http://github.com/symfonytuts/jobeet-docs>`_

Day 17: Search
==============

In day 14, we added some feeds to keep Jobeet users up-to-date with
new job posts. Today will help you to improve the user experience
by implementing the latest main feature of the Jobeet website: the
search engine.

The Technology
--------------

Before we jump in head first, let's talk a bit about the history of
symfony. We advocate a lot of best practices,
like tests and refactoring, and we also try to apply them to the
framework itself. For instance, we like the famous "Don't reinvent
the wheel" motto.

As a matter of fact, the symfony framework started its life four
years ago as the glue between two existing Open-Source softwares:
Mojavi and Propel. And every time we need to tackle a new problem,
we look for an existing library that does the job well before
coding one ourself from scratch.

Now, we want to add a search engine to Jobeet, and the Zend
Framework provides a great library, called
`Zend Lucene <http://framework.zend.com/manual/en/zend.search.lucene.html>`_,
which is a port of the well-know Java Lucene project. Instead of
creating yet another search engine for Jobeet, which is quite a
complex task, we will use Zend Lucene.

On the Zend Lucene documentation page, the library is described as
follows:

    ... a general purpose text search engine written entirely in PHP 5.
    Since it stores its index on the filesystem and does not require a
    database server, it can add search capabilities to almost any
    PHP-driven website. Zend\_Search\_Lucene supports the following
    features:

    
    -  Ranked searching - best results returned first
    -  Many powerful query types: phrase queries, boolean queries,
       wildcard queries, proximity queries, range queries and many others
    -  Search by specific field (e.g., title, author, contents)


-

    **NOTE** Today is not a tutorial about the Zend Lucene library, but
    how to integrate it into the Jobeet website; or more generally, how
    to integrate third-party libraries into a
    symfony project. If you want more information about this
    technology, please refer to the
    `Zend Lucene documentation <http://framework.zend.com/manual/en/zend.search.lucene.html>`_.


Installing and Configuring the Zend Framework
---------------------------------------------

The Zend Lucene library is part of the
Zend Framework. We will only install the Zend Framework into the
``lib/vendor/`` directory, alongside the symfony framework itself.

First, download the
`Zend Framework <http://framework.zend.com/download/overview>`_ and
un-archive the files so that you have a ``lib/vendor/Zend/``
directory.

    **NOTE** The following explanations have been tested with the
    1.10.3 version of the Zend Framework.


-

    **TIP** You can clean up the directory by removing everything but
    the following files and directories:

    
    -  ``Exception.php``
    -  ``Loader/``
    -  ``Autoloader.php``
    -  ``Search/``


Then, add the following code to the ``ProjectConfiguration`` class
to provide a simple way to register the Zend autoloader:

::

    <?php
    // config/ProjectConfiguration.class.php
    class ProjectConfiguration extends sfProjectConfiguration
    {
      static protected $zendLoaded = false;
    
      static public function registerZend()
      {
        if (self::$zendLoaded)
        {
          return;
        }
    
        set_include_path(sfConfig::get('sf_lib_dir').'/vendor'.PATH_SEPARATOR.get_include_path());
        require_once sfConfig::get('sf_lib_dir').'/vendor/Zend/Loader/Autoloader.php';
        Zend_Loader_Autoloader::getInstance();
        self::$zendLoaded = true;
      }
    
      // ...
    }

Indexing
--------

The Jobeet search engine should be able to return all jobs matching
keywords entered by the user. Before being able to search anything,
an index has to be built for the jobs; for
Jobeet, it will be stored in the ``data/`` directory.

Zend Lucene provides two methods to retrieve an index depending
whether one already exists or not. Let's create a helper method in
the ``JobeetJobPeer`` class that returns an existing index or
creates a new one for us: Zend Lucene provides two methods to
retrieve an index depending whether one already exists or not.
Let's create a helper method in the ``JobeetJobTable`` class that
returns an existing index or creates a new one for us:

::

    <?php

// lib/model/JobeetJobPeer.php //
lib/model/doctrine/JobeetJobTable.class.php static public function
getLuceneIndex() { ProjectConfiguration::registerZend();

::

      if (file_exists($index = self::getLuceneIndexFile()))
      {
        return Zend_Search_Lucene::open($index);
      }
    
      return Zend_Search_Lucene::create($index);
    }
    
    static public function getLuceneIndexFile()
    {
      return sfConfig::get('sf_data_dir').'/job.'.sfConfig::get('sf_environment').'.index';
    }

The ``save()`` method
~~~~~~~~~~~~~~~~~~~~~

Each time a job is created, updated, or deleted, the index must be
updated. Edit ``JobeetJob`` to update the index whenever a job is
serialized to the database:


.. raw:: html

   <?php
       // lib/model/JobeetJob.php
       public function save(PropelPDO $con = null)
       {
         // ...
   
         $ret = parent::save($con);
   
         $this->
   
updateLuceneIndex();

::

      return $ret;
    }


.. raw:: html

   <?php
       public function save(Doctrine_Connection $conn = null)
       {
         // ...
   
         $ret = parent::save($conn);
   
         $this->
   
updateLuceneIndex();

::

      return $ret;
    }

And create the ``updateLuceneIndex()`` method that does the actual
work:

::

    <?php

// lib/model/JobeetJob.php //
lib/model/doctrine/JobeetJob.class.php public function
updateLuceneIndex() { $index = JobeetJobPeer::getLuceneIndex();
$index = JobeetJobTable::getLuceneIndex();

::

      // remove existing entries
      foreach ($index->find('pk:'.$this->getId()) as $hit)
      {
        $index->delete($hit->id);
      }
    
      // don't index expired and non-activated jobs
      if ($this->isExpired() || !$this->getIsActivated())
      {
        return;
      }
    
      $doc = new Zend_Search_Lucene_Document();
    
      // store job primary key to identify it in the search results
      $doc->addField(Zend_Search_Lucene_Field::Keyword('pk', $this->getId()));
    
      // index job fields
      $doc->addField(Zend_Search_Lucene_Field::UnStored('position', $this->getPosition(), 'utf-8'));
      $doc->addField(Zend_Search_Lucene_Field::UnStored('company', $this->getCompany(), 'utf-8'));
      $doc->addField(Zend_Search_Lucene_Field::UnStored('location', $this->getLocation(), 'utf-8'));
      $doc->addField(Zend_Search_Lucene_Field::UnStored('description', $this->getDescription(), 'utf-8'));
    
      // add job to the index
      $index->addDocument($doc);
      $index->commit();
    }

As Zend Lucene is not able to update an existing entry, it is
removed first if the job already exists in the index.

Indexing the job itself is simple: the primary key is stored for
future reference when searching jobs and the main columns
(``position``, ``company``, ``location``, and ``description``) are
indexed but not stored in the index as we will use the real objects
to display the results.

##ORM## Transactions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

What if there is a problem when indexing a job or if the job is not
saved into the database? Both ##ORM## and Zend Lucene will throw an
exception. But under some circumstances, we might have a job saved
in the database without the corresponding indexing. To prevent this
from happening, we can wrap the two updates in a transaction and
rollback in case of an error:


.. raw:: html

   <?php
       // lib/model/JobeetJob.php
       public function save(PropelPDO $con = null)
       {
         // ...
   
         if (is_null($con))
         {
           $con = Propel::getConnection(JobeetJobPeer::DATABASE_NAME, Propel::CONNECTION_WRITE);
         }
   
         $con->
   
beginTransaction(); try { :math:`$ret = parent::save($`con);

::

        $this->updateLuceneIndex();
    
        $con->commit();
    
        return $ret;
      }
      catch (Exception $e)
      {
        $con->rollBack();
        throw $e;
      }
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJob.class.php
       public function save(Doctrine_Connection $conn = null)
       {
         // ...
   
         $conn = $conn ? $conn : $this->
   
getTable()->getConnection(); $conn->beginTransaction(); try {
:math:`$ret = parent::save($`conn);

::

        $this->updateLuceneIndex();
    
        $conn->commit();
    
        return $ret;
      }
      catch (Exception $e)
      {
        $conn->rollBack();
        throw $e;
      }
    }

``delete()``
~~~~~~~~~~~~

We also need to override the ``delete()`` method to remove the
entry of the deleted job from the index:


.. raw:: html

   <?php
       // lib/model/JobeetJob.php
       public function delete(PropelPDO $con = null)
       {
         $index = JobeetJobPeer::getLuceneIndex();
   
         foreach ($index->
   
find('pk:'.$this->getId()) as $hit) {
:math:`$index->delete($`hit->id); }

::

      return parent::delete($con);
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJob.class.php
       public function delete(Doctrine_Connection $conn = null)
       {
         $index = JobeetJobTable::getLuceneIndex();
   
         foreach ($index->
   
find('pk:'.$this->getId()) as $hit) {
:math:`$index->delete($`hit->id); }

::

      return parent::delete($conn);
    }

### Mass delete

Whenever you load the fixtures with the
``propel:data-load`` task, symfony removes all the existing job
records by calling the ``JobeetJobPeer::doDeleteAll()`` method.
Let's override the default behavior to also delete the index
altogether:

::

    <?php
    // lib/model/JobeetJobPeer.php
    public static function doDeleteAll($con = null)
    {
      if (file_exists($index = self::getLuceneIndexFile()))
      {
        sfToolkit::clearDirectory($index);
        rmdir($index);
      }
    
      return parent::doDeleteAll($con);
    }

Searching
---------

Now that we have everything in place, you can reload the fixture
data to index them:

::

    $ php symfony propel:data-load

    **TIP** For Unix-like users: as the index is modified from the
    command line and also from the web, you must change the index
    directory permissions accordingly depending on your configuration:
    check that both the command line user you use and the web server
    user can write to the index directory.


-

    **NOTE** You might have some warnings about the ``ZipArchive``
    class if you don't have the ``zip`` extension compiled in your PHP.
    It's a known bug of the ``Zend_Loader`` class.


Implementing the search in the frontend is a piece of cake. First,
create a route:

::

    [yml]
    job_search:
      url:   /search
      param: { module: job, action: search }

And the corresponding action:

::

    <?php
    // apps/frontend/modules/job/actions/actions.class.php
    class jobActions extends sfActions
    {
      public function executeSearch(sfWebRequest $request)
      {
        $this->forwardUnless($query = $request->getParameter('query'), 'job', 'index');

:math:`$this->jobs = JobeetJobPeer::getForLuceneQuery($`query);
:math:`$this->jobs = Doctrine_Core::getTable('JobeetJob') ➥ ->getForLuceneQuery($`query);
}

::

      // ...
    }

    **NOTE** The new ``forwardUnless()`` method forwards the user to
    the ``index`` action of the ``job`` module if the ``query`` request
    parameter does not exist or is empty.

    It's just an alias for the following longer statement:

    if (!$query = $request->getParameter('query')) {
    $this->forward('job', 'index'); }


The template is also quite straightforward:

::

    <?php
    // apps/frontend/modules/job/templates/searchSuccess.php
    <?php use_stylesheet('jobs.css') ?>
    
    <div id="jobs">
      <?php include_partial('job/list', array('jobs' => $jobs)) ?>
    </div>

The search itself is delegated to the ``getForLuceneQuery()``
method:


.. raw:: html

   <?php
       // lib/model/JobeetJobPeer.php
       static public function getForLuceneQuery($query)
       {
         $hits = self::getLuceneIndex()->
   
find($query);

::

      $pks = array();
      foreach ($hits as $hit)
      {
        $pks[] = $hit->pk;
      }
    
      $criteria = new Criteria();
      $criteria->add(self::ID, $pks, Criteria::IN);
      $criteria->setLimit(20);
    
      return self::doSelect(self::addActiveJobsCriteria($criteria));
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJobTable.class.php
       public function getForLuceneQuery($query)
       {
         $hits = self::getLuceneIndex()->
   
find($query);

::

      $pks = array();
      foreach ($hits as $hit)
      {
        $pks[] = $hit->pk;
      }
    
      if (empty($pks))
      {
        return array();
      }
    
      $q = $this->createQuery('j')
        ->whereIn('j.id', $pks)
        ->limit(20);
    
      $q = $this->addActiveJobsQuery($q);
    
      return $q->execute();
    }

After we get all results from the Lucene index, we filter out the
inactive jobs, and limit the number of results to ``20``.

To make it work, update the layout:

::

    <?php
    // apps/frontend/templates/layout.php
    <h2>Ask for a job</h2>
    <form action="<?php echo url_for('job_search') ?>" method="get">
      <input type="text" name="query" value="<?php echo $sf_request->getParameter('query') ?>" id="search_keywords" />
      <input type="submit" value="search" />
      <div class="help">
        Enter some keywords (city, country, position, ...)
      </div>
    </form>

    **NOTE** Zend Lucene defines a rich query language that supports
    operations like Booleans, wildcards, fuzzy search, and much more.
    Everything is documented in the
    `Zend Lucene manual <http://framework.zend.com/manual/en/zend.search.lucene.query-api.html>`_


Unit Tests
--------------------------

What kind of unit tests do we need to create to test the search
engine? We obviously won't test the Zend Lucene library itself, but
its integration with the ``JobeetJob`` class.

Add the following tests at the end of the ``JobeetJobTest.php``
file and don't forget to update the number of tests at the
beginning of the file to ``7``:

::

    <?php
    // test/unit/model/JobeetJobTest.php
    $t->comment('->getForLuceneQuery()');
    $job = create_job(array('position' => 'foobar', 'is_activated' => false));
    $job->save();

$jobs = JobeetJobPeer::getForLuceneQuery('position:foobar'); $jobs
=
Doctrine\_Core::getTable('JobeetJob')->getForLuceneQuery('position:foobar');
:math:`$t->is(count($`jobs), 0, '::getForLuceneQuery() does not
return non activated jobs');

::

    $job = create_job(array('position' => 'foobar', 'is_activated' => true));
    $job->save();

$jobs = JobeetJobPeer::getForLuceneQuery('position:foobar'); $jobs
=
Doctrine\_Core::getTable('JobeetJob')->getForLuceneQuery('position:foobar');
:math:`$t->is(count($`jobs), 1, '::getForLuceneQuery() returns jobs
matching the criteria'); :math:`$t->is($`jobs[0]->getId(),
$job->getId(), '::getForLuceneQuery() returns jobs matching the
criteria');

::

    $job->delete();

$jobs = JobeetJobPeer::getForLuceneQuery('position:foobar'); $jobs
=
Doctrine\_Core::getTable('JobeetJob')->getForLuceneQuery('position:foobar');
:math:`$t->is(count($`jobs), 0, '::getForLuceneQuery() does not
return deleted jobs');

We test that a non activated job, or a deleted one does not show up
in the search results; we also check that jobs matching the given
criteria do show up in the results.

Tasks
----------------

Eventually, we need to create a task to cleanup the index from
stale entries (when a job expires for example) and optimize the
index from time to time. As we already have a cleanup task, let's
update it to add those features:

::

    <?php
    // lib/task/JobeetCleanupTask.class.php
    protected function execute($arguments = array(), $options = array())
    {
      $databaseManager = new sfDatabaseManager($this->configuration);

// cleanup Lucene index $index = JobeetJobPeer::getLuceneIndex();

::

      $criteria = new Criteria();
      $criteria->add(JobeetJobPeer::EXPIRES_AT, time(), Criteria::LESS_THAN);
      $jobs = JobeetJobPeer::doSelect($criteria);

// cleanup Lucene index $index = JobeetJobTable::getLuceneIndex();

::

      $q = Doctrine_Query::create()
        ->from('JobeetJob j')
        ->where('j.expires_at < ?', date('Y-m-d'));
    
      $jobs = $q->execute();

foreach ($jobs as :math:`$job) { if ($`hit =
:math:`$index->find('pk:'.$`job->getId())) {
:math:`$index->delete($`hit->id); } }

::

      $index->optimize();
    
      $this->logSection('lucene', 'Cleaned up and optimized the job index');
    
      // Remove stale jobs

:math:`$nb = JobeetJobPeer::cleanup($`options['days']);

::

      $this->logSection('propel', sprintf('Removed %d stale jobs', $nb));

:math:`$nb = Doctrine_Core::getTable('JobeetJob')->cleanup($`options['days']);

::

      $this->logSection('doctrine', sprintf('Removed %d stale jobs', $nb));

}

The task removes all expired jobs from the index and then optimizes
it thanks to the Zend Lucene built-in ``optimize()`` method.

Final Thoughts
--------------

Along this day, we implemented a full search engine with many
features in less than an hour. Every time you want to add a new
feature to your projects, check that it has not yet been solved
somewhere else.

First, check if something is not implemented natively in the
`symfony framework <http://www.symfony-project.org/api/1_4/>`_.
Then, check the
`symfony plugins <http://www.symfony-project.org/plugins/>`_. And
don't forget to check the
`Zend Framework libraries <http://framework.zend.com/manual/en/>`_
and the `ezComponent <http://ezcomponents.org/docs>`_ ones too.

Tomorrow we will use some unobtrusive JavaScripts to enhance the
responsiveness of the search engine by updating the results in
real-time as the user types in the search box. Of course, this will
be the occasion to talk about how to use AJAX with symfony.

**ORM**


