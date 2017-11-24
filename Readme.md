# Overview #

Drop-in server for [Google App Engine](http://code.google.com/appengine/) applications which exposes your data model via a REST
API with no extra work.

## Basic Features ##
  * Metadata browsing
    * List models: `GET "/metadata"`
    * Define model in XML Schema: `GET "/metadata/MyModel"`
  * Reading data (XML output)
    * Get specific instance: `GET "/MyModel/<key>"`
    * Get all instances (paged results): `GET "/MyModel"`
      * Results can be ordered using the "ordering" query param: `"?ordering=<propertyName>"`
    * Find multiple instances (paged results): `GET "/MyModel?<queryParams>"`
      * Results can be ordered (same as get all)
  * Modifying data (XML input)
    * Create new instance (returns key): `POST "/MyModel"`
      * Supports batch create by surrounding multiple instances with `<list>` element (returns keys)
    * Partial update instance (returns key): `POST "/MyModel/<key>"`
      * Supports batch update (same as create)
    * Complete update instance (returns key): `PUT "/MyModel/<key>"`
      * Supports batch update (sames as create)
    * Output of all update operations may be altered with "type" query param
      * Return the keys in an XML format: `"?type=structured"`
      * Return the entire updated models: `"?type=full"`
    * Delete instance: `"DELETE "/MyModel/<key"`

Check out the [Features](../../blob/wiki/Features.md) page for a more complete list of the features supported by the appengine rest server, including advanced features.

## Example ##

The [Boomi Demo](http://boomi-demo.appspot.com/) App Engine application is a fully working example application (based on the Google App Engine Greeting demo application).

  * Available types: http://boomi-demo.appspot.com/rest/metadata
  * Greeting schema: http://boomi-demo.appspot.com/rest/metadata/Greeting
  * Greeting instances: http://boomi-demo.appspot.com/rest/Greeting
  * Greeting instances with filter: http://boomi-demo.appspot.com/rest/Greeting?feq_author=bob@example.com&feq_date=2008-11-03T00:21:19.080553
  * Data accessible in both XML and JSON format (input can be specified using the HTTP "Content-Type" header, output can be specified using the HTTP "Accept" request header, e.g. "application/xml" or "application/json")
  
## Advanced Features ##
  * Working with individual properties
    * Read single property: `GET "/MyModel/<key>/<propertyName>"`
      * Output "Content-Type" header can be specified using input "Accept" header
    * Read single element of list property: `GET "/MyModel/<key>/<propertyName>/<index>"`
    * Write single property: `POST "/MyModel/<key>/<propertyName>"`
      * Same output options as instance updates
    * Write single element of list property: `POST "/MyModel/<key>/<propertyName>/<index>"`
    * All blob types are _not_ Base64 encoded
  * JSON support
    * Specify JSON input using header: `"Content-Type: application/json"`
    * Request JSON output using header: `"Accept: application/json"`
    * [JSONP](http://en.wikipedia.org/wiki/JSON#JSONP) support using "callback" query parameter: `"?callback=my_method"`
  * Simulate HTTP PUT/DELETE using POST with header `"X-HTTP-Method-Override: <realMethod>"`
  * Custom Authentication and Authorization (for multi-tenancy)
  * Filter returned fields using "include\_props" query parameter: `"?include_props=<prop1>,<prop2>,..."`
  * Extended BlobInfo support
    * Include extra info (as attributes) in BlobInfo reference property (creation, filename, etc.) using "blobinfo" query parameter: `"?blobinfo=info"`
    * Download actual BlobInfo data: `GET "/MyModel/<key>/<blobProperty>/content"`
    * Upload BlobInfo
      1. `POST "/MyModel/<key>/<blobProperty>/content` -> returns upload form
      1. `POST "<formUrl>"` (with actual data) -> redirect url
      1. `GET "<redirectUrl>"` -> normal update results
  * Optional ETags Support (as of the 1.0.7 release)
    * Must be enabled on the server using the `Dispatcher.enable_etags` property
    * For GET requests, an ETag header will be returned on the request which applies to the _entire_ response body.  additionally, the model elements themselves will now include an "etag" attribute which is specific to that model only.  so, for a single model retrieval, the header and model value will be the same.  for a query operation with multiple models, the header will be an aggregate value different from each model value.  GET requests will honor the "If-None-Match" header, which can either specify a single value (for single model GETs or an entire collection) or multiple values (for each individual model in a collection response).  if the "If-None-Match" header is matched, an http 304 response code will be returned.
    * For PUT/POST/DELETE requests, the "If-Match" header will be honored.  Like the "If-None-Match" header, this can be a single value or multiple values.  alternatively, the model specific etag values can be provided in the input models themselves (in an etag attribute).  If the input etags do not match, an http 412 response code will be returned and _no_ modifications will be made on the server side.  Additionally, PUT/POST responses will include updated etag information similar to GET requests.

## Setup ##

# Getting Started #

## Setup ##

Utilizing this library is extremely simple.  Assuming you have the library code installed under the directory "rest" within your application (i.e. `"rest/__init__.py"`), you would add the following to your main application code:

```
import rest

# add a handler for "rest" calls
application = webapp.WSGIApplication([
  <... existing webservice urls ...>
  ('/rest/.*', rest.Dispatcher)
], ...)

# configure the rest dispatcher to know what prefix to expect on request urls
rest.Dispatcher.base_url = "/rest"

# add all models from the current module, and/or...
rest.Dispatcher.add_models_from_module(__name__)
# add all models from some other module, and/or...
rest.Dispatcher.add_models_from_module(my_model_module)
# add specific models
rest.Dispatcher.add_models({
  "foo": FooModel,
  "bar": BarModel})
# add specific models (with given names) and restrict the supported methods
rest.Dispatcher.add_models({
  'foo' : (FooModel, rest.READ_ONLY_MODEL_METHODS),
  'bar' : (BarModel, ['GET_METADATA', 'GET', 'POST', 'PUT'],
  'cache' : (CacheModel, ['GET', 'DELETE'] })

# optionally use custom authentication/authorization
rest.Dispatcher.authenticator = MyAuthenticator()
rest.Dispatcher.authorizer = MyAuthorizer()

```

For some example implementations of custom authentication and authorization see [ExampleAuthenticator](ExampleAuthenticator.md) and [ExampleAuthorizer](ExampleAuthorizer.md).

## Client Usage ##

Once this server has been installed in your application, the basic usage is as follows (assuming you installed the REST API with the url prefix "/rest" as shown above).

  * Metadata Browsing
    * `GET http://<service>/rest/metadata`
      * Gets all known types
    * `GET http://<service>/rest/metadata/<typeName>`
      * Gets the `<typeName>` type profile (as XML Schema).  (If the model is an Expando model, the schema will include an "any" element).
  * Object Manipulation
    * `GET http://<service>/rest/<typeName>`
      * Gets the first page of `<typeName>` instances (number returned per page is defined by server).  The returned list element will contain an "offset" attribute.  If it has a value, that is the next offset to use to retrieve more results.  If it is empty, there are no more results.
    * `GET http://<service>/rest/<typeName>?offset=50`
      * Gets the page of `<typeName>` instances starting at offset 50 (0 based numbering).  The offset should generally be filled in from a previous request.
    * `GET http://<service>/rest/<typeName>?<queryTerm>[&<queryTerm>]`
      * Gets a page of `<typeName>` instances using a query filter created from the given query terms (with offset features mentioned above).
        * Multiple query terms will be AND'ed together to create the filter.
        * A query filter term has the structure: `f<op>_<propertyName>=<value>`
          * Examples:
            * `"feq_author=bob@example.com"` means include instances where the value of the "author" property is equal to "bob@example.com"
            * `"flt_count=37&fin_content=value1,value2"` means include instances where the value of the "count" property greater than "37" and the value of the content property is "value1" or "value2"
        * Available operations:
          * `"feq_" -> "equal to"`
          * `"flt_" -> "less than"`
          * `"fgt_" -> "greater than"`
          * `"fle_" -> "less than or equal to"`
          * `"fge_" -> "greater than or equal to"`
          * `"fne_" -> "not equal to"`
          * `"fin_" -> "in <commaSeparatedList>"`
        * Blob and Text properties may not be used in a query filter
    * `GET http://<service>/rest/<typeName>/<key>`
      * Gets the single `<typeName>` instance with the given `<key>`
    * `POST http://<service>/rest/<typeName>`
      * Create new `<typeName>` instance using the posted data which should adhere to the XML Schema for the type
      * Returns the key of the new instance by default.  With "?type=full" at the end of the url, returns the entire updated instance like a GET request.
    * `POST http://<service>/rest/<typeName>/<key>`
      * Partial update of the existing `<typeName>` instance with the given `<key>`.  Will only modify fields included in the posted xml data.
      * (Returns same as previous request)
    * `PUT http://<service>/rest/<typeName>/<key>`
      * Complete replacement of the existing `<typeName>` instance with the given `<key>`
      * (Returns same as previous request)
    * `DELETE http://<service>/rest/<typeName>/<key>`
      * Delete the existing `<typeName>` instance
      
# Example Authenticator #

The Dispatcher supports custom authentication by plugging in a custom Authenticator.  Below is an almost complete, example implementation of an Authenticator for doing HTTP Basic Authentication.  The missing piece (see the FIXME comment) is validating the given credentials against any stored information (the application would need to determine how to create and store these credentials).

You would utilize this Authenticator by setting it on the Dispatcher:
```
rest.Dispatcher.authenticator = BasicAuthenticator()
```

# BasicAuthenticator #

```
import rest
import logging
import base64

AUTHENTICATE_HEADER = "WWW-Authenticate"
AUTHORIZATION_HEADER = "Authorization"
AUTHENTICATE_TYPE = 'Basic realm="Secure Area"'
CONTENT_TYPE_HEADER = "Content-Type"
HTML_CONTENT_TYPE = "text/html"

class BasicAuthenticator(rest.Authenticator):
    """Example implementation of HTTP Basic Auth."""

    def __init__(self):
        super(BasicAuthenticator, self).__init__()

    def authenticate(self, dispatcher):

        user_arg = None
        pass_arg = None
        try:
            # Parse the header to extract a user/password combo.
            # We're expecting something like "Basic XZxgZRTpbjpvcGVuIHYlc4FkZQ=="
            auth_header = dispatcher.request.headers[AUTHORIZATION_HEADER]

            # Isolate the encoded user/passwd and decode it
            auth_parts = auth_header.split(' ')
            user_pass_parts = base64.b64decode(auth_parts[1]).split(':')
            user_arg = user_pass_parts[0]
            pass_arg = user_pass_parts[1]

        except Exception:
            # set the headers requesting the browser to prompt for a user/password:
            dispatcher.response.set_status(401, message="Authentication Required")
            dispatcher.response.headers[AUTHENTICATE_HEADER] = AUTHENTICATE_TYPE
            dispatcher.response.headers[CONTENT_TYPE_HEADER] = HTML_CONTENT_TYPE

            dispatcher.response.out.write("<html><body>401 Authentication Required</body></html>")
            raise rest.DispatcherException()

        # FIXME, writeme: if(valid user_arg,pass_arg):
        #     return

        dispatcher.forbidden()
```
# Example Authorizer #

The Dispatcher supports custom authorization by plugging in a custom Authorizer.  Below is an example implementation of an Authenticator controlling access to Model instances based on a UserProperty field named "owner" on every Model class.

You would utilize this Authorizer by setting it on the Dispatcher:
```
rest.Dispatcher.authorizer = OwnerAuthorizer()
```

# OwnerAuthorizer #

```
from google.appengine.api import users


class OwnerAuthorizer(rest.Authorizer):

    def can_read(self, dispatcher, model):
        if(model.owner != users.get_current_user()):
            dispatcher.not_found()

    def filter_read(self, dispatcher, models):
        return self.filter_models(models)

    def check_query(self, dispatcher, query_expr, query_params):
        query_params.append(users.get_current_user())
        if(not query_expr):
            query_expr = 'WHERE owner = :%d' % (len(query_params))
        else:
            query_expr += ' AND owner = :%d' % (len(query_params))
        return query_expr

    def can_write(self, dispatcher, model, is_replace):
        if(not model.is_saved()):
            # creating a new model
            model.owner = users.get_current_user()
        elif(model.owner != users.get_current_user()):
            dispatcher.not_found()

    def filter_write(self, dispatcher, models, is_replace):
        return self.filter_models(models)

    def can_delete(self, dispatcher, model_type, model_key):
        query = model_type.all(True).filter("owner = ", users.get_current_user()).filter("__key__ = ", model_key)
        if(len(query.fetch(1)) == 0):
            dispatcher.not_found()

    def filter_models(self, models):
        cur_user = users.get_current_user()
        models[:] = [model for model in models if model.owner == cur_user]
        return models
```
# Overview Namespace Authorizor #

The Dispatcher supports custom authorization by plugging in a custom Authorizer.  Below is an example implementation of an Authenticator controlling access to Model instances based on the currently specified namespace.  The [namespace can be set](http://code.google.com/appengine/docs/python/multitenancy/multitenancy.html#Setting_the_Current_Namespace) either globally or on a per-user basis (e.g. in a custom Authenticator).

You would utilize this Authorizer by setting it on the Dispatcher:
```
rest.Dispatcher.authorizer = NamespaceAuthorizer()
```

# NamespaceAuthorizer #

```
from google.appengine.api import namespace_manager


class NamespaceAuthorizer(rest.Authorizer):

    def can_read(self, dispatcher, model):
        if(model.key().namespace() != namespace_manager.get_namespace()):
            dispatcher.not_found()

    def filter_read(self, dispatcher, models):
        return self.filter_models(models)

    def can_write(self, dispatcher, model, is_replace):
        if(model.is_saved() and (model.key().namespace() != namespace_manager.get_namespace())):
            dispatcher.not_found()

    def filter_write(self, dispatcher, models, is_replace):
        return self.filter_models(models)

    def can_delete(self, dispatcher, model_type, model_key):
        if(model_key.namespace() != namespace_manager.get_namespace()):
            dispatcher.not_found()

    def filter_models(self, models):
        cur_ns = namespace_manager.get_namespace()
        models[:] = [model for model in models if model.key().namespace() == cur_ns]
        return models
```
