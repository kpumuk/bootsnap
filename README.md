# Bootsnap

**Beta-quality. See [the last section of this README](#trustworthiness).**

Bootsnap is a library that plugs into a number of ruby and (optionally) `ActiveSuport` and `YAML`
methods. These methods are modified to cache results of expensive computation, and can be grouped
into two broad categories:

* [Path Pre-Scanning](#path-pre-scanning)
    * `Kernel#require` and `Kernel#load` are modified to eliminate `$LOAD_PATH` scans.
    * `ActiveSupport::Dependencies.{autoloadable_module?,load_missing_constant,depend_on}` are
      overridden to eliminate scans of `ActiveSupport::Dependencies.autoload_paths`.
* [Compilation caching](#compilation-caching)
    * `RubyVM::InstructionSequence.load_iseq` is implemented to cache the result of ruby bytecode
      compilation.
    * `YAML.load_file` is modified to cache the result of loading a YAML object in MessagePack format
      (or Marshal, if the message uses types unsupported by MessagePack).

### Path Pre-Scanning

*(This work is a minor evolution of [bootscale](https://github.com/byroot/bootscale)).*

Upon initialization of bootsnap or modification of the path (e.g. `$LOAD_PATH`),
`Bootsnap::LoadPathCache` will fetch a list of requirable entries from a cache, or, if necessary,
perform a full scan and cache the result.

Later, when we run (e.g.) `require 'foo'`, ruby *would* iterate through every item on our
`$LOAD_PATH` `['x', 'y', ...]`,  looking for `x/foo.rb`, `y/foo.rb`, and so on. Bootsnap instead
looks at all the cached requirables for each `$LOAD_PATH` entry and substitutes the full expanded
path of the match ruby would have eventually chosen.

If you look at the syscalls generated by this behaviour, the net effect is that what would
previously look like this:

```
open  x/foo.rb # (fail)
# (imagine this with 500 $LOAD_PATH entries instead of two)
open  y/foo.rb # (success)
close y/foo.rb
open  y/foo.rb
...
```

becomes this:

```
open y/foo.rb
...
```

Exactly the same strategy is employed for methods that traverse
`ActiveSupport::Dependencies.autoload_paths` if the `autoload_paths_cache` option is given to
`Bootsnap.setup`.

The following diagram flowcharts the overrides that make the `*_path_cache` features work.

![Flowchart explaining
Bootsnap](https://cloud.githubusercontent.com/assets/3074765/24532120/eed94e64-158b-11e7-9137-438d759b2ac8.png)

Bootsnap classifies path entries into two categories: stable and volatile. Volatile entries are
scanned each time the application boots, and their caches are only valid for 30 seconds. Stable
entries do not expire -- once their contents has been scanned, it is assumed to never change.

The only directories considered "stable" are things under the ruby install prefix
(`RbConfig::CONFIG['prefix']`, e.g. `/usr/local/ruby` or `~/.rubies/x.y.z`), and things under the
`Gem.path` (e.g. `~/.gem/ruby/x.y.z`). Everything else is considered "volatile".

In addition to the [`Bootsnap::LoadPathCache::Cache`
source](https://github.com/Shopify/bootsnap/blob/master/lib/bootsnap/load_path_cache/cache.rb),
this diagram may help clarify how entry resolution works:

![How path searching works](https://cloud.githubusercontent.com/assets/3074765/24532143/18278cd6-158c-11e7-8250-78d831df70db.png)

It's also important to note how expensive `LoadError`s can be. If ruby invokes
`require 'something'`, but that file isn't on `$LOAD_PATH`, it takes `2 *
$LOAD_PATH.length` filesystem accesses to determine that. Bootsnap caches this
result too, raising a `LoadError` without touching the filesystem at all.

### Compilation Caching

*(A simpler implementation of this concept can be found in [yomikomu](https://github.com/ko1/yomikomu)).*

Ruby has complex grammar, an parsing it is not a particularly cheap operation. Since 1.9, ruby has
translated ruby source to an internal bytecode format, which is then executed by the ruby VM. Since
2.2, ruby [exposes an API](https://ruby-doc.org/core-2.3.0/RubyVM/InstructionSequence.html) that
allows caching that bytecode. This allows us to bypass the relatively-expensive compilation step on
subsequent loads of the same file.

We also noticed that we spend a lot of time loading YAML documents during our application boot, and
that MessagePack and Marshal are *much* faster at deserialization than YAML, even with a fast
implementation. We use the same strategy of compilation caching for YAML documents, with the
equivalent of ruby's "bytecode" format being a MessagePack document (or, in the case of YAML
documents with types unsupported by MessagePack, a Marshal stream).

These compilation results are stored using `xattr`s on the source files. This is likely to change in
the future, as it has some limitations (notably precluding Linux support except where the user feels
like changing mount flags). However, this is a very performant implementation. 

Whereas before, the sequence of syscalls generated to `require` a file would look like:

```
open    /c/foo.rb -> m
fstat64 m
close   m
open    /c/foo.rb -> o
fstat64 o
fstat64 o
read    o
read    o
...
close   o
```

With bootsnap, we get:

```
open      /c/foo.rb -> n
fstat64   n
fgetxattr n
fgetxattr n
close     n
```

Bootsnap writes two `xattrs` attached to each file read:

* `user.aotcc.value`, the binary compilation result; and
* `user.aotcc.key`, a cache key to determine whether `user.aotcc.value` is still valid.

The key includes several fields:

* `version`, hardcoded in bootsnap. Essentially a schema version;
* `compile_option`, which changes with `RubyVM::InstructionSequence.compile_option` does;
* `data_size`, the number of bytes in `user.aotcc.value`, which we need to read it into a buffer
  using `fgetxattr(2)`;
* `ruby_revision`, the version of ruby this was compiled with; and
* `mtime`, the last-modification timestamp of the source file when it was compiled.

If the key is valid, the result is loaded from the value. Otherwise, it is regenerated and clobbers
the current cache.

This diagram may help elucidate things:

![Compilation Caching](https://burkelibbey.s3.amazonaws.com/bootsnap-compile-cache.png)

### Putting it all together

Imagine we have this file structure:

```
/
├── a
├── b
└── c
    └── foo.rb
```

And this `$LOAD_PATH`:

```
["/a", "/b", "/c"]
```

When we call `require 'foo'` without bootsnap, ruby would generate this sequence of syscalls:


```
open    /a/foo.rb -> -1
open    /b/foo.rb -> -1
open    /c/foo.rb -> n
close   n
open    /c/foo.rb -> m
fstat64 m
close   m
open    /c/foo.rb -> o
fstat64 o
fstat64 o
read    o
read    o
...
close   o
```

With bootsnap, we get:

```
open      /c/foo.rb -> n
fstat64   n
fgetxattr n
fgetxattr n
close     n
```

If we call `require 'nope'` without bootsnap, we get:

```
open    /a/nope.rb -> -1
open    /b/nope.rb -> -1
open    /c/nope.rb -> -1
open    /a/nope.bundle -> -1
open    /b/nope.bundle -> -1
open    /c/nope.bundle -> -1
```

...and if we call `require 'nope'` *with* bootsnap, we get...

```
# (nothing!)
```

## Usage

Add `bootsnap` to your `Gemfile`:

```ruby
gem 'bootsnap'
```

Next, add this to your boot setup immediately after `require 'bundler/setup'` (i.e. as early as
possible: the sooner this is loaded, the sooner it can start optimizing things)

```ruby
require 'bootsnap'
Bootsnap.setup(
  cache_dir:            'tmp/cache', # Path to your cache
  development_mode:     ENV['MY_ENV'] == 'development',
  load_path_cache:      true,        # Should we optimize the LOAD_PATH with a cache?
  autoload_paths_cache: true,        # Should we optimize ActiveSupport autoloads with cache?
  disable_trace:        false,       # Sets `RubyVM::InstructionSequence.compile_option = { trace_instruction: false }`
  compile_cache_iseq:   true,        # Should compile Ruby code into ISeq cache?
  compile_cache_yaml:   true         # Should compile YAML into a cache?
)
```

**Protip:** You can replace `require 'bootsnap'` with `BootLib::Require.from_gem('bootsnap',
'bootsnap')` using [this trick](https://github.com/Shopify/bootsnap/wiki/Bootlib::Require). This
will help optimize boot time further if you have an extremely large `$LOAD_PATH`.

## Trustworthiness

We use the `*_path_cache` features in production and haven't experienced any issues in a long time.

The `compile_cache_*` features work well for us in development on macOS, but probably don't work on
Linux at all.

`disable_trace` should be completely safe, but we don't really use it because some people like to
use tools that make use of `trace` instructions.

| feature | where we're using it |
|-|-|
| `load_path_cache` | everywhere |
| `autoload_path_cache` | everywhere |
| `disable_trace` | nowhere, but it's safe unless you need tracing |
| `compile_cache_iseq` | development, unlikely to work on Linux |
| `compile_cache_yaml` | development, unlikely to work on Linux |
