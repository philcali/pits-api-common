# Ophis Pteretos

[![CodeQL](https://github.com/philcali/ophis-py/actions/workflows/codeql-analysis.yml/badge.svg)](https://github.com/philcali/ophis-py/actions/workflows/codeql-analysis.yml)
[![Python package](https://github.com/philcali/ophis-py/actions/workflows/python-package.yml/badge.svg)](https://github.com/philcali/ophis-py/actions/workflows/python-package.yml)
[![codecov](https://codecov.io/gh/philcali/ophis-py/graph/badge.svg?token=B9HXJVMLCN)](https://codecov.io/gh/philcali/ophis-py)

This is common Python bindings for AWS based applications. The name
"Ophis Pteretos" is Greek for flying serpent, since... this library
is intended for the cloud. Supported in this library are helpers for

- AWS Gateway routing
- DynamoDB repositories

Using `ophis-py`, the heavy lifting for spinning up any sized project
behind an API Gateway frontend is reduced to a few lines of code.

## API Gateway Routing

The core of the API Gateway router is found in the `Router` class. In
typical applications, you would put this in your module `__init__.py`:

``` python
from ophis.router import Router


api = Router()
```

Now defining your route is as easy as:

``` python
@api.route('/greet')
def greet():
    return {
        'message': 'Hi, person'
    }
```

Adding path param evaluation:

``` python
@api.route('/greet/{name}')
def greet(name):
    return {
        'message': f'Hi, {name}'
    }
```

### Context

The library makes heavy use of `contextvars` for contextual evaluation.
Some default ones for `request`, `response`, and `app_context` exist for
reading context aware request and setting response data as needed. The
`app_context` is simply a container to evaluate dependencies to routes.

``` python
from ophis.globals import request, response, app_context

app_context.inject('thing_data', ThingDatabase())

@api.route('/things')
def list_things(thing_data):
    limit = int(request.queryparams.get('limit', '100'))
    next_token = request.queryparams.get('nextToken', None)
    if 0 <= limit > 100:
        response.status_code = 400
        return {
            'message': 'The limit cannot be less than 1 or greater than 100.'
        }
    resp = thing_data.query(
        request.account_id(),
        params=QueryParams(limit=limit, next_token=next_token))
    return {
        'items': resp.items,
        'next_token': resp.next_token
    }
```

## DynamoDB

The library allows a complete integration with DynamoDB using the `database` module.
The library does much of the heavy lifting, making an assumption around CRUD
(create, read, update, and delete) operations. A repository allows for adjacency
list building as well as multiple tables. An example one for `Things` might look
like:

``` python
from ophis.database import Repository

class ThingDatabase(Repository):
    def __init__(self, table=None):
        super.__init__(self, table=table, type="Things", fields_to_keys={
            'thingName': 'SK'
        })
```

With the above definition, the following operations can be performed:

``` python
import boto3
from ophis.database import QueryParams

ddb = boto3.resource('dynamodb')
table = ddb.Table('MyTableName')

thing_data = ThingDatabase(table=table)

# create, hashed by an AWS account ID for example
thing1 = thing_data.create('hashid', item={'thingName': 'MyThing1'})

# throws conflict
thing_data.create('hashid', item={'thingName': 'MyThing1'})

# update
thing_data.update('hashid', item={**thing1, 'newField': 'Updated'})

# read
thing_data.get('hashid', item_id=thing1['thingName'])

results = thing_data.list('hashid', query=QueryParams(limit=10))
for item in results.items:
    print(item)

results.next_token # can be used for pagination

# delete
thing_data.delete('hashid', item_id=thing1['thingName'])
```

Note that the pagination token is aes/gcm encrypted based on the last evaluated token
and the contents on the query call (for a multi-tenant application, this would include
the AWS account ID). This means that pagination tokens can be transmitted safely
as query parameters to read operations, by design.
