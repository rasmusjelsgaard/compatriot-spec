# Problems solved by Compatriot

## The problem
Contemporary IDEs and CI/CD solutions make it easy to execute test runs to test software at different levels (unit test, integration tests, etc). They also make it easy to produce reporting artifacts, i.e. nUnit reports, code coverage reports etc.  
However, just as no man is an island, (almost) no piece of software exists as a single binary with no compile- or build-time dependencies.  
The problem: it requires a lot of manual work to compile useful information about what software compositions have been tested.

## The solution
Compatriot solves this by providing a simple way to store and query information about software composition tests.  
It does this by storing test run results as facts about whether a given software composition configruation has been tested.  
Compatriot can then be queried about the test run state of specific software compositions.  
In their simplest form, these queries always result one of three states:

1. PASS
2. FAIL
3. UNKNOWN

Compatriot also supports advanced queries that results in tuples of states. See below for a more detailed explanation of this.

## Use cases ##
1. Store test results from local development and pipelines in Compatriot so this intel gathering happens automatically.
2. Query Compatriot to figure if software components that are assumed to be compatible were tested at any point.
3. Add extra here.

### Features not in scope
Compatriot does not come with any built-in support for CI/CD logic such as failing pipelines based on test results, nor does it gather information about the actual composition of software packages. It has no notion of "flaky" tests or similar concepts. These concerns are left to the tools "on top of" Compatriot.

## Examples using the Compatriot CLI
The examples below assume that:
- Compatriot has been setup to point to a redis store beforehand. This is where the data you provide as input to Compatriot is stored.
  
