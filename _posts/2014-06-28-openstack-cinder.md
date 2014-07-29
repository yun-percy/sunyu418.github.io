---
layout: post
title: "openstack cinder api源码分析"
description: ""
category: 
tags: []
---
{% include JB/setup %}

cinder api service负责接受和处理Rest请求，并将请求放入RabbitMQ队列。<br/>
下面是cinder-api的启动脚本：
{% highlight py linenos%}
"""Starter script for Cinder OS API."""

import eventlet
eventlet.monkey_patch()

import os
import sys

from oslo.config import cfg

possible_topdir = os.path.normpath(os.path.join(os.path.abspath(sys.argv[0]),
                                   os.pardir,
                                   os.pardir))
if os.path.exists(os.path.join(possible_topdir, "cinder", "__init__.py")):
    sys.path.insert(0, possible_topdir)

from cinder.openstack.common import gettextutils
gettextutils.install('cinder', lazy=False)

# Need to register global_opts
from cinder.common import config  # noqa
from cinder.openstack.common import log as logging
from cinder import rpc
from cinder import service
from cinder import utils
from cinder import version


CONF = cfg.CONF


if __name__ == '__main__':
    CONF(sys.argv[1:], project='cinder',
         version=version.version_string())
    logging.setup("cinder")
    utils.monkey_patch()

    rpc.init(CONF)
    launcher = service.process_launcher()
	# WSGI 服务
    server = service.WSGIService('osapi_volume')
    launcher.launch_service(server, workers=server.workers or 1)
    launcher.wait()
{% endhighlight %}
从代码中可以看出cinder-api实际上是一个WSGI服务。WSGI的官方定义是，the Python Web Server Gateway Interface。从名字就可以看出来，这东西是一个Gateway，也就是网关。网关的作用就是在协议之间进行转换。 也就是说，WSGI就像是一座桥梁，一边连着web服务器，另一边连着用户的应用。

![WSGI Service]({{ ASSET_PATH }}../img/wsgi.jpg)

WSGI app用来处理Web请求，它是一个callable对象。

来看service.WSGIService()的__init__()方法：
{% highlight py linenos %}
class WSGIService(object):
    """Provides ability to launch API from a 'paste' configuration."""

    def __init__(self, name, loader=None):
        """Initialize, but do not start the WSGI server.

        :param name: The name of the WSGI server given to the loader.
        :param loader: Loads the WSGI application using the given name.
        :returns: None

        """
        self.name = name
        self.manager = self._get_manager()
        self.loader = loader or wsgi.Loader()
		# 加载Application
        self.app = self.loader.load_app(name)
		# 从配置中读取相关参数
        self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")
        self.port = getattr(CONF, '%s_listen_port' % name, 0)
        self.workers = getattr(CONF, '%s_workers' % name, None)
        if self.workers < 1:
            LOG.warn(_("Value of config option %(name)s_workers must be "
                       "integer greater than 1.  Input value ignored.") %
                     {'name': name})
            # Reset workers to default
            self.workers = None
        self.server = wsgi.Server(name,
                                  self.app,
                                  host=self.host,
                                  port=self.port)

class Loader(object):
    """Used to load WSGI applications from paste configurations.
	   从paste配置文件中加载WSGI application
	"""

    def __init__(self, config_path=None):
        """Initialize the loader, and attempt to find the config.

        :param config_path: Full or relative path to the paste config.
        :returns: None

        """
        config_path = config_path or CONF.api_paste_config
        self.config_path = utils.find_config(config_path)

    def load_app(self, name):
        """Return the paste URLMap wrapped WSGI application.

        :param name: Name of the application to load.
        :returns: Paste URLMap object wrapping the requested application.
        :raises: `cinder.exception.PasteAppNotFound`

        """
        try:
			# 调用paste deploy中的loadapp从paste配置文件中加载
            return deploy.loadapp("config:%s" % self.config_path, name=name)
        except LookupError as err:
            LOG.error(err)
            raise exception.PasteAppNotFound(name=name, path=self.config_path)
{% endhighlight %}
Paste Deployment是一种机制，通过loadapp函数和一个配置文件或者egg包来载入WSGI app。

以api v2为例，来看一下paste配置：
{% highlight ini linenos %}
[composite:openstack_volume_api_v2]
use = call:cinder.api.middleware.auth:pipeline_factory
noauth = request_id faultwrap sizelimit noauth apiv2
keystone = request_id faultwrap sizelimit authtoken keystonecontext apiv2
keystone_nolimit = request_id faultwrap sizelimit authtoken keystonecontext apiv2
。。。
[app:apiv2]
paste.app_factory = cinder.api.v2.router:APIRouter.factory
{% endhighlight %}

如果是从Web传过来的请求中api是v2，那么处理这个请求的就是cinder.api.v2.router.APIRouter.factory生成的callable对象。

