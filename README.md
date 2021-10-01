# node-red-web-server-examples #

This collection of [Node-RED](https://nodered.org/) examples (with a matching [Postman](https://www.postman.com/) collection for testing) is mainly intended for my students, but parts of it may also be of more general interest.

It is the continuation of another set of [HTTP(S)-related examples](https://github.com/rozek/node-red-http-server-examples) and explains how to implement some basic file-based web servers with or without custom HTTP endpoints and with or without cookie-based authorization. The next step in this series is a collection of examples for [basic REST services](https://github.com/rozek/node-red-rest-server-examples), also built with Node-RED.

For this series, it is assumed that the reader already installed Node-RED (as described in [Getting Started](https://nodered.org/docs/getting-started/)), optionally secured the editor (as shown in [Securing Node-RED](https://nodered.org/docs/user-guide/runtime/securing-node-red)) and started using it (as explained in [Creating your first flow](https://nodered.org/docs/tutorials/first-flow))

> Nota bene: this work is currently in progress, do not expect it to be finished before midth of October 2021

### Required Extensions ###

For these examples to be run, it is necessary to install the following extension:

* [node-red-contrib-reusable-flows](https://github.com/ollixx/node-red-contrib-reusable-flows)<br>"Reusable Flows" allow multiply needed flows to be defined once and then invoked from multiple places

## Examples ##

All example specifications are stored in JSON format and may easily be imported into a Node-RED workspace. Preferrably, you should open a separate tab and insert them there.

To test the examples, a [Postman collection](https://raw.githubusercontent.com/rozek/node-red-web-server-examples/main/PostmanCollection.json) is included, which may easily be imported into a running [Postman](https://www.postman.com/) instance. After the import, you should open the collection's "Variables" section and set the `BaseURL` to the base URL of your NodeRED instance (by default, it is set to `127.0.0.1:1880`, which should work out-of-the-box for most Node-RED installations). If your Node-RED instance has been configured to require basic authentication, you should also set the variables `Username` and `Password`)

Alternatively, other tools like [cURL](https://curl.se/) may be used as well.

### Simple (and open) Web Server ###

The first example in this series is a simple web server delivering files found in a configured directory.

> For this example to work, please copy file `FileTypeMappings.json` and folder `public` into the working directory of your Node-RED instance

The server is principally "open" for everybody as not extra authorization is needed to get responses for sent requests.

A simple path normalization asserts that clients may only request files from within the configured directory and may not inspect the whole file system.

If the requested path is that of a directory, the client is redirected to a file "index.html" in that directory. If no file exists at the requested path, a response with the status code 404 (Not Found) is sent back.

The server recognizes many different file types by looking at their type suffixes and sets the "Content-Type" header accordingly (see file [FileTypeMappings.json](https://raw.githubusercontent.com/rozek/node-red-web-server-examples/main/FileTypeMappings.json) for the complete set) 

Unforeseen errors are caught by a generic "catch" node and result in a response with status code 500 ("Internal Server Error")

Node-RED's debug console serves as a kind of "logger" and informs about any served files or any errors that occur.

![](examples/simple-web-server.png)

Simply import the [example from this repo](examples/simple-web-server.json) into your Node-RED workspace and deploy. You may then test your server either by navigating your browser to `{{BaseURL}}/simple-web-server` or using the included Postman collection. Currently, the `public` folder contains two files only, but you are free to add as many files as you like.

> Please note: usually, it is not a good idea to use Node-RED just for static file serving - there are lots of other solutions which are even simpler to install and configure and are much more performant and secure than this example. If, however, a server with custom endpoints is needed which *also* serves some static files, this example may be quite practical.

### Custom (and still open) Web Server ###

As mentioned before, Node-RED becomes interesting as soon as servers with custom endpoints will be needed. The following example demonstrates such a scenario: in addition to the part mentioned before which serves static files only, three new endpoints are introduced that *compute* responses rather than load them from a file system:

* `{{BaseURL}}/custom-web-server/server-time` simply sends the current server time in ISO format,
* `{{BaseURL}}/custom-web-server/echo` responds with a JSON object containing important parts of the `msg.req` object for the received request,
* `{{BaseURL}}/custom-web-server/client-address` determines the IP address of the connected client and returns it

Because of the way in which these endpoints are recognized, more endpoints may easily be added (or unwanted ones removed). Static file serving is still included and starts if no custom endpoint was found - the flow needed is the same as in the previous example, which is why it has been connected using a "link" node: in this way, several implementations may use the same sequence of nodes without having to copy them.

> For this example to work, please copy file `FileTypeMappings.json` and folder `public` into the working directory of your Node-RED instance

![](examples/custom-web-server.png)

The following part is already known from the previous example:

![](examples/serving-files.png)

To test this server, just import the ["custom web server" example](examples/custom-web-server.json) into your Node-RED workspace and deploy. It works similar to the "simple Web Server" presented before - with the exception, that requests to `{{BaseURL}}/custom-web-server/server-time`, `{{BaseURL}}/custom-web-server/echo` or `{{BaseURL}}/custom-web-server/client-address` will now be processed internally even if files with these names exist in the `public` directory. The included Postman collection may help you testing.

### Closed Web Server ###

Often, web servers are not completely open for everybody - instead, some (or all) entry points are protected and require clients to authenticate themselves before being authorized to access these resources.

Protecting entry points usually implies some kind of authorization management. In the following example, Node-RED developers (like you) first have to explicitly create new users themselves (without specifying any initial password for them), and may later also reset user passwords or delete users - all this is done within the Node-RED server workspace. Without prior authorization, users created this way are only allowed to define their passwords or delete themselves as long as no password is set - this is done through HTTP endpoints (where it is assumed that these users are in contact with a Node-RED developer who informs them about the appropriate time). For all other operations users have to "login" first (presenting their user name and current password). They may then change their password, delete themselves - and navigate to the other endpoints this server offers.

This kind of "developer-managed authorization handling" gives developers full control over the users they permit, but still keeps user passwords secret from them. On the other hand, users have full control over their passwords and may also delete their accounts - but no user is allowed to register him/herself. 

Authorization itself is handled using "Cookies": upon "login" a "Token" is created which allows the logged-in user to access non-public endpoints for a certain period until he/she has to log-in again.

The token itself consists of the requesting user's name and an expiration time (in this example, tokens remain valid for 2 minutes). While being sent in readable form, these details are protected using a "message digest" which is also part of the token cookie. As a consequence, any attempt to modify the cookie will immediately invalidate the token. The "secret" this message digest is based on is randomly generated upon Node-RED startup: any redeployment will therefore automatically invalidate any existing tokens.

In order to make this behaviour more comfortable for the user, any successfully authorized operation also "refreshes" the access token. Thus, a new login will only be required after two minutes of inactivity.

User data is kept in a JSON file called `Users.json` within the working directory of the running Node-RED instance. It will be read upon start-up and written whenever any user-related setting changes. The file contains a "dictionary" of user names with password hashes as values. These hashes are computed using PBKDF2 with random salt - this avoids having to store passwords in plain text form and makes automated dictionary attacks more difficult.

User names are arbitrary - this server does not make any assumptions about their format (except that they must not contain neither colons (":") nor control characters). Their length should however be limited since cookies are not allowed to occupy more than 4093 characters (including their name and some other cookie details) - assuming a length limit of 2048 bytes seems like a good idea.

![](examples/closed-web-server-I.png)
![](examples/closed-web-server-II.png)

The following part is already known from the previous examples:

![](examples/serving-files.png)

To test this server, just import the ["closed web server" example](examples/closed-web-server.json) into your Node-RED workspace and deploy. The included Postman collection contains several requests which allow you act like a user, any "administrative" operations have to be performed using Node-RED itself.

In this example, *all* endpoints (except those needed for authorization management) demand a successful authorization before being processed. In practice, there is often a mixture of public and private endpoints - a situation which may easily be implemented by combining flows from this example and the previous one.

## License ##

[MIT License](LICENSE.md)
