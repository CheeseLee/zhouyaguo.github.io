---
layout: post
title: "Compute Node资源使用情况"
comments: true
categories:
- openstack
---

表`compute_nodes`保存了compute node的资源使用情况，主要包括几个方面：

- hypervisor信息
	- hypervisor_type (xen)
	- hypervisor_version (4001000)
- 计算
	- cpu_info （例如： {"arch": "x86\_64", "features": [], "topology": {}} ）
	- 物理vcpus数量 （os.sysconf('SC_NPROCESSORS_ONLN')）
	- vcpus_used （利用libvirt connection找出所有的domain，累加vcpus）
- 内存
	- host_memory_total （libvirt connection getInfo()获得）
	- memory_mb_used  （利用libvirt connection找出所有的domain，累加已使用mem。如果是xen，还算上domain0的内存）
- 磁盘
	- FLAGS.instances_path的总空间  (os.statvfs(path))
	- FLAGS.instances_path的已使用空间
	- FLAGS.instances_path的剩余空间

compute_nodes表结构
-------------------

```
CREATE TABLE `compute_nodes` (
  `created_at` datetime default NULL,
  `updated_at` datetime default NULL,
  `deleted_at` datetime default NULL,
  `deleted` tinyint(1) default NULL,
  `id` int(11) NOT NULL auto_increment,
  `service_id` int(11) NOT NULL,
  `vcpus` int(11) NOT NULL,
  `memory_mb` int(11) NOT NULL,
  `local_gb` int(11) NOT NULL,
  `vcpus_used` int(11) NOT NULL,
  `memory_mb_used` int(11) NOT NULL,
  `local_gb_used` int(11) NOT NULL,
  `hypervisor_type` mediumtext NOT NULL,
  `hypervisor_version` int(11) NOT NULL,
  `cpu_info` mediumtext NOT NULL,
  `disk_available_least` int(11) default NULL,
  `free_ram_mb` int(11) default NULL,
  `free_disk_gb` int(11) default NULL,
  `current_workload` int(11) default NULL,
  `running_vms` int(11) default NULL,
  `hypervisor_hostname` varchar(255) default NULL,
  PRIMARY KEY  (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8
```

HostState代码
------------

```python
class HostState(object):
    """Manages information about the compute node through libvirt"""
    def __init__(self, read_only):
        super(HostState, self).__init__()
        self.read_only = read_only
        self._stats = {}
        self.connection = None
        self.update_status()

    def get_host_stats(self, refresh=False):
        """Return the current state of the host.

        If 'refresh' is True, run update the stats first."""
        if refresh:
            self.update_status()
        return self._stats

    def update_status(self):
        """Retrieve status info from libvirt."""
        LOG.debug(_("Updating host stats"))
        if self.connection is None:
            self.connection = get_connection(self.read_only)
        data = {}
        data["vcpus"] = self.connection.get_vcpu_total()
        data["vcpus_used"] = self.connection.get_vcpu_used()
        data["cpu_info"] = utils.loads(self.connection.get_cpu_info())
        data["disk_total"] = self.connection.get_local_gb_total()
        data["disk_used"] = self.connection.get_local_gb_used()
        data["disk_available"] = data["disk_total"] - data["disk_used"]
        data["host_memory_total"] = self.connection.get_memory_mb_total()
        data["host_memory_free"] = (data["host_memory_total"] -
                                    self.connection.get_memory_mb_used())
        data["hypervisor_type"] = self.connection.get_hypervisor_type()
        data["hypervisor_version"] = self.connection.get_hypervisor_version()

        self._stats = data

        return data
```
