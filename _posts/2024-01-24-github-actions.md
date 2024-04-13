---
title: Analysis of Github Actions
subtitle: enforcing immutability in your workflows
layout: post
author: martoc
image: https://martoc.github.io/blog/images/github.webp
---

The examination of GitHub Actions involves a comprehensive evaluation of its functionalities, features, and overall effectiveness. GitHub Actions is a powerful tool for automating workflows within the GitHub platform, enabling seamless integration and continuous delivery processes. It facilitates the automation of tasks such as code compilation, testing, and deployment, contributing to an efficient and streamlined development pipeline.

In conducting an analysis, it is essential to assess the various components and capabilities that GitHub Actions offers. This includes examining workflow configuration, understanding the use of actions, and evaluating the integration with version control repositories. Additionally, considerations should be given to factors such as scalability, flexibility, and ease of use.

Furthermore, the analysis should delve into the benefits and potential drawbacks associated with GitHub Actions. Assessing its impact on development speed, code quality, and collaboration among team members provides valuable insights into its real-world applicability.

In summary, a thorough analysis of GitHub Actions involves exploring its features, understanding its configuration, and evaluating its impact on development processes. This ensures a comprehensive understanding of its capabilities and aids in making informed decisions regarding its adoption in software development workflows.

GitHub offers three distinct methods for creating actions: 
JavaScript, compose, and container. 
Each of them possesses its own advantages 
and disadvantages, and, on the whole, 
all of them enforce mutability in your runs.

## JavaScript

This model installs Node.js on the running container. 
It is natively supported by the action engine, 
and therefore, GitHub manages the installation process. 
One positive aspect is that the JavaScript code can be 
packaged into a single file, eliminating the need to 
install dependencies.

