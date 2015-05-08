# TextRender

Generating structured text is best done with [`CAF::TextRender`][caf_textrender_docs].
This document guides through the usage and testing of `CAF::TextRender`.

Using `ncm-metaconfig`, which is the metacomponent built around
`CAF::TextRender`, is described [here][metaconfig].

[caf_textrender_docs]: http://docs-test-caf.readthedocs.org/en/latest/CAF/CAF::TextRender
[metaconfig]: https://github.com/quattor/configuration-modules-core/Metaconfig.md

# CAF::TextRender

Basic usage has 2 main modes:
 * generate text : the `CAF::TextRender` instance has auto-stringification

```perl
use CAF::TextRender;
my $module = 'mymodule';
my $trd = CAF::TextRender->new($module, $contents, log => $self);
print "$trd"; # stringification
```

 * write text to file : get a `CAF::FileWriter` instance with text from `CAF::TextRender` instance

```perl
use CAF::TextRender;
$module = "mymodule";
$trd = CAF::TextRender->new($module, $contents, log => $self);
my $fh = $trd->filewriter('/some/path');
die "Problem rendering the text" if (!defined($fh));
$fh->close();
```

Besides the logger, the 2 main parameters are the `module` and the `contents`.
The `contents` is a hash-reference with the data that is used to generate
the text (e.g. from a `$cfg->getElement('/some/pan/path')->getTree()`).

The `module` is what defines how the text is generated.

It is either one of the following reserved values
 * *json* (using `JSON::XS`)
 * *yaml* (using `YAML::XS`),
 * *properties* (using `Config::Properties`),
 * *tiny* (using `Config::Tiny`),
 * *general* (using `Config::General`)

(The builtin modules can have issues with reproducability, e.g. ordering or a default timestamp.)

Or, for any other value, `Template::Toolkit` (TT) is used,
and the `module` then indicates the relative path of the template to use.
The absolute path of the TT files is determined by 2 optional parameters:
the absolute `includepath` (defaults to `/usr/share/templates/quattor`)
shouldn't be modified, but the `relpath` (defaults to `metaconfig`) should.

A module `mytest/main` with relpath `mycode` will use a
TT file `/usr/share/templates/quattor/mycode/mytest/main.tt`.
The `relpath` is important for creating the TT files: when the
`INCLUDE` directive is used, TT searches starting from the `includepath`,
so in this example the `main.tt` might look like

```
[% data.name %]
[% INCLUDE 'shared/data' %]
```

which will look for the absolute file `/usr/share/templates/quattor/shared/data.tt`.

`CAF::TextRender` does not allow you to include files from a directory lower then `relpath` (e.g. `module` `../cleverhack` will not work).

## Template::Toolkit

[`Template::Toolkit`][TT_home] is a templating framework

Example template
```
Hello [% world %]

```

with content a perl hashref

```perl
{ world => 'Quattor' }
```

will generate

```
$ perl -e 'use Template; my $tttext="Hello [% world %]\n"; Template->new()->process(\$tttext, { world => "Quattor" });'
Hello Quattor
```

Further information on TT:
 * [A nice writeup of the basics of TT][TT_basics_pnce]
 * [TT examples section][TT_home_examples]
 * [Older TT PCmag article][TT_linuxmag_old] (but some examples are outdated)
 * [ncm-metaconfig TT files][ncm_metaconfig_TT_subdir] (TT files are in the subdirectories)

[TT_home]: http://www.template-toolkit.org/index.html
[TT_home_examples]: http://www.template-toolkit.org/about.html#section_Examples
[TT_basics_pnce]: http://www.physics.umd.edu/pnce/pcs-docs/WebDesign/tt_basics.html
[TT_linuxmag_old]: http://www.stonehenge.com/merlyn/LinuxMag/col60.html
[ncm_metaconfig_TT_subdir]: https://github.com/quattor/configuration-modules-core/tree/master/ncm-metaconfig/src/main/metaconfig

### Minimal version

As quattor supports `EL5` and because the templating framework is deeply integrated in e.g. `CCM`, the minimal
required version of the TT framework is 2.18.

This is a rather old version, with some notable missing VMethods compared to recent ones, in particular the scalar
methods `.lower` and `.upper` do not work, one should use resp `FILTER lower` and `FILTER upper`.

Value based unittests are essential to detect any differences across the supported OS.

### Newline / chomp behaviour

TT can easily generate unwanted/unneeded newlines.
The [`chomp` behaviour][TT_whitespace_chomp] can be summarised as follows

Name     |  Tag Modifier
---------|--------------
NONE     |       +
ONE      |       -
COLLAPSE |       =
GREEDY   |       ~

[TT_whitespace_chomp]: http://www.template-toolkit.org/docs/manual/Config.html#section_PRE_CHOMP_POST_CHOMP



# Test::Quattor::RegexpTest

Testing the generated text (and in particular the TT files used to generate it)
can be done via regular expressions and e.g. the `like` method from `Test::More`.

[`Test::Quattor::RegexpTest`][regexptest_docs] provides an easy way to do this.

A `RegexpTest` is a text file with 3 blocks separated by a `---` marker.

The first block is the description, the second block a list of flags (one per line)
and the third block has all the regular expressions.

