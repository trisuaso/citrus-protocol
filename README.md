# 🍊 Citrus

**Citrus** a simple protocol for schema-based federation.

## Structure

Everything should be tagged with a unique ID. These IDs should be formatted similar to the examples below:

```
{service_id}:{uuid/hash}
```

The `service_id` is the domain name of the website the ID belongs to originally. For content hosted on <https://neospring.org>, this would look like `neospring.org:...`.

For the second part, you can use either a random UUID or a hash of something related to the data the ID identifies.

When the backend receives a request for some content, it should first look for if it begins with a service ID. If it doesn't have this, it should be assumed that the content belongs to the host server itself. If it does begin with a service ID, then the host server should fetch this data from the server that is specified there and then return it like it came from the host server.

The client should never be concerned with *where* content comes from. It *should* still be accessible to the client, but it should not be made the big point. Including unnecessary information about host servers and such is likely to confuse people and distract them from the content itself.

## Schemas

This structure brings up a question: how do servers know that the server they're sending the request to is going to send back a result that they can understand?

This is where **schemas** come in. Before the request is sent to the other server, the host server should send a "verification" request. This verification request should just be a request to the server's `/.well-known/citrus/citrus.toml` file.

### `citrus.toml`

The Citrus configuration file is used to identify a server to other server's which support the Citrus protocol.

When another server requests `/.well-known/citrus/citrus.toml` on a server, the server **must** return a file which looks something like this:

```toml
[server]
name = "Alice's server"
id = "example.com"

[[schemas]]
id = "net.rainbeam.structs.Question"
location = "schemas/structs/question.toml"

[[schemas]]
id = "net.rainbeam.structs.Response"
location = "schemas/structs/response.toml"
```

In the `server` key, you'll find some basic information about the server itself. The *name* field represents the name of the server itself. This isn't expected to be used anywhere but in UI that references the server without directly showing its URL. It should not be relied on since it could be set to anything. The *id* field is just the domain name of the server that this file belongs to.

In the `schemas` array, you'll find information about the schemas that are supported by this server. This data includes an ID of the schema in reverse domain name notation, the location of the schema (relative to `{server.id}/.well-known/citrus`), and the type of the file pointed to by `location`.

### Schema structure

Schemas are a simple definition of a data structure which will be returned by the server's API. Schemas must be defined in TOML. A schema must look like this TOML example:

```toml
id = "net.rainbeam.structs.Question"

[struct]
# This example provides the schema for net.rainbeam.structs.question.
# Everything in the `struct` field is variable, and should just resolve to a `HashMap<String, String>`
# A schema only must have an `id` and `struct` field. Whatever is inside the struct does not matter.
author = "net.rainbeam.structs.profle"
recipient = "net.rainbeam.structs.profle"
content = "data.String"
id = "data.CitrusID"
ip = "data.String"
timestamp = "data.u128"
context = "net.rainbeam.structs.QuestionContext"
```

As shown in this example, the type definitions of a schema's `struct` *also* consist of values in reverse domain name notation. For simple language values such as strings or numbers, the `data.` name is used. Everything that doesn't start with data is expected to be provided somewhere in the `schemas` field of the server's `citrus.toml` file so that they can be found.

The ID `data.CitrusID` is used to represent a field which should be the ID of something else (following the previously defined ID format). This should be deserialized in the same way as `data.String`.

In reality, a server is not expected to go through and check the type of everything in a schema. If the server only wants to make sure that the data is receiving from a server is correct, then it should just be sufficient to make sure the server *says* that it supports the correct schema ID when it is listing its schemas in its `citrus.toml` file.

Schema validation is **not** a replacement for proper error handling on the server to deal with failures in parsing the content returned from another server.

## Users and data control

Citrus does not have a set definition of "Users", nor does it have a definition of a user's "inbox"/"outbox". All of these features are completely up to the server which is sending the requests to the other server.

It is important to note that schemas are designed for documenting the data that other servers have access to **only**. Anything related to authentication should happen on the host server itself, without other servers acting as a proxy. **User tokens should never be sent to another server!**

Imagine we had an app in which users could create posts and other users could like their posts. When a user's post is like, they'll also receive a notification telling them that somebody has liked their post. To do this, we'll need a schema for users, posts, likes, and notifications.

With this data structure in mind, our `citrus.toml` file would likely look something like this:

```toml
[server]
name = "Alice's server"
id = "example.com"

[[schemas]]
id = "com.example.structs.User"
location = "schemas/structs/users.toml"

[[schemas]]
id = "com.example.structs.Post"
location = "schemas/structs/post.toml"

[[schemas]]
id = "com.example.structs.Like"
location = "schemas/structs/like.toml"

[[schemas]]
id = "com.example.structs.Notification"
location = "schemas/structs/notification.toml"
```

In our user's schema, we might choose to share a `username` field, `id` field, and some extra metadata fields. We would never share the user's password, salt, IP addresses, or any of their authentication tokens.

It should be assumed that we won't need to send any `POST` requests to the other server for users, posts, or notifications. In this example, we have no concept of "comments" or "replies", so we don't need to worry about that.

For likes, however, we will need to create data on the other server **and** our server. When we like content, we need to store the user **and** the ID (citrus ID format) of the post that they liked on our server. On the other server, when we receive this request, we need to store a `like` object containing the ID of the user (citrus ID format) and the ID of the post that they liked. We also need to handle sending the notification to the user, since the server that owns the user object should manage all the data belonging to that user data that it possibly can.

There is no reason for the server which doesn't have the user to be managing private data for the user, such as notifications. This should all be done when the server which *does* own the user receives information from the server that doesn't.

By doing things this way, the server that sent the like knows that its user liked a post which is *not* on it, but is instead on another server. The other server knows that a post which *is* on it received a like, and it knows that the user who liked it is on another server.

## Schema APIs

Servers can optionally define an `api` field in the `SchemaPointer` of any schema. This field is used to include vital information to allow other apps to know how to use your app's API through the `citrus.toml` file.

In a schema's API definition, you **must** include the URL, method, and expected body of each request you could send to the API that this schema is supported by. You are allowed to define multiple API endpoints for a single schema. It is the job of the Citrus client on the remote server to read your API endpoints and evaluate the best one for their use case.

Here's a quick example:

```toml
[server]
name = "Alice's server"
id = "example.com"

[[schemas]]
id = "net.rainbeam.structs.Question"
location = "schemas/structs/question.toml"

[schemas.api.create]
method = "POST"
url = "/api/v1/questions"
body = "{\"content\":\"<field>\"}"
```

In the API definition, we provide the method, URL, and body template of the API endpoint. In the API body, we can specify that data is expected to be replaced by using `<field>`. When a Citrus client is sending a request using this API endpoint, all data given to it (in a `Vec<String>`, for example) should fill in by replacing a single `<field>` value (from left to right).