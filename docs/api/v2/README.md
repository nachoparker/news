# External API v2 (Draft)

The **News app** offers a RESTful API which can be used to sync folders, feeds and items. The API also supports [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) which means that you can access the API from your browser using JavaScript.

In addition, an updater API is exposed which enables API users to run feed updates in parallel using a REST API or ownCloud console API.

## API Stability Contract

The API level will **change** if the following occurs:

* a required HTTP request header is added
* a required request parameter is added
* a field of a response object is removed
* a field of a response object is changed to a different datatype
* an HTTP response header is removed
* an HTTP response header is changed to a different datatype
* the meaning of an API call changes (e.g. /sync will not sync any more but show a sync timestamp)

The API level will **not change** if:

* a new HTTP response header is added
* an optional new HTTP request header is added
* a new response parameter is added (e.g. each item gets a new field "something": 1)
* The order of the JSON attributes is changed on any level (e.g. "id":3 is not the first field anymore, but the last)

You have to design your app with these things in mind!:

* **Don't depend on the order** of object attributes. In JSON it does not matter where the object attribute is since you access the value by name, not by index
* **Don't limit your app to the currently available attributes**. New ones might be added. If you don't handle them, ignore them
* **Use a library to compare versions**, ideally one that uses semantic versioning

## Authentication
Because REST is stateless you have to re-send user and password each time you access the API. Therefore running ownCloud **with SSL is highly recommended** otherwise **everyone in your network can log your credentials**.

The base URL for all calls is:

    https://yourowncloud.com/index.php/apps/news/api/v2

All defined routes in the Specification are appended to this url. To access the sync for instance you'd use the following url:

    https://yourowncloud.com/index.php/apps/news/api/v2/sync

