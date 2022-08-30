# Zenoh Plugins
## Preface
`zenohd`, Zenoh's reference router daemon, supports plugins to extend its behaviour. For example, the `storage_manager` plugin facilitates turning a router into a storage node by specifying key expressions to subscribe to; while the `rest` plugin allows `zenohd` to behave as a server for Zenoh's REST API.

These two plugins are bundled with Zenoh's releases by default, but many others may be created for a variety of reasons. This RFC's goal is to help users implement their own plugins.

> ⚠️ While `zenohd` plugins have some advantages that will be discussed bellow, they pass some Rust types over the DLL boundary.  
> Since Rust's ABI is unstable, this means that unless both `zenohd` and the plugins it's required to load are compiled by the same compiler with the same dependency versions, the plugin and `zenohd` may use different layouts for some types, leading to potential undefined behaviour (likely to cause segfaults).  
> Plugin developpers and users therefore should be careful around plugins.

## Why write a plugin over a standalone application embedding Zenoh?
Here are a few reasons why you might want to write a plugin instead of an application that embeds Zenoh:
- To have fewer nodes in your infrastructure: a plugin does not add a node, becoming part of `zenohd` instead. If several plugins are co-located on the same node and have to communicate together, these communications will be entirely done in RAM.
- Get hot-loading and hot-configuration for cheap:
  - hot-loading: `zenohd` only loads plugins when a configuration exists for them, and drops them if that configuration ceases to exist; that means you can keep your node running while starting and stopping your plugin at will.
    > ⚠️ note that some implementations of `dlopen`/`dlclose` may cache the dynamic library, making hot-loading not guaranteed to reload the associated file.
  - hot-configuration: `zenohd` informs plugins whenever a request to change their configuration is made, letting them vet the changes before they are applied and react to the new configuration easily.

## Life-cycle of a plugin
### Startup
When `zenohd` starts up, or when its configuration changes, it lists the names of the plugins that have been added or removed from its configuration. It will then for each added plugin load the appropriate library, and call the `load_plugin` function. Note that you should never write this function yourself, intead using `zenoh_plugin_trait::declare_plugin!(MyPlugin)` where `MyPlugin` is a type that implements `zenoh::plugins::ZenohPlugin`.

`zenoh::plugins::ZenohPlugin` is just an appropriately typed version of the `zenoh::plugins::Plugin`, for which you only need to define two properties:
- The `STATIC_NAME` constant must be a static string that may be used to refer to your plugin when linking it statically with `zenohd`. It's highly unlikely that it will ever be needed, but you should try to make this name unique.
- The `start` method, which is used to start your plugin.  
  It takes 2 arguments: the `name` under which your plugin was instanciated, and the `runtime` that you may use with `zenoh::Session::init` to construct a `zenoh::Session` that shares its internals with the host's session. The runtime also contains the configuration, so that you may inspect your plugin's configuration.  
  Your `start` method should be rather short lived, spawning the tasks that you may need, and returning your `RunningPlugin` which `zenohd` will use to communicate configuration changes to your plugin, as well as to allow it to respond to adminspace status queries.

### Deletion
When your plugin's configuration is completely deleted, the `RunningPlugin` you returned when `start` was called will be dropped, which will be your signal to clean up your plugin.

### Configuration changes
In the meantime, if your plugin's configuration is about to change, it will be notified through the validation function it will have returned when its `config_checker` method was called. This function should accept 3 arguments:
- the `path` within the plugin's config that's the root of changes,
- the `current` configuration,
- the `expected` new configuration.

This function may return one of 3 things:
- `Err` if the plugin rejects that configuration change.
- `Ok(None)` if the plugin accepted that configuration change.
- `Ok(Some(value))` if the plugin wishes for an other configuration to be saved instead of the `expected` one.