那么cinder是如何将不同的url请求map到不同的app来处理的呢？
cinder.api.v2.router.APIRouter的父类是cinder.api.v2.openstack.APIRouter，cinder.api.v2.openstack.APIRouter的父类是cinder.wsgi.Router。
{% highlight py linenos %}
class Router(object):
    """WSGI middleware that maps incoming requests to WSGI apps."""

    def __init__(self, mapper):
        """Create a router for the given routes.Mapper.

        Each route in `mapper` must specify a 'controller', which is a
        WSGI app to call.  You'll probably want to specify an 'action' as
        well and have your controller be an object that can route
        the request to the action-specific method.

        Examples:
          mapper = routes.Mapper()
          sc = ServerController()

          # Explicit mapping of one route to a controller+action
          mapper.connect(None, '/svrlist', controller=sc, action='list')

          # Actions are all implicitly defined
          mapper.resource('server', 'servers', controller=sc)

          # Pointing to an arbitrary WSGI app.  You can specify the
          # {path_info:.*} parameter so the target app can be handed just that
          # section of the URL.
          mapper.connect(None, '/v1.0/{path_info:.*}', controller=BlogApp())

        """
        self.map = mapper
        self._router = routes.middleware.RoutesMiddleware(self._dispatch,
                                                          self.map)

    @webob.dec.wsgify(RequestClass=Request)
    def __call__(self, req):
        """Route the incoming request to a controller based on self.map.

        If no match, return a 404.

        """
        return self._router

    @staticmethod
    @webob.dec.wsgify(RequestClass=Request)
    def _dispatch(req):
        """Dispatch the request to the appropriate controller.

        Called by self._router after matching the incoming request to a route
        and putting the information into req.environ.  Either returns 404
        or the routed WSGI app's response.

        """
        match = req.environ['wsgiorg.routing_args'][1]
        if not match:
            return webob.exc.HTTPNotFound()
        app = match['controller']
        return app
{% endhighlight %}
来看cinder.api.v2.openstack.APIRouter：
{% highlight py linenos %}
class APIRouter(base_wsgi.Router):
    """Routes requests on the API to the appropriate controller and method."""
    ExtensionManager = None  # override in subclasses

    @classmethod
    def factory(cls, global_config, **local_config):
        """Simple paste factory, :class:`cinder.wsgi.Router` doesn't have."""
        return cls()

    def __init__(self, ext_mgr=None):
        if ext_mgr is None:
            if self.ExtensionManager:
                ext_mgr = self.ExtensionManager()
            else:
                raise Exception(_("Must specify an ExtensionManager class"))

        mapper = ProjectMapper()
        self.resources = {}
		# 创建routes
        self._setup_routes(mapper, ext_mgr)
		# 创建扩展routes
        self._setup_ext_routes(mapper, ext_mgr)
        self._setup_extensions(ext_mgr)
        super(APIRouter, self).__init__(mapper)

    def _setup_ext_routes(self, mapper, ext_mgr):
        for resource in ext_mgr.get_resources():
            LOG.debug(_('Extended resource: %s'),
                      resource.collection)

            wsgi_resource = wsgi.Resource(resource.controller)
            self.resources[resource.collection] = wsgi_resource
            kargs = dict(
                controller=wsgi_resource,
                collection=resource.collection_actions,
                member=resource.member_actions)

            if resource.parent:
                kargs['parent_resource'] = resource.parent

            mapper.resource(resource.collection, resource.collection, **kargs)

            if resource.custom_routes_fn:
                resource.custom_routes_fn(mapper, wsgi_resource)

    def _setup_extensions(self, ext_mgr):
        for extension in ext_mgr.get_controller_extensions():
            collection = extension.collection
            controller = extension.controller

            if collection not in self.resources:
                LOG.warning(_('Extension %(ext_name)s: Cannot extend '
                              'resource %(collection)s: No such resource'),
                            {'ext_name': extension.extension.name,
                             'collection': collection})
                continue

            LOG.debug(_('Extension %(ext_name)s extending resource: '
                        '%(collection)s'),
                      {'ext_name': extension.extension.name,
                       'collection': collection})

            resource = self.resources[collection]
            resource.register_actions(controller)
            resource.register_extensions(controller)

    def _setup_routes(self, mapper, ext_mgr):
        raise NotImplementedError
{% endhighlight %}

再来看cinder.api.v2.router.APIRouter：
{% highlight py linenos %}
class APIRouter(cinder.api.openstack.APIRouter):
    """Routes requests on the API to the appropriate controller and method."""
    ExtensionManager = extensions.ExtensionManager

    def _setup_routes(self, mapper, ext_mgr):
        self.resources['versions'] = versions.create_resource()
        mapper.connect("versions", "/",
                       controller=self.resources['versions'],
                       action='show')

        mapper.redirect("", "/")

        self.resources['volumes'] = volumes.create_resource(ext_mgr)
        mapper.resource("volume", "volumes",
                        controller=self.resources['volumes'],
                        collection={'detail': 'GET'},
                        member={'action': 'POST'})

        self.resources['types'] = types.create_resource()
        mapper.resource("type", "types",
                        controller=self.resources['types'])

        self.resources['snapshots'] = snapshots.create_resource(ext_mgr)
        mapper.resource("snapshot", "snapshots",
                        controller=self.resources['snapshots'],
                        collection={'detail': 'GET'},
                        member={'action': 'POST'})

        self.resources['limits'] = limits.create_resource()
        mapper.resource("limit", "limits",
                        controller=self.resources['limits'])

        self.resources['snapshot_metadata'] = \
            snapshot_metadata.create_resource()
        snapshot_metadata_controller = self.resources['snapshot_metadata']

        mapper.resource("snapshot_metadata", "metadata",
                        controller=snapshot_metadata_controller,
                        parent_resource=dict(member_name='snapshot',
                                             collection_name='snapshots'))

        mapper.connect("metadata",
                       "/{project_id}/snapshots/{snapshot_id}/metadata",
                       controller=snapshot_metadata_controller,
                       action='update_all',
                       conditions={"method": ['PUT']})

        self.resources['volume_metadata'] = \
            volume_metadata.create_resource()
        volume_metadata_controller = self.resources['volume_metadata']

        mapper.resource("volume_metadata", "metadata",
                        controller=volume_metadata_controller,
                        parent_resource=dict(member_name='volume',
                                             collection_name='volumes'))

        mapper.connect("metadata",
                       "/{project_id}/volumes/{volume_id}/metadata",
                       controller=volume_metadata_controller,
                       action='update_all',
                       conditions={"method": ['PUT']})
{% endhighlight %}
扩展的一些api是通过extensions.ExtensionManager实现的。