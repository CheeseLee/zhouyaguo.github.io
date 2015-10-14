---
layout: post
title: "[openstack] sync_power_states"
comments: true
categories:
- openstack
---

如果直接调用`xm`命令对虚拟机操作，有可能导致真实状态和`instances`表中的`power_state`不一致。

ComputeManager中启动了一个定时任务来解决该问题。当间隔时间到达时（默认间隔是60s*10=600s），执行步骤如下：

1. 从instances表中取出所有vm信息，从libvirt api中取出vm个数，二者进行对比。
1. 针对每一个nova记录的instance，如果发现在xen中已不存在，继续判断是否是正在MIGRATING，如果确实已不存在，则设置vm\_power\_state的状态为`NOSTATE`
1. 将最新的vm\_power\_state更新到db中。
1. 修改db中vm\_state
	- 如果最新的vm\_power\_state为SHUTOFF、SHUTDOWN、CRASHED，并且db中记录的是ACTIVE时，则把db中的vm\_state修改为`SHUTOFF`
	- 如果最新的vm\_power\_state为BLOCKED、RUNNING，则把db中的vm_state修改为`ACTIVE`


```python
@manager.periodic_task(ticks_between_runs=10)
def _sync_power_states(self, context):
	"""Align power states between the database and the hypervisor.

	The hypervisor is authoritative for the power_state data, but we don't
	want to do an expensive call to the virt driver's list_instances_detail
	method. Instead, we do a less-expensive call to get the number of
	virtual machines known by the hypervisor and if the number matches the
	number of virtual machines known by the database, we proceed in a lazy
	loop, one database record at a time, checking if the hypervisor has the
	same power state as is in the database. We call eventlet.sleep(0) after
	each loop to allow the periodic task eventlet to do other work.

	If the instance is not found on the hypervisor, but is in the database,
	then it will be set to power_state.NOSTATE.
	"""
	db_instances = self.db.instance_get_all_by_host(context, self.host)

	num_vm_instances = self.driver.get_num_instances()
	num_db_instances = len(db_instances)

	if num_vm_instances != num_db_instances:
		LOG.warn(_("Found %(num_db_instances)s in the database and "
				"%(num_vm_instances)s on the hypervisor.") % locals())

	for db_instance in db_instances:
		# Allow other periodic tasks to do some work...
		greenthread.sleep(0)
		db_power_state = db_instance['power_state']
		try:
			vm_instance = self.driver.get_info(db_instance)
			vm_power_state = vm_instance['state']
		except exception.InstanceNotFound:
			# This exception might have been caused by a race condition
			# between _sync_power_states and live migrations. Two cases
			# are possible as documented below. To this aim, refresh the
			# DB instance state.
			try:
				u = self.db.instance_get_by_uuid(context,
												db_instance['uuid'])
				if self.host != u['host']:
					# on the sending end of nova-compute _sync_power_state
					# may have yielded to the greenthread performing a live
					# migration; this in turn has changed the resident-host
					# for the VM; However, the instance is still active, it
					# is just in the process of migrating to another host.
					# This implies that the compute source must relinquish
					# control to the compute destination.
					LOG.info(_("During the sync_power process the "
							"instance %(uuid)s has moved from "
							"host %(src)s to host %(dst)s") %
							{'uuid': db_instance['uuid'],
								'src': self.host,
								'dst': u['host']})
				elif (u['host'] == self.host and
					u['vm_state'] == vm_states.MIGRATING):
					# on the receiving end of nova-compute, it could happen
					# that the DB instance already report the new resident
					# but the actual VM has not showed up on the hypervisor
					# yet. In this case, let's allow the loop to continue
					# and run the state sync in a later round
					LOG.info(_("Instance %s is in the process of "
							"migrating to this host. Wait next "
							"sync_power cycle before setting "
							"power state to NOSTATE")
							% db_instance['uuid'])
				else:
					LOG.warn(_("Instance found in database but not "
							"known by hypervisor. Setting power "
							"state to NOSTATE"), locals(),
							instance=db_instance)
					vm_power_state = power_state.NOSTATE
			except exception.InstanceNotFound:
				# no need to update vm_state for deleted instances
				continue

		if vm_power_state == db_power_state:
			continue
# primton:
# poweroff
#            if (vm_power_state in (power_state.NOSTATE,
#                                   power_state.SHUTOFF,
#                                   power_state.SHUTDOWN,
#                                   power_state.CRASHED)
		if (vm_power_state in (power_state.SHUTOFF,
							power_state.SHUTDOWN,
							power_state.CRASHED)
			and db_instance['vm_state'] == vm_states.ACTIVE):
			self._instance_update(context,
								db_instance["id"],
								power_state=vm_power_state,
								vm_state=vm_states.SHUTOFF)
		# primton:
		# change the VM status
		elif(vm_power_state == power_state.BLOCKED or vm_power_state == power_state.RUNNING):
			self._instance_update(context,
								db_instance["id"],
								power_state=vm_power_state,
								vm_state=vm_states.ACTIVE)
		else:
			self._instance_update(context,
								db_instance["id"],
								power_state=vm_power_state)
```

power状态集
-----------

```python
"""The various power states that a VM can be in."""

#NOTE(justinsb): These are the virDomainState values from libvirt
NOSTATE = 0x00
RUNNING = 0x01
BLOCKED = 0x02
PAUSED = 0x03
SHUTDOWN = 0x04
SHUTOFF = 0x05
CRASHED = 0x06
SUSPENDED = 0x07
FAILED = 0x08
BUILDING = 0x09

# TODO(justinsb): Power state really needs to be a proper class,
# so that we're not locked into the libvirt status codes and can put mapping
# logic here rather than spread throughout the code
_STATE_MAP = {
    NOSTATE: 'pending',
    RUNNING: 'running',
    BLOCKED: 'blocked',
    PAUSED: 'paused',
    SHUTDOWN: 'shutdown',
    SHUTOFF: 'shutdown',
    CRASHED: 'crashed',
    SUSPENDED: 'suspended',
    FAILED: 'failed to spawn',
    BUILDING: 'building',
}
```
