---
layout:     post
title:      "Ray Notes"
subtitle:   "ray source code notes - storage"
date:       2020-02-11
author:     "Huang Yu'an"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - framework
    - ray
    - distrubute computing
---

# Ray notes - Storage

[toc]

### How to get value according to `ray.get(objectID)`?

```python
    worker = global_worker
    values = worker.get_objects(object_ids, timeout=timeout)
```

in Worker::get_objects()

```python
     data_metadata_pairs = self.core_worker.get_objects(
            object_ids, self.current_task_id, timeout_ms)
     return self.deserialize_objects(data_metadata_pairs, object_ids)
```

### How to put value according, ray.put()?

```python
object_id = worker.put_object(value)

if not weakref and not worker.mode == LOCAL_MODE:
    object_id.set_buffer_ref(
    	worker.core_worker.get_objects([object_id],
    									worker.current_task_id))
return object_id

```

in Worker::put_object(),

```python
   def put_object(self, value, object_id=None):
       ......
        serialized_value = self.get_serialization_context().serialize(value)
        return self.core_worker.put_serialized_object(
            serialized_value, object_id=object_id)
```

### Where are value/reference of objects stored?

1. open a space to store the new data and test if this object_id data has been created, if yet, do nothing and report it.

```cython
 cdef:
            CObjectID c_object_id
            shared_ptr[CBuffer] data
            shared_ptr[CBuffer] metadata
        metadata = string_to_buffer(serialized_object.metadata)
        total_bytes = serialized_object.total_bytes
        object_already_exists = self._create_put_buffer(
            metadata, total_bytes, object_id, &c_object_id, &data)
```

Using the Plasma In-Memory Object Store from C++.

```c++
/* in src/ray/core_worker/store_provider/plasma_store_provider.cc:55 */    

Status CoreWorkerPlasmaStoreProvider::Create(const std::shared_ptr<Buffer> &metadata,
                                             const size_t data_size,
                                             const ObjectID &object_id,
                                             std::shared_ptr<Buffer> *data) {
    ...
    arrow::Status status =
        store_client_.Create(plasma_id, data_size, metadata ? metadata->Data() : nullptr,
                             metadata ? metadata->Size() : 0, &arrow_buffer);
    ...
}
```

2. If it is new object, after allocate a new place, write the serialized data to it, then call Seal() to make it immutable.

```cython
if not object_already_exists:
            write_serialized_object(serialized_object, data)
            with nogil:
                check_status(
                    self.core_worker.get().Seal(c_object_id))
        return ObjectID(c_object_id.Binary())

```

3. Finally return the object id.

### Where Global Control Store (GCS) stored in? One machine (head) or multiple?

### How Object Table, Task Table, Function Table implement in GCS?