Compatriot requires that packages be specified using the [purl notation](https://github.com/package-url/purl-spec). For brevity in the examples I am using a pseudo-purl notation: <package-atom>-<major.minor.patch>, i.e. module-a-1.0.0 instead of the full purl.

### Example 1: Storing test results
*In the example below we assign the fact that module-a, 1.0.0 works (according to our tests at least)*

```
$ compatriot "module-a-1.0.0 PASS"
```
**Some interesting things to note:**

1. Compatriot assignments always begin with a purl. 
2. Compatriot assignments always nd with a test state value from the enum PASS, FAIL, UNKNOWN.

No output is given, the command returns exit code 0.

### Example 2: Storing test results for composited packages ###
*In the example below we store the fact that module-a, v1.0.0, containing module-b in version 1.10.1 and module-c in version 2.4.2 was tested OK.* 

 ```
$ compatriot "module-a-1.0.0 (module-b-1.10.1 module-c-2.4.2) PASS"
``` 
**Some interesting things to note:**

1. Composite assignments must adhere to the same rules as assignments to a single purl.
2. There is no transitive implication of assignments. The assignment above does *not* mean that "module-b-1.10.1 PASS" gets stored in Compatriot.

No output is given, the command returns exit code 0.

### Example 3: Query test status of package ###
*In the example below we query compatriot for its knowledge about a specific version of the package 'module-a'. This package has been tested and was tested ok.*

 ```
$ compatriot "module-a-1.0.0"
$ PASS
``` 
**Some interesting things to note:**

1. Compatriot has no notion of timestamping, so it is entirely possible that if you re-run the query that produced a PASS result you might end up getting a different result.
2. Compatriot has no security logic built in. Anyone with access to the frontend or database can overwrite values at any point in time.
3. If this specific package had never been tested, we would get status UNKNOWN. It's entirely possible to assign the value UNKNOWN to a package at any point in time.

## Examples of more advanced queries ##
Compatriot supports globs in the version part of the package url. This is useful when you don't have full knowledge about releases of a particular package.

### Example 4: Query test status of package 'module-a', version 1.0.? (wildcard patch version) ###
*In this example we find all versions of the package 'module-a' that starts with 1.0*

 ```
$ compatriot "module-a-1.0.?"
$ ---------------------------------
$ | module-a-1.0.1 |   PASS     |
$ | module-a-1.0.2 |   PASS     |
$ | module-a-1.0.3 |   PASS     |
$ | module-a-1.0.4 |   FAIL     |
$ | module-a-1.0.5 |   PASS     |
$ | module-a-1.0.6 |   UNKNOWN  |
$ ---------------------------------
 ```

**Some interesting things to note:**
The ordering of results is not guaranteed and has no intrinsic meaning, i.e. the results to the query in this example could just as well look like this:

 ```
$ ---------------------------------
$ | module-a-1.0.6   |   UNKNOWN  |
$ | module-a-1.0.5   |   PASS     |
$ | module-a-1.0.3   |   PASS     |
$ | module-a-1.0.2   |   PASS     |
$ | module-a-1.0.4   |   FAIL     |
$ | module-a-1.0.1   |   PASS     |
$ ---------------------------------
 ```

### Example 5: Query test status of package 'module-a', version 1.0.?, considering composites in specific versions ###
*Here we try to find the composited versions of 'module-a' with fixed versions of the composities.*

 ```
$ compatriot "module-a-1.0.? (module-b-1.10.1 module-c-2.4.2)"
$ ---------------------------------
$ | module-a-1.0.1   |   PASS     |
$ | module-a-1.0.2   |   PASS     |
$ | module-a-1.0.3   |   PASS     |
$ | module-a-1.0.4   |   FAIL     |
$ | module-a-1.0.5   |   PASS     |
$ | module-a-1.0.6   |   UNKNOWN  |
$ ---------------------------------
 ```

 **Some interesting things to note:**
1. This query could produce an empty result set if no such composites exist.
2. It is allowed to use wildcard in composite versions as well.
3. The '++' after the package url is a shorthand, so instead of:

 ``` 
$ ------------------------------------------------------------------
$ | module-a-1.0.0 (module-b-1.10.1 module-c-2.4.2)   |   PASS     |
$ ------------------------------------------------------------------
 ```

we have simply

``` 
$ ----------------------------------
$ | module-a-1.0.0++  |   PASS     |
$ ----------------------------------
```

Each package that has a glob in the version specification is included in the output.  
The '++' indicates that there are more packages in the query result that are not being shown.

The query would *not* consider module-a-1.0.6-rc1, because it does not match the ? qualifier.


## Examples of how to specify test run scopes in assignments and queries
New functionality often starts its life in a developer's machine before ending up in a branch. And so testing often (hopefully) begins here. It is useful to be able to distinguish between the results of a specific set of tests run on a local dev machine vs. the same tests executed in CI/CD pipeline vs. acceptance tests run in a customer's pre-production environment and so on.

Compatriot solves this by way of scope that can be specified for both assignment and retrieval of facts.

### Example 6: Assign the result of test of a single package in local scope
*In the following example we store the fact that module-a, version 1.0.0 was tested ok whilst running the tests locally at our machine*

```
$ compatriot --scope local "module-a-1.0.0 OK"
```

**Some interesting things to note:**
1. More than one scope must be specified, in this case they are separated by commas.
2. At least one scope must be specified.
3. The default scope is all, which always includes all other scopes.
4. Valid scopes are: local, integrated, functional, non-functional, release, deploy, all.
5. The scopes are "just" labels, they serve only to filter queries as you will see in the next examples.

### Example 7: Store results in local scope and query in other scopes.
```
$ compatriot --scope local "module-a-1.0.0 OK"
$ compatriot --scope local,integrated "module-a-1.0.0"
$ UNKNOWN
$ compatriot "module-a-1.0.0"
$ UNKNOWN
```

**Some interesting things to note:**
1. If at least one scope would return UNKNOWN, the result of the query will be UNKNOWN.

### Example 8: 
```
$ compatriot "module-a-1.0.0 PASS"
$ compatriot --scope integrated "module-a-1.0.0"
$ PASS
```

## Architecture: Component layout
Compatriot code is split into components in a coarse-grained manner by overall responsibility. The overall goal of this structure is to (mostly) interact with the lexer which has a very stable interface.

![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/rasmusjelsgaard/compatriot-spec/master/component-overview.puml?token=AAZOVYYQYZWETF2EGLS3YIC5TBPFS)

# Project layout and distributables
Compatriot consists of four main repositories:

## compatriot-cli
Command line interface to lib-compatriot.

## compatriot-web
Web interface to lib-compatriot. 

## lib-compatriot 
Provides the parser functionality to lex and parse compatriot expressions and make sure they are syntactically correct. Depends on compatriot-persistence for the actual data storage and retrieval.

## compatriot-persistence
Provides storage and query functionality for lib-compatriot.

# Ideas for integrations on top of Compatriot
Much of the value Compatriot provides comes from integrations. Below are some of the examples I imagine:

1. Slurper that can consume [CycloneDX s-BOMs](#https://cyclonedx.org/#example-sbom) and store their contents in Compatriot. Would be combined with a slurper for test runs to figure out the test run results. 
2. Visual tool to make an overview of what versions of software are not only supposedly compatible but also have been tested to be proven compatible (according to tests).
3. Visual tool that helps build more complex queries.
4. Hardware + software compatibility extensions so that it becomes possible to store information about test runs on different hardware specs as well.
  