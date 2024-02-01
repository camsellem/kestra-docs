---
title: Working Directory
icon: /docs/icons/dev.svg
---

This task allows you to run multiple tasks sequentially in the same working directory. It is useful when you want to share files from Namespace Files or from a Git repository across multiple tasks.

## When to use the `WorkingDirectory` task

By default, all Kestra tasks are **stateless**. If one task generates files, those files won’t be available in downstream tasks unless they are persisted in internal storage. Upon each task completion, the temporary directory for the task is purged. This behavior is generally useful as it keeps your environment clean and dependency free, and it avoids potential privacy or security issues when exposing some data generated by a task to other processes.

Despite the benefits of the stateless execution, in certain scenarios, **statefulness** is **desirable**. Imagine that you want to execute several Python scripts, and each of them generates some output data. Another script combines that data as part of an ETL/ML process. Executing those related tasks in the same working directory and **sharing state** between them is helpful for the following reasons:
- You can attach namespace files to the `WorkingDirectory` task and use them in all downstream tasks. This allows you to work the same way you would work on your local machine, where you can import modules from the same directory.
- Within a `WorkingDirectory`, you can **clone** your entire **GitHub branch** with multiple modules and configuration files needed to run several scripts and **reuse** them across multiple downstream tasks.
- You can **execute** multiple scripts **sequentially** on the same worker or in the same container, minimizing latency.
- **Output artifacts** of each task (such as CSV, JSON or Parquet files you generate in your script) are directly available to other tasks without having to persist them within the internal storage. This is because all child tasks of the `WorkingDirectory` task share the same file system.

The `WorkingDirectory` task allows you to:
1. Share files from Namespace Files or from a Git repository across multiple tasks
2. Run multiple tasks sequentially in the same working directory
3. Share data across multiple tasks without having to persist it in internal storage.

---

## The `LocalFiles` task to manage `inputs` and `outputs` for your script code


The `LocalFiles` task is meant to be used inside the `WorkingDirectory` task. This task is equivalent to using the `inputFiles` and `outputFiles` properties on the `WorkingDirectory`, `Script` or `Commands` tasks.

For example, the `inputs` property can be used to add input files that might be needed in your script. Imagine that you want to add a custom `requirements.txt` file that contains exact pip package versions, as in this example:

```yaml
id: pip
namespace: dev

tasks:
  - id: wdir
    type: io.kestra.core.tasks.flows.WorkingDirectory
    tasks:
    - id: pip
      type: io.kestra.core.tasks.storages.LocalFiles
      inputs:
        requirements.txt: |
          kestra>=0.6.0
          pandas>=1.3.5
          requests>=2.31.0

    - id: pythonScript
      type: io.kestra.plugin.scripts.python.Script
      docker:
        image: python:3.11-slim
      beforeCommands:
        - pip install -r requirements.txt > /dev/null
      warningOnStdErr: false
      script: |
        import requests
        import kestra
        import pandas as pd

        print(f"requests version: {requests.__version__}")
        print(f"pandas version: {pd.__version__}")
        methods = [i for i in dir(kestra.Kestra) if not i.startswith("_")]
        print(f"Kestra methods: {methods}")
```

Adding such additional configuration files is possible by using the `inputs` property of the `LocalFiles` task. You provide:
- the **file name** as **key**
- the **file's content** as **value**

In the example shown above, the key is `requirements.txt` and the value is the typical content of a [`requirements` file](https://pip.pypa.io/en/stable/reference/requirements-file-format/#requirements-file-format) listing `pip` packages and their version constraints.


### Using input files to pass data from a trigger to a script task

Another use case for input files is when your custom scripts need input coming from other tasks or triggers.

Consider the following example flow that runs when a new object with the prefix `"raw/"` arrives in the S3 bucket `"declarative-orchestration"`:

```yaml
id: s3TriggerCommands
namespace: blueprint
description: process CSV file from S3 trigger

tasks:
  - id: wdir
    type: io.kestra.core.tasks.flows.WorkingDirectory
    tasks:
      - id: cloneRepo
        type: io.kestra.plugin.git.Clone
        url: https://github.com/kestra-io/examples
        branch: main

      - id: local
        type: io.kestra.core.tasks.storages.LocalFiles
        inputs:
          data.csv: "{{ trigger.objects | jq('.[].uri') | first }}"

      - id: python
        type: io.kestra.plugin.scripts.python.Commands
        description: this script reads a file `data.csv` from S3 trigger
        docker:
          image: ghcr.io/kestra-io/pydata:latest
        warningOnStdErr: false
        commands:
          - python scripts/clean_messy_dataset.py

      - id: output
        type: io.kestra.core.tasks.storages.LocalFiles
        outputs:
          - "*.csv"
          - "*.parquet"

triggers:
  - id: waitForS3object
    type: io.kestra.plugin.aws.s3.Trigger
    bucket: declarative-orchestration
    maxKeys: 1
    interval: PT1S
    filter: FILES
    action: MOVE
    prefix: raw/
    moveTo:
      key: archive/raw/
    accessKeyId: "{{ secret('AWS_ACCESS_KEY_ID') }}"
    secretKeyId: "{{ secret('AWS_SECRET_ACCESS_KEY') }}"
    region: "{{ secret('AWS_DEFAULT_REGION') }}"

```

Because the `LocalFiles` is a dedicated task independent of the Python task, it can be used to pass the S3 object key (*downloaded to Kestra's internal storage by the S3 trigger*) as a local file to any downstream task. Note that we didn't have to hardcode anything specific to Kestra in the Python script from GitHub. That script remains pure Python that you can run anywhere. Kestra's trigger logic is stored along with orchestration and infrastructure configuration in the YAML flow definition.

This separation of concerns (*i.e. not mixing orchestration and business logic*) makes your code easier to test and keeps your business logic vendor-agnostic.


### Managing output files: `LocalFiles.outputs`

Using the previous example, note how the `LocalFiles` can be used to output any file as downloadable artifacts. [This script from GitHub](https://github.com/kestra-io/examples/blob/main/scripts/clean_messy_dataset.py#L12) outputs a file named `clean_dataset.csv`. However, you don't have to hardcode that specific file name. Instead, a simple [Glob](https://en.wikipedia.org/wiki/Glob_(programming)) expression can be used to define that you want to output any CSV or Parquet file generated by that script:

```yaml
      - id: output
        type: io.kestra.core.tasks.storages.LocalFiles
        outputs:
          - "*.csv"
          - "*.parquet"
```
If any files matching that  [Glob](https://en.wikipedia.org/wiki/Glob_(programming)) patterns are generated during the flow's Execution, they will be available for download from the **Outputs** tab.