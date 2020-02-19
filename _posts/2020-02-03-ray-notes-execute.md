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







### ! How it schedule task by considering the dependency?

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



Log when worker running a task submit by python driver

```
ffffffff20200214-080712.4076 
Log file created at: 2020/02/14 08:07:12
Running on machine: eagle-sane
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0214 08:07:12.942349  4076 core_worker.cc:100] Initializing worker 0200ffffffffffffffffffffffffffffffffffff
I0214 08:07:12.943464  4076 redis_gcs_client.cc:156] RedisGcsClient Connected.
I0214 08:07:12.943727  4076 grpc_server.cc:44] driver server started, listening on port 39267.
W0214 08:07:13.006904  4076 reference_count.cc:43] Tried to decrease ref count for nonexistent object ID: 6384c61fe1808d6e840ffe2e3e1f000000000000
I0214 08:07:13.008214  4076 task_manager.cc:16] Adding pending task ec32723a47dc4020ffffffff0200
I0214 08:07:13.008227  4076 direct_task_transport.cc:9] Submit task ec32723a47dc4020ffffffff0200
I0214 08:07:13.008232  4076 direct_task_transport.cc:11] Task dependencies resolved ec32723a47dc4020ffffffff0200
I0214 08:07:13.009481  4114 direct_task_transport.cc:119] Lease granted ec32723a47dc4020ffffffff0200
I0214 08:07:13.009619  4114 direct_task_transport.cc:34] Connected to 172.28.176.156:43411
I0214 08:07:13.009642  4114 direct_task_transport.cc:169] Pushing normal task ec32723a47dc4020ffffffff0200
I0214 08:07:13.012215  4114 task_manager.cc:63] Completing task ec32723a47dc4020ffffffff0200
I0214 08:07:13.012658  4076 core_worker.cc:458] Plasma GET timeout -1
I0214 08:07:13.013561  4076 task_manager.cc:16] Adding pending task 71f1719a7c120256ffffffff0200
I0214 08:07:13.013576  4076 direct_task_transport.cc:9] Submit task 71f1719a7c120256ffffffff0200
I0214 08:07:13.013581  4076 direct_task_transport.cc:11] Task dependencies resolved 71f1719a7c120256ffffffff0200
I0214 08:07:13.014401  4114 direct_task_transport.cc:119] Lease granted 71f1719a7c120256ffffffff0200
I0214 08:07:13.014441  4114 direct_task_transport.cc:169] Pushing normal task 71f1719a7c120256ffffffff0200
I0214 08:07:13.015753  4114 task_manager.cc:63] Completing task 71f1719a7c120256ffffffff0200
I0214 08:07:13.015815  4076 core_worker.cc:458] Plasma GET timeout -1
I0214 08:07:13.501974  4114 core_worker.cc:322] Sending 0 object IDs to raylet.
I0214 08:07:13.517566  4076 logging.cc:174] Uninstall signal handlers.
```

A complete log when running a task with id: `8ba9f5eac6758849ffffffff0400`.

```
(base) hya@eagle-sane:/tmp/ray/session_latest/logs$ cat * | grep 8ba9f5eac6758849ffffffff0400

I0218 13:36:26.833839 10574 direct_actor_transport.cc:218] Received task Type=NORMAL_TASK, Language=PYTHON, function_descriptor=__main__,,sqrt, task_id=8ba9f5eac6758849ffffffff0400, job_id=0400, num_args=2, num_returns=1
    
I0218 13:36:26.827762 10552 node_manager.cc:1573] Worker lease request 8ba9f5eac6758849ffffffff0400
I0218 13:36:26.827847 10552 node_manager.cc:1936] Submitting task: task_spec={Type=NORMAL_TASK, Language=PYTHON, function_descriptor=__main__,,sqrt, task_id=8ba9f5eac6758849ffffffff0400, job_id=0400, num_args=2, num_returns=1}, task_execution_spec={num_forwards=0}
I0218 13:36:26.827889 10552 scheduling_queue.cc:356] Added task 8ba9f5eac6758849ffffffff0400 to placeable queue
I0218 13:36:26.827987 10552 scheduling_queue.cc:224] Removed task 8ba9f5eac6758849ffffffff0400 from placeable queue
I0218 13:36:26.828006 10552 scheduling_queue.cc:356] Added task 8ba9f5eac6758849ffffffff0400 to ready queue
I0218 13:36:26.828032 10552 node_manager.cc:2292] Assigning task 8ba9f5eac6758849ffffffff0400 to worker with pid 10574, worker id: 512b80c644db0a5f2e60fd76c3a33c60641090e6
I0218 13:36:26.828058 10552 node_manager.cc:1579] Worker lease request DISPATCH 8ba9f5eac6758849ffffffff0400
I0218 13:36:26.828191 10552 node_manager.cc:2320] Finished assigning task 8ba9f5eac6758849ffffffff0400 to worker 512b80c644db0a5f2e60fd76c3a33c60641090e6
I0218 13:36:26.828214 10552 node_manager.cc:2947] FinishAssignTask: 8ba9f5eac6758849ffffffff0400
I0218 13:36:26.828234 10552 scheduling_queue.cc:224] Removed task 8ba9f5eac6758849ffffffff0400 from ready queue
I0218 13:36:26.828256 10552 scheduling_queue.cc:356] Added task 8ba9f5eac6758849ffffffff0400 to running queue
I0218 13:36:26.828274 10552 task_dependency_manager.cc:226] Task 8ba9f5eac6758849ffffffff0400 no longer blocked
   [worker time] 
I0218 13:36:26.837070 10552 node_manager.cc:2346] Finished task 8ba9f5eac6758849ffffffff0400
I0218 13:36:26.837095 10552 scheduling_queue.cc:224] Removed task 8ba9f5eac6758849ffffffff0400 from running queue
I0218 13:36:26.837154 10552 task_dependency_manager.cc:409] Task execution 8ba9f5eac6758849ffffffff0400 canceled
```

