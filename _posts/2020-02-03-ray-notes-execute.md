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
    - distribute computing
---

# Ray notes - Execute

[toc]

### Important Data Structures

- CoreWorker
- Plasma_store
- Raylet
- Redis

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

Start a worker:

> main_loop() ==> run_task_loop() ==> `StartExecutingTasks()`

```cpp
class CoreWorker{
boost::asio::io_service task_execution_service_;
std::shared_ptr<gcs::GcsClient> gcs_client_;
std::shared_ptr<ActorManager> actor_manager_;
std::shared_ptr<worker::Profiler> profiler_;

std::shared_ptr<raylet::RayletClient> local_raylet_client_;
std::shared_ptr<TaskManager> task_manager_;
std::unique_ptr<CoreWorkerRayletTaskReceiver> raylet_task_receiver_;
std::unique_ptr<CoreWorkerDirectTaskReceiver> direct_task_receiver_;
};
```

```cpp
void CoreWorker::StartExecutingTasks() { 
	task_execution_service_.run(); 
}
```

The very important method:

```cpp
void CoreWorkerRayletTaskReceiver::HandleAssignTask(
    const rpc::AssignTaskRequest &request, rpc::AssignTaskReply *reply,
    rpc::SendReplyCallback send_reply_callback)
```

It is this function really run each remote function (task).



But Who  invoke this function?

```cpp
 task_execution_service_.post([=] {
      raylet_task_receiver_->HandleAssignTask(request, reply, send_reply_callback);
    });
```

By doing this, task_execution_service_ will asynchronously invoke this task.







### !!!How it schedule task by considering the dependency?

```cpp
 resolver_.ResolveDependencies(task_spec, [this, task_spec]() {
    RAY_LOG(DEBUG) << "Task dependencies resolved " << task_spec.TaskId();
    absl::MutexLock lock(&mu_);
    // Note that the dependencies in the task spec are mutated to only contain
    // plasma dependencies after ResolveDependencies finishes.
    const SchedulingKey scheduling_key(
        task_spec.GetSchedulingClass(), task_spec.GetDependencies(),
        task_spec.IsActorCreationTask() ? task_spec.ActorCreationId() : ActorID::Nil());
    auto it = task_queues_.find(scheduling_key);
    if (it == task_queues_.end()) {
      it = task_queues_.emplace(scheduling_key, std::deque<TaskSpecification>()).first;
    }
    it->second.push_back(task_spec);
    RequestNewWorkerIfNeeded(scheduling_key);
  });
```

Using `LocalDependencyResolver` object method `void ResolveDependencies(TaskSpecification &task,                       std::function<void()> on_complete);` to resolve the dependency. After resolve the dependency, it will **`really submit`** this task.

In the method **`ResolveDependencies`**:

it ....

