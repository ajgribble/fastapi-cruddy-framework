<a name="readme-top"></a>

<!-- PROJECT LOGO -->
<div align="center">
  <h2 align="center">FastAPI - Cruddy Framework</h2>
  <a href="https://github.com/mdconaway/fastapi-cruddy-framework">
    <img src="https://raw.githubusercontent.com/mdconaway/fastapi-cruddy-framework/master/logo.png" alt="Logo">
  </a>
  <br/>
</div>

<!-- ABOUT THE PROJECT -->
## About Cruddy Framework

[![Product Name Screen Shot][product-screenshot]](https://github.com/mdconaway/fastapi-cruddy-framework)

`fastapi-cruddy-framework` is a companion library to [FastAPI](https://fastapi.tiangolo.com/) designed to bring the development productivity of [Ruby on Rails](https://rubyonrails.org/), [Ember.js](https://emberjs.com/) or [Sails.js](https://sailsjs.com/) to the [FastAPI](https://fastapi.tiangolo.com/) ecosystem. Many of the design patterns base themselves on [Sails.js](https://sailsjs.com/) "policies," [Sails.js](https://sailsjs.com/) model lifecycle events, [sails-ember-rest](https://github.com/mdconaway/sails-ember-rest) automatic CRUD routing, and [Ember.js](https://emberjs.com/) [REST-Adapter](https://api.emberjs.com/ember-data/release/classes/RESTAdapter) feature sets. By default, data sent to and from the auto-magic CRUD routes are expected to conform to the [Ember.js](https://emberjs.com/) Rest Envelope and Linked-data relationship specification. This specification is highly readable for front-end developers, allows for an expressive over-the-wire query syntax, and embeds self-describing relationship URL links in each over-the-wire record to help data stores automatically generate requests to fetch or update related records. This library is still in an alpha/beta phase, so use at your own risk. All CRUD actions and relationship types are currently supported, though there may be unexpected bugs. Please report any bugs under "issues."


TODO: All the documentation and E2E tests. Maybe more comments. Maybe more features.

See the examples folder for a quick reference of high level setup. It currently contains a fully functional fastapi server which uses fastapi-cruddy-framework and the sqlite adapter. It even shows how to override incoming post data to do things like hash a user's password during initial registration using a simple drop-in policy function.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ABOUT THE PROJECT -->
## Installation

The fastapi-cruddy-framework module can be installed using poetry...

```
poetry add fastapi-cruddy-framework
```

Or pip.

```
pip install fastapi-cruddy-framework
```

After that, you can import and use all of the classes outlined below.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CLASSES -->
## Cruddy Exports/Imports

Cruddy-framework provides users the following classes and helper functions to scaffold out a project. (For recommended project structure, see the "examples" folder in the Github repo)

```python
# MASTER ROUTER GENERATOR
CreateRouterFromResources
# RESOURCE AND REGISTRY
Resource
ResourceRegistry
CruddyResourceRegistry
# CONTROLLER / CONTROLLER HELPERS
CruddyController
ControllerCongifurator
# REPOSITORY
AbstractRepository
# DATABASE ADAPTERS
BaseAdapter
SqliteAdapter
MysqlAdapter
PostgresqlAdapter
# TYPES / MODELS / SCHEMAS
T
UUID
RelationshipConfig
CruddyGenericModel
BulkDTO,
MetaObject
PageResponse
ResponseSchema
CruddyModel
CruddyIntIDModel
CruddyUUIDModel
ExampleUpdate
ExampleCreate
ExampleView
Example
# MODULE LOADER HELPERS
getModuleDir
getDirectoryModules
# HELPERS
pluralizer
uuid6
uuid7
get_pk
possible_id_types
lifecycle_types
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CreateRouterFromResources -->
### CreateRouterFromResources

This instance factory creates and returns a fully-wired fastapi `APIRouter` which sub-routes all `Resource` instances created in your project. Theoretically, you can create and mount multiple routers this way, but each router needs to be provided all of the `Resource` instances required to fully resolve the relationships it may care about. The recommended way to structure your project is to keep all resources required for a route set contained within a single folder. This factory is then provided the relative path to a resource set folder, starting with your application's `main` module, and will return the fully interconnected route set while also instantiating all of your resource modules. Typically, it is a good idea to import all of your router instances in your main module, as they will need to be "connected" to your fastapi server <i>WITHIN</i> the `startup` hook. This is critical, as the resource registry (discussed below) cannot fully resolve relationships until after SQLAlchemy is aware of all models. This occurs in-between launching your main module and the `startup` hook.

Example:
```python
import my_app
from fastapi_cruddy_framework import CreateRouterFromResources
from fastapi import FastAPI, APIRouter


my_router: APIRouter = CreateRouterFromResources(
    # (REQUIRED) application_module is of "ModuleType" type, and should be a pointer to your main app module
    application_module=my_app, 
    # (OPTIONAL) resource_path is of "str" type, and should specify a relative path from application_module
    # to the location of your "resources" that will be auto-loaded and bundled under this router
    # tree.
    resource_path="resources",
    # (OPTIONAL) common_resource_name is of "str" type, and should describe the common export value in each
    # resource file where the router factory can find your "Resource" instances. Use this if you want to name
    # all of your resource objects something other than "resource"
    common_resource_name="resource"
)

app = FastAPI(title="My App", version="1")

@app.on_event("startup")
async def bootstrap():
  app.include_router(my_router)

# fin!
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- Resource -->
### Resource

The `Resource` class is the fundamental building block of fastapi-cruddy-framework. Your resource instances define the union of your models, resource "controller" (which is a fastapi router with baked-in CRUD logic), business policies, repository abstraction layer, any resource lifecycle hook, and database adapter. Fortunately for you, the user, everything is essentially ready-to-go out of the box. Like [sails-ember-rest](https://github.com/mdconaway/sails-ember-rest) or [Ruby on Rails](https://rubyonrails.org/), you can now focus all of your development time on creating reusable policies (which contain your business logic that lies just above your CRUD endpoints), defining your models, and extending your resource controllers to add one-off actions like "login" or "change password". Lifecycle actions allow you to alter query configurations or record data before or after it is persisted to a database, or perform some other task before replying to the user. All of your resources should be loaded by the router factory (above) to ensure that relationships and routes are resolved in the correct order. Don't forget, <b>only plug the master router into your application in the fastapi `startup` hook!</b>


<b>Resource Nuances:</b>
* Defining your policies is done at definition time!
* Lifecycle actions occur immediately before and after any database interaction your CRUD controllers make
* Lifecycle actions passed into the Resource constructor to interact with your queries or data <b>MUST</b> be `async` functions.
* Policies are run in the exact order in which they are included in the `List` sent to the resource definition.
* `policies_universal` apply to ALL CRUD routes, and always run <i>BEFORE</i> action specific policy chains.
* Action specific policies run <i>AFTER</i> all `policies_universal` have resolved successfully.
* Each endpoint is protected by `policies_universal` + `policies_<action>`.
* One-to-Many and Many-to-Many sub-routes (like /users/{id}/posts) will be protected by the policy chain: `user.policies_universal` + `user.policies_get_one` + `posts.policies_get_many`. Security, security, security!
* Blocking user REST modification of certain relationships via the default CRUD controller is also done at definition time!
* `protected_relationships` is a `List[str]` with each string indicating a one-to-many or many-to-many relationship that should not be allowed to update via the default CRUD actions.
* You should define your application-wide adapter elsewhere and pass it into the resource instance.
* Resources cannot span different databases.


<b>Available Policy Chain Definitions:</b>
* `policies_universal`
* `policies_create`
* `policies_update`
* `policies_delete`
* `policies_get_one`
* `policies_get_many`


<b>Available ASYNC Lifecycle Hooks:</b>
* `lifecycle_before_create`
* `lifecycle_after_create`
* `lifecycle_before_update`
* `lifecycle_after_update`
* `lifecycle_before_delete`
* `lifecycle_after_delete`
* `lifecycle_before_get_one`
* `lifecycle_after_get_one`
* `lifecycle_before_get_all`
* `lifecycle_after_get_all`
* `lifecycle_before_set_relations`
* `lifecycle_after_set_relations`


<b>Available Relationship Blocks:</b>
* `protected_relationships`


<b>Updating Relationships:</b>
* You can update relationships via either CREATE or UPDATE actions against each base resource! 


As you will discover, your resource's create and update models will automatically gain "shadow" properties where one-to-many and many-to-many relationships exist. These properties expect a client to send a list of IDs that specify the foreign records that relate to the target record. So - if a user is a member of many groups, and a group can have many users, you could update the users in a group by sending a property `"users": [1,2,3,4,5]` within the `group` payload object you send to the `POST /groups` or `PATCH /groups` routes/actions. It will all be clear when you look at the SWAGGER docs generated for your API.

<b>Lifecycle hooks</b>

The following lifecycle hook methods, which can be defined in user-space code, receive the following information from fastapi-cruddy-framework:

`lifecycle_before_create` - Record without an ID. Values altered on this record in the lifecycle hook will be persisted to the DB.

`lifecycle_after_create` - Record with an ID, as returned from the database.

`lifecycle_before_update` - A key-values dictionary to be applied to the database, and the primary key id of the record which will be updated. Values altered in the dictionary will be applied to the DB update.

`lifecycle_after_update` - Record with an ID, as returned from the database

`lifecycle_before_delete` - Record with an ID, as returned from the database.

`lifecycle_after_delete` - Record with an ID, as returned from the database. This record no longer exists in the database.

`lifecycle_before_get_one` - A primary key value that will be used to fetch the record from the database.

`lifecycle_after_get_one` - Record with an ID, as returned from the database.

`lifecycle_before_get_all` - Recieves a query configuration object. Any user-space modifications to this object will impact the query made by fastapi-cruddy-framework. This method is also invoked when a foreign Resource queries a relationship that affects the Resource where you plug in this hook.

`lifecycle_after_get_all` - Receives a BulkDTO object, containing the database objects retrieved by a get_all query, as well as the query metadata. This method is also invoked when a foreign Resource queries a relationship that affects the Resource where you plug in this hook.

`lifecycle_before_set_relations` - Receives a relationship configuration object which containts information about the record id affected, the relationship being altered, and the new list of relations for this relationship type.
```
{
    "id": id, # The database id whos relationship are about to be altered (of your defined PK type)
    "relation": relation, # The relationship that is about to change (string)
    "relations": relations # An array of foreign ids that will now define this relationship (Framework will attempt to discard old relations)
}
```

`lifecycle_after_set_relations` - Receives a completed mapping of the affected relational change, which can be used to echo changes to other databases or services.

```
{
    "model": model, # The CruddyModel affected by this relationship change
    "relation_conf": relation_conf, # The configuration object from lifecycle_before_set_relations
    "relation_type": MANYTOMANY, # An SQL Alchemy relationship-type identifier (MANYTOMANY or ONETOMANY)
    "related_table": foreign_table, # The table that ultimately represents the far-side of this relationship (not the join table!)
    "related_field": field_name, # The field on the related_table that represents the far side of the relationship
    "updated_db_count": result # The number of records now in the database associated with this relationship. If the number is different than the length of relation_conf.relations, you probably have a non-nullable field on the far-side of this relationship.
}
```

Resource Definition Options (And Defaults!):
```python
id_type: Union[Type[int], Type[UUID]] = int,
# You SHOULD pass in 'adapter'
adapter: Union[BaseAdapter, SqliteAdapter, MysqlAdapter, PostgresqlAdapter, None] = None,
# The following adapter specific options will probably get removed. You don't need to pass them in.
# They exist solely in the event you are defining disparate resources and want the resources to 
# automatically build their own adapters. This is probably not a great idea.
adapter_type: Literal["mysql", "postgresql"] = "postgresql",
db_mode: Literal["memory", "file"] = "memory",
db_path: Union[str, None] = None,
connection_uri="",
pool_size=4,
max_overflow=64,
# link_prefix will be applied at the beginning of each relationship link on each record.
# This can help with things like sub-domains, or CORS with your API, and will allow you 
# to point your relationships endpoints at a complete URL. You could pass in something like
# https://api.mydomain.com, which would make a relationship link look like
# https://api.mydomain.com/resource/{id}/relationship
link_prefix="",
# Path specifies where this resource resides within the API. This is generated for you by
# default. Only change if you know what you are doing. Ember.js would expect a resource path
# to be the pluralized name of its base model. So a 'user' resource should be accessible at 
# '/users', and all of its sub-routes and actions are nested under that route.
path: str = None,
# The "tags" list corresponds with the fastapi "tags" list. You can alter this if needed.
# It is defined for you initially as the singular name of your resource model. User -> 'user'
tags: List[str] = None,
# The next four options are mandatory. 'create_model' specifies the inner schema that is 
# allowed to be sent to the create endpoint by a user. It will be auto-wrapped in a REST
# envelope schema. 'update_model ' specifies the inner schema that is allowed to be sent 
# to the update endpoint by a user. It too will be auto-wrappted in a REST envelope schema.
# 'resource_model' is your base model, which includes all possible fields of your model and
# has table=True specified. 'response_schema' defines the fields of your model to return
# to the client during all CRUD transactions. 'response_schema' will be wrapped in REST
# envelope in both single and many responses. Only 'resource_model' should have a table!!
# All of your models should descend from CruddyModel, which is a simple SQLModel class. 
resource_create_model: CruddyModel = ExampleCreate,
resource_update_model: CruddyModel = ExampleUpdate,
resource_model: CruddyModel = Example,
response_schema: CruddyModel = ExampleView,
# 'response_meta_schema' allows you to remap the "meta" values returned to the client for
# any paginated routes. You shouldn't NEED to change this, but you can if you want.
response_meta_schema: CruddyGenericModel = MetaObject,
# 'protected_relationships' will ban-hammer relationship fields specified from gaining
# an auto-magic update property. This will prevent users from creating or updating these
# relationships via the default CRUD actions. You will need to build other business logic
# to manage creating or changing protected relationships elsewhere in your application.
# Protected relationships will still be viewable at their designated GET routes.
protected_relationships: List[str] = [],
# The following options allow you to pass in your Sails.js-like policy chains, which will
# run before all of your endpoints (in the case of universal), or in front of only specific 
# endpoints that match the action specified. These policies can be used for nearly any purpose,
# from triggering other APIs and services, protecting endpoints to ensure only the correct
# users can alter data, or to intercept and even modify data before it gets to a default CRUD
# action! (Like hashing a user's password based on the plain-text password they send to register)
policies_universal: List[Callable] = [],
policies_create: List[Callable] = [],
policies_update: List[Callable] = [],
policies_delete: List[Callable] = [],
policies_get_one: List[Callable] = [],
policies_get_many: List[Callable] = [],
# The disable_<endpoint> options allow app developers to simply abort automatic generation of select 
# CRUD endpoints on the resource's controller. For instance, to make a write-once collection a 
# developercould set disable_update to True, which would cause the resource to abort building a route 
# for PATCH resource/{id}. Be aware of the overall impact of endpoints you totally disable!
disable_create: bool = False,
disable_update: bool = False,
disable_delete: bool = False,
disable_get_one: bool = False,
disable_get_many: bool = False,
# 'controller_extension' is the mount point for user-defined actions to-be-added to this resource's
# controller/router. Pass in your class definition and it will be instantiated at the appropriate
# time! See "CruddyController" example below!
controller_extension: CruddyController = None,
# The following lifecycle hooks can each recieve an async function which will be invoked before or 
# after the target lifecycle event. Generally, whatever values are passed to the lifecycle hook are
# alterable WITHIN the hook so that userspace code can alter the behavior of the lifecycle based on
# app level concerns. This allows apps to do things like: hash a user password, force certain 
# relationships to always exist, force "many" queries to obey sensible limits, commit log entries,
# send messages to queues for processing based on CRUD events, or generally handle unforseen 
# circumstances.
lifecycle_before_create: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_create: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_before_update: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_update: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_before_delete: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_delete: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_before_get_one: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_get_one: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_before_get_all: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_get_all: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_before_set_relations: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
lifecycle_after_set_relations: Optional[Callable[..., Coroutine[Any, Any, Any]]] = None,
```


Below is an example for creating a `user` resource. The best way to organize your app would be to place the definition for your user resource in a folder like `my_app/resources/user.py`, where the name of your application is `my_app`. As you saw earlier in the description for `CreateRouterFromResources` you would then load this user resource file by simply specifying `application_module=my_app` and `resource_path="resources"`. Your `fastapi-cruddy-framework` project would then auto-magically load your resource file(s), create dynamic routes to create, read, update, and delete this resource, and further create sub-routes within this resource to browse, query and update all of the relationships for your resource.

Example:
```python
from fastapi_cruddy_framework import Resource, UUID
from my_app.adapters import sqlite
from my_app.models.user import (
    User,
    UserCreate,
    UserUpdate,
    UserView,
)
from my_app.schemas.response import MetaObject
from my_app.policies.verify_session import verify_session
from my_app.policies.hash_user_password import (
    hash_user_password,
)


resource = Resource(
    id_type=UUID,
    adapter=sqlite,
    resource_update_model=UserUpdate,
    resource_create_model=UserCreate,
    resource_model=User,
    response_schema=UserView,
    response_meta_schema=MetaObject,
    protected_relationships=["posts"],
    policies_universal=[verify_session],
    policies_create=[hash_user_password]
)

# fin!
```

Easy, right?

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ResourceRegistry -->
### ResourceRegistry

The `ResourceRegistry` class should be invisible to the average user. There are no input parameters when creating a registry, and by default Cruddy defines its own library-internal registry. The registry exists to perform the following functions:

* Maintain a map of all resources available to `fastapi-cruddy-framework`
* Trigger `resolve` for all resources after SQL Alchemy finishes computing the relationship properties on each SQLModel.
* Plugin to the `Resource` class, so that each `Resource` you define can automatically call `ResourceRegistry.register()` when you define it. This is all "under the hood".


<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CruddyResourceRegistry -->
### CruddyResourceRegistry

The `CruddyResourceRegistry` is a framework-created instance of the `ResourceRegistry` class, exposed as an export so that application builders can acquire framework component instance references dynamically. It manages all of the resources and other subordinate components the active Cruddy framework is aware of. This framework internal registry instance is extremely helpful for looking up resource, controller, and model classes and instances <i>without</i> causing a litany of circular import issues in app code. Due to the asynchronous nature of how cruddy framework must initialize, application developers should use the `CruddyResourceRegistry` to "lookup" objects in any app code that needs to use a fully functional framework instance. What's a good example of where this is useful? Acquiring a resource's repository instance from within a policy so you can run database checks <i>before</i> a CRUD action occurs. There are many other applicable scenarios. See example, below.

```python
# This is an example policy file that could live in your project at policies/load_user_into_session.py
from fastapi import Request
from fastapi_cruddy_framework import AbstractRepository, CruddyResourceRegistry, BulkDTO

async def load_user_into_session(request: Request):
    some_value = "that identifies a user"
    user_repository: AbstractRepository = CruddyResourceRegistry.get_repository_by_name(model_name="User")
    repo_dto: BulkDTO = await user_repository.get_all(limit=1, where={"some_field": some_value})
    if len(repo_dto.data) == 0:
        raise Exception("User not found!")
    request.session.update({"user": repo_dto.data.pop()})
```

The available registry lookup function signatures are:

* `get_model_by_name(model_name: str) -> Union[CruddyModel, None]`
* `get_relationships_by_name(model_name: str) -> Union[Dict, None]`
* `get_resource_by_name(model_name: str) -> Union[Resource, None]`
* `get_repository_by_name(model_name: str) -> Union[AbstractRepository, None]`
* `get_controller_by_name(model_name: str) -> Union[APIRouter, None]`
* `get_controller_extension_by_name(model_name: str) -> Union[CruddyController, None]`


Make sure that the `model_name` string you pass to the registry EXACTLY mirrors the class name for your base table `CruddyModel`. So for a model with a class of `User` you would pass in `model_name="User"`. Pay attention to the capitalization!


<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CruddyController -->
### CruddyController

The `CruddyController` is a base class with a single method `setup` that applications can leverage to "extend" the default CRUD route controllers as needed. This class is needed to control application flow during the resolution boot cycle, and to allow you to share methods between resources like a mixin. Controllers should generally be placed within a project folder named "controllers". Controller classes can be imported in your resource file, and passed to the resource at definition time. Don't worry, the `Resource` class will create a controller instance for you!

Example `my_app/controllers/User.py`:
```python
from fastapi import Depends
from fastapi_cruddy_framework import CruddyController, CruddyGenericModel
from my_app.policies.verify_session import verify_session

class HelloSchema(CruddyGenericModel):
    hello: str = "world"

class UserController(CruddyController):
    def setup(self):
        # You can extend controller actions here!
        # You can also access:
        # self.resource
        # self.repository
        # self.adapter
        # self.controller
        @self.controller.get(
            "/hello", response_model=HelloSchema, dependencies=[Depends(verify_session)]
        )
        async def hello():
            return HelloSchema(hello="world")
```

Example Continued `my_app/resources/User.py`:
```python
from fastapi_cruddy_framework import Resource, UUID
from my_app.adapters import sqlite
from my_app.models.user import (
    User,
    UserCreate,
    UserUpdate,
    UserView,
)
from my_app.schemas.response import MetaObject
from my_app.controllers.user import UserController
from my_app.policies.verify_session import verify_session
from my_app.policies.hash_user_password import (
    hash_user_password,
)


resource = Resource(
    adapter=sqlite,
    response_schema=UserView,
    response_meta_schema=MetaObject,
    resource_update_model=UserUpdate,
    resource_create_model=UserCreate,
    resource_model=User,
    protected_relationships=["posts"],
    id_type=UUID,
    policies_universal=[verify_session],
    policies_create=[hash_user_password],
    controller_extension=UserController
)
```

Notice that you don't need to instantiate your controller!


`CruddyController` extension classes passed to the `Resource` definition will be `setup()` <i>BEFORE</i> the auto-generated CRUD routes but <i>AFTER</i> SQL Alchemy has resolved model relationships. This ensures that your user-defined routes receive priority for incoming HTTP requests. If extension classes were not loaded first, then the CRUD handlers would almost always intercept the incoming request first.


<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ControllerCongifurator -->
### ControllerCongifurator

The `ControllerCongifurator` is a configuration function invoked by the `Resource` class after SQL Alchemy has resolved all model relationships. You shouldn't need to interact with this function, but if you're a super advanced user, or wunderkind, maybe you will find a reason to need this. In essence, this function builds out all of the basic CRUD logic for a resource, after the resource has constructed a repository and generated the shadow schemas for your models. This is where your CRUD routes and sub-routes are auto-magically configured.

The controller/router configured by each of your `Resource` objects will allow the base resource or its relationships to be queried from the client via an arbitrarily complex `where` object (JSON encoded query parameter). 

Invalid attributes or ops are just dropped. (May change in the future)

Improvements that will be made in the near future:
1. Conditional table joins for relationships to...
2. Make resources searchable with joined relationships via dot notation in the `where` object!
3. Maybe throw an error if a bad search field is sent? (Will help UI devs)

Clients can build an arbitrarily deep query with a JSON dictionary, sent via a query parameter in a JSON object that generally contains all possible filter operators along with "and," "or," and "not" conditions. 

Field level and boolean operators begin with a * character. This will nearly always translate down to the sqlalchemy level, where it is up to the model class to determine what operations are possible on each model attribute. The top level query of a `where` object is an implicit AND. To do an OR, the base key of the search must be `*or`, as in the below examples:

`/resource?where={"*or":{"first_name":"bilbo","last_name":"baggins"}}`

`/resource/{id}/relationship?where={"*or":{"first_name":{"*contains":"bilbo"},"last_name":"baggins"}}`

`/resource?where={"*or":{"first_name":{"*endswith":"bilbo"},"last_name":"baggins","*and":{"email":{"*contains":"@"},"first_name":{"*contains":"helga"}}}}`

`/resource?where={"*or":{"first_name":{"*endswith":"bilbo"},"last_name":"baggins","*and":[{"email":{"*contains":"@"}},{"email":{"*contains":"helga"}}]}}`


The following query would be an implicit *and:

`/resource?where=[{"first_name":{"*endswith":"bilbo"}},{"last_name":"baggins"}]`


As would the following query:

`/resource/{id}/relationship?where={"first_name":{"*endswith":"bilbo"},"last_name":"baggins"}`


<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- AbstractRepository -->
## AbstractRepository

The `AbstractRepository` is a helpful way to interact with the data layer of your models. It contains all of the underlying functions that the `Resource` and `Controller` objects use to query, create, update, and delete your database information. Each `Resource` you define will automatically create an `AbstractRepository` instance that manages it. This can be accessed in your application at `your_resource_instance.repository`. The methods available to you via this repository instance are:

```python
# User functions accessible from any resource's 'AbstractRepository'
async def create(data: CruddyModel)

async def get_by_id(id: Union[UUID, int, str])

async def update(id: Union[UUID, int, str], data: CruddyModel)

async def delete(id: Union[UUID, int, str])

async def get_all(page: int = 1, limit: int = 10, columns: List[str] = None, sort: List[str] = None, where: Json = None)

async def get_all_relations(id: Union[UUID, int, str] = ..., relation: str = ..., relation_model: CruddyModel = ..., page: int = 1, limit: int = 10, columns: List[str] = None, sort: List[str] = None, where: Json = None)

async def set_many_many_relations(id: Union[UUID, int, str], relation: str = ..., relations: List[Union[UUID, int, str]] = ...)

async def set_one_many_relations(id: Union[UUID, int, str], relation: str = ..., relations: List[Union[UUID, int, str]] = ...)
```

Generally, these functions do about what you would expect them to do. More documentation will be added to describe their function soon. Please read nuances below, however, as it applies to how x-to-Many relationships are managed via the automatic CRUD routes.


<b>Important AbstractRepository Nuances</b>

* `set_many_many_relations` and `set_one_many_relations` both destroy and then re-create the x-to-Many relationships they target. If a `user` with the id of 1 was a member of `groups` 1, 2, and 3, then calling `await user_repository.set_many_many_relations(1, 'groups', [4,5,6])` would result in `user` 1 being a member of only groups 4,5, and 6 after execution. Client applications should be aware of this functionality, and always send ALL relationships that should still exist during any relational updates.


<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[product-screenshot]: https://raw.githubusercontent.com/mdconaway/fastapi-cruddy-framework/master/screenshot.png