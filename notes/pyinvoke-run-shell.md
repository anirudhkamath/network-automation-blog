# Using PyInvoke to run shell tasks

One of the advantages of developing software applications using Python is that there is a library or tool for doing nearly everything in the development process. In my experience, the process of building a software application goes something like this:

1. Programming the core entrypoint functions of the app (your `def main()`, or main Python executable)
2. Developing base classes, auxiliary classes and their member methods and properties (if you subscribe to the object oriented programming way of thinking) OR helper functions if your application is more functional than object oriented
3. Weaving together those methods and classes developed into logical library directories/files that can be imported into your main executable, or each other
4. Defining your project's [packaging configuration](https://packaging.python.org/en/latest/flow/#the-configuration-file) if you intend to distribute your application package on PyPI (Python packaging index) or your own packaging index
5. Building tests
6. Developing API endpoints, proto defintions, etc.
7. Developing build artifacts (containerization of the application, or manifests for a Kubernetes deployment)
8. ...

To complete all these steps, developers must use multiple tools (think Django/Flask, Python-poetry/pip(x)/setuptools, pytest, docker, kubectl, ...) in their arsenal. It may be that many of the command line commands used to generate required artifacts and results may be long, multi-line commands with multiple flags, piped, multi stage, etc. For a new developer joining the team, being able to replicate these commands and steps can be quite annoying.
This where tools such as `make` and `pyinvoke` come in handy as it allows developers to provide "aliases" for such commands or workflows.

## About PyInvoke

