# TECHNIQUE 1 GNU/UNIX-style command-line arguments
Although the built-in flag package is useful and provides the basics, it doesn’t provide
flags in the manner most of us have come to expect. The difference in user interac-
tion between the style used in Plan 9 and the style in Linux- and BSD -based systems is
enough to cause users to stop and think. This is often due to problems they’ll encoun-
ter when trying to mix short flags, such as those used when executing a command like
ls -la .

## PROBLEM
Those of us who write for non-Windows systems will likely be working with a UNIX
variant, and our users will expect UNIX -style flag processing. How do you write Go
command-line tools that meet users’ expectations? Ideally, you want to do this with-
out writing one-off specialized flag processing.

## SOLUTION
This common problem has been solved in a couple of ways. Some applications, such
as Docker, the software container management system, have a subpackage containing
code to handle Linux-style flags. In some cases, these are forks of the Go flag pack-
age, in which the extra needs are added in. But maintaining per-project implementa-
tions of argument parsing results is an awful lot of duplicated effort to repeatedly
solve the same generic problem.
The better approach is to use an existing library. You can import several stand-
alone packages into your own application. Many are based on the flag package pro-
vided by the standard library and have compatible or similar interfaces. Importing
one of these packages and using it is faster than altering the flag package and main-
taining the difference.

## DISCUSSION
In the following examples, you’ll see two packages with slightly different approaches.
The first attempts to keep API compatibility with the Go flag package, whereas the
second breaks from it.

### GNUFLAG
Similar to the flag package, the launchpad.net/gnuflag package brings GNU -style
(Linux) flags to Go. Don’t let the name fool you. The license isn’t the GPL license, typ-
ical of GNU -based programs and libraries, but rather a BSD -style license in line with
the Go license.
Several forms of flags are supported when using gnuflag , including these:
- -f for a single-letter or short flag
- -fg for a group of single-letter flags
- --flag for a multiletter or long flag name
- --flag x for a long flag with a value passed in as x
- -f x or -fx for a short flag with a passed-in value

The gnuflag package has almost the exact same API as the flag package, with one
notable difference. The Parse function has an additional argument. Otherwise, it’s a
drop-in replacement. The flag package will parse flags between the command name
and the first nonflag argument. For example, in listing 2.4, the -s and -n flags are
defined, and -n takes a value. When flag parses foo , it has reached a nonflag argu-
ment and stops parsing. That means -bar won’t be parsed to see if it’s a flag. The
gnuflag package has an option to continue parsing that would allow it to find flags
positioned where -bar is.

> Listing 2.4 flag package flag parsing
```sh
$ flag_based_cli –s –n Buttercup foo -bar
```

To switch from the flag package to the gnuflag package, you need to change only
two things in an application. First, instead of importing the flag package, import
launchpad.net/gnuflag . Then, add an additional first argument to the Parse func-
tion of either true or false . A value of true looks for flags anywhere in the com-
mand, and false behaves like the flag package. That’s it.

> NOTE lanuchpad.net uses the Bazaar version-control system. You need to
have this installed in order for Go to fetch the package. More details are avail-
able at http://bazaar.canonical.com/.

### GO-FLAGS
Some community flag packages break from the conventions used in the flag package
within the standard library. One such package is github.com/jessevdk/go-flags . It
provides Linux- and BSD -style flags, providing even more features than those in
gnuflag , but uses an entirely different API and style. Features include the following:
- Short flags, such as –f , and groups of short flags, such as -fg .
- Multiple-letter or long flags, such as --flag .
- Supporting option groups.
- Generating well-formed help documentation.
- Passing values for flags in multiple formats such as -p/usr/local , -p /usr
/local , and -p=/usr/local .
- The same option can, optionally, appear more than once and be stored multi-
ple times on a slice.
To illustrate using this flag package, the following listing shows our Spanish-capable
Hello World console application rewritten using go-flags .

> Listing 2.5 Using go-flags
```go
package main
import (
    "fmt"
    flags "github.com/jessevdk/go-flags" // Imports go-flags aliased to the name flags
)
var opts struct { // [1]Struct containing the defined flags
    Name string `short:"n" long:"name" default:"World" description:"A name to say hello to."`
    Spanish bool `short:"s" long:"spanish"  description:"Use Spanish Language"`
}
func main() {
    flags.Parse(&opts) // [2]Parses the flag values into the struct
    if opts.Spanish == true { // [3]
        fmt.Printf("Hola %s!\n", opts.Name)
    } else {
        fmt.Printf("Hello %s!\n", opts.Name)
    }
}

```
The first big difference you’ll notice from previous techniques is how flags are
defined. In this instance, they’re defined as properties on a struct [1] . The property
names provide access to the values in your application. Information about a flag, such
as short name, long name, default value, and description, are gathered using reflec-
tion to parse the space-separated key-value pairs following a property on a struct. This
key-value-pair parsing capability is provided by the reflect package in the standard
library. For example, opts.Name is mapped to a flag that has a short name -n , a long
name --name , a default value of World , and a description used for help text.
For the values to be available on the opts struct, the Parse function needs to be
called with the opts struct passed in [2] . After that, the properties on the struct can be
called normally with flag values or their defaults available [3].