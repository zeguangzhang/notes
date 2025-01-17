Python 线程策略
```
import threading
import time

def worker():
    print("Thread execution starts")
    time.sleep(5)  # This simulates a time-consuming task
    print("Thread execution ends")

# create a thread by specifying the target function
t = threading.Thread(target=worker)

# start the thread
t.start()

print("Main thread execution ends")
```

```
# -*- coding: utf-8 -*-

from time import sleep, perf_counter
import random
from concurrent.futures import ThreadPoolExecutor, as_completed


def task(tid, r):

    print(f'task {tid} started, sleeping {r} secs')
    sleep(r)

    return f'finished task {tid}, slept {r}'

start = perf_counter()

with ThreadPoolExecutor() as executor:

    futures = []

    futures.append(executor.submit(task, 1, 4))
    futures.append(executor.submit(task, 2, 4))
    futures.append(executor.submit(task, 3, 30))


    for res in as_completed(futures):
        print(res.result())

finish = perf_counter()

print(f"It took {finish-start} second(s) to finish.")
```