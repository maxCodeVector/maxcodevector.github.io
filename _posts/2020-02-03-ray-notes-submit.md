---
layout:     post
title:      "Ray Notes"
subtitle:   "ray source code notes - submit"
date:       2020-02-03
author:     "Huang Yu'an"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - framework
    - ray
    - distrubute computing
---

# Ray notes - Submit

[toc]

### What `ray.init()` does?

if `driver_mode == LOCAL_MODE`

```python
    # in python/ray/worker.py
    
    _global_node = ray.node.LocalNode()
```

else if `redis_address is None`, start a new cluster, (**default behavior**)

```python
	# Start the Ray processes. We set shutdown_at_exit=False because we
    # shutdown the node in the ray.shutdown call that happens in the atexit
    # handler. We still spawn a reaper process in case the atexit handler
    # isn't called.
    _global_node = ray.node.Node(
        head=True,
        shutdown_at_exit=False,
        spawn_reaper=True,
        ray_params=ray_params)
```

else connecting an existing cluster.

```python
  _global_node = ray.node.Node(
            ray_params,
            head=False,
            shutdown_at_exit=False,
            spawn_reaper=False,
            connect_only=True)
```

then call connect() to connect this node, call some hook if any

```python
    connect(
        _global_node,
        mode=driver_mode,
        log_to_driver=log_to_driver,
        worker=global_worker,
        driver_object_store_memory=driver_object_store_memory,
        job_id=job_id,
        internal_config=json.loads(_internal_config)
        if _internal_config else {})
```



When creating Node, pass local_mode to RayParams

>local_mode (bool): True if the code should be executed serially
>without Ray. This is useful for debugging.

class: ray.node.Node(): create a node, may be start some new process.

```python
	# python/ray/node.py

	def __init__(self,
                 ray_params,
                 head=False,
                 shutdown_at_exit=True,
                 spawn_reaper=True,
                 connect_only=False):
        """Start a node.

        Args:
            ray_params (ray.params.RayParams): The parameters to use to
                configure the node.
            head (bool): True if this is the head node, which means it will
                start additional processes like the Redis servers, monitor
                processes, and web UI.
            shutdown_at_exit (bool): If true, spawned processes will be cleaned
                up if this process exits normally.
            spawn_reaper (bool): If true, spawns a process that will clean up
                other spawned processes if this process dies unexpectedly.
            connect_only (bool): If true, connect to the node without starting
                new processes.
        """
```

### How to connect a Node (local & remote)?

- Some basis check, eg. init twice?

- create redis client if it is not Local Model

  ```python
  worker.redis_client = node.create_redis_client()
  ```

- if it is Worker Model, do not specify the job id

- if it is Local Mode, random a job id

- if it is Driver Mode

  ```python
          # This is the code path of driver mode.
          if job_id is None:
              # TODO(qwang): use `GcsClient::GenerateJobId()` here.
              job_id = JobID.from_int(
                  int(worker.redis_client.incr("JobCounter")))
          # When tasks are executed on remote workers in the context of multiple
          # drivers, the current job ID is used to keep track of which job is
          # responsible for the task so that error messages will be propagated to
          # the correct driver.
          worker.worker_id = ray.utils.compute_driver_id_from_job(
              job_id).binary()
  ```

-  Create an object for interfacing with the global state.

  ```
  ray.state.state._initialize_global_state(
      node.redis_address, redis_password=node.redis_password)
  ```

- initial `GlableControlStateOptions` and `CoreWorker`

  ```python
      gcs_options = ray._raylet.GcsClientOptions(
              redis_address,
              int(redis_port),
              node.redis_password,
      )
      worker.core_worker = ray._raylet.CoreWorker(
          (mode == SCRIPT_MODE),
          node.plasma_store_socket_name,
          node.raylet_socket_name,
          job_id,
          gcs_options,
          node.get_logs_dir_path(),
          node.node_ip_address,
          node.node_manager_port,
      )
  ```

- initial raylet client:

  ```
  worker.raylet_client = ray._raylet.RayletClient(worker.core_worker)
  ```

- start some diamond threads (log, print messages, listener etc)



### How remote decorator works?

**remote(*args, \*\*kwargs)**

when simply using @ray.remote, use

```python
return make_decorator(worker=worker)(args[0])
```

if there are some arguments, use

```python
    return make_decorator(
        num_return_vals=num_return_vals,
        num_cpus=num_cpus,
        num_gpus=num_gpus,
        memory=memory,
        object_store_memory=object_store_memory,
        resources=resources,
        max_calls=max_calls,
        max_reconstructions=max_reconstructions,
        max_retries=max_retries,
        worker=worker)
```