[PyInvoke](https://docs.pyinvoke.org/en/stable/) or Invoke is a Python library that provides a high level Pythonic interface to interact with your host shell. This means that you can use pyinvoke to expose repeatable shell based tasks as Pythonic processes. This is useful as it marries the advantages that come with programming in a high level general purpose programming language such as Python with the interactive nature of your shell program. Pythonic features such as **exception handling**, **cross platform portability**, its **standard/vendored libraries**, and **extensibility** make `pyinvoke` an extremely handy tool for Python developers.

> [Installing invoke](https://www.pyinvoke.org/installing.html)

### Invoke tasks

Invoke is designed to look for a `tasks.py` file in a given project directory to understand what commands it can expose for the user to *invoke*. This file should be programmed with the various **tasks** (or commands as referenced) the user can run with the invoke program: `invoke <task-xyz>` where `task_xyz` is defined with the `@task` decorator:

```python
from invoke import task  # the wrapper around the high level API to the shell subprocesses.

# decorate your task function as such.
@task
def task_xyz(ctx, a, b):
    """
    Task XYZ
    
    This task can be called `invoke task-xyz -a value_a -b value_b`
    The flags `a` and `b` are string values.
    If other types need to be used, then a default value of the type can be given to the flag arg:

    def task_xyz(ctx, a=5, b=False):
    """
```

> [Dashes v/s underscores for invoke tasks](https://docs.pyinvoke.org/en/stable/concepts/namespaces.html#dashes-vs-underscores)

This `task()` wrapper allows the programmer to provide the following helpful properties for the command task (there are more options but I think these are used more frequently):

- `help`: `dict` holding informational descriptions for each flag the task exposes
- `optional`: The flags that can optionally hold a value of multiple types depending on what the user provides. Mostly used if you want an option to serve both as a boolean (true/false) but also be able to accept string values, such as a file name for logging or URL string if you need the command to send an HTTP request. This can help keep your high level API task command concise and not require an inordinate number of flags
- `pre`: The task that must be run before this task starts (pre-processor)
- `post`: The task that must be run after this task completes (post-processor)
- `iterable`: If a flag can accept an array/list of values, then it should be declared as an iterable

An example:

```python

@task
def pre_task():
    print("pre task")

@task
def post_task():
    print("post task")

@task(
    help={
        "http": "Send request as an HTTP request, defaults to `localhost:8080` if this flag is invoked as boolean",
        "log_file": "Log into a file, defaults to `/opt/app/file.log` if flag is invoked as boolean",
    },
    optional=["http", "log"],
    pre=[pre_task],
    post=[post_task],
    iterable=[],
)
def test_api_endpoint(context, http=None, log=None):  # default of None means it is a string
    """
    Task to test an API endpoint
    """
    if http:
        http_url = "http://localhost:8080"
        if isinstance(http, str):
            http_url = http
        logfile = "/opt/app/file.log"
        if log:
            if isinstance(log, str):
                logfile = log
            http_resp: dict = make_http_request(url=http_url, logfile=logfile)
        else:
            http_resp: dict = make_http_request(url=http_url)
    else:
        ...  # Python Ellipsis

```

### The context argument

Invoke tasks require a `context` argument which is the first positional argument in the task defintion. This context allows for the sharing of "global" data coming from invoke configuration with the commands run via the `run()` method that is meant to run processes in the shell. This context is helpful in cases such as when the process needs to be executed in a custom shell program, or in case the `sudo`/root password prompt needs to be handled to complete the shell process.

> https://docs.pyinvoke.org/en/stable/getting-started.html#why-context

### Running shell commands inside tasks

Now that we understand invoke's tasks and the context arg for those tasks, we finally come to the crux of invoke, running commands in the target shell. This is where the context provides information as to how to run our shell processes. Inside the task, you can run a shell command as such:

```python
@task
def mytask(ctx):
    ctx.run("echo hello world")
```

### Using invoke to build your own CLI programs

Invoke can also be used to create your own binary executables as their own **programs**. [Check this out](https://docs.pyinvoke.org/en/stable/concepts/library.html) for documentation from the invoke project about the same.

## Network automation example

To outline how invoke tasks can be helpful in a project, I describe a small exercise of wrapping Batfish questions against network configurations inside invoke tasks that can be used by network administrators to get summaries for their network.

The project tree holds directories for regions with network sites in these regions. The site level directories hold the network configs that Batfish will interpret. [Batfish](https://pybatfish.readthedocs.io/en/latest/index.html) has proven to be a helpful tool for network administrators to gain summaries or answers to certain "questions" about their network configurations -- summaries can include whether BGP sessions between two (say, iBGP) peers are compatible or misconfigured (maybe an incorrect peer AS or peer IP address), or summaries of access policies to certain networks based on ACLs configured on the network devices.

Project tree:

```bash
.
├── Dockerfile-batfish
├── poetry.lock
├── pyproject.toml
├── regions
│   ├── africa
│   ├── asia
│   │   └── bangalore
│   │       └── configs
│   │           ├── router1.cfg
│   │           ├── switch1.cfg
│   │           └── switch2.cfg
│   ├── australia
│   ├── europe
│   ├── north_america
│   │   └── nyc
│   │       └── configs
│   │           ├── core-rtr01.cfg
│   │           ├── core-rtr02.cfg
│   │           ├── edge-firewall01.cfg
│   │           ├── edge-sw01.cfg
│   │           └── internet-rtr01.cfg
│   └── south_america
└── tasks.py
```

- This Python project is packaged with dependencies and its virtual environment using [python-poetry](poetry-package-manager.md)
- There is a `Dockerfile` that specifies the build layers for building the Batfish docker image so we can run the Batfish server inside a docker container
- There is a `tasks.py` file holding our invoke tasks, described below
- There is a `regions/` directory that holds subdirectories/folders for each network site in that region (one site is a network in this example) -- these subdirectories hold network configurations that Batfish will analyze for the user. Our configuration files are from Cisco network devices running IOS/IOS-XE/ASA operating systems

PyBatfish will be invoked by the invoke tasks developed to be able to interpret the network configurations in their regional directories. The BGP sessions analyzer task look as such:

```python
@task(
    pre=[
        tasks.call(docker_build, dockerfile="Dockerfile-batfish"),
        tasks.call(docker_run, image="batfish/latest", publish_exposed_ports=["8888", "9996", "9997"]),
    ],
    post=[tasks.call(docker_stop),]
)
def bgp_sessions(ctx, region="", site=""):
    '''
    Analyze BGP sessions for a site using Batfish.
    '''
    if not site or not region:
        sys.exit("Site/region not given -- do not know which site to analyze BGP sessions for, exiting!")

    # ctx.run("mkdir -p /tmp/batfish/snapshots")  # create snapshots directory

    bf = Session(host="localhost", ssl=False, verify_ssl_certs=False)
    bf.set_network(f"{site}")

    SNAPSHOT_DIR = f'./regions/{region}/{site}/'
    tmp_uuid = uuid.uuid1().hex
    bf.init_snapshot(SNAPSHOT_DIR, name=f'snapshot-bgpsessions-{str(datetime.date.today())}-{tmp_uuid}', overwrite=True)

    print("running bgp session status")
    result = bf.q.bgpSessionStatus().answer().frame()
    
    print("BGP sessions analysis:\n")
    for res in result.iloc:
        print(res)
        print("\n")
```

A run down of what is going on here:

- The task here is called `bgp_sessions` and accepts two flags, `region` and `site`. The region and site are required to know which network Batfish should analyze
- This task [interacts with the Batfish service](https://pybatfish.readthedocs.io/en/latest/notebooks/interacting.html) that we run using a docker container. The task is configured to run tasks `docker_build` and `docker_run` as specified in the `pre` section of the task decorator, and `docker_stop` in the `post` section -- these are defined as such:

  ```python
  @task()
  def docker_build(ctx, dockerfile="", tag=None):
      """
      Docker build task.
      """
      if not dockerfile:
          sys.exit("Dockerfile not provided to docker build, exiting")
  
      if not tag:
          docker_tag = "latest"
      else:
          docker_tag = tag
  
      command = f"docker build -t batfish/{docker_tag} -f {dockerfile} ."
      ctx.run(command)
      print("\n\n Docker build completed\n\n")
  
  @task(
      optional=["publish_exposed_ports"]  # user can also provide a ports list with this.
  )
  def docker_run(ctx, image="", name="", publish_exposed_ports=False):
      if not image:
          sys.exit("Docker image not provided to docker run, exiting")
  
      if not name:
          docker_name="batfish"
      else:
          docker_name=name
  
      if publish_exposed_ports:
          if isinstance(publish_exposed_ports, list):
              # user has provided a list
              ports = ""
              for port in publish_exposed_ports:
                  if not isinstance(port, str):
                      sys.exit(f"Port provided is not of type str: {port}, exiting")
                  ports += f" -p {port}:{port}"
              command = f"docker run{ports} --detach --rm --name {docker_name} {image}"
          else:
              command = f"docker run -d -P --rm --name {docker_name} {image}"
      else:
          command = f"docker run -d --rm --name {docker_name} {image}"
  
      ctx.run(command)
  
      print("\n\nDocker run done\n\n")

  @task()
  def docker_stop(ctx, name="batfish"):
      ctx.run(f"docker stop {name}")
  ```

  Here `docker_build` will build an image with a given tag from a dockerfile. It is designed to build an image with the name `batfish`. `docker_run` will create an image from a given docker image and name it as per the provided container name; it also allows the invoker to provide a list of ports (provided as a list of strings) that must be exposed to the host the containers reside on -- if the invoker just calls this flag without any value then the container simply publishes the ports that are exposed as per the Dockerfile to ephemeral ports on the host. The Dockerfile looks like this:

  ```Dockerfile
  FROM batfish/allinone

  RUN mkdir /opt/app/
  COPY regions/ /opt/app/
  
  EXPOSE 8888
  EXPOSE 9997
  EXPOSE 9998
  ```

- finally the Batfish analyzed result is printed to the console and the user sees what Batfish inferred from the network configurations in the site within the region specified by the user

Once these are sufficiently defined, a user who did not develop the functionalities mentioned above can simply install the `invoke` tool as mentioned in the beginning of this article, and run

```bash
invoke bgp-sessions -r north_america -s nyc
```

or

```bash
invoke bgp-sessions --region north_america --site nyc
```

To run the invoke task to analyze the BGP sessions configured for the network in site `nyc` within region `north_america` in the directory structure. The result will appear as such:

```bash
Your snapshot was successfully initialized but Batfish failed to fully recognized some lines in one or more input files. Some unrecognized configuration lines are not uncommon for new networks, and it is often fine to proceed with further analysis. You can help the Batfish developers improve support for your network by running:

    bf.upload_diagnostics(dry_run=False, contact_info='<optional email address>')

to share private, anonymized information. For more information, see the documentation with:

    help(bf.upload_diagnostics)
running bgp session status
BGP sessions analysis:

Node                  internet-rtr01
VRF                          default
Local_AS                       12122
Local_Interface                 None
Local_IP                        None
Remote_AS                      65000
Remote_Node                     None
Remote_Interface                None
Remote_IP               104.34.34.34
Address_Families                  []
Session_Type          EBGP_SINGLEHOP
Established_Status    NOT_COMPATIBLE
```

The network admin/user who runs this invoke task can now see, without having to go through the configuration files or understand the syntax of such configurations, that there is a session configured on `internet-rtr01` in the default VRF with the ASN 12122, peering with remote AS 65000/remote IP 104.34.34.34 as a single hop eBGP session. As Batfish could not find this remote IP in the configurations it analyzed, it flagged the status of such a session as `NOT_COMPATIBLE` -- Batfish has documented what possible statuses it can show and the meaning of such [here](https://pybatfish.readthedocs.io/en/latest/notebooks/routingProtocols.html#BGP-Session-Status).

The power of invoke is displayed here -- we are able to wrap workflows or tasks that may otherwise be handled manually inside a high level task that can be invoked by a user that just wants to use the software application. It provides a simple user interface for the user that is command line based.

## Final notes

Invoke is a multifaceted Python library that can do more than just expose tasks that can be run with the `invoke` command, it can also allow developers to create application binaries that provide subcommands that can be used in CI or CD workflows. It is a project that has drawn inspiration from `make`, Ruby's `rake`, and its predecessor `fabric` -- to provide a high level API to run commands and processes on host shells (fabric is actually designed to also run on remote shells!).

You can find the code I described above [here](https://github.com/anirudhkamath/batfish_playground).

Some helpful places to understand more about invoke:

- [Invoke FAQs](https://www.pyinvoke.org/faq.html)
- \[advanced\] [configuring invoke](https://docs.pyinvoke.org/en/stable/concepts/configuration.html)
- [about tasks](https://docs.pyinvoke.org/en/stable/concepts/invoking-tasks.html)
- \[advanced\] [building CLI programs with invoke as a Python library](https://docs.pyinvoke.org/en/stable/concepts/library.html)
- [understanding the context arg for tasks](https://docs.pyinvoke.org/en/stable/getting-started.html#why-context)
- [Get started](https://docs.pyinvoke.org/en/stable/getting-started.html)

  