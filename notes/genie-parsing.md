This post intends to cover the basics behind how the Genie parser works, and how to write a simple Genie parser to interpret data from the CLI of a Cisco IOS-XE router.

# pyATS and Genie

pyATS was developed at Cisco to provide an end to end automation ecosystem to effectively manage and run tests for network infrastructure. Alongside pyATS came Genie- a library solution to allow developers to leverage core pyATS functionality to write extensive tests, validation scripts, and to allow programmatic interfacing to network devices.

This post will cover the topic of parsing data from network devices using Genie, and writing a parser using the Genie SDK.

# How does Genie parse data?

As parsing is done to convert raw textual data into data objects that can be consumed programmatically, there are 2 main components to any parser that you may come across (such as [TextFSM](textfsm.md)):

1. One that defines the data structure that comes out as a result of parsing must be defined.
2. One that powers the actual act of parsing, encompassing the capture data from text information given as output from a network device (or text file).

To allow developers to build their own parsers, or to use community driven parsers, Genie provides 2 packages for the same. These packages are:

- **genie.metaparser**: In modern day network topologies, devices are capable of being interacted with using the CLI, and multiple other protocols such as NETCONF and RESTCONF. Hence, the data format coming out of these devices, depending on the use case, can be CLI data, XML, or YANG output. The metaparser allows the creation of a standard output schema irrespective of what was parsed in what format.
- **genie.libs.parser**: This package includes all the Python classes that power the parsing of data extracted from network devices.