Credentials are passed as an HTTP header using [HTTP basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication#Client_side):

    Authorization: Basic $CREDENTIALS

where $CREDENTIALS is:

    base64(USER:PASSWORD)

This authentication/authorization method will be the recommended default until core provides an easy way to do OAuth

## Request Format
The required request headers are:
* **Accept**: application/json

Any request method except GET:
* **Content-Type**: application/json; charset=utf-8

Any route that allows caching:
* **If-None-Match**: an Etag, e.g. 6d82cbb050ddc7fa9cbb659014546e59. If no previous Etag is known, this header should be omitted

The request body is either passed in the URL in case of a **GET** request (e.g.: **?foo=bar&index=0**) or as JSON, e.g.:

```json
{
    "foo": "bar",
    "index": 0
}
```

## Response Format
The status codes are:
* **200**: Everything went fine
* **304**: In case the resource was not modified, contains no response body
* **403**: ownCloud Error: The provided authorization headers are invalid. No **error** object is available.
* **404**: ownCloud Error unless specified otherwise: The route can not be found. This can happen if the app is disabled or because of other reasons. No **error** object is available.
* **4xx**: There was an app related error, check the **error** object if specified
* **5xx**: ownCloud Error: A server error occurred. This can happen if the server is in maintenance mode or because of other reasons. No **error** object is available.

The response headers are:
* **Content-Type**: application/json; charset=utf-8
* **Etag**: A string containing a cache header of maximum length 64, e.g. 6d82cbb050ddc7fa9cbb659014546e59

The response body is a JSON structure that looks like this:

```js
{
    "data": {
        // payload is in here
    },
    // if an error occured
    "error": {
        "code": 1,  // an error code that is unique in combination with
                    // the HTTP status code to distinguish between multiple error types
        "message": "Folder exists already"  // a translated error message depending on the user's configured server locale
    }
}
```

## Security Guidelines
Read the following notes carefully to prevent being subject to security exploits:
* All string fields in a JSON response unless explicitly noted otherwise are provided in without sanitation. This means that if you do not escape it properly before rendering you will be vulnerable to [XSS](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29) attacks
* Basic Auth headers can easily be decrypted by anyone since base64 is an encoding, not an encryption. Therefore only send them if you are accessing an HTTPS website or display an easy to understand warning if the user chooses HTTP

## Syncing
All routes are given relative to the base API url, e.g.: **/sync** becomes  **https://yourowncloud.com/index.php/apps/news/api/v2/sync**

There are two usecases for syncing:
* **Initial sync**: the user does not have any data at all
* **Syncing local and remote changes**: the user has synced at least once and wants submit and receive changes

### Initial Sync
The intial sync happens when a user adds an ownCloud account in your app. In that case you want to download all folders, feeds and unread/starred items. To do this, make the following request:

* **Method**: GET
* **Route**: /sync
* **HTTP headers**:
  * **Accept: "application/json"**
  * Authorization headers

This will return the following status codes:
* **200**: Successully synced
* Other ownCloud errors, see **Response Format**

and the following HTTP headers:
* **Content-Type**: application/json; charset=utf-8
* **Etag**: A string containing a cache header of maximum size 64, e.g. 6d82cbb050ddc7fa9cbb659014546e59

and the following request body:
```js
{
    "data": {
        "folders": [{
            "id": 3,
            "name": "funny stuff"
        }, /* etc */],
        "feeds": [{
            "id": 4,
            "name": "The Oatmeal - Comics, Quizzes, & Stories",
            "faviconLink": "http://theoatmeal.com/favicon.ico",
            "folderId": 3,
            "ordering": 0,
            "isPinned": true,
            "error": {
                "code": 1,
                "message": ""
            }
        }, /* etc */],
        "items": [{
            "id": 5,
            "url": "http://grulja.wordpress.com/2013/04/29/plasma-nm-after-the-solid-sprint/",
            "title": "Plasma-nm after the solid sprint",
            "author": "Jan Grulich (grulja)",
            "publishedAt": "2005-08-15T15:52:01+0000",
            "updatedAt": "2005-08-15T15:52:01+0000",
            "enclosures": [{
                "mime": "video/webm",
                "url": "http://video.webmfiles.org/elephants-dream.webm"
            }],
            "body": "<p>At first I have to say...</p>",
            "feedId": 4,
            "isUnread": true,
            "isStarred": true,
            "fingerprint": "08ffbcf94bd95a1faa6e9e799cc29054"
        }, /* etc */]
    }
}
```

Each resource's (aka folder/feed/item) attributes are explained in separate chapters.

**Important**: Read the **Security Guidelines**

### Sync Local And Remote Changes
After the initial sync the app has all folders, feeds and items. Now you want to push changes and retrieve updates from the server. To do this, make the following request:

* **Method**: POST
* **Route**: /sync
* **HTTP headers**:
  * **Content-Type: "application/json; charset=utf-8"**
  * **Accept: "application/json"**
  * **If-None-Match: "6d82cbb050ddc7fa9cbb659014546e59"** (Etag from the previous request to the /sync route)
  * Authorization headers

with the following request body:

```js
{
    "items": [{
            // read and starred
            "id": 5,
            "isStarred": false,
            "isRead": true,
            "fingerprint": "08ffbcf94bd95a1faa6e9e799cc29054"
        }, {
            // only read
            "id": 6,
            "isRead": true,
            "fingerprint": "09ffbcf94bd95a1faa6e9e799cc29054"
        }, {
            // only starred
            "id": 7,
            "isStarred": false,
            "fingerprint": "18ffbcf94bd95a1faa6e9e799cc29054"
    },/* etc */]
}
```

If no items have been read or starred, simply leave the **items** array empty, e.g.:

```js
{
    "items": []
}
```

The response will be the same as in the initial sync except if an item's fingerprint is the same as in the database: This means that the contents of the item did not change and in order to preserve bandwidth, only the status is added to the item, e.g.:

```js
{
    "data": {
        "folders": [/* new or updated folders here */],
        "feeds": [/* new or updated feeds here */],
        "items": [{
                "id": 5,
                "isStarred": false,
                "isRead": true,
        }, /* etc */]
    }
}
```
However if an item did change, the full item will be sent to the client

If the HTTP status code was either in the **4xx** or **5xx** range, the exact same request needs to be retried when doing the next sync.


**Important**: Read the **Security Guidelines**


## Folders
Folders are represented using the following data structure:
```json
{
    "id": 3,
    "name": "funny stuff"
}
```

The attributes mean the following:
* **id**: 64bit Integer, id
* **name**: Abitrary long text, folder's name

### Deleting A Folder
To delete a folder, use the following request:
* **Method**: DELETE
* **Route**: /folders/{id}
* **Route Parameters**:
  * **{id}**: folder's id

The following response is being returned:

Status codes:
* **200**: Folder was deleted successfully
* **404**: Folder with given id was not found, no error object
* Other ownCloud errors, see **Response Format**

In case of an HTTP 200, the deleted folder is returned in full in the response, e.g.:

```json
{
    "data": {
        "folder": {
            "id": 3,
            "name": "funny stuff"
        }
    }
}
```
### Creating A Folder
To create a folder, use the following request:
* **Method**: POST
* **Route**: /folders

with the following request body:
```json
{
    "name": "Folder name"
}
```

The following response is being returned:

Status codes:
* **200**: Folder was created successfully
* **400**: Folder creation error, check the error object:
  * **code**: 1 folder name is empty
* **409**: Folder with given name exists already
* Other ownCloud errors, see **Response Format**

In case of an HTTP 200 or 409, the created or already existing folder is returned in full in the response, e.g.:

```json
{
    "data": {
        "folder": {
            "id": 3,
            "name": "funny stuff"
        }
    }
}
```
### Changing A Folder
The following attributes can be changed on the folder:
* **name**

To change any number of attributes on a folder, use the following request and provide as much attributes that can be changed as you want:
* **Method**: PATCH
* **Route**: /folders/{id}
* **Route Parameters**:
  * **{id}**: folder's id

with the following request body:
```json
{
    "name": "New folder name"
}
```

The following response is being returned:

Status codes:
* **200**: Folder was created successfully
* **400**: Folder creation error, check the error object:
  * **code**: 1 folder name is empty
* **409**: Folder with given name exists already
* Other ownCloud errors, see **Response Format**

In case of an HTTP 200 or 409, the changed or already existing folder is returned in full in the response, e.g.:

```json
{
    "data": {
        "folder": {
            "id": 3,
            "name": "funny stuff"
        }
    }
}
```


## Feeds
Feeds are represented using the following data structure:

```json
{
    "id": 4,
    "name": "The Oatmeal - Comics, Quizzes, & Stories",
    "faviconLink": "http://theoatmeal.com/favicon.ico",
    "folderId": 3,
    "ordering": 0,
    "isPinned": true,
    "error": {
        "code": 1,
        "message": ""
    }
}
```

The attributes mean the following:
* **id**: 64bit Integer, id
* **name**: Abitrary long text, feed's name
* **faviconLink**: Abitrary long text, feed's favicon location, **null** if not found
* **folderId**: 64bit Integer, the feed's folder or **0** in case no folder is specified
* **ordering**: 64bit Integer, overrides the feed's default ordering:
  * **0**: Default ordering
  * **1**: Oldest first ordering
  * **2**: Newest first ordering
* **isPinned**: Boolean, Used to list certain feeds before others. Feeds are first ordered by their **isPinned** value (true before false) and then by their name in alphabetical order
* **error**: error object, only present if an error occurred:
  * **code**: The error code:
    * **1**: Error occured during feed update
  * **message**: Translated error message depending on the user's configured server locale


### Deleting A Feed
To delete a feed, use the following request:
* **Method**: DELETE
* **Route**: /feeds/{id}
* **Route Parameters**:
  * **{id}**: feed's id

The following response is being returned:

Status codes:
* **200**: Feed was deleted successfully
* **404**: Feed with given id was not found, no error object
* Other ownCloud errors, see **Response Format**


In case of an HTTP 200, the deleted feed is returned in full in the response, e.g.:

```json
{
    "data": {
        "feed": {
            "id": 4,
            "name": "The Oatmeal - Comics, Quizzes, & Stories",
            "faviconLink": "http://theoatmeal.com/favicon.ico",
            "folderId": 3,
            "ordering": 0,
            "isPinned": true,
            "error": {
                "code": 1,
                "message": ""
            }
        }
    }
}
```

### Creating A feed
TBD
### Changing A Feed
TBD

## Items

Items can either be in the format of:
```json
{
    "id": 5,
    "url": "http://grulja.wordpress.com/2013/04/29/plasma-nm-after-the-solid-sprint/",
    "title": "Plasma-nm after the solid sprint",
    "author": "Jan Grulich (grulja)",
    "publishedAt": "2005-08-15T15:52:01+0000",
    "updatedAt": "2005-08-15T15:52:01+0000",
    "enclosures": [{
        "mime": "video/webm",
        "url": "http://video.webmfiles.org/elephants-dream.webm"
    }],
    "body": "<p>At first I have to say...</p>",
    "feedId": 4,
    "isUnread": true,
    "isStarred": true,
    "fingerprint": "08ffbcf94bd95a1faa6e9e799cc29054"
}
```

or if they did not change in the following format:

```json
{
    "id": 5,
    "isUnread": true,
    "isStarred": true
}
```

The attributes mean the following:
* **id**: 64bit Integer, id
* **url**: Abitrary long text, location of the online resource
* **title**: Abitrary long text, item's title
* **author**: Abitrary long text, name of the author/authors
* **publishedAt**: String representing an ISO 8601 DateTime object, when the item was published
* **updateddAt**: String representing an ISO 8601 DateTime object, when the item was updated
* **enclosures**: A list of enclosure objects,
  * **mime**: Mimetype
  * **url**: Abitrary long text, location of the enclosure
* **body**: Abitrary long text, **sanitized (meaning: does not have to be escape)**, contains the item's content
* **feedId**: 64bit Integer, the item's feed it belongs to
* **isUnread**: Boolean, true if unread, false if read
* **isStarred**: Boolean, true if starred, false if not starred
* **fingerprint**: 64 ASCII characters, hash that is used to determine if an item is the same as an other one. The following behavior should be implemented:
  * Items in a stream (e.g. All items, folders, feeds) should be filtered so that no item with the same fingerprint is present.
  * When marking an item read, all items with the same fingerprint should also be marked as read.


## Updater
Instead of using the built in, slow cron updater you can use the parallel update API to update feeds. The API can be accessed through REST or ownCloud console API.

The API should be used in the following way:

* Clean up before the update
* Get all feeds and user ids
* For each feed and user id, run the update
* Clean up after the update

The reference [implementation in Python](https://github.com/owncloud/news-updater) should give you a good idea how to design your own updater.

If the REST API is used, Authorization is required via Basic Auth and the user needs to be in the admin group.
If the ownCloud console API is used, no authorization is required.

### Clean Up Before Update
This is used to clean up the database. It deletes folders and feeds that are marked for deletion.

**Console API**:

    php -f /path/to/owncloud/occ news:updater:before-update

**REST API**:

* **Method**: GET
* **Route**: /updater/before-update

### Get All Feeds And User Ids
This call returns pairs of feed ids and user ids.

**Console API**:

    php -f /path/to/owncloud/occ news:updater:all-feeds

**REST API**:

* **Method**: GET
* **Route**: /updater/all-feeds

Both APIs will return the following response body or terminal output:

```js
{
    "data": {
        "feeds": [{
          "id": 3,
          "userId": "john"
        }, /* etc */]
    }
}
```

### Update A User's Feed
After all feed ids and user ids are known, feeds can be updated in parallel.

**Console API**:
* **Positional Parameters**:
  * **{feedId}**: the feed's id
  * **{userId}**: the user's id


    php -f /path/to/owncloud/occ news:updater:update-feed {feedId} {userId}

**REST API**:

* **Method**: GET
* **Route**: /updater/update-feed?feedId={feedId}&userId={userId}
* **Route Parameters**:
  * **{feedId}**: the feed's id
  * **{userId}**: the user's id

### Clean Up After Update
This is used to clean up the database. It removes old read articles which are not starred.

**Console API**:

    php -f /path/to/owncloud/occ news:updater:after-update

**REST API**:

* **Method**: GET
* **Route**: /updater/after-update

## Meta Data
The retrieve meta data about the app, use the following request:

* **Method**: GET
* **Route**: /

The following response is being returned:

Status codes:
* **200**: Meta data accessed successfully
* Other ownCloud errors, see **Response Format**


In case of an HTTP 200, the the following response is returned:

```json
{
    "version": "9.0.0",
    "issues": {
        "improperlyConfiguredCron": false
    },
    "user": {
        "userId": "john",
        "displayName": "John Doe",
        "avatar": {
            "data": "asdiufadfasdfjlkjlkjljdfdf",
            "mime": "image/jpeg"
        }
    }
}
```

The attributes mean the following:
* **version**: Abitrary long text, News app version
* **issues**: An object containing a dictionary of issues which need to be displayed to the user:
  * **improperlyConfiguredCron**: Boolean, if true this means that no feed updates are run on the server because the updater is misconfigured
* **user**: user information:
  * **userId**: Abitrary long text, the login name
  * **displayName**: Abitrary long text, the full name like it's displayed in the web interface
  * **avatar**: an avatar object, null if none is set
    * **data**: Abitrary long text, the user's image encoded as base64
    * **mime**: Abitrary long text, avatar mimetype