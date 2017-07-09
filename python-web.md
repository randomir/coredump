# Python web related (requests, Flask, Django, etc.)


## Question: [Python Flask how to pass a dynamic parameter to a decorator](https://stackoverflow.com/q/44994612/404556)

I am using python flask framework. I write a decorator which will be need a parameter, and this parameter will be dynamic.

my decorator like below, will be get a key ,and using the key fetch data from redis.

    def redis_hash_shop_style(key):
        def fn_wrapper(f):
            @wraps(f)
            def decorated_function(*args, **kwargs):
                data = redis_hash(key)
                return data
            return decorated_function
    return fn_wrapper

and I have a class to using this decorater, code like this

    class ShopAreaAndStyleListAPI(Resource):
        @redis_hash_shop_style(key='shop_{}_style'.format(g.city.id))
        def get(self):
            # if not found from redis, query from mysql
            pass

As you see, my decorator need a parameter named `key`, and I pass the key like this 

    @redis_hash_shop_style(key='shop_{}_style'.format(g.city.id))

`g.city.id` will be get the city's id, if everything is ok, the key will be like this `shop_100_style`,

but I got the error:

    class ShopAreaAndStyleListAPI(Resource):
    File "xx.py", line 659, in ShopAreaAndStyleListAPI

    @redis_hash_shop_style(key='shop_{}_style'.format(g.city.id))

    File "/Users/xx/.virtualenvs/yy/lib/python2.7/site-packages/werkzeug/local.py", line 347, in __getattr__
    return getattr(self._get_current_object(), name)
    File "/Users/xx/.virtualenvs/yy/lib/python2.7/site-packages/werkzeug/local.py", line 306, in _get_current_object
    return self.__local()
    File "/Users/xx/.virtualenvs/yy/lib/python2.7/site-packages/flask/globals.py", line 44, in _lookup_app_object
    raise RuntimeError(_app_ctx_err_msg)
    RuntimeError: Working outside of application context.

    This typically means that you attempted to use functionality that 
    needed to interface with the current application object in a way.  
    To solve this set up an application context with app.app_context().  
    See the documentation for more information.

I am quite confused, in flask, how to pass a dynamic parameter to a decorator?


## Answer

If we check the docs for [flask application global](http://flask.pocoo.org/docs/0.12/api/#application-globals), `flask.g`, it says:

> To share data that is valid for one request only from one function to another, a global variable
> is not good enough because it would break in threaded environments. Flask provides you with a
> **special object** that ensures it is **only valid for the active request** and that will return
> different values for each request.

This is achieved by using a thread-local proxy (in
[`flask/globals.py`](https://github.com/pallets/flask/blob/master/flask/globals.py#L61)):

    g = LocalProxy(partial(_lookup_app_object, 'g'))

The other thing we should keep in mind is that Python is executing the first pass of our decorator
during the "compile" phase, outside of any request, or `flask` application. That means `key`
argument get assigned a value of  `'shop_{}_style'.format(g.city.id)` when your application starts
(when your class is being parsed/decorated), outside of `flask` request context.

But we can easily delay accessing to `flask.g` by using a lazy proxy, which fetches the value only
when used, via callback function. Let's use the one already bundled with `flask`, the
[`werkzeug.local.LocalProxy`](http://werkzeug.pocoo.org/docs/0.12/local/#werkzeug.local.LocalProxy):

    from werkzeug.local import LocalProxy
    
    class ShopAreaAndStyleListAPI(Resource):
        @redis_hash_shop_style(key=LocalProxy(lambda: 'shop_{}_style'.format(g.city.id)))
        def get(self):
            # if not found from redis, query from mysql
            pass

In general (for non-`flask` or non-`werkzeug` apps), we can use a similar `LazyProxy` from the
[`ProxyTypes`](https://pypi.python.org/pypi/ProxyTypes/0.9) package.

Unrelated to this, you'll also want to fix your `redis_hash_shop_style` decorator to not only fetch
from `redis`, but to also update (or create) the value if stale (or non-existing), by calling the
wrapped `f()` when appropriate.
