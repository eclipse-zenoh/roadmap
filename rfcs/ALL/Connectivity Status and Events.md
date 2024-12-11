# Connectivity Status and Events

> 🔬 **Unstable**<br/>
> The resource keys and value formats defined in this document are **unstable**. They may change it in a future release.

The Zenoh `Session` provides to the application layer:
- access to the current connectivity status of a Zenoh `Session`:
    - The currently opened *transport sessions*
    - The currently opened *transport links*
- notifications of Zenoh `Session` connectivity events:
    - The opening of a new *transport session*
    - The closing of a *transport session*
    - The opening of a new *transport link*
    - The closing of a *transport link*

Those statuses and events are made available through resources in the following key space: `@/<local_zid>/session/**`.
So that the connectivity status of a Zenoh `Session` can be accessed through queries. And the connectivity events of a Zenoh `Session` can be accessed through `Subscribers`.

## Connectivity status

The Zenoh user `Session` embeds a `Queryable` with the following configuration:
- key_expr: `@/<local_zid>/session/**`
- allowed_origin: `SessionLocal`

For each currently open unicast *transport session*, the `Queryable` will reply the following resource to matching queries: 
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Transport`](#the-transport-type)

For each currently open unicast *transport link*, the `Queryable` will reply the following resource to matching queries: 
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>/link/<link_id>`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Link`](#the-link-type)

### Example

Using the `z_get` example: 

```bash
$ z_get -s @/*/session/**
Opening session...
Sending Query '@/*/session/**'...
>> Received ('@/F36E4392B8274714BAA1A4F425F0FEED/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> Received ('@/F36E4392B8274714BAA1A4F425F0FEED/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/6763370205260260746': '{"src":"tcp/127.0.0.1:55555","dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true}')
```

## Connectivity Events

### Transport Events

Each time a new unicast *transport session* is established with a new remote process (router, peer or client), a `put` is sent to matching `SessionLocal` subscribers:
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>`
- kind: `Put`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Transport`](#the-transport-type)

Each time a unicast *transport session* is closed, a `delete` is sent to matching `SessionLocal` subscribers:
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>`
- kind: `Delete`
- value: 
  - encoding: `<empty>`
  - payload: `<empty>`

### Link Events

Each time a new unicast *transport link* is established to a remote `Endpoint`, a `put` is sent to matching `SessionLocal` subscribers:
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>/link/<link_id>`
- kind: `Put`
- value: 
  - encoding: `application/json`
  - payload: a json encoded [`Link`](#the-link-type)

Each time a unicast *transport link* is closed, a `delete` is sent to matching `SessionLocal` subscribers:
- key: `@/<local_zid>/session/transport/unicast/<remote_zid>/link/<link_id>`
- kind: `Delete`
- value: 
  - encoding: `<empty>`
  - payload: `<empty>`

### Example

Using the `z_sub` example:

```bash
$ z_sub -k @/*/session/** 
Opening session...
Declaring Subscriber on '@/*/session/**'...
Enter 'q' to quit...
>> [Subscriber] Received PUT ('@/CE54FD5443C2432C840204E6D16DF877/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> [Subscriber] Received PUT ('@/CE54FD5443C2432C840204E6D16DF877/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/3786076588163193314': '{"src":"tcp/127.0.0.1:55555","dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true}')
>> [Subscriber] Received DELETE ('@/CE54FD5443C2432C840204E6D16DF877/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/3786076588163193314': '')
>> [Subscriber] Received DELETE ('@/CE54FD5443C2432C840204E6D16DF877/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '')
```

## Using the `AdvancedSubscriber`

When accessing connectivity events by declaring a `Subscriber`, it may happen that some *transport sessions* and *transport links* were opened before the `Subscriber` declaration. In order to access connectivity events that occurred before the `Subscriber` declaration as well as future connectivity events, a `zenoh-ext` `AdvancedSubscriber` can be used.

## Example

Using the `z_advanced_sub` example:

```bash
$ z_advanced_sub -k @/*/session/**
Opening session...
Declaring AdvancedSubscriber on @/*/session/**
Press CTRL-C to quit...
>> [Subscriber] Received PUT ('@/5AFCADB62D234A56ADB999E13E1D4392/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '{"zid":"CDDC6B8CAC0D4D52A0FB55DCD693F293","whatami":"router","is_qos":true,"is_shm":true}')
>> [Subscriber] Received PUT ('@/5AFCADB62D234A56ADB999E13E1D4392/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/15974018027047406507': '{"src":"tcp/127.0.0.1:5555"},"dst":"tcp/127.0.0.1:7447","group":null,"mtu":65535,"is_reliable":true,"is_streamed":true')
>> [Subscriber] Received DELETE ('@/5AFCADB62D234A56ADB999E13E1D4392/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293/link/15974018027047406507': '')
>> [Subscriber] Received DELETE ('@/5AFCADB62D234A56ADB999E13E1D4392/session/transport/unicast/CDDC6B8CAC0D4D52A0FB55DCD693F293': '')
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
