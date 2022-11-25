# Connectivity Status and Events

<div style="background: #FFF5D6; padding: 3px; margin-bottom: 5px; font-size: 0.875rem;">
  <span class="emoji">🔬</span> <b>Unstable</b><br/>
  The resource keys and value formats defined in this document are <b>unstable</b>. They may change it in a future release.
</div>

The Zenoh `Session` provides to the application layer:
- access to the current connectivity status of a Zenoh `Session`:
    - The currently opened *transport sessions*
    - The currently opened *transport links*
- notifications of Zenoh `Session` connectivity events:
    - The opening of a new *transport session*
    - The closing of a *transport session*
    - The opening of a new *transport link*
    - The closing of a *transport link*

Those statuses and events are made available through resources in the following key space: `@/session/<local_zid>/**`.
So that the connectivity status of a Zenoh `Session` can be accessed through queries. And the connectivity events of a Zenoh `Session` can be accessed through `Subscribers`.

## Connectivity status

The Zenoh user `Session` embeds a `Queryable` with the following configuration:
- key_expr: `@/session/<local_zid>/**`
- allowed_origin: `SessionLocal`

For each currently open unicast *transport session*, the `Queryable` will reply the following resource to matching queries: 
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Transport`](#the-transport-type)

For each currently open unicast *transport link*, the `Queryable` will reply the following resource to matching queries: 
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>/link/<link_id>`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Link`](#the-link-type)

### Example

Using the `z_get` example: 

```bash
$ z_get -s @/session/**
Opening session...
Sending Query '@/session/**'...
>> Received ('@/session/F36E4392B8274714BAA1A4F425F0FEED/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> Received ('@/session/F36E4392B8274714BAA1A4F425F0FEED/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/6763370205260260746': '{"src":"tcp/127.0.0.1:55555","dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true}')
```

## Connectivity Events

### Transport Events

Each time a new unicast *transport session* is established with a new remote process (router, peer or client), a `put` is sent to matching `SessionLocal` subscribers:
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>`
- kind: `Put`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Transport`](#the-transport-type)

Each time a unicast *transport session* is closed, a `delete` is sent to matching `SessionLocal` subscribers:
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>`
- kind: `Delete`
- value: 
  - encoding: `<empty>`
  - payload: `<empty>`

### Link Events

Each time a new unicast *transport link* is established to a remote `Endpoint`, a `put` is sent to matching `SessionLocal` subscribers:
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>/link/<link_id>`
- kind: `Put`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Link`](#the-link-type)

Each time a unicast *transport link* is closed, a `delete` is sent to matching `SessionLocal` subscribers:
- key: `@/session/<local_zid>/transport/unicast/<remote_zid>/link/<link_id>`
- kind: `Delete`
- value: 
  - encoding: `<empty>`
  - payload: `<empty>`

### Example

Using the `z_sub` example:

```bash
$ z_sub -k @/session/** 
Opening session...
Declaring Subscriber on '@/session/**'...
Enter 'q' to quit...
>> [Subscriber] Received PUT ('@/session/CE54FD5443C2432C840204E6D16DF877/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> [Subscriber] Received PUT ('@/session/CE54FD5443C2432C840204E6D16DF877/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/3786076588163193314': '{"src":"tcp/127.0.0.1:55555","dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true}')
>> [Subscriber] Received DELETE ('@/session/CE54FD5443C2432C840204E6D16DF877/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/3786076588163193314': '')
>> [Subscriber] Received DELETE ('@/session/CE54FD5443C2432C840204E6D16DF877/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '')
```

## Using the `QueryingSubscriber`

When accessing connectivity events by declaring a `Subscriber`, it may happen that some *transport sessions* and *transport links* were openned before the `Subscriber` declaration. In order to access connectivity events that occured before the `Subscriber` declaration as well as future connectivity events, a `zenoh-ext` `QueryingSubscriber` can be used.

## Example

Using the `z_query_sub` example:

```bash
$ z_query_sub -k @/session/**
Opening session...
Declaring QueryingSubscriber on @/session/** with an initial query on @/session/**
Enter 'd' to issue the query again, or 'q' to quit...
>> [Subscriber] Received PUT ('@/session/5AFCADB62D234A56ADB999E13E1D4392/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> [Subscriber] Received PUT ('@/session/5AFCADB62D234A56ADB999E13E1D4392/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/15974018027047406507': '{"src":"tcp/127.0.0.1:5555"},"dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true')
>> [Subscriber] Received DELETE ('@/session/5AFCADB62D234A56ADB999E13E1D4392/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/15974018027047406507': '')
>> [Subscriber] Received DELETE ('@/session/5AFCADB62D234A56ADB999E13E1D4392/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '')
```

## Connectivity types

### The `Transport` type

```json
{
    "type": "object",
    "properties": {
        "zid": {
        "type": "string",
        "description": "The zid of the remote zenoh process (base64)."
        },
        "whatami": {
        "type": "string",
        "description": "The type of the remote zenoh process (router, peer or client)."
        },
        "is_qos": {
        "type": "boolean"
        }
        "is_shm": {
        "type": "boolean"
        }
    }
}
```

### The `Link` type

```json
{
    "type": "object",
    "properties": {
        "src": {
        "type": "string",
        "description": "The local Endpoint of the link."
        },
        "dst": {
        "type": "string",
        "description": "The remote Endpoint of the link."
        },
        "group": {
        "type": "string",
        },
        "mtu": {
        "type": "integer",
        },
        "is_reliable": {
        "type": "boolean",
        },
        "is_streamed": {
        "type": "boolean",
        }
    }
}
```
