# Async in Python (including Celery)


## Question: [How to spawn a celery task from previous celery task?](https://stackoverflow.com/q/45353574/404556)

I maybe using celery incorrectly. But the chatbot I am developing requires celery with redis for async tasks. This is the framework I am using: http://microsoftbotframework.readthedocs.io/en/latest/asynctasks/.

My particular use case currently requires me to run a celery task **forever** and wait for some arbitrary amount of time in between, which ranges from 30 minutes to  3 days. Something like this 

    @celery.task
    def myAsyncMethod():
        while true:
            timeToWait = getTimeToNextAlarm()
            sleep(timeToWait)
            sendOutMessages()

Basically, I have an async process which never exits. I am pretty sure shouldn't be using celery like this. **So my question is, how do I create a celery task which processes the first task, spawns a task and submits it to celery queue and exits.** Basically, something like this:

    @celery.task
    def myImprovedTask():
        timeToWait = getTimeToNextAlarm()
        sleep(timeToWait)
        sendOutMessages()
        myImprovedTask().delay()    # recursive call to async method for next event

Not necessarily recursive or even like this, but something which is the way celery originally is intended to be used (for short lived tasks I believe?)

Tl;dr: How do I create a celery task from within another task and make the original task exit? 


## Answer

If you want to run another task from your initial task, just call it as you would usually do with [`Task.delay()`](http://docs.celeryproject.org/en/latest/reference/celery.app.task.html#celery.app.task.Task.delay), or [`Task.apply_async()`](http://docs.celeryproject.org/en/latest/reference/celery.app.task.html#celery.app.task.Task.apply_async):

    @celery.task
    def myImprovedTask():
        timeToWait = getTimeToNextAlarm()
        sleep(timeToWait)
        sendOutMessages()
        myImprovedTask.delay()

It doesn't matter if you call the same task again. It gets enqueued with `delay()`, your original task returns, and then the next task in queue gets run.

---

All this is under an assumption *you are actually calling your Celery tasks asynchronously*. Sometimes that's not a case, a common culprit being the [`task-always-eager`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#task-always-eager) config option. By default it's disabled, **but** (from the [docs](http://docs.celeryproject.org/en/latest/userguide/configuration.html#task-always-eager)):

> If **`task_always_eager`** is `True`, **all tasks will be executed locally by blocking until the task returns**. `apply_async()` and `Task.delay()` will return an `EagerResult` instance, that emulates the API and behavior of `AsyncResult`, except the result is already evaluated.

> That is, tasks will be executed locally **instead of being sent to the queue.**

So, be sure your Celery config includes:

    task_always_eager = False

