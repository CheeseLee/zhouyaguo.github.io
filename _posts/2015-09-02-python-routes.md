---
layout: post
title: "Routes"
comments: true
categories:
- openstack
---

先上nova的代码：

```python
class APIRouter(nova.api.openstack.APIRouter):
    """
    Routes requests on the OpenStack API to the appropriate controller
    and method.
    """
    ExtensionManager = extensions.ExtensionManager

    def _setup_routes(self, mapper):
        self.resources['versions'] = versions.create_resource()
        mapper.connect("versions", "/",
                    controller=self.resources['versions'],
                    action='show')

        mapper.redirect("", "/")

        self.resources['consoles'] = consoles.create_resource()
        mapper.resource("console", "consoles",
                    controller=self.resources['consoles'],
                    parent_resource=dict(member_name='server',
                    collection_name='servers'))

        self.resources['servers'] = servers.create_resource()
        mapper.resource("server", "servers",
                        controller=self.resources['servers'],
                        collection={'detail': 'GET'},
                        member={'action': 'POST'})

        self.resources['ips'] = ips.create_resource()
        mapper.resource("ip", "ips", controller=self.resources['ips'],
                        parent_resource=dict(member_name='server',
                                             collection_name='servers'))

        self.resources['images'] = images.create_resource()
        mapper.resource("image", "images",
                        controller=self.resources['images'],
                        collection={'detail': 'GET'})

        self.resources['limits'] = limits.create_resource()
        mapper.resource("limit", "limits",
                        controller=self.resources['limits'])

        self.resources['flavors'] = flavors.create_resource()
        mapper.resource("flavor", "flavors",
                        controller=self.resources['flavors'],
                        collection={'detail': 'GET'})

        self.resources['image_metadata'] = image_metadata.create_resource()
        image_metadata_controller = self.resources['image_metadata']

        mapper.resource("image_meta", "metadata",
                        controller=image_metadata_controller,
                        parent_resource=dict(member_name='image',
                        collection_name='images'))

        mapper.connect("metadata", "/{project_id}/images/{image_id}/metadata",
                       controller=image_metadata_controller,
                       action='update_all',
                       conditions={"method": ['PUT']})

        self.resources['server_metadata'] = server_metadata.create_resource()
        server_metadata_controller = self.resources['server_metadata']

        mapper.resource("server_meta", "metadata",
                        controller=server_metadata_controller,
                        parent_resource=dict(member_name='server',
                        collection_name='servers'))

        mapper.connect("metadata",
                       "/{project_id}/servers/{server_id}/metadata",
                       controller=server_metadata_controller,
                       action='update_all',
                       conditions={"method": ['PUT']})

```

map.resource("message", "messages")产生的规则表现为：

	```
	GET    /messages        => messages.index()    => url("messages")
	POST   /messages        => messages.create()   => url("messages")
	GET    /messages/new    => messages.new()      => url("new_message")
	PUT    /messages/1      => messages.update(id) => url("message", id=1)
	DELETE /messages/1      => messages.delete(id) => url("message", id=1)
	GET    /messages/1      => messages.show(id)   => url("message", id=1)
	GET    /messages/1/edit => messages.edit(id)   => url("edit_message", id=1)
	```


Nova的Rest API
------------

- versions.VersionV2

	```
	GET      /              show
	```

- servers.Controller

	```
	GET      /servers       index
	POST     /servers       create
	PUT      /servers/1     update
	DELETE   /servers/1     delete
	GET      /servers/1     show
	GET      /server/detail detail
	```

- consoles.Controller

	```
	GET      /servers/1/consoles       index
	POST     /servers/1/consoles       create
	PUT      /servers/1/consoles/1     update
	DELETE   /servers/1/consoles/1     delete
	GET      /servers/1/consoles/1     show
	```

- ips.Controller

	```
	GET      /servers/1/ips       index
	POST     /servers/1/ips       create
	PUT      /servers/1/ips/1     update
	DELETE   /servers/1/ips/1     delete
	GET      /servers/1/ips/1     show
	```

- images.Controller

	```
	GET      /images       index
	POST     /images       create
	PUT      /images/1     update
	DELETE   /images/1     delete
	GET      /images/1     show
	GET      /images/detail detail
	```

- limits.Controller

	```
	GET      /limits       index
	POST     /limits       create
	PUT      /limits/1     update
	DELETE   /limits/1     delete
	GET      /limits/1     show
	```

- flavors.Controller

	```
	GET      /flavors       index
	POST     /flavors       create
	PUT      /flavors/1     update
	DELETE   /flavors/1     delete
	GET      /flavors/1     show
	GET      /flavors/detail detail
	```

- image_metadata.Controller

	```
	GET      /images/1/metadata       index
	POST     /images/1/metadata       create
	PUT      /images/1/metadata/1     update
	DELETE   /images/1/metadata/1     delete
	GET      /images/1/metadata/1     show
	PUT      /{project_id}/images/{image_id}/metadata   update_all
	```

- server_metadata.Controller

	```
	GET      /servers/1/metadata       index  
	POST     /servers/1/metadata       create  
	PUT      /servers/1/metadata/1     update  
	DELETE   /servers/1/metadata/1     delete  
	GET      /servers/1/metadata/1     show  
	PUT      /{project_id}/servers/{server_id}/metadata update_all  
	```

示例
----

```
curl -i http://172.17.140.73:8774/v1.1/ -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: 2685100346d845c2b687373b7cedbd07"

curl -i http://172.17.140.73:5000/v2.0/tokens -H "Content-Type: application/json" --data '{"auth": {"tenantName": "tenant", "passwordCredentials": {"username": "admin", "password": "admin"}}}'

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/servers/detail -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: 2685100346d845c2b687373b7cedbd07"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/servers -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/servers/1d5460ac-8e36-458b-96e2-1919eaf1af78 -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/servers/da8a4222-3da6-44c2-98af-84903bc7c95c/ips -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/images  -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/images/f790adba-fca3-4cd4-a99e-e47947b3114b/metadata  -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/flavors  -H "X-Auth-Token: 0fc6a5907ad440e886f9342178fb3c49"

```

nova extensions
----------------

nova中还有很多extensions，也提供Rest API，以keypairs为例：

```
class KeypairController(object):
    """ Keypair API controller for the OpenStack API """

    @wsgi.serializers(xml=KeypairTemplate)
    def create(self, req, body):
        ...

    def delete(self, req, id):
        ...

    @wsgi.serializers(xml=KeypairsTemplate)
    def index(self, req):
        ...

class Keypairs(extensions.ExtensionDescriptor):
    """Keypair Support"""

    name = "Keypairs"
    alias = "os-keypairs"
    namespace = "http://docs.openstack.org/compute/ext/keypairs/api/v1.1"
    updated = "2011-08-08T00:00:00+00:00"

    def get_resources(self):
        resources = []

        res = extensions.ResourceExtension(
                'os-keypairs',
                KeypairController())

        resources.append(res)
        return resources
```

示例：

```
curl -i http://172.17.140.73:8774/v1.1/596ef07ab9a34ed1bf5f88c1db4de7bc/os-keypairs -X GET -H "X-Auth-Project-Id: tenant" -H "X-Auth-Token: a455de1d83964984842d80b64439099f"
```


Reference
==========

- [https://routes.readthedocs.org/en/latest/](https://routes.readthedocs.org/en/latest/)
