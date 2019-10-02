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

