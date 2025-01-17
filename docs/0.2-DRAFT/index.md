# Workflow Testing RO-Crate

Version: 0.2-DRAFT

Workflow Testing RO-Crate is a specialization of [Workflow RO-Crate](https://w3id.org/workflowhub/workflow-ro-crate/1.0) that supports additional metadata related to the testing of computational workflows. [LifeMonitor](https://crs4.github.io/life_monitor/) uses Workflow Testing RO-Crate as an exchange format that allows RO-Crate authors to describe test suites associated with workflows.


## Introduction

LifeMonitor monitors the execution of workflow test **suites** on one or more Continuous Integration (CI) **services** (e.g., [GitHub Actions](https://docs.github.com/en/actions)). We refer to the execution of a test suite on a CI service as test **instance**. The test suite itself is usually described by a **definition** that follows the requirements of a specific test **engine**, such as [Planemo](https://planemo.readthedocs.io/en/latest/test_format.html).


## Concepts

This section uses terminology from the [RO-Crate 1.1 specification](https://w3id.org/ro/crate/1.1).

Workflow Testing RO-Crate extends the [RO-Crate 1.1 context](https://www.researchobject.org/ro-crate/1.1/context.jsonld) with types and properties defined in the [test RO-Terms vocabulary](https://github.com/ResearchObject/ro-terms/blob/master/test/vocabulary.csv). To add mappings for these terms to an RO-Crate, specify the `@context` as follows:

```json
"@context": [
    "https://w3id.org/ro/crate/1.1/context",
    "https://w3id.org/ro/terms/test"
],
```

A Workflow Testing RO-Crate MUST be a valid [Workflow RO-Crate](https://w3id.org/workflowhub/workflow-ro-crate/1.0) (e.g., it has to contain a *Main Workflow*). In addition, it MUST refer to one or more [test suites](#test-suite) from the [root data entity](https://www.researchobject.org/ro-crate/1.1/root-data-entity.html) via the `mentions` property:

```json
{
    "@id": "./",
    "@type": "Dataset",
    "mentions": [
        {"@id": "#test1"},
        {"@id": "#test2"}
    ],
    ...
}
```


### Test suite

A _test suite_ describes a set of tests for a computational workflow. It is represented by a contextual entity of type `TestSuite`. A test suite MUST refer either to one or more [test instances](#test-instance) (via the `instance` property) or to a [test definition](#test-definition) (via the `definition` property) or both. Additionally, a test suite SHOULD refer to the tested workflow via `mainEntity`.

```json
{
    "@id": "#test1",
    "@type": "TestSuite",
    "mainEntity": {"@id": "sort-and-change-case.ga"},
    "instance": [
        {"@id": "#test1_1"},
        {"@id": "#test1_2"}
    ],
    "definition": {"@id": "test/test1/sort-and-change-case-test.yml"}
}
```

If the `mainEntity` property is missing, consumers should assume that the suite refers to the main workflow (pointed to by the `mainEntity` property of the root data entity).


### Test instance

A _test instance_ is a specific job that executes a [test suite](#test-suite) on a [test service](#test-service). It is represented by a contextual entity of type `TestInstance`. A test instance MUST refer to: a test service via the `runsOn` property; the base URL of the specific test service deployment where it runs via the `url` property; the relative URL of the test project via the `resource` property:

```json
{
    "@id": "#test1_1",
    "@type": "TestInstance",
    "runsOn": {"@id": "https://w3id.org/ro/terms/test#JenkinsService"},
    "url": "http://example.org/jenkins",
    "resource": "job/tests/"
}
```

For information on the exact format supported by LifeMonitor, see [LifeMonitor-specific features and requirements](#lifemonitor-specific-features-and-requirements).


### Test service

A _test service_ is a software service where tests can be run. It is represented by a contextual entity of type `TestService`:

```json
{
    "@id": "https://w3id.org/ro/terms/test#JenkinsService",
    "@type": "TestService",
    "name": "Jenkins",
    "url": {"@id": "https://www.jenkins.io"}
}
```

For information on the test services supported by LifeMonitor, see [LifeMonitor-specific features and requirements](#lifemonitor-specific-features-and-requirements).


### Test definition

A _Test definition_ is a file that describes how to run a [test suite](#test-suite). In the RO-Crate metadata, it is represented by a [data entity](https://www.researchobject.org/ro-crate/specification/1.1/data-entities) whose type MUST include `TestDefinition` and  `File`. A test definition MUST refer to the [test engine](#test-engine) it is written for via `conformsTo` and to the engine's version via `engineVersion`:

```json
{
    "@id": "test/test1/my-test.yml",
    "@type": ["File", "TestDefinition"],
    "conformsTo": {"@id": "https://w3id.org/ro/terms/test#PlanemoEngine"},
    "engineVersion": ">=0.70"
},
```


### Test engine

A _test engine_ is a software application that runs workflow tests according to a definition. It is represented by a context entity of type `SoftwareApplication`:

```json
{
    "@id": "https://w3id.org/ro/terms/test#PlanemoEngine",
    "@type": "SoftwareApplication",
    "name": "Planemo",
    "url": {"@id": "https://github.com/galaxyproject/planemo"}
}
```


## LifeMonitor-specific features and requirements

### Test service types

LifeMonitor currently supports monitoring tests executed on:

[GitHub Actions](https://docs.github.com/en/actions)

```json
{
    "@id": "https://w3id.org/ro/terms/test#GithubService",
    "@type": "TestService",
    "name": "Github Actions",
    "url": {"@id": "https://github.com"}
}
```

[Travis CI](https://travis-ci.com)

```json
{
    "@id": "https://w3id.org/ro/terms/test#TravisService",
    "@type": "TestService",
    "name": "Travis CI",
    "url": {"@id": "https://www.travis-ci.com"}
}
```

[Jenkins](https://www.jenkins.io)

```json
{
    "@id": "https://w3id.org/ro/terms/test#JenkinsService",
    "@type": "TestService",
    "name": "Jenkins",
    "url": {"@id": "https://www.jenkins.io"}
}
```

To fetch test build data from the CI service, LifeMonitor needs to be pointed to the specific project's endpoint via the `TestInstance` properties.

In the case of GitHub Actions, `url` must be set to `"https://api.github.com"`, while `resource` must be in the form:

```
repos/<OWNER>/<REPO NAME>/actions/workflows/<YAML FILE NAME>
```

For instance, [fair-crcc-send-data](https://github.com/crs4/fair-crcc-send-data) has a GitHub Actions workflow, `.github/workflows/main.yml`, that runs tests for the scientific workflow hosted in the repository. To have the test runs monitored by LifeMonitor, the `TestInstance` entry needs to be set up as follows:

```json
{
    "@id": "#my-test",
    "@type": "TestInstance",
    "url": "https://api.github.com",
    "resource": "repos/crs4/fair-crcc-send-data/actions/workflows/main.yml",
    "runsOn": {"@id": "https://w3id.org/ro/terms/test#GithubService"},
    "name": "My Test"
}
```

For Travis CI builds, set `url` to `https://travis-ci.com` and `resource` to `github/<OWNER>/<REPO NAME>` or `repo/<REPO ID>`. For Jenkins builds, set `url` to the base URL of the Jenkins instance (e.g., `"https://jenkins.example.org"`) and `resource` to the project's relative URL (e.g., `"job/my_tests"`).


## Example

```json
{
    "@context": [
        "https://w3id.org/ro/crate/1.1/context",
        "https://w3id.org/ro/terms/test"
    ],
    "@graph": [
        {
            "@id": "ro-crate-metadata.json",
            "@type": "CreativeWork",
            "about": {
                "@id": "./"
            },
            "conformsTo": {
                "@id": "https://w3id.org/ro/crate/1.1"
            }
        },
        {
            "@id": "./",
            "@type": "Dataset",
            "name": "sort-and-change-case",
            "description": "sort lines and change text to upper case",
            "license": "Apache-2.0",
            "mainEntity": {
                "@id": "sort-and-change-case.ga"
            },
            "hasPart": [
                {
                    "@id": "sort-and-change-case.ga"
                },
                {
                    "@id": "LICENSE"
                },
                {
                    "@id": "README.md"
                },
                {
                    "@id": "test/test1/sort-and-change-case-test.yml"
                }
            ],
            "mentions": [
                {
                    "@id": "#test1"
                }
            ]
        },
        {
            "@id": "sort-and-change-case.ga",
            "@type": [
                "File",
                "SoftwareSourceCode",
                "ComputationalWorkflow"
            ],
            "programmingLanguage": {
                "@id": "#galaxy"
            },
            "name": "sort-and-change-case"
        },
        {
            "@id": "LICENSE",
            "@type": "File"
        },
        {
            "@id": "README.md",
            "@type": "File"
        },
        {
            "@id": "#galaxy",
            "@type": "ComputerLanguage",
            "name": "Galaxy",
            "identifier": {
                "@id": "https://galaxyproject.org/"
            },
            "url": {
                "@id": "https://galaxyproject.org/"
            }
        },
        {
            "@id": "#test1",
            "name": "test1",
            "@type": "TestSuite",
            "mainEntity": {
                "@id": "sort-and-change-case.ga"
            },
            "instance": [
                {"@id": "#test1_1"}
            ],
            "definition": {"@id": "test/test1/sort-and-change-case-test.yml"}
        },
        {
            "@id": "#test1_1",
            "name": "test1_1",
            "@type": "TestInstance",
            "runsOn": {"@id": "https://w3id.org/ro/terms/test#JenkinsService"},
            "url": "http://example.org/jenkins",
            "resource": "job/tests/"
        },
        {
            "@id": "test/test1/sort-and-change-case-test.yml",
            "@type": [
                "File",
                "TestDefinition"
            ],
            "conformsTo": {"@id": "https://w3id.org/ro/terms/test#PlanemoEngine"},
            "engineVersion": ">=0.70"
        },
        {
            "@id": "https://w3id.org/ro/terms/test#JenkinsService",
            "@type": "TestService",
            "name": "Jenkins",
            "url": {"@id": "https://www.jenkins.io"}
        },
        {
            "@id": "https://w3id.org/ro/terms/test#PlanemoEngine",
            "@type": "SoftwareApplication",
            "name": "Planemo",
            "url": {"@id": "https://github.com/galaxyproject/planemo"}
        }
    ]
}
```
