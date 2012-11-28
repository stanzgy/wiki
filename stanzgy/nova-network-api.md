# How nova-network api works

This page briefly indicates how nova-network api call rpc functions, for more detailed rpc information please refer to [[nova-network-rpc-registry]] and [[inside-openstack-rpc]].

## Simple RPC structure of nova-network:


		  Publisher                                                                                               Consumer
	+--------------------+                                                                                       +---------+
	|                    |                                                                                       |         |
	|        ....        |                                   +-----------+                                       |   ...   |
	|                    |       register topic "network"    |           |    call topic "network" to respond    |         |
	|   NetworkManager   |     <-------------------------->  |           |  <--------------------------------->  |   API   |
	|   NetworkManager   |                                   | RPC queue |                                       |   API   |
	|   NetworkManager   |       register "network.host"     |           |    call "network.host" to respond     |   API   |
	|                    |     <-------------------------->  |           |  <--------------------------------->  |         |
	|        ....        |                                   +-----------+                                       |   ...   |
	|                    |                                                                                       |         |
	+--------------------+                                                                                       +---------+

## How API call RPC in code 

### messages with routing key "topic" in code
> nova / nova / network / api.py

```python
class API(base.Base):

	...
    def get_all(self, context):
        return rpc.call(context,
                        FLAGS.network_topic,	# FLAGS.network_topic = "network"
                        {'method': 'get_all_networks'})
	...

    def release_floating_ip(self, context, address,
                            affect_auto_assigned=False):
        """Removes floating ip with address from a project. (deallocates)"""
        rpc.cast(context,
                 FLAGS.network_topic,			# FLAGS.network_topic = "network"
                 {'method': 'deallocate_floating_ip',
                  'args': {'address': address,
                           'affect_auto_assigned': affect_auto_assigned}})
    ...
```


### messages with routing key "topic.host" in code
> nova / nova / network / manager.py

```python
class RPCAllocateFixedIP(object):

    def _allocate_fixed_ips(self, context, instance_id, host, networks, **kwargs):
	
		...
		topic = self.db.queue_get_for(context,
									  FLAGS.network_topic,
									  host)
		args = {}
		args['instance_id'] = instance_id
		args['network_id'] = network['id']
		args['address'] = address
		args['vpn'] = vpn

		green_pool.spawn_n(rpc.call, context, topic,
						   {'method': '_rpc_allocate_fixed_ip',
							'args': args})
		...
```
