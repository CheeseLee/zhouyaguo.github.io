---
layout: post
title: "[openstack] 定时任务"
comments: true
categories:
- openstack
---

以compute服务为例
----------------

在compute服务启动过程，有一个步骤是启动定时任务，时间间隔是periodic\_interval，具体做的事情是periodic_tasks

```python
if self.periodic_interval:
	periodic = utils.LoopingCall(self.periodic_tasks)
	periodic.start(interval=self.periodic_interval, now=False)
	self.timers.append(periodic)
```

periodic_interval默认是 `60s`

```python
cfg.IntOpt('periodic_interval',
		default=60,
		help='seconds between running periodic tasks'),
```

periodic_tasks是什么？

```python
def periodic_tasks(self, raise_on_error=False):
	"""Tasks to be run at a periodic interval."""
	ctxt = context.get_admin_context()
	self.manager.periodic_tasks(ctxt, raise_on_error=raise_on_error)
```

以下代码可以看出：tasks是\_periodic\_tasks，同时还加入了 `ticks_to_skip` 的概念

```python
def periodic_tasks(self, context, raise_on_error=False):
	"""Tasks to be run at a periodic interval."""
	for task_name, task in self._periodic_tasks:
		full_task_name = '.'.join([self.__class__.__name__, task_name])

		ticks_to_skip = self._ticks_to_skip[task_name]
		if ticks_to_skip > 0:
			LOG.debug(_("Skipping %(full_task_name)s, %(ticks_to_skip)s"
						" ticks left until next run"), locals())
			self._ticks_to_skip[task_name] -= 1
			continue

		self._ticks_to_skip[task_name] = task._ticks_between_runs
		LOG.debug(_("Running periodic task %(full_task_name)s"), locals())

		try:
			task(self, context)
		except Exception as e:
			if raise_on_error:
				raise
			LOG.exception(_("Error during %(full_task_name)s: %(e)s"),
						locals())
```

那么task和\_ticks\_to\_skip的值来自哪里？答案在下面的代码中，即取自属性\_periodic\_task

```python
class ManagerMeta(type):
    def __init__(cls, names, bases, dict_):
        """Metaclass that allows us to collect decorated periodic tasks."""
        super(ManagerMeta, cls).__init__(names, bases, dict_)

        # NOTE(sirp): if the attribute is not present then we must be the base
        # class, so, go ahead an initialize it. If the attribute is present,
        # then we're a subclass so make a copy of it so we don't step on our
        # parent's toes.
        try:
            cls._periodic_tasks = cls._periodic_tasks[:]
        except AttributeError:
            cls._periodic_tasks = []

        try:
            cls._ticks_to_skip = cls._ticks_to_skip.copy()
        except AttributeError:
            cls._ticks_to_skip = {}

        for value in cls.__dict__.values():
            if getattr(value, '_periodic_task', False):
                task = value
                name = task.__name__
                cls._periodic_tasks.append((name, task))
                cls._ticks_to_skip[name] = task._ticks_between_runs
```

以`_sync_power_states`为例，`@manager.periodic_task(ticks_between_runs=10)` 的作用是为ComputeManager设置属性\_periodic\_task、\_ticks\_between\_runs

```python
@manager.periodic_task(ticks_between_runs=10)
def _sync_power_states(self, context):
```

```python
def periodic_task(*args, **kwargs):
    """Decorator to indicate that a method is a periodic task.

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

至此可以看出\_sync\_power\_states方法的定时执行间隔为 `60s*10=600s`
