# Writing simpleconfig files: the basics {: #writing_simpleconfig_files }

This page explains all the basic information about how to write
simpleconfig files.

For software builds that follow established build patterns, an
simpleconfig is all that you need to create in order to build and install
the software and the corresponding module file.

Luckily, the majority of software delivery mechanisms are being designed
around either autotools or CMake or, perhaps, some simple file
extraction/copy pattern. In that case, a *generic simpleblock* can be
leveraged; see [Overview of generic simpleblocks][generic_simpleblocks].

Yet, in case the software build calls for more elaborate steps
(scientific software never fails to surprise us in this regard), a
software-specific simpleblock may be required; see
[Implementing simpleblocks][implementing_simpleblocks].

## What is an simpleconfig (file)? {: #what_is_an_simpleconfig }

An simpleconfig file serves as a *build specification* for
SimpleBuild.

It consists of a plain text file (in Python syntax) with mostly
*key-value* assignment to define **simpleconfig parameters**.

Simpleconfigs typically follow a (fixed) strict naming scheme, i.e.
`<name>-<version>[-<toolchain>][<versionsuffix>].eb`.

The `-<toolchain>` label (which includes the toolchain name and version)
is omitted when a [`system` toolchain][system_toolchain] is
used. The `<versionsuffix>` label is omitted when the version suffix is
empty.

!!! note
    the filename of an simpleconfig is only important w.r.t.
    dependency resolution (`--robot`), see [Enabling dependency resolution][use_robot].

Example:

``` python
# simpleconfig file for GCC v4.8.3
name = 'GCC'
version = '4.8.3'
...
```

!!! tip
    Comments can be included in simpleconfig files using the hash
    (`#`) character (just like in Python code).

## Available simpleconfig parameters

About 50 different (generic) simpleconfig parameters are supported
currently. An overview of all available simpleconfig parameters is
available via the `-a` command line option.

By default, the parameters specific to generic (default) simpleblock
`ConfigureMake` are included; using `--simpleblock`/`-e` parameters that
are specific to a particular simpleblock can be consulted.

See [Available simpleconfig parameters][vsd_avail_simpleconfig_params] for more details.

Example:

``` console
$ eb -a -e Binary
Available simpleconfig parameters (* indicates specific for the Binary SimpleBlock)
MANDATORY
---------
[..]
name:           Name of software (default: None)
[...]
SIMPLEBLOCK-SPECIFIC
------------------
install_cmd(*):     Install command to be used. (default: None)
[...]
```

## Mandatory simpleconfig parameters

A handful of simpleconfig parameters are *mandatory*:

- **name, version**: specify what software (version) to build
- **homepage, description**: metadata (used for module help)
- **toolchain**: specifies name and version of compiler toolchain to
    use
    - format: dictionary with name/version keys, e.g.,
        `{'name': 'foo', 'version': '1.2.3'}`
    - a list of supported toolchains can be found
        [here](version-specific/toolchains.md)

Remarks:

- some others are planned to be required in the future
    - *docurls, software license, software license urls*

Example:

``` console
name = 'HPL'
version = '2.0'

homepage = 'http://www.netlib.org/benchmark/hpl/'
description = "High Performance Computing Linpack Benchmark"

toolchain = {'name': 'goolf', 'version': '1.4.10'}
[...]
```

## Common simpleconfig parameters

This section includes an overview of some commonly used (optional)
simpleconfig parameters.

### Source files, patches and checksums {: #common_simpleconfig_param_sources }

- **sources**: list of source files (filenames only)
- **source urls**: list of URLs where sources can be downloaded
- **patches**: list of patch files to be applied (`.patch` extension)
- **checksums**: list of checksums for source and patch files

Remarks:

