---
layout:     post
title:      "Ray Notes"
subtitle:   "ray source code notes - execute"
date:       2020-02-11
author:     "Huang Yu'an"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - framework
    - ray
    - distrubute computing
---

# Ray notes - Execute

[toc]

### How `submit_task()` works in CoreWorker?:

```cython
from ray.includes.libcoreworker cimport CCoreWorker

class CoreWorker:
    def submit_task(self,
                    function_descriptor,
                    args,
                    int num_return_vals,
                    c_bool is_direct_call,
                    resources,
                    int max_retries):
        cdef:
            unordered_map[c_string, double] c_resources
            CTaskOptions task_options
            CRayFunction ray_function
            c_vector[CTaskArg] args_vector
            c_vector[CObjectID] return_ids

        with self.profile_event(b"submit_task"):
            prepare_resources(resources, &c_resources) # some resources
            task_options = CTaskOptions(
                num_return_vals, is_direct_call, c_resources)
            ray_function = CRayFunction(
                LANGUAGE_PYTHON, string_vector_from_list(function_descriptor))
            prepare_args(args, &args_vector) # push arguments

            with nogil:
                check_status(self.core_worker.get().SubmitTask(
                    ray_function, args_vector, task_options, &return_ids,
                    max_retries))

            return VectorToObjectIDs(return_ids)
```



python/ray/includes/libcoreworker.pxd:

```cython
cdef extern from "ray/core_worker/core_worker.h" nogil:
    cdef cppclass CCoreWorker "ray::CoreWorker":
        CCoreWorker(const CWorkerType worker_type, const CLanguage language,
                    const c_string &store_socket,
                    const c_string &raylet_socket, const CJobID &job_id,
                    const CGcsClientOptions &gcs_options,
                    const c_string &log_dir, const c_string &node_ip_address,
                    int node_manager_port,
                    CRayStatus (
                        CTaskType task_type,
                        const CRayFunction &ray_function,
                        const unordered_map[c_string, double] &resources,
                        const c_vector[shared_ptr[CRayObject]] &args,
                        const c_vector[CObjectID] &arg_reference_ids,
                        const c_vector[CObjectID] &return_ids,
                        c_vector[shared_ptr[CRayObject]] *returns) nogil,
                    CRayStatus() nogil,
                    c_bool ref_counting_enabled)
        void Disconnect()
        CWorkerType &GetWorkerType()
        CLanguage &GetLanguage()

        void StartExecutingTasks()

        CRayStatus SubmitTask(
            const CRayFunction &function, const c_vector[CTaskArg] &args,
            const CTaskOptions &options, c_vector[CObjectID] *return_ids,
            int max_retries)
```

src/ray/core_worker/core_worker.cc:

```cpp
Status CoreWorker::SubmitTask(const RayFunction &function,
                              const std::vector<TaskArg> &args,
                              const TaskOptions &task_options,
                              std::vector<ObjectID> *return_ids, int max_retries) {
  TaskSpecBuilder builder;
  const int next_task_index = worker_context_.GetNextTaskIndex();
  const auto task_id =
      TaskID::ForNormalTask(worker_context_.GetCurrentJobID(),
                            worker_context_.GetCurrentTaskID(), next_task_index);

  const std::unordered_map<std::string, double> required_resources;
  // TODO(ekl) offload task building onto a thread pool for performance
  BuildCommonTaskSpec(
      builder, worker_context_.GetCurrentJobID(), task_id,
      worker_context_.GetCurrentTaskID(), next_task_index, GetCallerId(), rpc_address_,
      function, args, task_options.num_returns, task_options.resources,
      required_resources,
      task_options.is_direct_call ? TaskTransportType::DIRECT : TaskTransportType::RAYLET,
      return_ids);
  TaskSpecification task_spec = builder.Build();
  if (task_options.is_direct_call) {
    task_manager_->AddPendingTask(GetCallerId(), rpc_address_, task_spec, max_retries);
    return direct_task_submitter_->SubmitTask(task_spec);
  } else {
    return local_raylet_client_->SubmitTask(task_spec);
  }
}
```

#### Some definition:

```cpp
// Language of a task or worker.
enum Language {
  PYTHON = 0;
  JAVA = 1;
  CPP = 2;
}

// Type of a worker.
enum WorkerType {
  WORKER = 0;
  DRIVER = 1;
}

// Type of a task.
enum TaskType {
  // Normal task.
  NORMAL_TASK = 0;
  // Actor creation task.
  ACTOR_CREATION_TASK = 1;
  // Actor task.
  ACTOR_TASK = 2;
}
```

to be continued...



### How `CoreWorker` execute each task really?

a worker run main_loop will receive and execute task.

```python
# in class Worker
    def main_loop(self):
        """The main loop a worker runs to receive and execute tasks."""

        def sigterm_handler(signum, frame):
            shutdown(True)
            sys.exit(1)

        signal.signal(signal.SIGTERM, sigterm_handler)
        self.core_worker.run_task_loop()
        sys.exit(0)
```

Normally we have these processes:

1. **(ray start --head)**

   - process run `python/ray/log_monitor.py`

   - process run `python/ray/reporter.py`

   - raylet process with several child process (worker processes)

   - plasma_store_server process

   - raylet_monitor process

   - two redis-server processes

2. **(ray start redis_addr=xxxxx, join to the head node's cluster)**

3. **(ray.init(ip_addr=xxxxx), connect this cluster)**

4. **(ray.init(), create local worker process)**

   - process run `python/ray/ray_process_reaper.py`::==>`self.start_reaper_process()`
   - process run `python/ray/monitor.py`
   - two redis-server processes::==>`self.start_head_processes()`
   - process run `python/ray/log_monitor.py`
   - process run `python/ray/reporter.py`
   - raylet process with serveral child worker processes
   - plasma_store_server process
   - raylet_monitor process ::==>`self.start_ray_processes()`


```python
# in class ray.node.Node
    def start_head_processes(self):
        """Start head processes on the node."""
        logger.debug(
            "Process STDOUT and STDERR is being redirected to {}.".format(
                self._logs_dir))
        assert self._redis_address is None
        # If this is the head node, start the relevant head node processes.
        self.start_redis()
        self.start_monitor()
        self.start_raylet_monitor()
        # The dashboard is Python3.x only.
        if PY3 and self._ray_params.include_webui:
            self.start_dashboard()

    def start_ray_processes(self):
        """Start all of the processes on the node."""
        logger.debug(
            "Process STDOUT and STDERR is being redirected to {}.".format(
                self._logs_dir))

        self.start_plasma_store()
        self.start_raylet()
        if PY3:
            self.start_reporter()

        if self._ray_params.include_log_monitor:
            self.start_log_monitor()
```



### How it schedule task by considering the dependency?