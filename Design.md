# What problem does Compatriot solve (Work in progress) #

## The problem ##
Contemporary CI/CD solutions make it easy and appealing to execute test runs that test software at different levels (unit test, integration tests, etc). They also make it easy to produce reporting artifacts, i.e. nUnit reports, code coverage reports etc.  
However, just as no man is an island, (almost) no piece of software exists as a single binary with no compile- or build-time dependencies.  
The problem: it requires a lot of manual work to compile useful information about what configurations of software compositions have been tested and the test results.

## Solution ##
Compatriot solves this by providing a simple way to store and query information about software composition tests.
It does this by storing the results as tests runs as facts about whether a given software composition configruation have been tested. Compatriot can then be queried about the state of specific software compositions. In their simplest form, these queries always result one of three states:

1. OK
2. FAILED
3. UNKNOWN

Compatriot also supports advanced queries that results in tuples of states. See below for a more detailed explanation of this.

## Use cases ##
1. Store test results from pipeline in Compatriot so this intel gathering happens automatically.
2. Query Compatriot to figure if software components that are assumed to be compatible were tested at any point.
3. Add extra here.

### Features not in scope ###
Compatriot does not come with any built-in support for CI/CD logic such as failing pipelines based on test results, nor does it gather information about the actual composition of software packages. It has no notion of "flaky" tests or similar concepts. These concerns are left to the tools "on top of" Compatriot.

### Examples using the Compatriot CLI ###
The examples below assume compatriot has been setup to point to a redis store beforehand.

### Example 1: Storing test results ###
*In the example below we store the fact that module-a, 1.0.0 works (according to our tests at least)*

```
$ compatriot (module-a-1.0.0) [(module-b-1.10.1 module-c-2.4.2)] = OK
```

### Example 2: Storing test results for composited component ###
*In the example below we store the fact that module-a, v1.0.0, containing module-b in version 1.10.1 and module-c in version 2.4.2 was tested OK.* 

 ```
$ compatriot (module-a-1.0.0) [(module-b-1.10.1 module-c-2.4.2)] = OK
``` 

### Example: Query which if a specific version was tested ###
*In the example below we query compatriot for its knowledge about a specific version of module-a. This has been tested and was ok.

 ```
$ compatriot module-a-1.0.0
$ OK
``` 

If this specific purl had never been tested, we would get status UNKNOWN.

