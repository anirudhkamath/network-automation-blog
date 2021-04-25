# Software packaging using Poetry

As we keep using more open source software, it becomes a challenge to effectively develop, test, and use the multiple software projects you wish to contribute to. They usually work with multiple dependencies that need to be installed for the project to work as expected. These dependencies can come packaged in some ways:

- In a `requirements.txt` file that has the package names with the required version/URLs that need to be installed.
- Using Docker, by packaging a Dockerfile along with the project that provides a way through which a Docker image can be built, using which containers can be spun up with the required dependencies installed in it.

While it is a neat approach, the Docker way requires the developer to have Docker installed and to be able to troubleshoot any Docker issues that may come up, if any. This approach is ideal in cases where the software application requires the use of extra services such as a database.

The issue with the first approach comes when 2 or more of your projects use the same dependency, but different versions. To not encounter any problems in this case, project specific virtual environments need to be maintained. To maintain this manually for multiple projects can be a taxing affair- this is where Poetry comes in to simplify things.

## What does Poetry do?

In a way, Poetry automates a lot of specifics in handling dependencies for your project. It provides a command line tool to add/remove new dependencies to your project, install project dependencies, and initialize your project software package in a simple way through the command line.

Essentially, what Poetry does is it maintains a virtual environment for your project, in which it installs and maintains all project dependencies by itself. The difference is that Poetry does this on the fly, rather than having you maintain a separate virtual environment directory for every project on your file system.

## Working with Poetry

To install Poetry, ensure you have Python installed in your environment, along with [cURL](https://curl.se/). Run this command:

```bash
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python -
```

To test if Poetry has installed properly, run `poetry --version` to see Poetry tell you which version has been installed.

> Poetry can be updated by running `poetry self update`, or `poetry self update <version>` to install a specific version

To set up a new Poetry enabled project, `poetry new <project-name>` is there to help you. It creates a directory with the following structure

```bash
(ins)~ ➜ tree <project-name>/
<project-name>/
├── README.rst
├── <project-name>
│   └── __init__.py
├── pyproject.toml
└── tests
    ├── __init__.py
    └── test_<project-name>.py

2 directories, 5 files
```

To enable Poetry in an existing project, move to the root of the project and run `poetry init`.

```bash
cd <project-name>
poetry init
```

Upon doing this, Poetry shall ask you some questions about the project specifics, using which it shall create a `pyproject.toml` file, and write some entries into a file called `poetry.lock` if dependencies are specified when answering the questions Poetry has for you.

```bash
(ins)<project-name> ➜ poetry init

This command will guide you through creating your pyproject.toml config.

Package name [poetry-practice]:  test
Version [0.1.0]:
Description []:  Test description
Author [Random Person <random_person@somewhere.com>, n to skip]:
License []:
Compatible Python versions [^3.8]:

Would you like to define your main dependencies interactively? (yes/no) [yes] no
Would you like to define your development dependencies interactively? (yes/no) [yes] no
Generated file

[tool.poetry]
name = "test"
version = "0.1.0"
description = "Test description"
authors = ["Random Person <random_person@somewhere.com>"]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"


Do you confirm generation? (yes/no) [yes] yes
```

When being prompted, simply pressing enter makes Poetry take the default value (specified in the `[]` brackets).

To specify dependencies and make sure Poetry installs them, use the `poetry add` command within the project directory. For this project, I may want to install Netmiko 3.2.0:

```bash
(ins)<project-name> ➜ poetry add netmiko==3.2.0
Creating virtualenv test-wQCgqYUH-py3.8 in /home/username/.cache/pypoetry/virtualenvs

Updating dependencies
Resolving dependencies... (2.9s)

Writing lock file

Package operations: 12 installs, 0 updates, 0 removals

  • Installing pycparser (2.20)
  • Installing cffi (1.14.5)
  • Installing six (1.15.0)
  • Installing bcrypt (3.2.0)
  • Installing cryptography (3.4.7)
  • Installing pynacl (1.4.0)
  • Installing future (0.18.2)
  • Installing paramiko (2.7.2)
  • Installing pyserial (3.5)
  • Installing scp (0.13.3)
  • Installing textfsm (1.1.0)
  • Installing netmiko (3.2.0)
```

Upon checking in the `pyproject.toml` file, it can be seen that `netmiko 3.2.0` is installed in the project:

```toml
[tool.poetry]
name = "test"
version = "0.1.0"
description = "Test description"
authors = ["Random Person <random_person@somewhere.com>"]

[tool.poetry.dependencies]
python = "^3.8"
netmiko = "3.2.0"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

The fun thing with Poetry is when you may want to run some specific commands within your Poetry virtual environment and have it work with your dependencies- this may be some unit tests for your project, or maybe a linter to make sure your files are in compliance with linting standards. For example, `pytest` when added as a dependency for the project can be run using Poetry as such:

```bash
poetry add pytest # adds and installs pytest as a dependency
poetry run pytest # run unit tests
```

To enter the virtual environment Poetry has created for the project, run `poetry shell` within the project. To exit the virtual environment once activated, just run `exit`, as you would within a shell environment.

To ensure all developers in the team have the same dependencies with the required versions, make sure the `pyproject.toml` and `poetry.lock` file are included in vesion control and pushed to remote repositories where other engineers can pull from.

When an engineer wants to install project dependencies to use the project, running `poetry install` within the project installs them within the Poetry virtual environment for the project.

> To install dependencies only and not the project itself- run `poetry install --no-root`

To remove dependencies, running `poetry remove <package-name>` does the job within the Poetry virtual environment.

To view more Poetry CLI options, the documentation can be found [here](https://python-poetry.org/docs/cli/).
