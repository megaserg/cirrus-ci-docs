# Writing Tasks

Task defines where and how your scripts will be executed. Let's check line-by-line an example of `.cirrus.yml` configuration file first:

```yaml
task:
  container:
    image: gradle:4.3.0-jdk8
    cpu: 8
    memory: 20G
  script: gradle test
```

Example above defines a single task that will be scheduled and executed on Community Cluster using `gradle:4.3.0-jdk8` Docker image.
Only one user defined script instruction to run `gradle test` will be executed. Pretty simple, isn't it?

A `task` simply defines a [compute service](deoc/supported-computing-services.md) to schedule the task on and 
a sequence of [`script`](#script-instruction) and [`cache`](#cache-instruction) instructions that will be executed.

Please read topics below if you want better understand what's doing on in a more complex `.cirrus.yml` configuration file like this:

```yaml
# global default
container:
  image: node:latest
  
lint_task:      
  node_modules_cache:
    folder: node_modules
    fingerprint_script: cat yarn.lock
    populate_script: yarn install

  test_script: yarn run lint

test_task:
  container:
    matrix:
      image: node:latest
      image: node:8.3.0
      
  node_modules_cache:
    folder: node_modules
    fingerprint_script: cat yarn.lock
    populate_script: yarn install

  test_script: yarn run test
  

publish_task:
  depends_on: 
    - test 
    - lint
  only_if: $BRANCH == "master"
  
  node_modules_cache:
    folder: node_modules
    fingerprint_script: cat yarn.lock
    populate_script: yarn install

  publish_script: yarn run publish
```

# Script Instruction

`script` instruction executes commands via `shell` on Unix or `batch` on Windows. `script` instruction can be named by
adding a name as a prefix. For example `test_script` or `my_very_specific_build_step_script`. Naming script instructions
helps gather more granular information about task execution. Cirrus CI will use it in future to auto-detect performance 
regressions.

Script commands can be specified as a single string value or a list of string values in `.cirrus.yml` configuration file
like in an example below:

```yaml
check_task:
  compile_script: gradle --parallel classes testClasses 
  check_script:
    - printenv
    - gradle check
``` 

# Cache Instruction

`cache` instruction allows to save some folder in cache based on a fingerprint and reuse it during the next execution 
of the task with the same fingerprint. `cache` instruction can be named the same way as `script` instruction.

Here is an example: 

```yaml
test_task:
  node_modules_cache:
    folder: node_modules
    fingerprint_script: cat yarn.lock
    populate_script: yarn install
  test_script: yarn run test
```

`fingerprint_script` is an optional field that can specify a script that will be executed and console output of which
will be used as a fingerprint for the given task. By default task name is used as a fingerprint value.

`fingerprint_script` is an optional field that can specify a script that will be executed to populate the cache. 
`fingerprint_script` should create `folder`.

!> Note that cache folder will be archived and uploaded only in the very end of the task execution once all instructions succeed.

Which means the only difference between example above and below is that `yarn install` will always be executed in the 
example below where in the example above only when `yarn.lock` has changes.

```yaml
test_task:      
  node_modules_cache:
    folder: node_modules
    fingerprint_script: cat yarn.lock
  install_script: yarn install
  test_script: yarn run test
```

# Environment Variables

Environment variables can be configured under `environment` keyword in `.cirrus.yml` file. Here is an example:

```yaml
echo_task:
  environment:
    FOO: Bar
  echo_script: echo $FOO   
```

Also some default environment variables are pre-defined:

Name | Value / Description
---  | ---
CI | true
CIRRUS_CI | true
CONTINUOUS_INTEGRATION | true
CIRRUS_BRANCH | Branch name. For example `master`
CIRRUS_BUILD_ID | Unique build ID
CIRRUS_CHANGE_IN_REPO | Git SHA
CIRRUS_TASK_NAME | Task name
CIRRUS_TASK_ID | Unique task ID
CIRRUS_REPO_NAME | Repository name. For example `my-library`
CIRRUS_REPO_OWNER | Repository owner(an organization or a user). For example `my-organization`
CIRRUS_REPO_FULL_NAME | Repository full name. For example `my-organization/my-library`
CIRRUS_REPO_CLONE_URL | URL used for cloning. For example `https://github.com/my-organization/my-library.git`
CIRRUS_WORKING_DIR | Working directory where
CIRRUS_HTTP_CACHE_HOST | Host and port number on which [local HTTP cache](#http-cache) can be accessed on
      
# Encrypted Variables

It is possible to securely add sensitive information to `.cirrus.yml` file. Encrypted variables are only available to
builds initialized or approved by users with write permission to a corresponding repository.

In order to encrypt a variable go to repository's settings page via clicking settings icon ![](https://storage.googleapis.com/material-icons/external-assets/v4/icons/svg/ic_settings_white_24px.svg)
on a repository's main page (for example https://cirrus-ci.org/github/my-organization/my-repository) and follow instructions.

!> Only users with `WRITE` permissions can add encrypted variables to a repository.

An encrypted variable will be presented in a form like `ENCRYPTED[qwerty239abc]` which can be safely committed within `.cirrus.yml` file:

```yaml
publish_task:
  environemnt:
    AUTH_TOKEN: ENCRYPTED[qwerty239abc]
  script: ./publish.sh
```

Cirrus CI encrypts variables with a unique per repository 256-bit encryption key so forks and even repositories within
the same organization cannot re-use them.

# Matrix Modification

Sometimes it's useful to run the same task against different software versions. Or run different batches of tests based
on an environment variable. For cases like these `matrix` modification comes very handy. It's possible to use `matrix`
keyword anywhere inside of a particular task to have multiple tasks based on the original one. Each new task will be created
from the original task by replacing the whole `matrix` YAML node with each `matrix`'s children separately. 

Let check an example of `.cirrus.yml`:

```yaml
test_task:
  container:
    matrix:
      image: node:latest
      image: node:8.3.0
  test_script: yarn run test
```

Which will be expanded into:

```yaml
test_task:
  container:
    image: node:latest
  test_script: yarn run test
  
test_task:
  container:
    image: node:8.3.0
  test_script: yarn run test
```

!> `matrix` modification can be used multiple times within a task.

`matrix` modification makes it easy to create some pretty complex testing scenarios like this:

```yaml
test_task:
  container:
    matrix:
      image: node:latest
      image: node:8.3.0
  environment:
    matrix:
      COMMAND: test
      COMMAND: lint
  node_modules_cache:
    folder: node_modules
    fingerprint_script: 
      - node --version
      - cat yarn.lock
    populate_script: yarn install
  test_script: yarn run $COMMAND
```

# Dependencies

Sometimes it might be very handy execute some tasks only after successful execution of other tasks. For such cases
it's possible specify for a task names of other tasks it depends on with `depends_on` keyword:

```yaml
lint_task:
  script: yarn run lint

test_task:
  script: yarn run test
  
publish_task:
  depends_on: 
    - test
    - lint
  script: yarn run publish
```

# Conditional Task Execution

Some tasks are meant to be executed for master or release branches only. In order to specify a condition when a task
should be executed please use `only_if` keyword:

```yaml
publish_task:
  only_if: $CIRRUS_BRANCH == 'master'
  script: yarn run publish
```

Currently only basic operators like `==`, `!=`, `&&`, `||` and unary `!` are supported in `only_if` expression.
[Environment variables](#environment-variables) can also be used as usually.

# HTTP Cache


