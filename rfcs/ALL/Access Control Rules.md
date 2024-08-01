# Access Control Rules 

Zenoh provides the option of controlling access to key-expressions (eg: `test/demo`) based on network interfaces (eg: `lo`) and other authentication mechanisms. The access control is managed by filtering messages: `put`, `delete`, `declare_subscriber`, `query`, `reply` and `declare_queryable`. This filteration can be applied on both incoming messages (`ingress`) and outgoing messages (`egress`).

Access control was added in Zenoh 0.11 and is described in the [Zenoh 0.11 Access Control Rules RFC](https://github.com/eclipse-zenoh/roadmap/blob/ca841fe219890bf73289089b520271d70ded89b6/rfcs/ALL/Access%20Control%20Rules.md). The following document describes the Zenoh 1.0 access control feature without highlighting the changes compared to the 0.11 version.

## Access Control Config

The typical ACL configuration in the config.json5 file looks something like this:

```json5
 access_control: {
  "enabled": true,
  "default_permission": "deny",

  "rules": 
  [
    {
      "id": "allow pub/sub on test/demo",
      "messages": [
        "put",
        "delete",
        "declare_subscriber",
      ],
      "flows":["egress","ingress"],
      "permission": "allow",
      "key_exprs": [
        "test/demo"
      ],
    },
 ],

 "subjects": [
  {
    "id": "loopback interface",
    "interfaces": [
      "lo0",
      "lo",
    ],
  },
  {
    "id": "usernames on any interface",
    "usernames": [
      "zenoh_instance_1",
      "zenoh_instance_2",
    ,]
  },
 ],

 "policies": [
  {
    "rules": ["allow pub/sub on test/demo"],
    "subjects": [
      "loopback interface",
      "usernames on any interface",
    ]
  }
 ],
}
```


The configuration has 5 fields:

1. **enabled**: true or false
2. **default_permission**: allow or deny 
3. **rules**: list of rules for explicitly specifying allow and deny permissions for messages
4. **subjects**: list of subject configurations to match with connected Zenoh instances
5. **policies**: list of associations between declared rules and declared subjects

The *enabled* field decides if the ACL is enabled or not. If it is set to `false`, no filtering of messages takes place and everything following that in the config is ignored.
The *default_permission* field provides the implicit permission for the filtering, i.e., this rule applies if no other matching rule is found for a message. If set to `allow`, it will allow all messages to pass through unless explicitly denied in the *rules* field. If set to `deny`, it blocks all messages and only allows those that are allowed by explicit rules provided in the *rules* section. If left empty or unspecified, it is treated as `deny`. The *default_permission* always has lower priority than explicit rules provided in the *rules* section.

The *rules* section itself has sub-fields: *id*, *messages*, *flows*, *permission*, *key_exprs*. The values provided in these fields set the explicit rules for the access control over individual messages:

* **id**: unique string identifier within the rules list.
* **messages**: supports six types of messsges - `put`, `delete`, `declare_subscriber`, `query`, `reply`, `declare_queryable`.
* **flows**: supports two values - `egress` and `ingress`. If this field is not provided, the rule will apply to both flows.
* **permission**: supports value `allow` or `deny`.
* **key_exprs**: supports values of any key type or key-expression (set of keys) type, eg: `temp/room_1`, `temp/**` etc. (see [Key_Expressions](https://github.com/eclipse-zenoh/roadmap/blob/main/rfcs/ALL/Key%20Expressions.md))

The *subjects* section has the following sub-fields: *id*, *interfaces*, *cert_common_names*, *usernames*. The values provided are matched with the characteristics of connected Zenoh instances, in order to identify which rules to apply on which instance's messages.

* **id**: unique string identifier within the subjects list.
* **interfaces**: list of local network interfaces through which the configured instance communicates with the remote instance to be matched. Supports all possible values for network interfaces, eg: `lo`, `lo0` etc. If this field is not provided, it will match instances connected on any network interfaces, or none (currently possible on certain link protocols, ex: *websocket*).
* **cert_common_names**: list of certificate common names which are matched with the respective certificate content of the remote instances connected using TLS or QUIC transport. This requires that the local instance has a valid TLS authentication configuration. If this field is not provided, it will match with instances that have any certificate common name or none.
* **usernames**: list of usernames to be matched with the authentication config of remote instances. This requires that the local instance has a valid user-password authentication configuration. If this field is not provided, it will match with instances that have any username or none.

Note that a subject with no *interfaces*, *cert_common_names* and *usernames* is valid, and matches all Zenoh instances (wildcard).

Finally, the *policies* section allows to associate (i.e apply) rules to subjects. It is a list of JSON objects containing each a *rules* and *subjects* list, which respectively contain identifiers of declared rules and declared subjects to be associated.

For example, in our sample config, the *default_permission* is set to deny and then the `"allow pub/sub on test/demo"` rule is added to explicitly allow certain behavior.  Here, a node connecting via the `lo0` interface will be allowed to `put`, `delete` and `declare_subscriber` on the `test/demo` key expression for both incoming and outgoing messages. However, if there is a node trying to send another message type (eg: `query`), it will be denied. Additionally, nodes connected via any interface and authentified with one of the listed usernames in the `"usernames on any interface"` subject are also allowed to `put`, `delete` and `declare_subscriber` on the `test/demo` key expression. This provides a granular access control over permissions, ensuring that only authorized devices or networks can perform allowed behavior. 

Internally, for each combination of *subject_combination* + *flow* + *message*  (example:`Interface("l0")`+`"egress"`+`"put"`) parsed from the config, we construct *allow* and *deny* KeTrees (key-expression tries). The *allow* KeTree is built using all the key-expressions provided in the config on which that *subject* is allowed to perform that *action* on a particular *flow*. On receiving an authorization request, the key-expression in the request is matched against the appropriate KeTrees to confirm authorization.

## Subject combinations

Subjects are mainly comprised of three lists of characteristics to be matched: *interfaces*, *cert_common_names* and *usernames*. In order to allow users to express combinations of characteristics without implementing a boolean logic language and parser, we've identified the following two rules for construction of subjects:

- Two items within the same list are considered a logical OR
- Two items across different lists are considered a logical AND

These rules are built based on the assumption that an individual message on which ACL rules are to be applied can only travel across one network interface, from/to an instance authentified with at most one certificate common name, and one username.

To produce all combinations that characterize a subject configuration based on the rules of construction noted above, the Cartesian product of the *interfaces*, *cert_common_names* and *usernames* lists is calculated. The following is an example of a subject configurations and its internal representation.

```json5
{
  "id": "example subject",
  "interfaces": [
    "lo0",
    "en0",
  ],
  "cert_common_names": [
    "example.zenoh.io"
  ],
  "usernames": [
    "zenoh-example1",
    "zenoh-example2",
  ],
  // This instance translates internally to this filter:
  // (interface="lo0" AND cert_common_name="example.zenoh.io" AND username="zenoh-example1") OR
  // (interface="lo0" AND cert_common_name="example.zenoh.io" AND username="zenoh-example2") OR
  // (interface="en0" AND cert_common_name="example.zenoh.io" AND username="zenoh-example1") OR
  // (interface="en0" AND cert_common_name="example.zenoh.io" AND username="zenoh-example2")
}
```

Combined with the OR combinations that can be expressed between configured subjects in the *policies* list, we believe this gives the users enough tools to cover most use-cases.

## Priority 

For each message, the access control checks are made in the following order of priority:
*explicit deny* rule > *explicit allow* rule > *default_permission* rule

1. Explicit rules with deny permissions are given topmost priority. This means that if the key-expression in the  authorization request matches against anything in the *deny* KeTree, it will be denied immediately. If there is no match, then the *explicit allow* rules are checked.
2. Explicit rules with allow permissions come next, i.e., if request matches anything in the *allow* tree then it will be allowed.
3. The *default_permission* value has least priority. It is applied only if there are no matches in the previous two steps.
    
    Note: if *default_permission* is set to `allow`, then there is no need for setting explicit allow rules, since they will be overlooked by the matcher anyways. The same logic does not apply if *default_permission* is set to `deny` and explicit deny, due to permission priorities (see pseudo-code below for more detail)"

The decision logic is explained by the following pseudocode:

```rust
fn is_allowed(key_expr) -> decision {
   if default_permission == deny {
      if !deny_ketree.match(key_expr) && allow_ketree.match(key_expr) {
            return true;
      } else {
          return false;
      }
   } else { 
      // default_permission = allow
      if deny_ketree.match(key_expr) {
         return false;
      } 
      // no need to check for allow_ketree here
      return true; 
   }
}
```

## Key-Expression Matching

All requests are macthed on keys and key expressions (for more information on key expressions, check out [Key_Expressions](https://github.com/eclipse-zenoh/roadmap/blob/main/rfcs/ALL/Key%20Expressions.md)). Therefore, it is important to understand how the key-expression matching works, since it ultimately decides the behavior of the access control logic. In matching a key-expression against a KeTree, it will match as a positive only if it is *included in* (*equal to* or *subset of*) the key-expressions specified in the KeTree. A partial match or being a superset will not result in a match.

The following table demonstrates how the matching will work on a key-expression(KE) in request and in the list of rules:

| KE in request | KE in list of rules | Match | Reason |
|---------------|---------------|-------|------|
| test/demo/a         | test/demo/a         | yes   | - |
| test/demo/a         | test/demo/*         | yes   | -  |
| test/\*/\*         | test/**          | yes   | - |
| test/demo/a         | **            | yes   | - |
| test/demo/a         | test/*/a         | yes   | -  |
| test/demo/*         | test/*/a         | no    | partial overlap |
| test/**          | test/demo/a         | no    |superset |

The same behaviour extends for our DSL support as well:

| KE in request | KE in list of rules | Match |Reason |
|---------------|---------------|-------|----|
| test/demo/a         | test/d$*/a         | yes   | -  | 
| test/demo/a         | t$*/**            | yes   | -  | 
| test/d$*/a        | t$*/**            | yes   | -  |
| test/demo/*         | test/d$*/a         | no    | partial overlap  |
| test/d$*/a          | test/demo/a         | no    | superset  |

For verbatims, the subpart of the key-expression starting with `@` has to be *exactly* the same in the list of rules for a match. The other parts of the key-expression follow matching rules as before.

| KE in request | KE in list of rules | Match |Reason|
|---------------|---------------|-------|-----|
| test/@demo/a         | test/demo/a         | no   | verbatim missing  | 
| test/demo/a         | test/@demo/a         | no   | verbatim missing | 
| test/@demo/a         | test/@demo/*         | yes   | -  | 
| test/@demo/a         | **            | no   | verbatim missing  | 
| test/@demo/a         | test/*/a         | no   | verbatim missing  | 
| test/@demo/*         | */@demo/a         | no    | partial overlap  | 

If the match happens then the result will be as set in the explicit rules. If not, then the default permission will take over. For example, if the default permission is `allow` and the key-expressions in the *explicit deny* rules are any of the key-expressions like `test/demo/a` , `test/demo/*` or `test/demo/**` then a request on `test/demo/a` will match with the *explicit deny* KeTree and will be denied. However, given the same list of rules, a request on `test/**` will not match (since it is a superset) and therefore the request will be allowed to go through. Therefore, extra care needs to be taken while devising the rules, and especially so when using wildcards.


## Performance

Given Zenoh's priority is performance, a lot of care was taken while adding access control features to the codebase, to keep the performance as high as possible. However, as with any other piece of software, security comes with a price in terms of performance. Having done multiple tests, we can share some tips to improve performance:

1. Keys (eg: `test/demo/a` ) are faster than key-expressions that use wildcards and DSL (eg: `test/demo/*` or`test/d$*`). Therefore, don't use them in your list of rules unless necessary. Verbatims are okay.
2. The number of chunks in a key-expression also affects the performance since it increases the depth of the KeTree to be searched. So `test/demo/a` will be faster than `test/demo/a/b/c`. This loss of performance is not as drastic as that of using wildcards and DSL. However, still try to keep the number of chunks as low as possible in the key expression.
3. Using both flows in the list of rules can cause a double verification of messages. If possible, only use a single flow in your rules.
4. Tip 3 can be applied to all the other fields as well, though the performance improvement will not be as drastic. You should keep the list of rules as specific as possible. If you don't need to use certain messages or flows, you can skip them in the list of rules. For example, if your scenario uses only publishers and subscribers, maybe you don't have to set rules for `query`, `reply` and `declare_queryable` in your access control rules.
