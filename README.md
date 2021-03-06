﻿dpq2
====
[![Build Status](https://travis-ci.org/denizzzka/dpq2.svg?branch=master)](https://travis-ci.org/denizzzka/dpq2)
[![Coverage Status](https://coveralls.io/repos/denizzzka/dpq2/badge.svg?branch=master)](https://coveralls.io/r/denizzzka/dpq2)
[![codecov.io](https://codecov.io/github/denizzzka/dpq2/coverage.svg?branch=master)](https://codecov.io/github/denizzzka/dpq2)

This is yet another attempt to create a good interface to PostgreSQL for the 
D programming language.

It adds only tiny overhead to the original low level library libpq but
make convenient use PostgreSQL from D.


Features
--------

* Text string arguments support
* Binary arguments support (including multi-dimensional arrays)
* Both text and binary formats of query result support
* Immutable query result for simplify multithreading
* Async queries support
* Reading of the text query results to native D text types
* Representation of the binary query results to native D types
 * Text types
 * Integer and decimal types
 * Some data and time types
 * JSON type (stored into vibe.data.json.Json)
* Conversion of values to BSON (into vibe.data.bson.Bson)
* Access to PostgreSQL's multidimensional arrays
* LISTEN/NOTIFY support

Building
--------

Bindings for libpq can be static or dynamic.

The static bindings are generated by default. Add `--config=dynamic`
to the `dub` parameters to generate dynamic bindings.

Example
-------
```D
#!/usr/bin/env rdmd

import dpq2;
import std.stdio: writeln;
import std.getopt;
import vibe.data.bson;

void main(string[] args)
{
    string connInfo;
    getopt(args, "conninfo", &connInfo);

    Connection conn = new Connection(connInfo);

    // Only text query result can be obtained by this call:
    auto answer = conn.exec(
        "SELECT now()::timestamp as current_time, 'abc'::text as field_name, "~
        "123 as field_3, 456.78 as field_4, '{\"JSON field name\": 123.456}'::json"
        );

    writeln( "Text query result by name: ", answer[0]["current_time"].as!PGtext );
    writeln( "Text query result by index: ", answer[0][3].as!PGtext );

    // It is possible to read values of unknown type using BSON:
    auto firstRow = answer[0];
    foreach(cell; rangify(firstRow))
    {
        writeln("bson: ", cell.as!Bson);
    }

    // Separated arguments query with binary result:
    QueryParams p;
    p.sqlCommand = "SELECT "~
        "$1::double precision as double_field, "~
        "$2::text, "~
        "$3::text as null_field, "~
        "array['first', 'second', NULL]::text[] as array_field, "~
        "$4::integer[] as multi_array, "~
        "'{\"float_value\": 123.456,\"text_str\": \"text string\"}'::json as json_value";
    
    p.argsFromArray = [
        "-1234.56789012345",
        "first line\nsecond line",
        null,
        "{{1, 2, 3}, {4, 5, 6}}"
    ];

    auto r = conn.execParams(p);
    
    writeln( "0: ", r[0]["double_field"].as!PGdouble_precision );
    writeln( "1: ", r[0][1].as!PGtext );
    writeln( "2.1 isNull: ", r[0][2].isNull );
    writeln( "2.2 isNULL: ", r[0].isNULL(2) );
    writeln( "3.1: ", r[0][3].asArray[0].as!PGtext );
    writeln( "3.2: ", r[0][3].asArray[1].as!PGtext );
    writeln( "3.3: ", r[0]["array_field"].asArray[2].isNull );
    writeln( "3.4: ", r[0]["array_field"].asArray.isNULL(2) );
    writeln( "4: ", r[0]["multi_array"].asArray.getValue(1, 2).as!PGinteger );
    writeln( "5.1 Json: ", r[0]["json_value"].as!Json);
    writeln( "5.2 Bson: ", r[0]["json_value"].as!Bson);

    // It is possible to read values of unknown type using BSON:
    for(auto column = 0; column < r.columnCount; column++)
    {
        writeln("column name: '"~r.columnName(column)~"', bson: ", r[0][column].as!Bson);
    }

    version(LDC) destroy(r); // before Derelict unloads its bindings (prevents SIGSEGV)
}
```
####Compile and run:
```
Running ./dpq2_example --conninfo=dbname=postgres
Text query result by name: 2016-02-23 15:22:29.024757
Text query result by index: 456.78
bson: "2016-02-23 15:22:29.024757"
bson: "abc"
bson: "123"
bson: "456.78"
bson: {"JSON field name":123.456}
0: -1234.57
1: first line
second line
2.1 isNull: true
2.2 isNULL: true
3.1: first
3.2: second
3.3: true
3.4: true
4: 6
5.1 Json: {"text_str":"text string","float_value":123.456}
5.2 Bson: {"text_str":"text string","float_value":123.456}
column name: 'double_field', bson: -1234.56789012345
column name: 'text', bson: "first line\nsecond line"
column name: 'null_field', bson: null
column name: 'array_field', bson: ["first","second",null]
column name: 'multi_array', bson: [[1,2,3],[4,5,6]]
column name: 'json_value', bson: {"text_str":"text string","float_value":123.456}
```