noted the later case return a function: make_decorator, while the former return the result of invoke this function.

**make_decorator()**

Wen making remote class, using the argument worker to create actor: 

```python
return worker.make_actor(function_or_class, num_cpus, num_gpus,
                         memory, object_store_memory, resources,
                         max_reconstructions)
```

when making remote function, create a `RemoteFunction` object

```python
return ray.remote_function.RemoteFunction(
    function_or_class, num_cpus, num_gpus, memory,
    object_store_memory, resources, num_return_vals, max_calls,
    max_retries)
```

**worker.make_actor()** ==> python/ray/actor.py

```python
    return ActorClass._ray_from_modified_class(
        Class, ActorClassID.from_random(), max_reconstructions, num_cpus,
        num_gpus, memory, object_store_memory, resources)
```

return an ActorClass object



#### Call the remote function:

```python
	# in ray.remote()
...
       def invocation(args, kwargs):
            if not args and not kwargs and not self._function_signature:
                list_args = []
            else:
                list_args = ray.signature.flatten_args(
                    self._function_signature, args, kwargs)

            if worker.mode == ray.worker.LOCAL_MODE:
                object_ids = worker.local_mode_manager.execute(
                    self._function, self._function_descriptor, args, kwargs,
                    num_return_vals)
            else:
                object_ids = worker.core_worker.submit_task(
                    self._function_descriptor_list, list_args, num_return_vals,
                    is_direct_call, resources, max_retries)

            if len(object_ids) == 1:
                return object_ids[0]
            elif len(object_ids) > 1:
                return object_ids

        if self._decorator is not None:
            invocation = self._decorator(invocation)

        return invocation(args, kwargs)
```

The self._function_descriptor_list: 

- module_name,
- class_name
- function_name
- function_source_hash

is_direct_call: it is **false by default** which is controlled by environment variable: `RAY_FORCE_DIRECT`

#### Instance the remote class (Actor)

```python
        # in ActorClass::_remote()
    	...
    	actor_id = worker.core_worker.create_actor(
                        function_descriptor.get_function_descriptor_list(),
                        creation_args, meta.max_reconstructions, resources,
                        actor_placement_resources, is_direct_call, max_concurrency,
                        detached, is_asyncio)

        actor_handle = ActorHandle(
        actor_id,
        meta.modified_class.__module__,
        meta.class_name,
        meta.actor_method_names,
        meta.method_decorators,
        meta.method_signatures,
        meta.actor_method_num_return_vals,
        actor_method_cpu,
        worker.current_session_and_job,
        original_handle=True)

        if name is not None:
            ray.experimental.register_actor(name, actor_handle)

        return actor_handle
```

return an ActorHandle object

#### Call the remote class (actor) function

```python
  return actor._actor_method_call(
                self._method_name,
                args=args,
                kwargs=kwargs,
                num_return_vals=num_return_vals)
```



### How create_actor() works in `CoreWorker`?

```cython
  def create_actor(self,
                     function_descriptor,
                     args,
                     uint64_t max_reconstructions,
                     resources,
                     placement_resources,
                     c_bool is_direct_call,
                     int32_t max_concurrency,
                     c_bool is_detached,
                     c_bool is_asyncio):
        cdef:
            CRayFunction ray_function
            c_vector[CTaskArg] args_vector
            c_vector[c_string] dynamic_worker_options
            unordered_map[c_string, double] c_resources
            unordered_map[c_string, double] c_placement_resources
            CActorID c_actor_id

        with self.profile_event(b"submit_task"):
            prepare_resources(resources, &c_resources)
            prepare_resources(placement_resources, &c_placement_resources)
            ray_function = CRayFunction(
                LANGUAGE_PYTHON, string_vector_from_list(function_descriptor))
            prepare_args(args, &args_vector)

            with nogil:
                check_status(self.core_worker.get().CreateActor(
                    ray_function, args_vector,
                    CActorCreationOptions(
                        max_reconstructions, is_direct_call, max_concurrency,
                        c_resources, c_placement_resources,
                        dynamic_worker_options, is_detached, is_asyncio),
                    &c_actor_id))

            return ActorID(c_actor_id.Binary())
```

It will invoke method `CoreWorker::CreateActor` to create the actor and generate relative actor id.



### Other useful reference

https://medium.com/distributed-computing-with-ray/how-ray-uses-grpc-and-arrow-to-outperform-grpc-43ec368cb385

At [Anyscale](http://anyscale.io/), we’re working on a number of enhancements for Ray 1.0, including:

- Support for tasks/actors written in C++ and Rust (in addition to Python and Java today).
- Distributed reference counting for shared memory objects.
- Accelerated point-to-point GPU transfers.