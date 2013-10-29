# Guide to YAML::Tiny testing

YAML::Tiny tests use several components:

* .t files
* Test libraries in t/lib
* YAML data files in t/data
* TestML data files in t/tml-*

The use of each is described below.

## .t files

The .t files are standard Perl test files.  They may use one or more of the
test libraries in t/lib.  They may only use modules available in Perl 5.8.1 or
later (and not subsequently deprecated), but they may use newer non-XS versions
of those modules as necessary to avoid known bugs.

Some .t files have complete inputs/outputs for their tests.  Others iterate
over .tml files in the t/tml-* directories.

A .t file should load Test::More and either use `done_testing` or provide a
test plan.  If tests iterate over external data, the use of `done_testing` is
preferred so that external data can be updated with new tests without needing
to also update a test plan.

Currently, the convention is to name .t files matching the pattern
qr/^\d\d_\w+\.t$/

## Test libraries

There are currently three test libraries in t/lib.  A .t file that uses one or
more of them should put `use lib 't/lib';` at the top of the .t file.  Test
libraries can assume that if they were loaded, that 't/lib' is already in @INC.

The test libraries are:

* TestUtils
* TestMLTiny
* TestBridge

The TestUtils library contains utility functions and some testing functions
that have not yet been migrated to TestMLBridge.  Going forward, new testing
functions should not be added (i.e. nothing that uses Test::More).

The TestMLTiny library contains functions for parsing and executing TestML
tests with callbacks.  TestML is a data-driven testing language; TestMLTiny
implements a small subset of its features. See the section on TestML, below,
for an example.  The TestMLTiny module must remain standalone.  It should only
use core Perl modules.  Generally speaking, for anything other than minor
and obvious bugs, please consult with Ingy before patching it.

The TestBridge library contains testing functions for use in .t files or to
be passed to TestMLTiny functions as callbacks.  Tests functions should not
include `done_testing`.  They should use `subtest` for any repetitive testing
that loops over test cases.  Callback should check for the expected test
points (see below) and skip a TML block if those points are not available.

## YAML data files in t/data

Files in the t/data directory are intended to test how YAML files are loaded
and decoded and typically need some custom test code to load the file and see
if the result matches expectations (successful or not).

If a real-world YAML file cannot be loaded due to character set encoding
issues, it should be placed in this directory for testing.  If a real-world
YAML file is ASCII or UTF-8 encoded, can be decoded successfully, but has
problems in parsing, it should be reduced to the smallest sample of YAML that
demonstrates the parsing problem and added to a .tml file for testing.  See
below for more details.

## TestML quick intro

TestML data files are UTF-8 encoded files with a .tml suffix that contain one
or more test "blocks".  Each block has a test label, and one or more 'test
points', usually representing input and expected output, and possibly
additional annotations or flags.

Here is an example of a .tml file with a single block:

    # This is a TestML block:    (this line is a comment)
    === This is the test label
    Lines until the first point are ignored

    # This is a "block" style point. All non-comment lines until next point
    # are the data for the 'yaml' point. The data ends with newline, and
    # trailing blank lines are trimmed.
    --- yaml
    ---
    foo: bar
    # a comment
    \# not a comment

    # This is the second point; "inline" style. The data after the colon goes
    # to end of line. Leading/trailing whitespace is trimmed.
    --- perl: [ { foo => 'bar' } ]

    # This is a point whose value is the empty string
    --- a_flag

    # This is the next block:
    === Another test case

The test label is provided on a line beginning with qr/^===/.  Test "points"
are provided in sections beginning with qr/^--- +\w+/.  All flush-left comment
lines are stripped.  Lines beginning with '\' are escaped.

Different tests expect different test points in a .tml file, based on the
specific test callback being used.

Many .tml files have the points 'yaml' and 'perl' as in the example above.  The
'yaml' point is a YAML document and the 'perl' point is a Data::Dumper Perl
data structure.  The test checks whether the YAML parses into a data structure
identical to the Perl one.  The 'a_flag' point is an annotation that the
testing callback can use to affect the run of a given test.

The semantics of points (including annotations) is specific to the callback
functions used to process test blocks.

# TestML data files in t/tml-*

TestML data files are organized into three directories:

* t/tml-spec — these test files are provided by the YAML spec maintainers and
should not be modified except to skip testing features that YAML::Tiny does not
support

* t/tml-local — these test files are YAML::Tiny's own unit tests; generally new
test cases for coverage or correctness should be added here

* t/tml-world — these test files represent "real world" YAML and their
corresponding expected Perl output

Generally, if a "real world" problem can be isolated to a particular snippet of
YAML, it's best to add it to a t/tml-local file (or create a new one).  If the
problem can only be seen in the context of the entire YAML document, include it
in t/tml-world.  If the problem relates to encoding, it should be put into
t/data instead.