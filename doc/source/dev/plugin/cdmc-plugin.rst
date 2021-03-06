..
      Except where otherwise noted, this document is licensed under Creative
      Commons Attribution 3.0 License.  You can view the license at:

          https://creativecommons.org/licenses/by/3.0/

.. _implement_cluster_data_model_collector_plugin:

========================================
Build a new cluster data model collector
========================================

Watcher Decision Engine has an external cluster data model plugin interface
which gives anyone the ability to integrate an external cluster data model
collector in order to extend the initial set of cluster data model collectors
Watcher provides.

This section gives some guidelines on how to implement and integrate custom
cluster data model collectors within Watcher.


Creating a new plugin
=====================

First of all, you have to extend the :class:`~.BaseClusterDataModelCollector`
base class which defines the :py:meth:`~.BaseClusterDataModelCollector.execute`
abstract method you will have to implement. This method is responsible for
building an entire cluster data model.

Here is an example showing how you can write a plugin called
``DummyClusterDataModelCollector``:

.. code-block:: python

    # Filepath = <PROJECT_DIR>/thirdparty/dummy.py
    # Import path = thirdparty.dummy

    from watcher.decision_engine.model import model_root
    from watcher.decision_engine.model.collector import base


    class DummyClusterDataModelCollector(base.BaseClusterDataModelCollector):

        def execute(self):
            model = model_root.ModelRoot()
            # Do something here...
            return model

This implementation is the most basic one. So in order to get a better
understanding on how to implement a more advanced cluster data model collector,
have a look at the :py:class:`~.NovaClusterDataModelCollector` class.

Define configuration parameters
===============================

At this point, you have a fully functional cluster data model collector.
By default, cluster data model collectors define an ``period`` option (see
:py:meth:`~.BaseClusterDataModelCollector.get_config_opts`) that corresponds
to the interval of time between each synchronization of the in-memory model.

However, in more complex implementation, you may want to define some
configuration options so one can tune the cluster data model collector to its
needs. To do so, you can implement the :py:meth:`~.Loadable.get_config_opts`
class method as followed:

.. code-block:: python

    from oslo_config import cfg
    from watcher.decision_engine.model import model_root
    from watcher.decision_engine.model.collector import base


    class DummyClusterDataModelCollector(base.BaseClusterDataModelCollector):

        def execute(self):
            model = model_root.ModelRoot()
            # Do something here...
            return model

        @classmethod
        def get_config_opts(cls):
            return super(
                DummyClusterDataModelCollector, cls).get_config_opts() + [
                cfg.StrOpt('test_opt', help="Demo Option.", default=0),
                # Some more options ...
            ]

The configuration options defined within this class method will be included
within the global ``watcher.conf`` configuration file under a section named by
convention: ``{namespace}.{plugin_name}``. In our case, the ``watcher.conf``
configuration would have to be modified as followed:

.. code-block:: ini

    [watcher_cluster_data_model_collectors.dummy]
    # Option used for testing.
    test_opt = test_value

Then, the configuration options you define within this method will then be
injected in each instantiated object via the  ``config`` parameter of the
:py:meth:`~.BaseClusterDataModelCollector.__init__` method.


Abstract Plugin Class
=====================

Here below is the abstract ``BaseClusterDataModelCollector`` class that every
single cluster data model collector should implement:

.. autoclass:: watcher.decision_engine.model.collector.base.BaseClusterDataModelCollector
    :members:
    :special-members: __init__
    :noindex:


Register a new entry point
==========================

In order for the Watcher Applier to load your new cluster data model collector,
the cluster data model collector must be registered as a named entry point
under the ``decision_engine.model.collector`` entry point of your ``setup.py``
file. If you are using pbr_, this entry point should be placed in your
``setup.cfg`` file.

The name you give to your entry point has to be unique.

Here below is how to register ``DummyClusterDataModelCollector`` using pbr_:

.. code-block:: ini

    [entry_points]
    watcher_cluster_data_model_collectors =
        dummy = thirdparty.dummy:DummyClusterDataModelCollector

.. _pbr: http://docs.openstack.org/developer/pbr/


Using cluster data model collector plugins
==========================================

The Watcher Decision Engine service will automatically discover any installed
plugins when it is restarted. If a Python package containing a custom plugin is
installed within the same environment as Watcher, Watcher will automatically
make that plugin available for use.

At this point, you can use your new cluster data model plugin in your
:ref:`strategy plugin <implement_strategy_plugin>` by using the
:py:attr:`~.BaseStrategy.collector_manager` property as followed:

.. code-block:: python

    # [...]
    dummy_collector = self.collector_manager.get_cluster_model_collector(
        "dummy")  # "dummy" is the name of the entry point we declared earlier
    dummy_model = collector.get_latest_cluster_data_model()
    # Do some stuff with this model
