# JDS-GTM
JDS (OSEHRA Release of eHMP) with GT.M support

# Authors
Christopher Edwards, David Wicksell.

GT.M modifications - Sam Habiel.

# Configuration Instructions
Import all routines into Cache or GT.M. If you use Cache, make sure that the %ut namespace has been mapped to JDS.

Run the following to configure and start JDS for the first time, replacing 9080 with the port on which you want JDS to listen:

```
D SETUP^VPRJCONFIG
D GO^VPRJRCL(9080)
```

This will set up JDS with a single listener on port 9080. From this point forward, you can start JDS with the following:

```
D START^VPRJ
```

# Testing JDS

JDS provides a REST endpoint that will confirm that JDS has been successfully configured. In order to access it, launch the following URL:

[http://localhost:9080/ping]

If you have configured JDS to listen on a port other than 9080, modify the above URL (as well as any further example URLs in this documentation) accordingly.

# JDS Documentation

JDS has built-in documentation, accessible at [http://localhost:9080/documentation/index]

This documentation is formatted for [API Blueprint](https://apiblueprint.org), but the API Blueprint renderers I've tried will only render the top-level documentation. I believe there may have been some sort of script or external tooling in the original JDS project for gathering and rendering the documentation into user-friendly HTML. To my knowledge, this has been lost to time.

# GDS Usage Addendum

JDS includes a component called the Generic Data Store, or GDS. This is an extremely powerful JSON database that--unlike the rest of JDS--is usable for storing and searching any JSON data of any format, not limited (as in the other parts of JDS) to a specific JSON schema (such as patient data). As the aforementioned built-in documentation is light in its treatment of GDS, this section will provide a helpful guide for getting started with basic GDS CRUD, housekeeping, and monitoring endpoints.

## The GDS Store

To store data in GDS, you will need to create a *store*. A GDS store can contain any number of JSON documents, and you may create as many stores as you wish.

### Creating a Store

To create a store, you will need to decide upon a name by which the store will be referred in subsequent API calls, and create it with an HTTP *PUT* request, as follows:

```
http://localhost:9080/mystore
```

If you issue a PUT request for the above URL, assuming a new, correctly-configured instance of JDS, GDS will create a store named *mystore* and respond with an HTTP status code of 201.

### Getting Store Information

Once a store is created, you may query JDS for basic information about the store. Again, using our *mystore* example from above, we will issue an HTTP *GET* request to the following URL:

```
http://localhost:9080/mystore
```

GDS will return a JSON document containing basic information about mystore:

```
{
    "committed_update_seq": 0,
    "compact_running": false,
    "data_size": 0,
    "db_name": "mystore",
    "disk_format_version": 1,
    "disk_size": 0,
    "doc_count": 0,
    "doc_del_count": 0,
    "instance_start_time": 0,
    "purge_seq": 0,
    "update_seq": 0
}
```

Note that the *db_name* field contains a string denoting the name of the store; in this case, *mystore*. Another very useful field in this output is the *doc_count* field, containing the number of JSON documents (zero in the above example) being stored in *mystore*.

### Adding Documents to a Store

#### The GDS UID

Having created a proper GDS store, it would be useful to know how to add JSON documents to it. For this, we need to become familiar with the concept of a *UID*. This is simply a unique identifier by which GDS references a distinct document. In traditional uses of JDS, this would be a four-character string of capital letters, followed by a semicolon, followed by a number. However, GDS **does not** currently enforce this. You are generally free to choose a UID for GDS according to your own needs and preferences.

You are responsible for choosing and creating your own UIDs and ensuring their uniqueness, as they must be passed to the relevant endpoints as part of the request URL.

#### Inserting a New Document

To insert a document into GDS, you will make an HTTP *PUT* request to the following GDS endpoint, replacing *{storeName}* with the name of the store to which you wish to insert (i.e. *mystore*), and *{uid}* with the UID you have chosen or generated for the new document:

```
http://localhost:9080/{storeName}/{uid}
```

Your JSON document must be the body of your *PUT* request, and its *Content-Type* header must be *application/json*. If the insertion of your new document succeeds, GDS will respond with HTTP status code 200. GDS will add a *uid* field to the stored document, containing the GDS UID of the document, so you must take care that your document itself *does not* contain a *uid* field.

#### Retrieving an Existing Document

Retrieving an existing document uses a URL identical to that used for inserting a new document, but instead of using an HTTP *PUT* request, you will use a *GET* request. If the requested UID exists in the store, your document requested will be returned with a Content-Type of *application/json* and an HTTP status code of 200.

#### Updating an Existing Document

Again, updating an existing document uses the same URL as inserting and retrieving, but instead of using an HTTP *PUT* or *GET*, you will use an HTTP *PATCH* request. Returned HTTP status code on success will again be 200.

#### Deleting an Existing Document

Once again, we will use the same URL, but with an HTTP *DELETE* request. Also returns HTTP status code 200 on successful completion.

#### Retrieving Multiple Documents

GDS of course allows you to retrieve multiple documents from a store at one time. The most basic of these operations will return everything in the store: certainly not recommended for extremely large datasets! However, the URL (using HTTP *GET*, and returning status code 200 on success) to do this is as follows, replacing *{storeName}* with the name of the GDS store from which you wish to retrieve:


```
http://localhost:9080/{storeName}/
```

This will return an *application/json* document similar to the one below:

```
{
  "items": [
    {
      "uid": "CAR;1",
      "make": "Buick",
      "model": "Electra 225",
      "year": 1976
    },
    {
      "uid": "CAR;2",
      "make": "Lincoln",
      "model": "Continental Mark VI",
      "year": 1980
    },
    {
      "uid": "CAR;3",
      "make": "Lincoln",
      "model": "Town Car",
      "year": 1996
    }
  ]
}
```

##### Limiting the Result Count

Let's say we only want to retrieve the first 2 records from the store. We can pass the *limit* parameter in the *GET* API's query string, as follows:

```
http://localhost:9080/{storeName}/?limit=2
```

Using the same store as above, this query would return the following *application/json* document:

```
{
  "items": [
    {
      "uid": "CAR;1",
      "make": "Buick",
      "model": "Electra 225",
      "year": 1976
    },
    {
      "uid": "CAR;2",
      "make": "Lincoln",
      "model": "Continental Mark VI",
      "year": 1980
    }
  ]
}
```

#### Using Filters

Filters really give GDS much of its great power. In order to use them, we must pass a *filter* argument through the *GET* URL's query string:

```
http://localhost:9080/{storeName}/?filter={filterString}
```

Using the store referenced above, and leveraging GDS filters, let's retrieve all Lincoln automobiles manufactured after 1978. In order to do this, we will make the following *GET* request:

```
http://localhost:9080/{storeName}/?filter=and(eq("make","Lincoln"),gt("year",1978))
```

This will return the following *application/json* document:

```
{
  "items": [
    {
      "uid": "CAR;2",
      "make": "Lincoln",
      "model": "Continental Mark VI",
      "year": 1980
    }
  ]
}
```

##### Filters Reference

JDS accepts double-quoted strings and unquoted strings as filter arguments. Use double-quoted strings when the argument contains non-alphanumeric characters.

Separate operators with commas and/or spaces. There may be an optional trailing comma.

An implicit and group surrounds the entire filter query parameter.

###### Grouping Operators

Grouping operators can be nested. For example, or(and(eq("foo","bar"),like("baz","quux")),in("ping","pong"))

An implicit and group surrounds the entire filter query parameter. For example, the following 2 filter query parameters are equivalent:

filter=eq(make,"Lincoln"),eq(model,"Town Car")
filter=and(eq(make,"Lincoln"),eq(model,"Town Car"))

Grouping operators can have any number of operands.

|Grouping operators|
|------------------|
|and(operators)    |
|or(operators)     |
|not(operators)    |

###### Comparison Operators

Comparison operators take arguments inside parentheses. The first argument is always what JDS field to inspect and the second argument is what value to check for.
Arguments can't be empty. Lists are comma-separated items that start with [ and end with ]

NOTE: all JDS filters must be submitted to JDS URI-encoded. The following examples are not URI-encoded.

|Operator |Arguments                |Usage                                      |Example                              |
|---------|-------------------------|-------------------------------------------|-------------------------------------|
|eq       |(field,value)            |equals, exact comparison                   |eq(make,"Lincoln")                   |
|ne       |(field,value)            |not equals, exact comparison               |ne(make,"Lincoln")                   |
|in       |(field,[list,of,values]) |inside list, exact comparison with list    |in(make,["Lincoln","Buick"])         |
|nin      |(field,[list,of,values]) |not inside list, exact comparison with list|nin(make,["Hyundai","Toyota"])       |
|exists   |(field)                  |exists                                     |exists(make)                         |
|gt       |(field,value)            |greater than                               |gt(year,1978)                        |
|gte      |(field,value)            |greater than or equal                      |gte(year,1978)                       |
|lt       |(field,value)            |lesser than                                |lt(year,1978)                        |
|lte      |(field,value)            |lesser than or equal                       |lte(year,1978)                       |
|between  |(field,low,high)         |between exclusive                          |between(year,1978,1999)              |
|like     |(field,value)            |M pattern match, case sensitive            |like(model,"Continental%")           |
|ilike    |(field,value)            |M pattern match, case insensitive          |ilike(model,"CONTINENTAL%")          |

% is the wildcard character

# Documentation To-Do List

In the future, this documentation will also cover indexes and templates, as time permits.