To build and use your own Genie parser, the metaparser is the main package that will be used. The methodology behind Genie parsers is covered well in [this piece of documentation](https://github.com/CiscoTestAutomation/genieparser/blob/master/CODING.md#genie-metaparser--genie-libs-parsers) found on the GitHub repository for [genieparser](https://github.com/CiscoTestAutomation/genieparser).

# Building the parser

To write a Genie parser, we need a file with 2 Python classes: one to define the parsed data schema, and the other that runs the actual parsing. Before developing the parser, create a virtual environment and install pyATS and Genie to make sure all required packages can be used to create your parser.

```bash
pip install pyats[full]
```

In this post I will describe the parsing of the most basic form of output from the `show inventory` command for an IOS-XE router. A much more comprehensive parser ships with `genie.libs.parser` in the genieparser GitHub project, and can be found [here](https://github.com/CiscoTestAutomation/genieparser/blob/master/src/genie/libs/parser/iosxe/show_platform.py).

## The schema class

This class inherits from the `MetaParser` class in the `genie.metaparser` module. The `MetaParser` class has a member variable called `schema`, which needs to be defined in this class.

```python
import re

# Metaparser
from genie.metaparser import MetaParser
from genie.metaparser.util.schemaengine import Optional, Any


# ====================================================
#  Schema for show inventory
# ====================================================
class ShowInventorySchema(MetaParser):
    """Schema for:
        show inventory"""

    schema = {
        'inventory': {
            Any(): {
                Optional('name'): str,
                Optional('description'): str,
                Optional('pid'): str,
                Optional('vid'): str,
                Optional('serial'): str,
            },
        },
    }
```

Here, the schema is a dictionary that defines a key called `inventory`, which will hold indexed key-value pairs of items in the device inventory. `Optional()` tells the schema that it is not necessary that this key be defined for every entry, and `Any()` tells the schema to match any data type for that key.

The schema is effective in conveying to developers and users of the parser the expected format of data that will be returned by the parser. One great thing about genie parsers is that the schemas of the outputs returned by the parser is well documented, and can be found [here](https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/parsers).

## The parser class

Genie parsers, like all other parsers, use [Regular Expressions (RegEx)](https://docs.python.org/3/library/re.html) for matching patterns found in text output. Which is why the `re` module was imported previously.

The parser class inherits from the schema class to get all context required to effectively parse the output returned by the device. The parser class does a couple of things:

- Gives the developer a way to define different parsing schemes for different data formats like CLI, XML, or YANG. Member methods such as `cli()` or `xml()` can be used to define this.
- Provides access to the raw output data returned by executing the command on the device. The command executed on the device is a class member variable called `cli_command`.

So all that is left is to loop over the lines in the output data and parse using RegEx!

```python
# ================================
# Parser for 'show inventory'
# ================================
class ShowInventory(ShowInventorySchema):
    
    cli_command = 'show inventory'

    def cli(self, output=None):

        if output is None:
            out = self.device.execute(self.cli_command)
        else:
            out = output


        # NAME: "Chassis", DESCR: "Cisco CSR1000V Chassis"
        # PID: CSR1000V          , VID: V00  , SN: 9ZSGNIG46EE

        # NAME: "module R0", DESCR: "Cisco CSR1000V Route Processor"
        # PID: CSR1000V          , VID: V00  , SN: JAB1303001C

        # NAME: "module F0", DESCR: "Cisco CSR1000V Embedded Services Processor"
        # PID: CSR1000V          , VID:      , SN:

        # pattern to capture name and description
        p1 = re.compile(r'\s*NAME\s*:\s*"(?P<name>.*)"\s*,\s*DESCR\s*:\s*"(?P<description>.*)"')

        # pattern to capture product ID, version ID, and serial number.
        p2 = re.compile(r'\s*PID\s*:\s*(?P<pid>\S+)\s*,\s*VID\s*:\s*(?P<vid>.*)\s*,\s*SN\s*:\s*(?P<serial>.*)\s*')

        parsed_dict = {}
        inventory_index = 0

        for line in out.splitlines():
            line = line.strip()

            result = p1.match(line)

            if result:
                # setdefault allows assigned var to set value of key (first arg) in dict
                inventory_dict = parsed_dict.setdefault('inventory',{}) \
                    .setdefault(inventory_index,{})
                group = result.groupdict()

                inventory_dict['name'] = group['name']
                inventory_dict['description'] = group['description']

                continue

            result = p2.match(line)

            if result:
                inventory_dict = parsed_dict.setdefault('inventory',{}) \
                    .setdefault(inventory_index,{})
                group = result.groupdict()

                inventory_dict['pid'] = group['pid']
                inventory_dict['vid'] = group['vid']
                inventory_dict['serial'] = group['serial']

                inventory_index = inventory_index + 1 # move onto next entry

                continue

        return parsed_dict
```

# Using the parser

PyATS works well with something called a testbed. This testbed is a YAML file that holds information about the devices that pyATS can work against and operate with. Using the `genie` command line tool, it is simple to make a small testbed.

1. In the directory where your parser is created, run `genie create testbed interactive --output testbed.yaml` to create a testbed file called `testbed.yaml`. This will start a prompt asking you to enter details about your devices so that it can make your testbed for you.
2. Once you answer all the questions asked by the prompt for your devices, a file called `testbed.yaml` is created. It may look something like this:

    ```yaml
    devices:
    dist-rtr01:
        connections:
        cli:
            ip: 10.10.20.175
            protocol: telnet
        credentials:
        default:
            password: <password>
            username: <username>
        enable:
            password: <password>
        os: iosxe
        type: iosxe
    dist-rtr02:
        connections:
        cli:
            ip: 10.10.20.176
            protocol: telnet
        credentials:
        default:
            password: <password>
            username: <username>
        enable: 
            password: <password>
        os: iosxe 
        type: iosxe
    ```

Once this parser file is created with the schema and parser class, it is easy to test its working by opening a Python shell in your virtual environment where `pyATS[full]` was installed.

1. Import the class from the parser created.

    ```bash
    (.genie-venv) user@COMPUTER:~/$ python
    Python 3.7.9 (default, Jan  3 2021, 14:00:24)
    [GCC 9.3.0] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> from show_inventory_parser import ShowInventory
    ```

2. To interact with the testbed, a module named `genie.testbed` comes in handy. Use it to load your testbed.

    ```bash
    >>> from genie import testbed
    >>> tb = testbed.load('testbed.yaml')
    ```

3. Create a device object from the testbed. Your testbed object has a dictionary loaded from the testbed YAML structure.

    ```bash
    >>> dev = tb.devices['dist-rtr01']
    ```

4. Open a connection to the device. Genie knows all about the OS and device type from the testbed file, which makes this easy.

    ```bash
    >>> dev.connect()
    ```

5. Pass the device object to the parser class, and call the parse method provided by the `MetaParser` class (which the parser class inherits) to do the parsing magic.

    ```bash
    >>> obj = ShowInventory(device=dev)
    >>> from pprint import pprint
    >>> pprint(obj.parse())

    2021-02-26 06:12:37,737: %UNICON-INFO: +++ dist-rtr02 with alias 'cli': executing command 'show inventory' +++
    show inventory
    NAME: "Chassis", DESCR: "Cisco CSR1000V Chassis"
    PID: CSR1000V          , VID: V00  , SN: 9DILH0XHO0N

    NAME: "module R0", DESCR: "Cisco CSR1000V Route Processor"
    PID: CSR1000V          , VID: V00  , SN: JAB1303001C

    NAME: "module F0", DESCR: "Cisco CSR1000V Embedded Services Processor"
    PID: CSR1000V          , VID:      , SN:


    dist-rtr02#
    {'inventory':
        {
            0: {
                'description': 'Cisco CSR1000V Chassis',
                'name': 'Chassis',
                'pid': 'CSR1000V',
                'serial': '9DILH0XHO0N',
                'vid': 'V00  '
            },
            1: {
                'description': 'Cisco CSR1000V Route Processor',
                'name': 'module R0',
                'pid': 'CSR1000V',
                'serial': 'JAB1303001C',
                'vid': 'V00  '
            },
            2: {
                'description': 'Cisco CSR1000V Embedded Services Processor',
                'name': 'module F0',
                'pid': 'CSR1000V',
                'serial': '',
                'vid': ''
            }
        }
    }
    ```