# lww-element-set
An Python implementation of lww-set (Last-Writer-Wins Element Set). lww-set is an Operation-based [Conflict-Free Replicated Date Type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)(CRDT). It can be used to achieve strong eventual consistency and monotonicy (i.e., no rollbacks).

### Operations
The lww-set provides the following operations:
- add(element, timestamp): Adds an element associated with its  timestamp in the set.
- remove(element, timestamp): Removes an element associated with its  timestamp in the set.
- exist(element): returns True if the element exists, False otherwise. The criteria is whether the element's most recent operation was an add.  
- get(): returns a list of existing elements in lww-set

lww-set consists of two separate underlying sets: add_set and remove_set. Each set records entries of (element, timestamp). When inserting an element to add_set or remove_set, create a new entry if the element does not exist. Otherwise, update the corresponding entry's timestamp if input is more recent (i.e., the timestamp argument is larger).

In order to satisfy the CRDT properties, i.e., Associativity, Commutativity and Idempotence, lww-set defines the following combinations of operations and their resulting states.

| Original state | Operation   | Resulting state |
|----------------|-------------|-----------------|
| A(a,1) R()     | add(a,0)    | A(a,1) R()      |
| A(a,1) R()     | add(a,1)    | A(a,1) R()      |
| A(a,1) R()     | add(a,2)    | A(a,2) R()      |
| A() R(a,1)     | add(a,0)    | A(a,0) R(a,1)   |
| A() R(a,1)     | add(a,1)    | A(a,1) R(a,1)   |
| A() R(a,1)     | add(a,2)    | A(a,2) R(a,1)   |
| A() R(a,1)     | remove(a,0) | A() R(a,1)      |
| A() R(a,1)     | remove(a,1) | A() R(a,1)      |
| A() R(a,1)     | remove(a,2) | A() R(a,2)      |
| A(a,1) R()     | remove(a,0) | A(a,1) R(a,0)   |
| A(a,1) R()     | remove(a,1) | A(a,1) R(a,1)   |
| A(a,1) R()     | remove(a,2) | A(a,1) R(a,2)   |

Note: The above table is different from the one in [Roshi](https://github.com/soundcloud/roshi) which does instant garbage collection when inserting a new element. The above table makes sure lww-set meets strictly the **Commutativity** property, i.e., a+b = b+a, the order of applying operations does not matter. However, because there is no garbage collection, there can be redundant copies of elements in both add_set and remove_sets, which wastes space.

### Semantics of lww-set add/remove operations
Unlike any other usual set data structure, lww-set's add/remove operation semantic is slightly different. When you call add() or remove(), what actually happens internally under the hood is record the add/remove operations in the underlying sets. The below two cases are possible

- A remove() operation of (element, timestamp) is recorded in remove_set before the corresponding element exists in add_set.
- A previously existing element has been deleted in lww-set, i.e., the element in remove_set has a higher timestamp than in add_set. However, a client can still issue the remove() on that element multiple times and the timestamp of that element in remove_set will be updated multiple times.

These above scanarios are allowed in lww-set because we position lww-set as a layer where operations come in asynchronously. Because of the CRDT properties, the order of the operation arrival time does not matter, so all replicated lww-set will eventually converge.

#### Return Values of add/remove()

However, in the API design, a natural question arises: What value should lww-set remove operation return? We are facing the dilemma of using usual set semantic vs. lww-set state changes.

For example, when add set and remove set are both empty, i.e., A() R(), we perform Remove(a,1) operation on the lww-set. From an external view, we are deleting a non-existing element in lww-set, so it should return False in this semantic. But what actually happens is remove set will record an R operation and the internal result is changed to: A() R(a,1). So it indeed **changes** the state of lww-set and it should return True.

If the return value is based on the usual set add/remove semantic, then it may confuses users because some other processes calling the same method concurrently may have overwritten the element (with a more recent timestamp).

If the return value is based on whether lww-set internal state has been changed, then we probably should we enumerate all return values for all possible scenarios(the table above). However, exposing the internal states of lww-set violates the encapsulation.

The solution here is **add/remove methods always return None**. The caller of these methods does not know whether the operation succeeds or not based on the return value. The way to get a timely update is via calling get() or exist() method.

#### Eventuality
For add() or remove(), if the timestamp argument is the most recent (compared with
  all other elements), the operation                                                                                                                                         will eventually succeed. Otherwise, when other processes or                                                                                                                                             threads are invoking add() or remove() operations concurrently                                                                                                                                       , the timestamp of an element may be                                                                                                                                                    overwritten by another newer timestamp.  Therefore, each client won't know whether its
   add() or remove() operations succeeds because it does not have the
   ''big picture'' of whether there is more recent operation.  The return value is                                                                                                                                            always none and does not indicate the success of this                                                                                                                                                   operation. The only way to check is by calling exist() or get().

However, when calling exist() or get(), if no other processes/threads is                                                                                                                                               calling add() or remove() concurrently, it is guaranteed that                                                                                                                                             exist() or get() always returns the most recent result. But when there are concurrent operations, there is no such
guarantees and it is possible to get an
out-dated result.
After all on-the-fly operations complete,
calling exist() or get() will eventually return the up-to-date result.

## Synchronization considerations

### Locks in lww-set add() and remove()
The two underlying sets are implemented in Python dictionaries. Since Python's GIL guarantees that only one thread runs at a time, individual dictionary operations, for example, D[x] = y, are [atomic and thread-safe](http://effbot.org/pyfaq/what-kinds-of-global-value-mutation-are-thread-safe.htm).

However, there is no atomic test & set operations in Python dictionary. For lww-set add/remove operations which is to insert operations to add_sets/remove_sets, we need to protect the critical region (like the one below) from data race.

```
# Element test & set region
# race condition could happen in below snippet without locking
if element in target_set:                                                                                                                                                                               
    current_timestamp = target_set[element]                                                                                                                                                             
    if current_timestamp < timestamp:                                                                                                                                                                   
        target_set[element] = timestamp                                                                                                                                                                 
else:                                                                                                                                                                                                   
    target_set[element] = timestamp              
```

### No lock in exist() or get()
For exist() or get() implementations, they only need to read values from the Python dictionaries, i.e.,add_set and remove_set. Because each python dictionary operations is atomic (dicussed above), there is no need to protect them with a lock.

## References
1.  [Conflict-Free Replicated Date Type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) Wikipedia page.
2. [Roshi](https://github.com/soundcloud/roshi), an LWW-element-set CRDT implemented in Go.
