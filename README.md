# KFG

> Config for dummies

The goal of this project it to define a new type of config that is smart enough to catch some of the stupid things we do that cause outages or introduce weird bugs (because you are pointing at the wrong environment).

The idea is a config DSL which can be used to generate config files in a more maintainable way that constrains bloat and improves the safety of configuration files by placing constraints on values. The other half of this story is a CLi tool that can process these

## Status

Currently the status is just this spec. I am getting people to review it and give feedback. Based on that if I get enough feedback that it would solve actual problems people have and fills a gap in the developer tool chain, I'll build it. Initially I have been playing around with an implementation in F# but plan to do either a parallel implementation or even the primary implementation in Rust.

## Goals

- Allow for 1 configuration file that can generate config a file for multiple environments
- Allow easy adoption of the format by progressively starting to use features<sup>1</sup>
- Allow validation of values both as required and formats
- Verify correctness of environments<sup>2</sup>
- Useful error messages (and codes) for both humans and machines
- Support multiple formats for generated config types
- Easy exploration and summaries of config structures
- (?) Setting of environments based on a kfg file
- (?) Validation of schema when referencing external files
- (?) Allow combination of multiple files
- (?) Generate config types for multiple languages
- (?) Generate a schema description for easy extension for unsupported languages
- (?) IDE support... at least VSCode support
- (?) Library support for Rust and .NET where kfg files can be consumed directly with extensions for validation and input sources
- (?) Support key vaults as input sources

1. Allow progressive use of features (should be able to just rename the extension of an existing config and start using)*
2. If environment variables are required, the tool can check they are present

## Syntax

What follows is examples and explanations of the planned features.

### INPUTS

> Name is still up for debate, variables is a rather overloaded term. 

#### DEFINE

##### Example

*app.kfg*
```
DEFINE ENV:sql_server_address
DEFINE ARG:user_db_username
DEFINE ARG:user_db_password
```

The above example doesn't do much, so let's look at how we can use inputs to produce safer configs...

*app.kfg*
```
DEFINE ENV:sql_server_address <ipaddress> /// The IP Address for the MS SQL database containing the users
DEFINE ARG:user_db_username <minlength(8);nospecial> /// Username for a MS SQL user that has read/write access to the user table
DEFINE ARG:user_db_password <minlength(12)> /// password for provided user
{
    "dbs" : {
        "user_db" : {
            "server" : "{{ENV:sql_server_address}}",
            "username" : "{{ARG:user_db_username}}",
            "password" : "{{ARG:user_db_password}}"
        }
    }
}
```

This example defines 3 inputs, 1 environment variable and 2 that will be expected as arguments on the kfg CLi. These inputs also contain input constraints and doc comments.
The value add here is that the `kfg` CLi tool understands and uses these definitions, including the constraints and the doc comments to check input, environments, and provide human friendly usage of the CLi on a per kfg file basis.

The constraints are really a key part of the safety that KFG adds. The built-in constraints are:

- `optional` // Since all inputs are by default required, they can be marked as optional if needed (this may only make sense if null coalescing operator is supported which I am still on the fence about)
- `allowedvalues`
- `minlength`
- `maxlength`
- `isnumber`
- `isnumberinrange`
- `ishostname`
- `isurl`
- `isipaddress`
- `isemail`
- `isport`
- `allowswhitespace`
- `nospecial`
- `disallow`
- `regex`

#### TEMPLATE

In the previous example we used implicit templating because no keyword was used before the JSON text started. This will just be ingested as-is and the values in between {{ }} will be replaced.

The other option is to store the template in a separate file and load it using `TEMPLATE`.

##### Example

*app.kfg*
```
DEFINE ARG:password <minlength(12)> /// password for provided user
TEMPLATE "app.json"
```

This example shows the usage of an external json file as the output template where any occurrence of `{{ARG:password}}` would be replaced when generating an output.

#### NULL coalescing operator

> I am on the fence about supporting any kind of conditional logic since as I see it your config schema should be the same across all environments and values will always be only 1 correct value for a given environment.

To enforce safety and avoid using branching logic for evil, the idea would be that this operator only works on inputs that have the `allowedvalues` constraint.

##### Example

*app.kfg*
```
... other definitions for inputs
DEFINE ARG:db_conn <optional> /// The database connection string
DEFINE ENV:db_conn /// The database connection string
DEFINE ENV:environment <allowedvalues("dev","test","acc","prod")> /// The current environment this config is generated for
VAL:db_conn = ARG:db_conn ?? ENV:db_conn
TEMPLATE "app.json"
```

Now templates can use `{{VAL:db_conn}}` and it is possible for to pass in an ARG input that is used instead of the environment variable if supplied.

#### MAP & LIST

`MAP` allows for a kfg specific object structure that would allow for a output to multiple formats such as YAML & Json. Although something like XML would be supported, it would preclude the use of XML attributes.

##### Example

```
MAP [
  "key1" : "value1"
  "keyWithList" : LIST [1,2,3,4]
  "key3" : MAP [
      "innerKey" : "{{ARG:inner_value}}"
  ]
]
```

#### Vaults

> This section on vault is still vague. Ideas welcome.

This specific feature needs to be fleshed out a bit more as I need to look into commonalities and differences between vault implementations.

Possibilities are a keyword per vault type ie. for Azure Key Vault
```
DEFINE AKS:some_key
{{AKV:some_key}}
```

TODO: experiment and flesh this out

```
VAULT AKV:db_password "https://myvault.vault.azure.net"

MAP [
    "db_password" : "{{AKV:db_password}}"
]
```

This assumes Service Principal of executing process.

## CLi

The true value comes from the interaction of the format with the command-line tool. The goal is that each *kfg* file functions like it's own CLi tool that is determined by the `DEFINE` declarations in the *kfg* file.

To illustrate the features, let's walk through the workflow of a developer setting up the project for the first time, all the way to deploying to production.

### set

Scenario: A developer checks out the repo for a project for the first time. What environment variables are needed to run the application? On a project from a mature team this is covered in a setup script, or documentation. 
In either of those cases, it has taken time for someone to do that and keep it up-to-date with the applications current requirements. In most cases, the developer runs the application and starts hunting down why it failed.

Let's take a look at how `kfg` can streamline this.

> NOTE: This setup is just to demonstrate a lot of the features simply and does not indicate best practices for a build pipeline.

*app.kfg*
```
DEFINE ENV:sql_server_address <ipaddress> /// The IP Address for the MS SQL database containing the users
DEFINE ARG:db_username <minlength(8);nospecial> /// Username for a MS SQL user that has read/write access to the user table
DEFINE ARG:db_password <optional;minlength(12)> /// password for provided user
VAULT AKV:db_password "https://myvault.vault.azure.net"
VAL db_password = ARG:db_password ?? AKV:db_password
{
    "dbs" : {
        "user_db" : {
            "server" : "{{ENV:sql_server_address}}",
            "username" : "{{ARG:db_username}}",
            "password" : "{{VAL:udb_password}}"
        }
    }
}
```

Running the command

`kfg set app.kfg -i`

will prompt the developer to interactively set the `DEFINE ENV` values required for a valid setup.

### gen

Next, she can run

`kfg gen app.kfg app.json --db_username sa123456 --db_password sasa123456789

which will generate the needed *app.json* for the application.