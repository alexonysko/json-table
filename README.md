# json-table [![Build Status](https://travis-ci.org/micha/json-table.svg?branch=master)](https://travis-ci.org/micha/json-table)

**Jt** transforms JSON data structures into tables of columns and rows for
processing in the shell.

Extracting information from deeply nested JSON data is difficult and unreliable
with tools like **sed** and **awk**, and tools that are specially designed for
manipulating JSON are cumbersome to use in the shell because they either return
their results as JSON or introduce a new turing complete scripting language
that needs to be quoted and constructed via string interpolation.

**Jt** provides only what is needed to extract data from nested JSON data
structures and organize the data into a table. Tools like **cut**, **paste**,
**join**, **sort**, **uniq**, etc. can be used to efficiently reduce the
tabular data to produce the final result.

#### Features

* **Self contained** &mdash; statically linked, has no build or runtime dependencies.
* **Fast, small memory footprint** &mdash; efficiently process **large** JSON input.
* **Correct** &mdash; parser does not accept invalid JSON (see tests for details).
* **Streaming mode** &mdash; reads JSON objects one-per-line e.g., from log files.

#### Example

You can get an idea of what **jt** can do from this one-liner that produces
a table of ELB names to EC2 instance IDs from the complex JSON returned by the
`aws elb` tool:

```
$ aws elb describe-load-balancers \
  |jt LoadBalancerDescriptions [ LoadBalancerName % ] Instances InstanceId %
elb-1	i-94a6f73a
elb-2	i-b910a256
...
```

## INSTALL

Linux users can install prebuilt binaries from the release tarball:

```
sudo bash -c "cd /usr/local && wget -O - https://github.com/micha/json-table/raw/master/jt.tar.gz | tar xzvf -"
```

Otherwise, build from source:

```
make && make test && sudo make install
```

## DOCUMENTATION

See the [man page][man] or `man jt` in your terminal.

## EXAMPLES

Use the `@` command to print an object's keys:

```bash
cat <<EOT | jt @
{
  "foo": 100,
  "bar": 200,
  "baz": 300
}
EOT
```
```
foo
bar
baz
```

### Drill Down

Property names are also commands. Use `foo` here as a command to drill down
into the `foo` property and then use `@` to print its keys:

```bash
cat <<EOT | jt foo @
{
  "foo": {
    "bar": 100,
    "baz": 200
  }
}
EOT
```
```
bar
baz
```

When a property name conflicts with a `jt` command you must wrap the property
name with square brackets to drill down:

```bash
cat <<EOT | jt [@] @
{
  "@": {
    "bar": 100,
    "baz": 200
  }
}
EOT
```
```
bar
baz
```

It's okay if the property name itself has square brackets. Just wrap with more
brackets:

```bash
cat <<EOT | jt [[@]] @
{
  "[@]": {
    "bar": 100,
    "baz": 200
  }
}
EOT
```
```
bar
baz
```

### Extract

The `%` command prints the item at the top of the data stack. Note that when
the top item is a collection it is printed as JSON (insiginificant whitespace
removed):

```bash
cat <<EOT | jt %
{
  "foo": 100,
  "bar": 200
}
EOT
```
```
{"foo":100,"bar":200}
```

Drill down and print:

```bash
cat <<EOT | jt foo bar %
{
  "foo": {
    "bar": 100
  }
}
EOT
```
```
100
```

The `%` command can be used multiple times. The printed values will be delimited
by tabs:

```bash
cat <<EOT | jt % foo % bar %
{
  "foo": {
    "bar": 100
  }
}
EOT
```
```
{"foo":{"bar":100}}     {"bar":100}     100
```

### Save / Restore

The `[` and `]` commands provide a sort of `GOSUB` facility &mdash; the data
stack is saved by `[` and restored by `]`. This can be used to extract values
from different paths in the JSON as a single record:

```bash
cat <<EOT | jt [ foo % ] [ bar % ]
{
  "foo": 100,
  "bar": 200
}
EOT
```
```
100     200
```

### Iteration (Arrays)

`Jt` automatically iterates over arrays, running the program once for each
item in the array. This produces one tab-delimited record for each iteration,
separated by newlines:

```bash
cat <<EOT | jt [ foo % ] [ bar baz % ]
{
  "foo": 100,
  "bar": [
    {"baz": 200},
    {"baz": 300},
    {"baz": 400}
  ]
}
EOT
```
```
100     200
100     300
100     400
```

The `^` command includes the array index as a column in the result:

```bash
cat <<EOT | jt [ foo % ] [ bar ^ baz % ]
{
  "foo": 100,
  "bar": [
    {"baz": 200},
    {"baz": 300},
    {"baz": 400}
  ]
}
EOT
```
```
100     0       200
100     1       300
100     2       400
```

Note that `^` is scoped &mdash; it prints the index of the innermost enclosing
loop:

```bash
cat <<EOT | jt foo ^ bar ^ %
{
  "foo": [
    {"bar": [100, 200]},
    {"bar": [300, 400]}
  ]
}
EOT
```
```
0       0       100
0       1       200
1       0       300
1       1       400
```

### Iteration (Objects)

The `.` command iterates over the values of an object:

```bash
cat <<EOT | jt . %
{
  "foo": 100,
  "bar": 200,
  "baz": 300
}
EOT
```
```
100
200
300
```

