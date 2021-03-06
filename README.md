jsfs
====

A general-purpose, deduplicating filesystem with a REST interface, jsfs is intended to provide low-level filesystem functions for Javascript applications.  Additional functionality (private file indexes, token lockers, centralized authentication, etc.) are deliberately avoided here and will be implemented in a modular fashion on top of jsfs.

#STATUS
jsfs 2.x is a from-scratch rewrite of the original jsfs filesystem.  In its current form, it lacks the versioning and distributed functions of the original jsfs, however these features will return in later builds.  Additionally, the API for jsfs 2.x has been simplified while providing more functionality and an effort has been made to better comply with HTTP conventions.

#REQUIREMENTS
* Node.js

#CONFIGURATION
*  Clone this repository
*  Copy config.ex to config.js
*  Create the `blocks` directory (`mkdir blocks`)
*  Start the server (`node jsfs.js` foreman, pm2, etc.)

If you don't like storing the data in the same directory as the code (smart), edit config.js to change the location where data (blocks) are stored and restart jsfs for the changes to take effect. If you've already stored data in jsfs, you'll want to move the contents of the existing `blocks` directory to the new location or you'll loose the data.

#API

##HEADERS
jsfs uses several custom headers to control access to data and how it is stored.

###x-private
Set this header to `true` to mark files as private (they won't show up in directory listings).  *NOTE:* since private files don't show up in directory listings you'll have to keep track of the URL's yourself.  Additionally, to access private files a valid `x-access-token` header must be supplied with the `GET` request.

###x-encrypted
Set this header to `true` to encrypt stored data before it is stored on-disk.  Once enabled decryption happens automatically on `GET` requests and additional modifications via `PUT` will be encrypted as well. *NOTE:* encryption increases CPU utilization and reduces deduplication performance so use only when necissary.

###x-access-token
This header is used to authorize requests that modify existing files (`PUT`, `DELETE`).  The token is provided as part of the response when a new file is `POST`ed to a URL.  To perform further updates to a new file you'll need to keep track of this token.


##POST
Stores a new file at the specified URL.  If the file exists jsfs returns `405 method not allowed`.

###EXAMPLE
Request:

     curl -X POST --data-binary @Brinstar.mp3 "http://localhost:7302/music/Brinstar.mp3"

Response:
````
{
    "created": 1420309092678,
    "version": 0,
    "private": false,
    "encrypted": false,
    "access_token": "7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71",
    "content_type": "application/x-www-form-urlencoded",
    "file_size": 179186,
    "block_size": 1048576,
    "blocks": [
        "7653454f4c8c859bed57a44d59c6b536b0518192"
    ]
}
````

##GET
Retreives the file at the specified URL.  If the file does not exist a `404 not found` is returned.  If the URL ends with a trailing slash `/` a directory listing of non-private files stored at the specified location.

###EXAMPLE
Request (directory):

     curl http://localhost:7302/music/

Response:

````
[
    "Brinstar.mp3",
    "Opening.mp3",
    "Mother_Brain.mp3"
]
````

Request (file):

     curl -o Brinstar.mp3 http://localhost:730s/music/Brinstar.mp3

Response: 
The binary file is stored in new local file called `Brinstar.mp3`.

##PUT
Updates an existing file stored at the specified location.  This method requires authorization, so requests must include a valid `x-access-token` header for the specific file, otherwise `401 unauthorized` will be returned.  If the file does not exist `405 method not allowed` is returned.

###EXAMPLE
Request:

     curl -X PUT -H "x-access-token: 7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71"  --data-binary @Brinstar.mp3 "http://localhost:7302/music/Brinstar.mp3"

Result:
````
{
    "created": 1420309092678,
    "version": 0,
    "private": false,
    "encrypted": false,
    "access_token": "7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71",
    "content_type": "application/x-www-form-urlencoded",
    "file_size": 179186,
    "block_size": 1048576,
    "blocks": [
        "7653454f4c8c859bed57a44d59c6b536b0518192"
    ],
    "updated": 1420309368172
}
````

##DELETE
Removes the file at the specified URL.  This method requires authorization so requests must include a valid `x-access-token` header for the specified file.  If the token is not supplied or is invalid `401 unauthorized` is returend.  If the file does not exist `405 method not allowed` is returned.

###Example
Request:

     curl -X DELETE -H "x-access-token: 7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71" "http://localhost:7302/music/Brinstar.mp3"

Response
`HTTP 200` if sucessful.

##HEAD
Returns status and header information for the specified URL.

###Example
Request:

    curl -I "http://localhost:7302/music/Brinstar.mp3"

Response:
````
HTTP/1.1 200 OK
Content-Type: application/x-www-form-urlencoded
Content-Length: 179186
Date: Sat, 03 Jan 2015 18:24:53 GMT
Connection: keep-alive
````
