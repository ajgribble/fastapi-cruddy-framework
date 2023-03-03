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

`fastapi-cruddy-framework` is a companion library to FastAPI designed to bring the development productivity of Ruby on Rails, Ember.js or Sails.js to the FastAPI ecosystem. Many of the design patterns base themselves on Sails.js "policies," sails-ember-rest automatic CRUD routing, and Ember.js REST-Adapter feature sets. By default, data sent to and from the auto-magic CRUD routes are expected to conform to the Ember.js Rest Envelope / Linked-data specification. This specification is highly readable for front-end developers, allows for an expressive over-the-wire query syntax, and embeds self-describing relationship URL links in each over-the-wire record to help data stores automatically generate requests to fetch or update related records. This library is still in an alpha/beta phase, so use at your own risk. All CRUD actions and relationship types are currently supported, though there may be unexpected bugs. Please report any bugs under "issues."


TODO: All the documentation. See the examples folder for a quick reference of high level setup. It currently contains a fully functional fastapi server which uses fastapi-cruddy-framework and the sqlite adapter. It even shows how to override incoming post data to do things like hash a user's password during initial registration using a simple drop-in policy function.

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
# CONTROLLER HELPERS
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
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

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

### Resource

The `Resource` class is the fundamental building block of fastapi-cruddy-framework. Your resource instances define the union of your models, resource "controller" (which is a fastapi router with baked-in CRUD logic), business policies, repository abstraction layer, and database adapter. Fortunately for you, the user, everything is essentially ready-to-go out of the box. Like sails-ember-rest or Ruby on Rails, you can now focus all of your development time on creating reusable policies (which contain your business logic that lies just above your CRUD endpoints), defining your models, and extending your resource controllers to add one-off actions like "login" or "change password". All of your resources should be loaded by the router factory (above) to ensure that relationships and routes are resolved in the correct order. Don't forget, <b>only plug the master router into your application in the fastapi `startup` hook!</b>


<b>Resource Nuances:</b>
* Defining your policies is done at definition time!
* Policies are run in the exact order in which they are included in the `List` sent to the resource definition.
* `policies_universal` apply to ALL CRUD routes, and always run <i>BEFORE</i> action specific policy chains.
* Action specific policies run <i>AFTER</i> all `policies_universal` have resolved successfully.
* Each endpoint is protected by `policies_universal` + `policies_<action>`.
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


<b>Available Relationship Blocks:</b>
* `protected_relationships`


<b>Updating Relationships</b>
* You can update relationships via either CREATE or UPDATE actions against each base resource! 

As you will discover, your resource's create and update models will automatically gain "shadow" properties where one-to-many and many-to-many relationships exist. These properties expect a client to sent a list of IDs that specify the foreign records that relate to the target record. So - if a user is a member of many groups, and a group can have many users, you could update the users in a group by sending a property `"users": [1,2,3,4,5]` within the `group` payload object you send to the `POST /groups` or `PATCH /groups` routes/actions. It will all be clear when you look at the SWAGGER docs generated for your API.


Resource Definition Options (And Defaults!):
```python
id_type: Union[Type[int], Type[UUID]] = int,
# You SHOULD pass in 'adapter'
adapter: Union[BaseAdapter, SqliteAdapter, MysqlAdapter, PostgresqlAdapter, None] = None,
# The following adapter specific options will probably get removed. You don't need to pass them in.
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
# The remaining options allow you to pass in your Sails.js-like policy chains, which will
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
    policies_create=[hash_user_password],
)

# fin!
```

Easy, right?

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[product-screenshot]: https://raw.githubusercontent.com/mdconaway/fastapi-cruddy-framework/master/screenshot.png