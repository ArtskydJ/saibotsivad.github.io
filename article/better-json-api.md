---
title: How I use JSON API
description: I like the JSON-API specs a lot, but there are a couple spots to shore up. This is my take on it.
date: 2018-03-20 21:17
layout: article
published: true
---

I've worked on and designed many APIs over the years, mostly for small start-up type
of companies, and many projects seem like they start their API design with an "we try
to be consistent but anything goes" sort of approach.

As a project matures, inevitably the developers want to be more consistent in their
API design. Smart engineers will look for pre-existing design work, so they don't
have to reinvent everything, and many engineers settle on the
[JSON-API spec](http://jsonapi.org/).

I think this spec is pretty great, but in this article I'm not going to try to convince
you how awesome it is, I'm just going to tell you about how we use the JSON-API spec at
[From Now On](http://from-now-on.com/) (the place I'm currently working), and how we
address one particular shortcoming.

## General

In normal REST parlance, a call to something like `/things` is a call to the things
collection. Collection names are always plural, e.g. `/things` and never `/thing`.

Strictly speaking, the JSON-API spec (and numerous other people all over the internet)
do not support a REST route that creates or modifies multiple resource in one HTTP
request. I have found that in practice this is not a sustainable approach. You can't
expose a money-making API that doesn't allow some sort of batching.

Our goal was to modify the JSON-API spec to support creating/modifying multiple
requests in one HTTP request, but in a way that feels "right" and "intuitive" to
the original spec.

After many hours of discussion and iteration, we have mostly settled on the following
routes for a given `/things` collection:

### The Collection

These follow the JSON-API spec as normal:

* `GET /things`	Returns an array of one or more `thing` objects, potentially limited
	or modified by query parameters.
* `DELETE /things` This route is not supported.

> Note: Although deleting multiple resources in one request is not supported, we have
> not found this to be an issue, because we don't support completely destroying a
> resource. Instead, a resource is tagged with something like `objectState = 'DELETED'`
> and in our database queries we exclude resources with that type. In this way, to
> the user the resource is deleted, but is persisted in the database until periodic
> cleanup. This makes debugging much easier, and also means you can delete multiple
> resources by issuing a `PATCH`. (Discussed below.)

These routes stray from the JSON-API spec in one small but powerful way: they all take
a single `thing` object _or_ an array of `thing` objects on the `data` property:

* `POST /things` Creates the resource(s), returning the list of created resource(s).
* `PUT /things`	Replaces the resource(s), returning the list of modified resource(s).
* `PATCH /things` Applies a change to the resource(s), returning the list of modified resource(s).

Since the `data` object of the `/things` POST, PUT, and PATCH must be a
[resource object](http://jsonapi.org/format/#document-resource-objects),
it is safe to overload the `data` property: if the type of `data` is an object, it
is either an invalid object or a _single_ `thing` object, and if `data` is an array
it is either an invalid request or a _list_ of `thing` objects.

### Individual Items

These follow the JSON-API spec as normal:

* `GET /things/:thingId` Fetch a single `thing` object, potentially including
	other related resources.
* `POST /things/:thingId` This route is not supported.
* `PUT /things/:thingId` Takes an already existing `thign` and replaces that
	resource, returning the modified `thing` object.
* `PATCH /things/:thingId` Takes an already existing `thign` and applies a change
	to that resource, returning the modified `thing` object.
* `DELETE /things/:thingId` Destroy an existing `thing` from the database.

### Object Structure

Technically the JSON-API spec allows for fields like `meta`, `links`, etc. to
be placed at the root, like this:

```json
{
	"data": {
		"id": "thing-01",
		"type": "thing",
		"attributes": {
			"name": "John Jacob Jingleheimerschmidt"
		}
	},
	"meta": {
		"created": "2017-08-23T17:45:13Z"
	}
}
```

Or inside the `data` object, like this:

```json
{
	"data": {
		"id": "thing-01",
		"type": "thing",
		"attributes": {
			"name": "John Jacob Jingleheimerschmidt"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	}
}
```

However, for consistency we have found that normalizing requests/responses
to the second one makes more sense, so that we can more carefully overload
the `data` property for multiple resources.

Apart from that, strictly speaking if you make a request or receive a response
containing multiple resources and there are `meta`/etc. fields at the root
of the object, the only reasonable conclusion is that the `meta`/etc. fields
are metadata for _all_ resources in that request.

## Examples!

Examples make the world go around, so let's get started!

In these examples I'll be describing data related to sports, which is what we
do at [From Now On](http://from-now-on.com/).

### `GET /coaches` fetch list of resources

Fetching resources follows the JSON-API spec pretty strictly, with the restriction
of the data structure described above.

To fetch some coach objects you might do:

```
GET /api/secure/coaches?page[number]=2&page[limit]=2
```

And the response object might look something like this:

```
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	}],
	"links": {
		"prev": "http://site.com/api/secure/coaches?page[number]=1&page[limit]=2",
		"next": "http://site.com/api/secure/coaches?page[number]=3&page[limit]=2"
	}
}
```

### `POST /coaches` create a single resource

To create a single `coach` object associated with a team, you might do:

```
POST /api/secure/coaches
{
	"data": {
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		}
	}
}
```

The response would look pretty much the same as the request, but would
include an `id` on the created resource, and may include other information
such as a created date:

```
{
	"data": {
		"id": "f95d88cf-17d6-44fc-be6a-4f215b117573",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	}
}
```

### `POST /coaches` create multiple resources

To create multiple `coach` objects, you might do:

```
POST /api/secure/coaches
{
	"data": [{
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		}
	},{
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker"
		}
	}]
}
```

Again, the response would look pretty much the same as the request,
except with an `id` added to each resource, and potentially other
information included:

```
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	}]
}
```

### `PUT /coaches` replace a single resource

Updating with a PUT _replaces_ the existing resource. In practice, to avoid
collisions, I have personally found it necessary to include some sort of field for
[optimistic concurrency](https://en.wikipedia.org/wiki/Optimistic_concurrency_control).

If your data structure uses a "last updated" property, that usually
works well. That property goes in the `meta` property.

So to replace a single `coach` object, you might do:

```
PUT /api/secure/coaches
{
	"data": {
		"id": "f95d88cf-17d6-44fc-be6a-4f215b117573",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh",
			"score": 9001
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	}
}
```

And the response would look pretty much the same as the request, but
with additional/updated information in the `meta` object:

```
{
	"data": {
		"id": "f95d88cf-17d6-44fc-be6a-4f215b117573",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh",
			"score": 9001
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	}
}
```

### `PUT /coaches` replace multiple resources

To _replace_ multiple `coach` objects, you do the same:

```
PUT /api/secure/coaches
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh",
			"videoUrl": "https://www.youtube.com/watch?v=571BuZeeQjE"
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker",
			"videoUrl": "https://www.youtube.com/watch?v=Y9Omtvdfh2U"
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	}]
}
```

Again, the response would look pretty much the same as the request, except you
might update the `meta` properties, if that's in your data structure:

```
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	}]
}
```

### `PATCH /coaches` diff a single resource

Updating with a PATCH changes specific properties of an existing resource, according
to some pretty straightforward rules:

* If the PATCH resource object specifies a property, it is replaced.
* If the PATCH resource sets the property to `null`, it is deleted.
* In all other cases, all properties are left unchanged with a PATCH.

(Again, placing the optimistic concurrency control property in the `meta` object.)

So if you wanted to update the `score` on a single `coach` object, the
request would look something like this:

```
PATCH /api/secure/coaches
{
	"data": {
		"id": "f95d88cf-17d6-44fc-be6a-4f215b117573",
		"type": "coach",
		"attributes": {
			"score": 73
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	}
}
```

The response would include the _full_ resource object, after that diff
has been applied:

```
{
	"data": {
		"id": "f95d88cf-17d6-44fc-be6a-4f215b117573",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh",
			"score": 73
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	}
}
```

### `PATCH /coaches` diff multiple resources

To PATCH multiple `coach` objects, you might do:

```
PUT /api/secure/coaches
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"score": 73
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"score": 71
		},
		"meta": {
			"updated": "2017-08-23T17:45:13Z"
		}
	}]
}
```

The response here would be an array of resource objects, much like the response
to a PUT request:

```
{
	"data": [{
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh",
			"score": 73
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	},{
		"id": "cb4b6047-1b05-49cf-8766-30d6797b553d",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Mike",
			"lastName": "Money Maker",
			"score": 71
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z",
			"updated": "2017-11-07T23:07:52Z"
		}
	}]
}
```

### `GET /coaches/:coachId` fetch single resource

To fetch a single coach object you might do:

```
GET /api/secure/coaches/4f2c2a04-9974-47ef-9e7a-d3b12b3849f7
```

And the response object might look something like this:

```
{
	"data": {
		"id": "4f2c2a04-9974-47ef-9e7a-d3b12b3849f7",
		"type": "coach",
		"attributes": {
			"teamId": "91cfa28f-6551-f009-2f6a-59a55679c940",
			"firstName": "Froggy",
			"lastName": "Fresh"
		},
		"meta": {
			"created": "2017-08-23T17:45:13Z"
		}
	}
}
```

### `PUT /coaches/:coachId` replace single resource

This route is identical to the `PUT /coaches` route, except that
it can only ever take a single object on the `data` property.

### `PATCH /coaches/:coachId` diff single resource

This route is identical to the `PATCH /coaches` route, except that
it can only ever take a single object on the `data` property.

### `DELETE /coaches/:coachId` destroy single resource

A request to this might be simply:

```
DELETE /api/secure/coaches/4f2c2a04-9974-47ef-9e7a-d3b12b3849f7
```

A successful response would be an empty body.

---

So that's it. That's how we make our API mostly compliant with the JSON-API
specs, except adding a little functionality that is incredibly powerful.

I don't get on Twitter these days, but you can hit me up via email at `me@` + this
domain if you have questions or comments, and I'll try to get back to you.