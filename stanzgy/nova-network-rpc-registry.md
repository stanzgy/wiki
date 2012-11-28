# Openstack network service registry procedure

This page describes how nova-network register itself as a "network" service in rpc queues. 
In summary, all nova-network listen on two rpc message queues "`network`" and "`network.{host}`" to prepare to make a remote response.

For more detailed rpc information, please refer to [[inside-openstack-rpc]].

## how nova-network register itself in message queues

### NetworkManager call super class to register itself as "network" in `__init__()`
> nova / nova / network / manager.py

```python
class NetworkManager(manager.SchedulerDependentManager):
	...    
	def __init__(self, network_driver=None, *args, **kwargs):
		...
		super(NetworkManager, self).__init__(service_name='network',
												*args, **kwargs)
```

### `__init__()` in super class
> nova / nova / manager.py

	note that periodic_task decorator made _publish_service_capabilities() run periodically to notify network service
	
```python
class SchedulerDependentManager(Manager):
    """ Periodically send capability updates to the Scheduler services.

          Services that need to update the Scheduler of their capabilities
          should derive from this class. Otherwise they can derive from
          manager.Manager directly. Updates are only sent after
          update_service_capabilities is called with non-None values.

     """

    def __init__(self, host=None, db_driver=None, service_name='undefined'):
        self.last_capabilities = None
        self.service_name = service_name
        super(SchedulerDependentManager, self).__init__(host, db_driver)

    def update_service_capabilities(self, capabilities):
        """Remember these capabilities to send on next periodic update."""
        self.last_capabilities = capabilities

    @periodic_task
    def _publish_service_capabilities(self, context):
        """Pass data back to the scheduler at a periodic interval."""
        if self.last_capabilities:
            LOG.debug(_('Notifying Schedulers of capabilities ...'))
            api.update_service_capabilities(context, self.service_name,
                                self.host, self.last_capabilities)


def periodic_task(*args, **kwargs):
    """ Decorator to indicate that a method is a periodic task.

		This decorator can be used in two ways:

		1. Without arguments '@periodic_task', this will be run on every tick
		of the periodic scheduler.

		2. With arguments, @periodic_task(ticks_between_runs=N), this will be
		run on every N ticks of the periodic scheduler.
	"""
    def decorator(f):
        f._periodic_task = True
        f._ticks_between_runs = kwargs.pop('ticks_between_runs', 0)
        return f
```
