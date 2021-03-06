# The Draft Plugins Guide

A plugin is a tool that can be accessed through the `draft` CLI, but which is not part of the
built-in Draft codebase. This guide explains how to use and create plugins.

## An Overview

Draft plugins are add-on tools that integrate seamlessly with Draft. They provide a way to extend the
core feature set of Draft, but without requiring every new feature to be written in Go and added to
the core tool.

Draft plugins have the following features:

- They can be added and removed from a Draft installation without impacting the
  core Draft tool.
- They can be written in any programming language.
- They integrate with Draft, and will show up in `draft help` and other places.

Draft plugins live in `$DRAFT_HOME/plugins`.

The Draft plugin model is partially modeled on Git's plugin model. To that end, you may sometimes
hear `draft` referred to as the _porcelain_ layer, with plugins being the _plumbing_. This is a
shorthand way of suggesting that Draft provides the user experience and top level processing logic,
while the plugins do the "detail work" of performing a desired action.

## Installing a Plugin

A Draft plugin management system is in the works. But in the short term, plugins are installed by
copying the plugin directory into `$(draft home)/plugins`.

```console
$ cp -a myplugin/ $(draft home)/plugins/
```

If you have a plugin tar distribution, simply untar the plugin into the `$(draft home)/plugins`
directory.

## Building Plugins

In many ways, a plugin is similar to a chart. Each plugin has a top-level directory, and then a
`plugin.yaml` file.

```
$(draft home)/plugins/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh

```

In the example above, the `keybase` plugin is contained inside of a directory named `keybase`. It
has two files: `plugin.yaml` (required) and an executable script, `keybase.sh` (optional).

The core of a plugin is a simple YAML file named `plugin.yaml`. Here is a plugin YAML for a plugin
that adds support for Keybase operations:

```
name: "keybase"
version: "0.1.0"
usage: "Integrate Keybase.io tools with Draft"
description: |-
  This plugin provides Keybase services to Draft.
ignoreFlags: false
useTunnel: false
command: "$DRAFT_PLUGIN_DIR/keybase.sh"
```

The `name` is the name of the plugin. When Draft executes it plugin, this is the name it will use
(e.g. `draft NAME` will invoke this plugin).

_`name` should match the directory name._ In our example above, that means the plugin with
`name: keybase` should be contained in a directory named `keybase`.

Restrictions on `name`:

- `name` cannot duplicate one of the existing `draft` top-level commands.
- `name` must be restricted to the characters ASCII a-z, A-Z, 0-9, `_` and `-`.

`version` is the SemVer 2 version of the plugin.
`usage` and `description` are both used to generate the help text of a command.

The `ignoreFlags` switch tells Draft to _not_ pass flags to the plugin. So if a plugin is called
with `draft myplugin --foo` and `ignoreFlags: true`, then `--foo` is silently discarded.

The `useTunnel` switch indicates that the plugin needs a tunnel to draftd. This should be set to
`true` _anytime a plugin talks to draftd_. It will cause Draft to open a tunnel, and then set
`$DRAFT_HOST` to the right local address for that tunnel. But don't worry: if Draft detects that a
tunnel is not necessary because draftd is running locally, it will not create the tunnel.

Finally, and most importantly, `command` is the command that this plugin will execute when it is
called. Environment variables are interpolated before the plugin is executed. The pattern above
illustrates the preferred way to indicate where the plugin program lives.

There are some strategies for working with plugin commands:

- If a plugin includes an executable, the executable for a `command:` should be
  packaged in the plugin directory.
- The `command:` line will have any environment variables expanded before
  execution. `$DRAFT_PLUGIN_DIR` will point to the plugin directory.
- The command itself is not executed in a shell. So you can't oneline a shell script.
- Draft injects lots of configuration into environment variables. Take a look at
  the environment to see what information is available.
- Draft makes no assumptions about the language of the plugin. You can write it
  in whatever you prefer.
- Commands are responsible for implementing specific help text for `-h` and `--help`.
  Draft will use `usage` and `description` for `draft help` and `draft help myplugin`,
  but will not handle `draft myplugin --help`.

## Environment Variables

When Draft executes a plugin, it passes the outer environment to the plugin, and also injects some
additional environment variables.

Variables like `KUBECONFIG` are set for the plugin if they are set in the outer environment.

The following variables are guaranteed to be set:

- `DRAFT_PLUGIN`: The path to the plugins directory
- `DRAFT_PLUGIN_NAME`: The name of the plugin, as invoked by `draft`. So
  `draft myplug` will have the short name `myplug`.
- `DRAFT_PLUGIN_DIR`: The directory that contains the plugin.
- `DRAFT_BIN`: The path to the `draft` command (as executed by the user).
- `DRAFT_HOME`: The path to the Draft home.
- `DRAFT_PACKS_PATH`: The path to the Draft starter packs.
- `DRAFT_HOST`: The `domain:port` to draftd. If a tunnel is created, this
  will point to the local endpoint for the tunnel. Otherwise, it will point
  to `$DRAFT_HOST`, `--host`, or the default host (according to Draft's rules of
  precedence).

While `DRAFT_HOST` _may_ be set, there is no guarantee that it will point to the correct draftd
instance. This is done to allow the plugin developer to access `DRAFT_HOST` in its raw state when
the plugin itself needs to manually configure a connection.

## A Note on `useTunnel`

If a plugin specifies `useTunnel: true`, Draft will do the following (in order):

1. Parse global flags and the environment
2. Create the tunnel
3. Set `DRAFT_HOST`
4. Execute the plugin
5. Close the tunnel

The tunnel is removed as soon as the `command` returns. So, for example, a command cannot
background a process and assume that that process will be able to use the tunnel.

## A Note on Flag Parsing

When executing a plugin, Draft will parse global flags for its own use. Some of these flags are
_not_ passed on to the plugin.

- `--debug`: If this is specified, `$DRAFT_DEBUG` is set to `1`
- `--home`: This is converted to `$DRAFT_HOME`
- `--host`: This is converted to `$DRAFT_HOST`
- `--kube-context`: This is simply dropped. If your plugin uses `useTunnel`, this
  is used to set up the tunnel for you.

Plugins _should_ display help text and then exit for `-h` and `--help`. In all other cases, plugins
may use flags as appropriate.