- sources are downloaded (best effort), unless already available
- proxy settings are taken into account, since the [urllib2 Python
    package](https://docs.python.org/2/library/urllib2.html) is used for
    downloading (since SimpleBuild v2.0)
- patches need to be SimpleBuild-compatible
    - unified diff format (`diff -ruN`)
    - patched locations relative to unpacked sources
- see [Patches][common_simpleconfig_param_sources_patches] for more information on `patches`
- see [Checksums][common_simpleconfig_param_sources_checksums] for more information on `checksums`
- `sources` is usually specified as a list of strings representing
    filenames for source files, but other formats are supported too, see
    [Alternative formats for `sources`][common_simpleconfig_param_sources_alt]

Example:

``` python
name = 'HPL'
version = '2.2'

[...]

source_urls = ['http://www.netlib.org/benchmark/hpl']
sources = ['hpl-%(version)s.tar.gz']

# fix Make dependencies, so parallel build also works
patches = ['HPL_parallel-make.patch']

checksums = ['ac7534163a09e21a5fa763e4e16dfc119bc84043f6e6a807aba666518f8df440']

[...]
```

!!! note
    Rather than hardcoding the version (and name) in the list of sources,
    a string template `%(version)s` can be used, see also
    [Dynamic values for simpleconfig parameters][simpleconfig_param_templates].

#### Patches {: #common_simpleconfig_param_sources_patches }

Patches can be provided via the `patches` simpleconfig parameter (list). A
patch can be defined as:

- a `string`,
- a `tuple` of 2 elements, or
- a `dict`

The most straight-forward use-case is `string`, which contains the name
of the patch file (and must have `.patch` extension).

A `tuple` adds the possibility to specify where patch should be applied.
This is mostly needed if a patch file adds new files or it is generally
not possible to determine the starting directory. The first element is
the patch file and the second is either the patch level (as an integer)
which is used in the patch command (`patch -p<n>`) or a directory
relative to the unpacked source dir.

!!! note
    `tuple` also has a special use case if the patch file has any other extension than `.patch`.
    In this case, the first tuple argument is a file which should be
    copied to the unpacked source dir and the second tuple argument is
    the target path, where the files should be copied to (relative to
    the unpacked source dir). See below for an example of this use case.

A `dict` adds the ability to pass the `patch` command additional
arguments. For example, the `--binary` flag is needed to patch files
with CRLF endings. The `dict` has a required `filename` key, with
`level` and `opts` being optional ones.

!!! note
    Specifying only `filename` in `dict` is the same as using a plain `string` definition.
    Specifying `filename` and `level` is same as using a `tuple`
    definition.

Example:

``` python
patches = [
  # a simple patch file
  'name-versions-fix.patch',

  # when creating only new files by patch file, you need to specify the level:
  ('name-versions-fix.patch', 1),

  # apply the patch in a (sub-)subdirectory inside the source tree
  ('name-versions-fix.patch', 'src/subfolder'),

  # copy file to target_path folder
  ('Makefile', 'target_path'),

  # specify patch file and optionally level and opts for patch command
  {'filename': 'name-versions-fix.patch', 'level': 1, 'opts': '-l'}
]
```

#### Checksums {: #common_simpleconfig_param_sources_checksums }

Checksums for source files and patches can be provided via the
`checksums` simpleconfig parameter.

SimpleBuild does not enforce checksums to be available for all source
files and patches. Provided checksums will be 'consumed' first for the
specified sources (in order), and subsequently also for patches.

Nevertheless, providing checksums for *all* source files and patches is
highly recommended.

If checksums are provided, the checksum of the corresponding source
files and patches is verified to match.

The `checksums` simpleconfig parameter is usually defined as a list of
strings.

Until SimpleBuild v3.3.0, only MD5 checksums could be provided through a
list of strings. Since SimpleBuild v3.3.0, the checksum type is determined
by looking at the length of the string:

- 32-character strings are considered to be MD5 checksums (`md5`)
- 64-character strings are considered to be SHA256 checksums
    (`sha256`)
- (other lengths will result in an error message)

The intention is to move towards making `sha256` the recommended and
default checksum type.

Other checksum types are also supported: `adler32`, `crc32`, `sha1`,
`sha512`, `size` (filesize in bytes). To provide checksum values of a
specific type, elements of the `checksums` list can also be 2-element
tuples of the form `('<checksum type>', '<checksum value>')`. For
example:

``` python
checksums = [('sha512', 'f962008105639f58e9a4455c8057933ab0a5e2f43db8340ae1e1afe6dc2d24105bfca3b2e1f79cb242495ca4eb363c9820d8cea6084df9d62c4c3e5211d99266')]
```

##### Adding or replacing checksums using `--inject-checksums` {: #inject_checksums }

Using the `--inject-checksums` command line option, you can let
SimpleBuild add or update checksums in one or more simpleconfig files (which
is significantly more convenient than doing it manually).

With `--inject-checksums`, checksums are injected for all sources and
patches (if any), as well as for all sources & patches of every
extension listed in `exts_list` (if any, see
[Extensions][module_extensions]).

If the sources (& patches) are not available yet, SimpleBuild will try to
download them first; i.e., the `fetch` step is run prior to computing &
injecting the checksums.

A backup is created of every simpleconfig file that is touched by
`--inject-checksums`, to avoid accidental loss of information. Backups
are given an additional extension of the form
`.bak_<year><month><day><hour><min><sec>`.

!!! note
    To clean up backup simpleconfig files, you can use this one-liner:

    ``` shell
    find . -name '*.eb.bak_*' | xargs rm -v
    ```

    The `-v` option makes `rm` print the path of files that are being
    removed.

    **Do use this with care; just run** `find . -name '*.eb.bak_*'`
    **first in case of doubt!**

Multiple simpleconfigs can be specified when using `--inject-checksums`,
they will be processed in sequence. In addition, you can also combine
`--inject-checksums` with `--robot`, see
[Synergy between `--inject-checksums` and `--robot`][inject_checksums_robot_synergy].

###### Adding checksums when none are specified yet {: #inject_checksums_adding }

If the simpleconfig file does not specify any checksums yet, they are
simply injected after the `sources` (or `patches`, if present)
specification when `--inject-checksums` is used.

For example:

``` console
$ eb bzip2-1.0.6.eb --inject-checksums
== temporary log file in case of crash /tmp/eb-Vm6w3e/simplebuild-cAVQl6.log
== injecting sha256 checksums in /example/bzip2-1.0.6.eb
== fetching sources & patches for bzip2-1.0.6.eb...
== backup of simpleconfig file saved to /example/bzip2-1.0.6.eb.bak_20170824200906...
== injecting sha256 checksums for sources & patches in bzip2-1.0.6.eb...
== * bzip2-1.0.6.tar.gz: a2848f34fcd5d6cf47def00461fcb528a0484d8edef8208d6d2e2909dc61d9cd
== Temporary log file(s) /tmp/eb-Vm6w3e/simplebuild-cAVQl6.log* have been removed.
== Temporary directory /tmp/eb-Vm6w3e has been removed.
```

The backup simpleconfig file can be used to double-check the difference
between the original simpleconfig file and the one produced by
`--inject-checksums`:

``` console
$ diff -u /example/bzip2-1.0.6.eb.bak_20170824200906 /example/bzip2-1.0.6.eb
diff --git a//example/bzip2-1.0.6.eb.bak_20170824200906 b/example/bzip2-1.0.6.eb
index 46b2debed..2eb73f15a 100644
--- a/example/bzip2-1.0.6.eb.bak_20170824200906
+++ b/example/bzip2-1.0.6.eb
@@ -9,8 +9,9 @@ compressors), whilst being around twice as fast at compression and six times fas
 toolchain = SYSTEM
 toolchainopts = {'pic': True}

-sources = [SOURCE_TAR_GZ]
 source_urls = ['http://www.bzip.org/%(version)s/']
+sources = [SOURCE_TAR_GZ]
+checksums = ['a2848f34fcd5d6cf47def00461fcb528a0484d8edef8208d6d2e2909dc61d9cd']

 buildopts = "CC=gcc CFLAGS='-Wall -Winline -O3 -fPIC -g $(BIGFILES)'"
```

!!! note
    Along with injecting checksums, SimpleBuild will also reorder the `source_urls`, `sources`
    and `patches` specifications, in that order and if they are present,
    and include the `checksums` specification afterwards. This is done
    to facilitate working towards a uniform style in simpleconfig files,
    which also applies to the order of specified simpleconfig parameters.

###### Replacing existing checksums {: #inject_checksums_replacing }

When one or more checksums are already specified, SimpleBuild requires the
use of `--force` together with `--inject-checksums` to replace those
checksums. A clear warning will be printed to notify that existing
checksums will be replaced.

For example:

``` console
$ eb bzip2-1.0.6.eb --inject-checksums
== temporary log file in case of crash /tmp/eb-WhSwVH/simplebuild-HCODnl.log
== injecting sha256 checksums in /example/bzip2-1.0.6.eb
== fetching sources & patches for bzip2-1.0.6.eb...
ERROR: Found existing checksums, use --force to overwrite them
```

``` console
$ eb bzip2-1.0.6.eb --inject-checksums --force
== temporary log file in case of crash /tmp/eb-dS2QLa/simplebuild-JGxOzC.log
== injecting sha256 checksums in /example/bzip2-1.0.6.eb
== fetching sources & patches for bzip2-1.0.6.eb...

WARNING: Found existing checksums in bzip2-1.0.6.eb, overwriting them (due to use of --force)...

== backup of simpleconfig file saved to /example/bzip2-1.0.6.eb.bak_20170824203850...
== injecting sha256 checksums for sources & patches in bzip2-1.0.6.eb...
...
```

!!! note
    Any existing checksums are *blindly* replaced when
    `--inject-checksums --force` is used: the existing checksums are
    *not verified* to be correct as during normal use of SimpleBuild
    (since that would kind of defeat the purpose of
    `--inject-checksums`).
    In addition, it also doesn't matter whether or not checksums are
    available for all sources & patches: with `--inject-checksums`,
    checksums will be added for *all* sources and patches, including for
    extensions listed in `exts_list` (if any).

###### Synergy between `--inject-checksums` and `--robot` {: #inject_checksums_robot_synergy }

When `--inject-checksums` is combined with `--robot`, checksums are
injected for *each* simpleconfig file in the dependency graph for which no
module is available yet.

For example, to inject checksums in *every* simpleconfig file required to
build HPL 2.2 with the `foss/2017a` toolchain:

``` console
$ MODULEPATH= eb HPL-2.2-foss-2017a.eb --installpath /tmp/$USER/sandbox --inject-checksums --robot
== temporary log file in case of crash /tmp/eb-8HpJc3/simplebuild-H35khM.log
== resolving dependencies ...
...
== injecting sha256 checksums in /example/GCCcore-6.3.0.eb
...
== injecting sha256 checksums in /example/OpenMPI-2.0.2-GCC-6.3.0-2.27.eb
...
== injecting sha256 checksums in /example/FFTW-3.3.6-gompi-2017a.eb
...
== injecting sha256 checksums in /example/HPL-2.2-foss-2017a.eb
...
```

!!! note
    We are clearing `$MODULEPATH` and specifying a custom (empty) location to `--installpath` to
    avoid that SimpleBuild skips any simpleconfig because a corresponding
    module is already available.

###### Type of checksum to inject {: #inject_checksums_type }

By default, `--inject-checksums` will compute & inject `SHA256`
checksums, but a different checksum type can be specified as an argument
(e.g., `--inject-checksums md5`).

!!! note
    Because of the optional argument that can be passed to `--inject-checksums`,
    you should not specify an simpleconfig file name directly after the
    `--inject-checksums`, since it will be assumed to specify a checksum
    type, which will result in an error message like:

    ``` console
    $ eb --inject-checksums bzip2-1.0.6.eb
    Usage: eb [options] simpleconfig [...]

    eb: error: option --inject-checksums: invalid choice: 'bzip2-1.0.6.eb' (choose from 'adler32', 'crc32', 'md5', 'sha1', 'sha256', 'sha512', 'size')
    ```

#### Alternative formats for `sources` {: #common_simpleconfig_param_sources_alt }

In some cases, it can be required to provide additional information next
to the name of a source file, e.g., a custom extraction command (because
the one derived from the file extension is not correct), or an altername
filename that should be used to download the source file.

This can be specified using a Python dictionary value in the `sources`
simpleconfig parameter.

Since SimpleBuild v3.3.0, three keys are supported:

- `filename` (*mandatory*): filename of source file
- `download_filename`: filename that should be used when downloading
    this source file; the downloaded file will be saved using the
    `filename` value
- `extract_cmd`: custom extraction command for this source file
- `source_urls`: source URLs to consider for downloading this source
    file
- `git_config`: see
    [Downloading from a Git repository][common_simpleconfig_param_sources_git_config]

For example:

``` python
sources = [{
    'source_urls': ['https://example.com'],
    'filename': 'example-%(version)s.gz',
    'download_filename': 'example.gz',  # provided source tarball is not versioned...
    'extract_cmd': "tar xfvz %s",  # source file is actually a gzipped tarball (filename should be .tar.gz)
}]
```

!!! note
    Custom extraction commands can also be specified as a 2-element tuple, but this format has been deprecated
    in favour of the Python dictionary format described above; see also
    [Specifying source files as 2-element tuples to provide a custom extraction command][depr_sources_2_element_tuple].

#### Using `download_instructions` for user-side part of installation

In some cases, getting some of the files required for an installation
cannot be automated. Reasons for this include:

- there is a manual stage to combine multiple downloaded files into
    the required installation file
- the file requires a login to download

You can use the `download_instructions` parameter to specify steps for
the user to do. This parameter takes string value and prints it whenever
build fails because any file needed was not found. If
`download_instructions` is not specified, Simplebuild prints the default
message stating the paths that were tested.

``` python
download_instructions = """
  Step 1: Go to example.com and download example.jar.
  Step 2: Install example.jar (ensure Java is installed).
  Step 3: Go to the installation directory and create Tarball of its contents (tar -czvf example.tar.gz *).
  Step 4: Move created Tarball into the same directory, where your simpleconfig is located and run build again.
"""
```

#### Downloading from a Git repository {: #common_simpleconfig_param_sources_git_config }

Since SimpleBuild v3.7.0, support for downloading directly from a Git
repository is available.

When `git_config` is provided for a particular source file (see
[Alternative formats for `sources`][common_simpleconfig_param_sources_alt]),
SimpleBuild will create a source tarball after downloading the specified
Git repository.

The value for `git_config` is a Python dictionary, where the following
keys are *mandatory*:

- `url`: the URL where the Git repository is located
- `repo_name`: the name of the Git repository

The value for `filename` in the source specification *must* end in
`.tar.gz` (because a gzipped tarball will be created from the cloned
repository).

In addition, either of the following keys *must* also be defined:

- `tag`: the specific tag to download
- `commit`: the specific commit ID to download

If a recursive checkout should be made of the repository, the
`recursive` key can be set to `True`.

To also retain the `.git` directory (which holds the Git metadata for
the repository), you can set the `keep_git_dir` to `True` (supported
since SimpleBuild v4.2.0).

A different name for the top-level directory can be specified via
`clone_into`; by default the name of the repository is used.

Examples:

- creating a source tarball named `example-main.tar.gz` of the
    `main` branch of a (fictional) `test` repository from
    `https://agitserver.org/example`, and use `example-test` as
    top-level directory name:

    ``` console
    sources = [{
        'filename': 'example-main.tar.gz',
        'git_config': {
            'url': 'https://agitserver.org/example',
            'repo_name': 'test',
            'tag': 'main',
            'clone_into': 'example-test',
        },
    }]
    ```

- creating a source tarball named `example-20180920.tar.gz` of the
    recursive checkout of commit `abcdef12` of the `test` repository
    from `https://agitserver.org/example`:

    ``` console
    sources = [{
        'filename': 'example-20180920.tar.gz',
        'git_config': {
            'url': 'https://agitserver.org/example',
            'repo_name': 'test',
            'commit': 'abcdef12',
            'recursive': True,
            'keep_git_dir': True,
        },
    }]
    ```

!!! note
    Because the source tarball is created locally (by running `tar cfvz` on the directory containing
    the cloned repository), the (SHA256) checksum is not guaranteed to
    be the same across different systems.

    Whenever you have the option to download a source tarball (or
    equivalent) directly (for example from GitHub, which also allows
    downloading a tarball of a specific commit), we strongly recommend
    you to do so, `git_config` is intended for other Git repos.

### Dependencies {: #dependency_specs }

- **dependencies**: build/runtime dependencies
- **builddependencies**: build-only dependencies (not in module)
- **hiddendependencies**: dependencies via hidden modules (see also
    [Installing dependencies as hidden modules using `--hide-deps`][hide_deps])
- **osdependencies**: system dependencies (package names)

Remarks:

- modules must exist for all (non-system) dependencies
- (non-system) dependencies can be resolved via `--robot`
- format: `(<name>, <version>[, <versionsuffix>[, <toolchain>]])`

Example:

``` python
name = 'GTI'
...
toolchain = {'name': 'goolf', 'version': '1.5.14'}
dependencies = [('PnMPI', '1.2.0')]
builddependencies = [('CMake', '2.8.12', '', ('GCC', '4.8.2'))]
```

For each of the specified (build) dependencies, the corresponding module
will be loaded in the build environment defined by SimpleBuild. For the
*runtime* dependencies, `module load` statements will be included in the
generated module file.

!!! note
    By default, SimpleBuild will try to resolve dependencies using the same toolchain as specified for the
    software being installed. As of v3.0, if no simpleconfig exists to
    resolve a dependency using the default toolchain SimpleBuild will
    search for the dependency using a compatible subtoolchain.

    A different toolchain can be specified on a per-dependency level
    (cfr. the `CMake` build dependency in the example above).

    Alternatively, you can instruct SimpleBuild to use the most minimal
    (sub)toolchain when resolving dependencies, see
    [Using minimal toolchains for dependencies][minimal_toolchains].

#### Loading of modules for dependencies with a `system` toolchain

When a [`system` toolchain][system_toolchain] is used, the
modules for each of the (build) dependencies are *always* loaded,
regardless of the toolchain version (as opposed the behaviour with the
`dummy` toolchain in SimpleBuild versions prior to v4.0, see
[Motivation for deprecating the `dummy` toolchain][system_toolchain_motivation_deprecating_dummy]).

#### Specifying dependencies using `system` toolchain

To make SimpleBuild resolve a dependency using the `system` toolchain,
simply use the `SYSTEM` template constant as the 4th value in the tuple
representing the dependency specification.

For example, to specify PnMPI version 1.2.0 built with the `system`
toolchain as a (runtime) dependency:

``` python
dependencies = [('PnMPI', '1.2.0', '', SYSTEM)]
```

#### Using external modules as dependencies

Since SimpleBuild v2.1, specifying modules that are not provided via
SimpleBuild as dependencies is also supported. See
[Using external modules][using_external_modules] for more
information.

### Extensions {: #module_extensions }

Besides dependencies, which are found outside the software being built
but are part of the site's SimpleBuild installation, it is also possible
to incorporate extensions to the software within the build. This is done
via the `exts_list` array.

Each entry in `exts_list` is a three-component tuple, with the name and
version number, and a dictionary of configuration options for the entry:

``` python
exts_list = [
    ('name', 'version', { 'option':'value', 'option':'value' })
]
```

The latter may contain essentially any of the full simpleconfig
parameters, including `buildopts`, `installopts`, etc. Among those
options, the following exceptions and special cases should be noted:

- **nosource**: If set `True`, no download will be done
- **source_tmpl**: Template string for the file to be downloaded
    - default is `'%(name)s-%(version)s.tar.gz'`
    - `%(name)s` and `%(version)s` come from the `exts_list` entry
        (above)
- **sources**: Dictionary specifying details of where to download the
    extension
    - equivalent to a single entry from the simpleconfig `sources` list
    - preferred to use of `source_tmpl`
- **start_dir**: If not set, will be derived; the simpleconfig value
    will not be used

``` python
exts_list = [
    ('llvmlite', '0.26.0', {
        'source_urls': ['https://pypi.python.org/packages/source/l/llvmlite/'],
        'patches': ['llvmlite-0.26.0_fix-ffi-Makefile.patch'],
        'checksums': [
            '13e84fe6ebb0667233074b429fd44955f309dead3161ec89d9169145dbad2ebf',    # llvmlite-0.26.0.tar.gz
            '40e6fe6de48709b45daebf8082f65ac26f73a4afdf58fc1e8099b97c575fecae',    # llvmlite-0.26.0_fix-ffi-Makefile.patch
        ],
    }),
    ('singledispatch', '3.4.0.3', {
        'source_urls': ['https://pypi.python.org/packages/source/s/singledispatch/'],
        'checksums': ['5b06af87df13818d14f08a028e42f566640aef80805c3b50c5056b086e3c2b9c'],
    }),
    (name, version, {
        'source_urls': ['https://pypi.python.org/packages/source/n/numba/'],
        'checksums': ['c62121b2d384d8b4d244ef26c1cf8bb5cb819278a80b893bf41918ad6d391258'],
    }),
]
```

That third instance uses the `name` and `version` variables defined in
the simpleconfig file. Since SimpleBuild v4.2.2, a single-entry `sources`
dictionary (see [Alternative formats for `sources`][common_simpleconfig_param_sources_alt]) may be included in an `exts_list` entry. For example, to
download Git sources from a private repository and convert them to a
tar-ball for installation:

``` python
exts_defaultclass = 'PythonPackage'
exts_list = [
    ('pyCAP', '0.1', {
        'sources': {
            'filename': '%(name)s-%(version)s.tar.gz',
            'git_config': {
                'url': 'ssh://nero.stanford.edu/data/git/Analysis',
                'repo_name': 'pyCAP',
                'tag': '%(version)s',
            }
        }
    }),
]
```

Here, the template strings `%(name)s` and `%(version)s` will be
substituted from the `exts_list` entry elements ("pyCAP" and "0.1",
respectively), not from the simpleconfig values.

### Configure/build/install command options {: #configure_build_install_command_options }

- **configopts**: options for configure command
- **preconfigopts**: options used as prefix for configure command

In analogy to *configure*, also *build* and
*install* commands are tuneable:

- **buildopts, prebuildopts**: options for build command
- **installopts, preinstallopts**: options for install command

Example:

``` python
simpleblock = 'ConfigureMake'
...
# configure with: ./autogen.sh && ./configure CC="$CC" CFLAGS="$CFLAGS"
preconfigopts = "./autogen.sh && "
buildopts = 'CC="$CC" CFLAGS="$CFLAGS"'
# install with: make install PREFIX=<installation prefix>
installopts = 'PREFIX=%(installdir)s'
```

!!! note
    For more details w.r.t. use of string templates like
    `%(installdir)s`, see [Dynamic values for simpleconfig parameters][simpleconfig_param_templates].

#### List of configure/build/install options {: #configure_build_install_command_options_iterate }

In some cases, the *configure-build-install* cycle must be executed
multiple times during a single installation, using different options for
one or more steps.

SimpleBuild supports specifying a *list* of strings, each of which
specifying a particular set of options to use.

For example, to perform the installation procedure with three different
sets of configuration options:

``` python
configopts = [
    "--common-opt --one --one-more",
    "--common-opt --two",
    "--common-opt --three",
]
```

This way, SimpleBuild will perform the *configure-build-install* cycle
**three** times:

- configure using `--common-opt --one --one-more`, build and install
- configure using `--common-opt --two`, build and install on top of
    the existing installation
- configure using `--common-opt --three`, build and install once more
    on top of what is installed already

During this process, the environment is reset and the build directory is
cleaned up after each cycle, while the installation directory is left
untouched (in order to not destroy the result of earlier cycles).

If several `(pre){config|build|install}opts` parameters are defined as
being a list of strings, the number of items in the lists must be the
same. Any of these parameters defined as a single string value are just
reused for each of the cycles performed. For example:

``` python
simpleblock = 'ConfigureMake'
configopts = ['--one', '--two', '--three']
buildopts = 'lib'
preinstallopts = ['TYPE=one', 'TYPE=two', 'TYPE=three']
```

would result in:

- `./configure --prefix=... --one; make lib; TYPE=one make install`
- `./configure --prefix=... --two; make lib; TYPE=two make install`
- `./configure --prefix=... --three; make lib; TYPE=three make install`

An example use case of this is building FFTW with different precisions,
see the [FFTW simpleconfig
files](https://github.com/simplebuilders/simplebuild-simpleconfigs/tree/main/simplebuild/simpleconfigs/f/FFTW).

### Sanity check

Custom paths and commands to be used in the sanity check step can be
specified using the respective parameters. These are used to make sure
that an installation didn't (partly) fail unnoticed.

- **sanity_check_paths**: files/directories that must get installed
- **sanity_check_commands**: (simple) commands that must work when the
    installed module is loaded

Remarks:

- format: Python dictionary with (*only*) `files`/`dirs`
    keys
- values must be lists of (tuples of) strings, one of both **must** be
    non-empty
    - paths are *relative* to installation directory
    - for a path specified as a tuple, only one of the specified paths
        must be available
- default values:
    - paths: non-empty `bin` and `lib` or `lib64` directories
    - commands: none

Example:

``` python
sanity_check_paths = {
    'files': ["bin/xhpl"],
    'dirs': [],
}
```

### Simpleblock specification {: #writing_simpleconfigs_simpleblock_spec }

To make SimpleBuild use a specific (usually generic) simpleblock the
**simpleblock** parameter can be used.

By default, SimpleBuild will assume that the simpleblock to use can be
derived from the software name. For example: for `GCC`, SimpleBuild will
look for an simpleblock class named `EB_GCC` in the Python module
`simplebuild.simpleblocks.gcc`.

A list of available simpleblocks is available via `--list-simpleblocks` (see
also [List of available simpleblocks][list_simpleblocks]); generic
simpleblocks are the ones for which the name does *not* start with `EB_`.

Example:

``` python
simpleblock = 'CMakeMake'
name = 'GTI'
version = '1.2.0'
...
```

!!! tip
    It is highly recommended to use existing (generic) simpleblocks, where
    applicable. This avoids the need for creating (and maintaining) new
    simpleblocks. Typically, generic simpleblocks support several custom
    simpleconfig parameters which allow to steer their behavior (see also
    [All available simpleconfig parameters][avail_simpleconfig_params]).

Example:

``` python
simpleblock = 'Binary'
[...]
install_cmd = "./install.bin"
[...]
```

### Module class

The category to which the software belongs to can be specified using the
**moduleclass** simpleconfig parameter. By default, the `base` module
class is used (which should be replaced with a more appropriate
category).

SimpleBuild enforces that only known module classes can be specified (to
avoid misclassification due to typos).

The default list of module classes is available via
`--show-default-moduleclasses`; additional module classes can be defined
via the `--moduleclasses` configure option.

Example:

``` python
name = 'GCC'
[...]
moduleclass = 'compiler'
```

!!! note
    By default, SimpleBuild will create a symlink to the generated module file in a module class-specific path.
    This behavior is configurable through the module naming scheme being
    used.

!!! tip
    The module class may play a significant role in other aspects. For example, the alternative (hierarchical)
    module naming scheme `HierarchicalMNS` heavily relies on the
    `moduleclass` parameter for discriminating compilers and MPI
    libraries.

## Tweaking existing simpleconfig files

The ability to modify simpleconfig files on the fly with SimpleBuild,
provides a very powerful and flexible feature to describe builds,
without having to manually create all the input files.

Tweaking existing simpleconfigs can be done using the `--try-*` command
lines options. See [Tweaking existing simpleconfig files][tweaking_simpleconfigs_using_try] for more details.

Example:

- GCC version update:

    ``` shell
    eb GCC-4.9.0.eb --try-software-version=4.9.1
    ```

- install WRF + its dozen dependencies with a different toolchain (!):

    ``` shell
    eb WRF-3.5.1-ictce-5.3.0-dmpar.eb --try-toolchain=intel,2014b -r
    ```

## Dynamic values for simpleconfig parameters {: #simpleconfig_param_templates }

String templates are completed using the value of particular simpleconfig
parameters, typically `name` and/or `version`. These help to avoid
hardcoding values in multiple locations.

A list of available string templates can be obtained using
`--avail-simpleconfig-templates`.

Additionally, constants that can be used in simpleconfig files are
available via `--avail-simpleconfig-constants`.

Example:

``` python
name = 'GCC'
version = '4.8.3'
...
source_urls = [
    # http://ftpmirror.gnu.org/gcc/gcc-4.8.3
    'http://ftpmirror.gnu.org/%(namelower)s/%(namelower)s-%(version)s',
]
sources = [SOURCELOWER_TAR_GZ]  # gcc-4.8.3.tar.gz
...
```

!!! note
    Proper use of string templates is important, in particular to avoid hardcoding the software version
    in multiple locations of an simpleconfig file; this is critical to
    make `--try-software-version` behave as expected (see also
    [Tweaking existing simpleconfig files][tweaking_simpleconfigs_using_try]).

## Version-specific documentation relevant to simpleconfigs

- [Available config file constants][avail_cfgfile_constants]
- [Available simpleconfig parameters][vsd_avail_simpleconfig_params]
- [Constants available for simpleconfig files][avail_simpleconfig_constants]
- [License constants available for simpleconfig files][avail_simpleconfig_licenses]
- [List of available simpleblocks][vsd_list_simpleblocks]
- [List of available toolchain options][avail_toolchain_opts]
- [List of known toolchains](version-specific/toolchains.md)
- [List of supported software][list_software]
- [Overview of generic simpleblocks][generic_simpleblocks]
- [Templates available for simpleconfig files][avail_simpleconfig_templates]

## Contributing simpleconfigs

**Contribute your working simpleconfig files!**

Share your expertise with the community, avoid duplicate work,
especially if:

- the software package is not supported yet
- an existing simpleconfig needs (non-trivial) changes for a different
    version/toolchain
- it is a frequently used software package (compilers, MPI, etc.)

See [Contributing][contributing] for more information.
