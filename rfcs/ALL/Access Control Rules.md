# Access Control Rules 

Starting from 0.11.0 release, Zenoh provides the option of controlling access to key-expressions (eg: `test/demo` ) based on network interface (eg: `lo`). The access control is managed by filtering messages (denoted as **actions** in the acl config): `put`, `get`, `declare_subscriber`, `declare_queryable`. This filteration can be applied on both incoming messages (`ingress`) and outgoing messages (`egress`).

## Access Control Config

The typical ACL configuration in the config.json5 file looks something like this:

```json5
 access_control: {
  "enabled": true,
  "default_permission": "deny",
  "rules": 
  [
    {
      "actions": [
        "put","declare_subscriber"
      ],
      "flows":["egress","ingress"],
      "permission": "allow",
      "key_exprs": [
        "test/demo"
      ],
      "interfaces": [
        "lo0"
      ]
    },
 ]
}
```


The configuration has 3 fields:

1. **enabled**: true or false
2. **default_permission**: allow or deny 
3. **rules**: list of rules for explicitly specifying allow and deny permissions

The *enabled* field decides if the ACL is enabled or not. If it is set to `false`, no filtering of messages takes place and everything following that in the config is ignored.
The *default_permission* field provides the implicit permission for the filtering, i.e., this rule applies if no other matching rule is found for an *action*. If set to `allow`, it will allow all messages to pass through unless explicitly denied in the *rules* field. If set to `deny`, it blocks all messages and only allows those that are allowed by explicit rules provided in the *rules* section. If left empty or unspecified, it is treated as `deny`. The *default_permission* always has lower priority than explicit rules provided in the *rules* section.

The *rules* section itself has sub-fields: *actions*, *flows*, *permission*, *key_exprs*, *interfaces*. The values provided in these fields set the explicit rules for the access control:

* **actions**: supports four types of messsges - `put`, `get`, `declare_subscriber`, `declare_queryable`
* **flows**: supports two values - `egress` and `ingress`
* **permission**: supports value `allow` or `deny`
* **key_exprs**: supports values of any key type or key-expression (set of keys) type, eg: `temp/room_1`, `temp/**` etc. (see [Key_Expressions](https://github.com/eclipse-zenoh/roadmap/blob/main/rfcs/ALL/Key%20Expressions.md))
* **interfaces**: supports all possible value for network interfaces, eg: `lo`, `lo0` etc

For example, in our sample config, the *default_permission* is set to deny and then a rule is added to explicitly allow certain behavior.  Here, a node connecting via the `lo0` interface will be allowed to `put` and `declare_subscriber` on the `test/demo` key expression for both incoming and outgoing messages. However, if there is a node connected via another interface or trying to perform another action (eg: `get`), it will be denied. This provides a granular access control over permissions, ensuring that only authorized devices or networks can perform allowed behavior. 

Internally, for each combination of *interface* + *flow* + *action*  (example:`l0`+`egress`+`put`) in the rules vector, we construct *allow* and *deny* KeTrees (key-expression tries). The *allow* KeTree is built using all the key-expressions provided in the config on which that *interface* is allowed to do that *action* on a particular *flow*. On receiving an authorization request, the key-expression in the request is matched against the appropriate KeTrees to confirm authorization.

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

All requets are macthed on keys and key expressions (for more information on key expressions, check out [Key_Expressions](https://github.com/eclipse-zenoh/roadmap/blob/main/rfcs/ALL/Key%20Expressions.md)) . Therefore, it is important to understand how the key-expression matching works, since it ultimately decides the behavior of the access control logic. In matching a key-expression against a KeTree, it will match as a positive only if it is *included in* (*equal to* or *subset of*) the key-expressions specified in the KeTree. A partial match or being a superset will not result in a match.

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

Given Zenoh's priority is performance, a lot of care was taken while adding access control features to the codebase, to keep the performance as high as possible. However, as with any other piece of software, security comes with a price in terms of performance. However, having done multiple tests, we can share some tips to improve performance:

1. Keys (eg: `test/demo/a` ) are faster than key-expressions that use wildcards and DSL (eg: `test/demo/*` or`test/d$*`). Therefore, don't use them in your list of rules unless necessary. Verbatims are okay.
2. The number of chunks in a key-expression also affects the performance since it increases the depth of the KeTree to be searched. So `test/demo/a` will be faster than `test/demo/a/b/c`. This loss of performance is not as drastic as that of using wildcards and DSL. However, still try to keep the number of chunks as low as possible in the key expression.
3. Using both flows in the list of rules can results in checking of the messages twice. If possible, use only a single flow in your rules.
4. Tip 3 can be applied to all the other fields as well, though the performance improvement will not be as drastic. You should keep the list of rules as specific as possible. If you don't need to use certain actions or flows, you can skip them in the list of rules. For example, if your scenario uses only publishers and subscribers, maybe you don't have to set rules for `get` and `declare_queryable` in your access control rules.