An example RegexpTest looks like

    Verify mycode
    ---
    ---
    ^line 1
    ^line 3

with an empty flags block (using the defaults `ordered` and `multiline`).

If we create a file `src/test/resources/rt_mycode` with this content, we can now test
generated text against this RegexpTest using


```perl
use Test::Quattor::RegexpTest;
use CAF::TextRender;
my $module = 'mymodule';
my $trd = CAF::TextRender->new($module, $contents, log => $self);
my $rt = Test::Quattor::RegexpTest->new(
    regexp => 'src/test/resources/rt_mycode',
    text => "$trd",
    );
$rt->test();
```

With the default flags, each line is compiled as a multiline regular expression and matched against the text.
The matches are also checked if they are ordered. In the example above `line 3` is expected to match in the text
following `line 1`. But it does not need to be the next line (e.g. there could be a `line 2` in between).
Each match is a test and each verification of the ordering also.

[regexptest_docs]: http://docs-test-maven-tools.readthedocs.org/en/latest/maven-tools/RegexpTest/

# CCM::TextRender

Starting from the `15.4` release, one can render text using a `CCM::Element` instance as `contents`,
instead of hash references. (This is also what `ncm-metaconfig` (since 15.4) and the test
framework (since `1.44`) use).

For this purpose, the `EDG::WP4::CCM::TextRender` module was created as a drop-in replacement
for `CAF::TextRender` (`CCM::TextRender` is a subclass of `CAF::TextRender`; and a hash reference
as `contents` is still supported).

```perl
use EDG::WP4::CCM::TextRender;
my $element = $config->getElement("/my/path", element => {doublequote => 1});
my $module = 'mymodule';
my $trd = EDG::WP4::CCM::TextRender->new($module, $element, log => $self);
print "$trd"; # stringification
```

By default, `CCM::TextRender` will do the following
 * the is a `CCM` variable namespace inserted with following data and methods
  * `CCM.contents` a (copy of) the contents hash references. This gives you access to e.g. the varaibles via
```
[% FOREACH pair IN CCM.contents.pairs %]
[% pair.key %] = [% pair.value %]
[% END %]
```
   (By default, there is no clean way to get all the variables (i.e. keys from hash reference) passed via `contents`)
  * extra methods
   * `CCM.ref()` returns the (internal) perl type
   * `CCM.is_list()`, `CCM.is_hash` and `CCM.is_scalar()` return if something is resp.  a list, hash or scalar
   * `CCM.escape()` and `CCM.unescape()` the `escape` and `unescape` methods

* if contents is an element instance
  * use `$element->getTree` to generate the hash reference that is passed on as `contents` to TT;
    options for `getTree` are passed via the `element` option
  * all pan scalars (`boolean`, `string`, `long` and `double`) are converted to `CCM::TT::Scalar` instances
  * `CCM.element.path` a (printable) `CCM::Path` instance derived with `$element->getPath`  (new in (15.6))

## `element` option

Options for `getTree` are passed via as a hasref via the `element` option.

There are a number of predefined conversions
 * `doublequote`, `singlequote` wraps any (pan type) string in double or single quotes (not type aware)
 * `yesno`, `truefalse` (and the uppercase vairants `YESNO` and `TRUEFALSE`) converts boolean
    to resp. `yes`/`no` and `true`/`false`

For more details, see the `CCM::TextRender` documentation

## CCM::TT::Scalar

The `CCM::TT::Scalar` instances in TT give you access to the scalar types in TT via some custom VMethods
(together with the usual TT scalar VMethods).
Additional methods are (
 * `.is_boolean`, `.is_string`, `.is_double` and `is_long` return if the variable is resp. a boolean, string, double or long
 * `.get_value` return the value
 * `.get_type` return the type

Warning: when using `JSON` templates, access to the pan `long` and `double` requires CCM typed JSON (via the
`json_typed` configuration option in `ccm.conf` (and changing it requires a new profile or a `ccm-fetch --force`).

## pan format example

An example TT file is `pan` format `CCM/pan.tt` (since 15.6)
```
[% INCLUDE CCM/pan_element.tt data=CCM.contents path=CCM.element.path -%]
```

This starts with `data` and `path` as derived as the contents and the path of the element

Individual elements are dealt with via `CCM/pan_element.tt`
```
[%- IF CCM.is_scalar(data) -%]
[%-     type = data.get_type -%]
"[% path %]" = [% data %]; # [% type FILTER lower %]
[% # the only newline, one per element -%]
[%- ELSIF CCM.is_list(data) -%]
[%-     index = 0 -%]
[%-     FOREACH value IN data -%]
[%-         index = index +1 -%]
[%-          INCLUDE CCM/pan_element.tt data=value path=path.merge(index) -%]
[%-      END -%]
[%- ELSIF CCM.is_hash(data) -%]
[%-     FOREACH pair IN data.pairs -%]
[%-          INCLUDE CCM/pan_element.tt data=pair.value path=path.merge(pair.key) -%]
[%-      END -%]
[%- END -%]
```

`CCM::TextRender` has the `doublequote` element option is set to produce
a doublequoted string if `data` is a string and the `truefalse` option to
generate `true` or `false` value if `data` is a boolean.