An example action can be found at [martoc/my-custom-javascript-action](https://github.com/martoc/my-custom-javascript-action)

This action is used at [martoc/github-action-test](https://github.com/martoc/github-action-test/actions)

```yaml
name: javascript-version
run-name: ${{ github.event.head_commit.message }}
on: [push]
jobs:
  action:
    runs-on: ubuntu-22.04
    steps:
      - name: Run my javascript custom action
        uses: martoc/my-custom-javascript-action@3709a7f133773d77e83b4e151b77b142a73ce0a4
```

If you look at the logs for this execution, it looks like the nodejs is installed on the running container.

```
Run martoc/my-custom-javascript-action@3709a7f133773d77e83b4e151b77b142a73ce0a4
Mocking an AWS call via JS AWS SDK
```

Futher investigation shows that the nodejs is installed on the running container.

```bash
Run node --version || true
v18.19.0
```

However the version that the command requires is nodejs v20, if you run the same command on the container you will see that the version is v18.19.0
before and after the custom command is executed. Therefore the GitHub action engine for Javascript commands is installing nodejs on the running container or executing with nodejs v18.19.0. To check this I have added a command to the action to print the nodejs version.

```javscript
console.log('Node.js version: ' + process.version);
```

```bash
Run martoc/my-custom-javascript-action@799f06db8e843cbd3fc38b04fa3f6e4d19d615dd
Mocking an AWS call via JS AWS SDK
Node.js version: v20.8.1
```

## Composite

This configuration enables you to execute any shell script, 
including Python. However, Python must be installed in the 
running container, along with the dependencies required 
by your script.

An example action can be found at [martoc/my-custom-python-action](https://github.com/martoc/my-custom-python-action)

This action is used at [martoc/github-action-test](https://github.com/martoc/github-action-test/actions)

```yaml
name: 'Custom Python GitHub Action'
description: 'Calls Python script'
author: martoc
runs:
  using: 'composite'
  steps:
    - name: Install Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - name: Install Dependencies
      run: pip install -r ${{ github.action_path }}/requirements.txt
      shell: bash
    - name: call script
      id: aws-call
      run: python ${{ github.action_path }}/src/aws-call.py
      shell: bash
```

There are 3 steps in this action, the first one installs python, the second one installs the dependencies and the third one executes the python script.

```bash
Run martoc/my-custom-python-action@6442800227f376e3aa348572d7e4902173e861bf
Run actions/setup-python@v4
  with:
    python-version: 3.10
    check-latest: false
    token: ***
    update-environment: true
    allow-prereleases: false
Installed versions
  Successfully set up CPython (3.10.13)
Run pip install -r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt
Collecting boto3 (from -r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading boto3-1.34.28-py3-none-any.whl.metadata (6.6 kB)
Collecting botocore<1.35.0,>=1.34.28 (from boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading botocore-1.34.28-py3-none-any.whl.metadata (5.7 kB)
Collecting jmespath<2.0.0,>=0.7.1 (from boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading jmespath-1.0.1-py3-none-any.whl (20 kB)
Collecting s3transfer<0.11.0,>=0.10.0 (from boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading s3transfer-0.10.0-py3-none-any.whl.metadata (1.7 kB)
Collecting python-dateutil<3.0.0,>=2.1 (from botocore<1.35.0,>=1.34.28->boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading python_dateutil-2.8.2-py2.py3-none-any.whl (247 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 247.7/247.7 kB 6.6 MB/s eta 0:00:00
Collecting urllib3<2.1,>=1.25.4 (from botocore<1.35.0,>=1.34.28->boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading urllib3-2.0.7-py3-none-any.whl.metadata (6.6 kB)
Collecting six>=1.5 (from python-dateutil<3.0.0,>=2.1->botocore<1.35.0,>=1.34.28->boto3->-r /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/requirements.txt (line 1))
  Downloading six-1.16.0-py2.py3-none-any.whl (11 kB)
Downloading boto3-1.34.28-py3-none-any.whl (139 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 139.3/139.3 kB 18.1 MB/s eta 0:00:00
Downloading botocore-1.34.28-py3-none-any.whl (11.9 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 11.9/11.9 MB 72.0 MB/s eta 0:00:00
Downloading s3transfer-0.10.0-py3-none-any.whl (82 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 82.1/82.1 kB 13.0 MB/s eta 0:00:00
Downloading urllib3-2.0.7-py3-none-any.whl (124 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 124.2/124.2 kB 16.0 MB/s eta 0:00:00
Installing collected packages: urllib3, six, jmespath, python-dateutil, botocore, s3transfer, boto3
Successfully installed boto3-1.34.28 botocore-1.34.28 jmespath-1.0.1 python-dateutil-2.8.2 s3transfer-0.10.0 six-1.16.0 urllib3-2.0.7

Notice:  A new release of pip is available: 23.0.1 -> 23.3.2
Notice:  To update, run: pip install --upgrade pip
Run python /home/runner/work/_actions/martoc/my-custom-python-action/6442800227f376e3aa348572d7e4902173e861bf/src/aws-call.py
```

## Container 

In this mode, the execution is wrapped in a container, 
offering a more controlled environment for 
running your scripts. However, this container is built 
during runtime.

An example action can be found at [martoc/my-custom-container-action](https://github.com/martoc/my-custom-container-action)

This action is used at [martoc/github-action-test](https://github.com/martoc/github-action-test/actions)

```yaml
name: 'Custom container GitHub Action'
description: 'Calls container script'
author: martoc
runs:
  using: docker
  image: Dockerfile
```

There is only one step in this action, the execution is wrapped in a container using the Dockerfile.

```bash
Build container for action use: '/home/runner/work/_actions/martoc/my-custom-container-action/a926ec3166e87135ce845ba817965c1c8ba8fa16/Dockerfile'.
  /usr/bin/docker build -t d4da51:aae3b92beba644a59acbc8c2101f8b14 -f "/home/runner/work/_actions/martoc/my-custom-container-action/a926ec3166e87135ce845ba817965c1c8ba8fa16/Dockerfile" "/home/runner/work/_actions/martoc/my-custom-container-action/a926ec3166e87135ce845ba817965c1c8ba8fa16"
  #0 building with "default" instance using docker driver
  
  #1 [internal] load .dockerignore
  #1 transferring context: 2B done
  #1 DONE 0.0s
  
  #2 [internal] load build definition from Dockerfile
  #2 transferring dockerfile: 286B done
  #2 DONE 0.0s
  
  #3 [auth] library/alpine:pull token for registry-1.docker.io
  #3 DONE 0.0s
  
  #4 [internal] load metadata for docker.io/library/alpine:3.12
  #4 DONE 0.4s
  
  #5 [internal] load build context
  #5 transferring context: 90B done
  #5 DONE 0.0s
  
  #6 [1/3] FROM docker.io/library/alpine:3.12@sha256:c75ac27b49326926b803b9ed43bf088bc220d22556de1bc5f72d742c91398f69
  #6 resolve docker.io/library/alpine:3.12@sha256:c75ac27b49326926b803b9ed43bf088bc220d22556de1bc5f72d742c91398f69 done
  #6 extracting sha256:1b7ca6aea1ddfe716f3694edb811ab35114db9e93f3ce38d7dab6b4d9270cb0c
  #6 sha256:1b7ca6aea1ddfe716f3694edb811ab35114db9e93f3ce38d7dab6b4d9270cb0c 2.81MB / 2.81MB 0.1s done
  #6 sha256:c75ac27b49326926b803b9ed43bf088bc220d22556de1bc5f72d742c91398f69 1.64kB / 1.64kB done
  #6 sha256:cb64bbe7fa613666c234e1090e91427314ee18ec6420e9426cf4e7f314056813 528B / 528B done
  #6 sha256:24c8ece58a1aa807c0d8ea121f91cee2efba99624d0a8aed732155fb31f28993 1.47kB / 1.47kB done
  #6 extracting sha256:1b7ca6aea1ddfe716f3694edb811ab35114db9e93f3ce38d7dab6b4d9270cb0c 0.1s done
  #6 DONE 0.2s
  
  #7 [2/3] WORKDIR /usr/src
  #7 DONE 0.0s
  
  #8 [3/3] COPY entrypoint.sh .
  #8 DONE 0.0s
  
  #9 exporting to image
  #9 exporting layers
  #9 exporting layers 1.0s done
  #9 writing image sha256:c56abace66f1f3d2c8915b4cb09bd62d6095c2a0792900f063759073af46642c done
  #9 naming to docker.io/library/d4da51:aae3b92beba644a59acbc8c2101f8b14 done
  #9 DONE 1.0s
  Run martoc/my-custom-container-action@a926ec3166e87135ce845ba817965c1c8ba8fa16
/usr/bin/docker run --name d4da51aae3b92beba644a59acbc8c2101f8b14_cc3c9a --label d4da51 --workdir /github/workspace --rm -e "HOME" -e "GITHUB_JOB" -e "GITHUB_REF" -e "GITHUB_SHA" -e "GITHUB_REPOSITORY" -e "GITHUB_REPOSITORY_OWNER" -e "GITHUB_REPOSITORY_OWNER_ID" -e "GITHUB_RUN_ID" -e "GITHUB_RUN_NUMBER" -e "GITHUB_RETENTION_DAYS" -e "GITHUB_RUN_ATTEMPT" -e "GITHUB_REPOSITORY_ID" -e "GITHUB_ACTOR_ID" -e "GITHUB_ACTOR" -e "GITHUB_TRIGGERING_ACTOR" -e "GITHUB_WORKFLOW" -e "GITHUB_HEAD_REF" -e "GITHUB_BASE_REF" -e "GITHUB_EVENT_NAME" -e "GITHUB_SERVER_URL" -e "GITHUB_API_URL" -e "GITHUB_GRAPHQL_URL" -e "GITHUB_REF_NAME" -e "GITHUB_REF_PROTECTED" -e "GITHUB_REF_TYPE" -e "GITHUB_WORKFLOW_REF" -e "GITHUB_WORKFLOW_SHA" -e "GITHUB_WORKSPACE" -e "GITHUB_ACTION" -e "GITHUB_EVENT_PATH" -e "GITHUB_ACTION_REPOSITORY" -e "GITHUB_ACTION_REF" -e "GITHUB_PATH" -e "GITHUB_ENV" -e "GITHUB_STEP_SUMMARY" -e "GITHUB_STATE" -e "GITHUB_OUTPUT" -e "RUNNER_OS" -e "RUNNER_ARCH" -e "RUNNER_NAME" -e "RUNNER_ENVIRONMENT" -e "RUNNER_TOOL_CACHE" -e "RUNNER_TEMP" -e "RUNNER_WORKSPACE" -e "ACTIONS_RUNTIME_URL" -e "ACTIONS_RUNTIME_TOKEN" -e "ACTIONS_CACHE_URL" -e "ACTIONS_RESULTS_URL" -e GITHUB_ACTIONS=true -e CI=true -v "/var/run/docker.sock":"/var/run/docker.sock" -v "/home/runner/work/_temp/_github_home":"/github/home" -v "/home/runner/work/_temp/_github_workflow":"/github/workflow" -v "/home/runner/work/_temp/_runner_file_commands":"/github/file_commands" -v "/home/runner/work/github-action-test/github-action-test":"/github/workspace" d4da51:aae3b92beba644a59acbc8c2101f8b14
Mocking an AWS call via AWS CLI
```

One of the biggest advantages of this method is that the environment where your scripts run can be controlled. However the container is built during runtime, therefore it is not immutable. This only works in linux runners, if you try to run this action in a windows runner you will get the following error.

```bash
Run martoc/my-custom-container-action@a926ec3166e87135ce845ba817965c1c8ba8fa16
Error: Container action is only supported on Linux
```

# Conclusion

To maximize immutability during runtime, considering 
the mentioned options, the 
Container method provides a more 
controlled environment. However, 
it's essential to be aware that it 
still enforces mutability. 
To achieve maximum immutability, 
it's advisable to explore additional 
strategies or tools that specifically 
focus on immutable runtime environments 
for example Distributing binaries, where 
dependencies are fully contained and run directly 
in the running container, will diminish 
mutability to an HTTP pull.