When iterating over an object the `^` command prints the name of the current
property:

```bash
cat <<EOT | jt [ foo % ] [ bar . ^ % ]
{
  "foo": 100,
  "bar": {
    "baz": 200,
    "baf": 300,
    "qux": 400
  }
}
EOT
```
```
100     baz     200
100     baf     300
100     qux     400
```

The scope of `^` is similar when iterating over objects:

```bash
cat <<EOT | jt . ^ . ^ %
{
  "foo": {
    "bar": 100,
    "baz": 200
  }
}
EOT
```
```
foo     bar     100
foo     baz     200
```

### Explicit Iteration

Sometimes the implicit iteration over arrays is awkward:

```bash
cat <<EOT | jt . ^ . ^ %
{
  "foo": [
    {"bar":100},
    {"bar":200}
  ]
}
EOT
```
```
0       bar     100
1       bar     200
```

Should the first `^` be printing the array index (which it does, in this case)
or the object key (i.e. `foo`)? Explicit iteration with the `-a` flag eliminates
the ambiguity:

```bash
cat <<EOT | jt -a . . ^ . ^ %
{
  "foo": [
    {"bar":100},
    {"bar":200}
  ]
}
EOT
```
```
0       bar     100
1       bar     200
```

prints the array index and:

```bash
cat <<EOT | jt -a . ^ . . ^ %
{
  "foo": [
    {"bar":100},
    {"bar":200}
  ]
}
EOT
```
```
foo     bar     100
foo     bar     200
```

prints the object key, and

```bash
cat <<EOT | jt -a . ^ . ^ . ^ %
{
  "foo": [
    {"bar":100},
    {"bar":200}
  ]
}
EOT
```
```
foo     0       bar     100
foo     1       bar     200
```

prints both.

### JSON Streams

`Jt` automatically iterates over all entities in a [JSON stream][json-stream]
(a stream of JSON entities delimited by optional insignificant whitespace):

```bash
cat <<EOT | jt [ foo % ] [ bar % ]
{"foo": 100, "bar": 200}
{"foo": 200, "bar": 300}
{"foo": 300, "bar": 400}
EOT
```
```
100     200
200     300
300     400
```

Insignificant whitespace is ignored:

```bash
cat <<EOT | jt [ foo % ] [ bar % ]
{
  "foo": 100,
  "bar": 200
}{"foo":200,"bar":300}
    {
      "foo": 300,
      "bar": 400
    }
EOT
```
```
100     200
200     300
300     400
```

Within a JSON stream the `^` command prints the current stream index:

```bash
cat <<EOT | jt ^ [ foo % ] [ bar % ]
{"foo": 100, "bar": 200}
{"foo": 200, "bar": 300}
{"foo": 300, "bar": 400}
EOT
```
```
0       100     200
1       200     300
2       300     400
```

Note that one entity in the stream may result in more than one output record
when iteration is involved:

```bash
cat <<EOT | jt [ foo % ] [ bar % ]
{"foo":10,"bar":[100,200]}
{"foo":20,"bar":[300,400]}
EOT
```
```
10      100
10      200
20      300
20      400
```

### Nested JSON

The `+` command to parses JSON embedded in strings:

```bash
cat <<EOT | jt [ foo + bar % ] [ baz % ]
{"foo":"{\"bar\":100}","baz":200}
{"foo":"{\"bar\":200}","baz":300}
{"foo":"{\"bar\":300}","baz":400}
EOT
```
```
100     200
200     300
300     400
```

Note that `+` pushes the resulting JSON entity onto the data stack &mdash; it
does not mutate the original JSON:

```bash
cat <<EOT | jt [ foo + bar % ] %
{"foo":"{\"bar\":100}","baz":200}
{"foo":"{\"bar\":200}","baz":300}
{"foo":"{\"bar\":300}","baz":400}
EOT
```
```
100     {"foo":"{\"bar\":100}","baz":200}
200     {"foo":"{\"bar\":200}","baz":300}
300     {"foo":"{\"bar\":300}","baz":400}
```

### Joins

Notice the empty column &mdash; some objects don't have the `bar` key:

```bash
cat <<EOT | jt [ foo % ] [ bar % ]
{"foo":100,"bar":1000}
{"foo":200}
{"foo":300,"bar":3000}
EOT
```
```
100     1000
200
300     3000
```

Enable inner join mode with the `-j` flag. This removes output rows when a key
in the traversal path doesn't exist:

```bash
cat <<EOT | jt -j [ foo % ] [ bar % ]
{"foo":100,"bar":1000}
{"foo":200}
{"foo":300,"bar":3000}
EOT
```
```
100     1000
300     3000
```

Note that this does not remove rows when the key exists and the value is empty:

```bash
cat <<EOT | jt -j [ foo % ] [ bar % ]
{"foo":100,"bar":1000}
{"foo":200,"bar":""}
{"foo":300,"bar":3000}
EOT
```
```
100     1000
200
300     3000
```

## COPYRIGHT

Copyright © 2017 Micha Niskin. Distributed under the Eclipse Public License.

[man]: http://htmlpreview.github.io/?https://raw.githubusercontent.com/micha/json-table/master/jt.1.html
[json-stream]: https://en.wikipedia.org/wiki/JSON_Streaming#Concatenated_JSON
