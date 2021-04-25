This post intends to talk about using TextFSM, a tool that is used to parse textual data to create structured data objects.

# Introduction

The CLI on a network device is a good friend of a network engineer. It gives us the information we need in a no-nonsense manner, usually in the form of a table or some other structure that makes sense when you read it.

```
Taken from https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/command/iro-cr-book/ospf-s1.html#wp1717098229

Router# show ip ospf database
OSPF Router with id(192.168.239.66) (Process ID 300)
                 Displaying Router Link States(Area 0.0.0.0)
  Link ID       ADV Router    Age        Seq#     Checksum  Link count
172.16.21.6   172.16.21.6    1731    0x80002CFB    0x69BC       8
172.16.21.5   172.16.21.5    1112    0x800009D2    0xA2B8       5
172.16.1.2    172.16.1.2     1662    0x80000A98    0x4CB6       9
172.16.1.1    172.16.1.1     1115    0x800009B6    0x5F2C       1
172.16.1.5    172.16.1.5     1691    0x80002BC     0x2A1A       5
172.16.65.6   172.16.65.6    1395    0x80001947    0xEEE1       4
172.16.241.5  172.16.241.5   1161    0x8000007C    0x7C70       1
172.16.27.6   172.16.27.6    1723    0x80000548    0x8641       4
172.16.70.6   172.16.70.6    1485    0x80000B97    0xEB84       6
                Displaying Net Link States(Area 0.0.0.0)
  Link ID       ADV Router      Age        Seq#        Checksum
172.16.1.3  192.168.239.66     1245    0x800000EC      0x82E
                Displaying Summary Net Link States(Area 0.0.0.0)
  Link ID       ADV Router       Age        Seq#        Checksum
172.16.240.0   172.16.241.5    1152      0x80000077      0x7A05
172.16.241.0   172.16.241.5    1152      0x80000070      0xAEB7
172.16.244.0   172.16.241.5    1152      0x80000071      0x95CB
```

Above we can see the [OSPF database](https://community.cisco.com/t5/networking-documents/reading-and-understanding-the-ospf-database/ta-p/3145995) on a Cisco router, telling us about connected routes in the same area as the router, routes as advertised from the designated router, and route summaries at the end. While this makes sense to us, a computer does not understand the semantics behind this output. Using TextFSM, we can describe the manner using which information can be extracted from this output.

# What is TextFSM and how to use it

Going by the [TextFSM GitHub repository wiki](https://github.com/google/textfsm/wiki/TextFSM),

> TextFSM is a Python module that implements a template based state machine for parsing semi-formatted text.

This is a pretty loaded sentence, so let us break it down a bit.

- Template: A template is a document that describes how the input text data should be processed. It consists of values that need to be recorded, and states which outline a "skeleton" structure describing the structure of the data that needs to be parsed. For example, in our `show ip ospf database` example above, in the first sentence:

    ```
    OSPF Router with id(192.168.239.66) (Process ID 300)
    ```

    The variables here are the router ID (192.168.239.66), and the OSPF process ID (300). Every output of `show ip ospf database` on a Cisco device will have the other characters in this sentence, with only the router and OSPF process ID changing.

- State machine: In classical computer science, a state machine is a mathemtical abstraction (basically, a mathemtical instrument) used to design algorithms. What this means is that we, the holy designers of our parsing algorithm, have to define the various **states** that the parsing algorithm will go through while it is fed some input. Based on what state the algorithm is in, it can take some actions for the input it is fed based on **rules** defined within the state. This will make more sense later when I describe how to create a TextFSM template.

- Semi-formatted text: The example output given above is semi-formatted text- it has an informal format in how it is structured that we can notice, but it not formatted in a way where a computer program can interpret the semantic format. It is not like [JSON](https://www.json.org/json-en.html), [YAML](https://yaml.org/), or [XML](https://developer.mozilla.org/en-US/docs/Web/XML/XML_introduction), where the semantics of how the data is structured is already understood, or is described with the text.

Great, so putting this together, we can deduce that TextFSM is a tool that allows you to write templates to describe the data to be parsed, gives developers the flexibility to define the different states the template can parse in, and works on semi-formatted data that have an apparent textual format in which they are represented.

TextFSM uses [RegEx](https://docs.python.org/3/howto/regex.html) to parse data.

## TextFSM template definitions

A TextFSM template has 2 components:

### Values

These are the variables that need to be captured in the parsing process. TextFSM works in a manner where it builds a table of records that are parsed from semi-formatted text data, so declaring the values can be considered the same as declaring the columns of the table.
A value is declared in the following way:

```
Value [option[,option...]] name regex
```

> NOTE: The parts enclosed in square brackets `[]` mean that this part is OPTIONAL.

- `Value`: This is a keyword that tells TextFSM that a column that needs to be recorded is being defined in this line.
- option: Options tell TextFSM how to work with this value. TextFSM allows the following options that can be specified for a value in a comma separated manner:

    - `Filldown`: Take the last found value of this column and fill all successive empty values for this column in the table downwards until a row is found where the column has a non-empty value. This helps when a value needs to have some general entry for a column in every row recorded.
    - `Key`: This column/field contains a unique string identifier, which can be used to identify the row.
    - `List`: This value contains a list instead of a string value.
    - `Fillup`: Similar to filldown, but fills values upwards until a recorded non-empty value is found.
- `name`: The name of this column.
- `regex`: The RegEx pattern that this field should match with, so that TextFSM knows how to find this value to record it. `regex` should be enclosed in round brackets- like `(\d+)`, to match a pattern of 1 or more digits.

### State definitions

After defining the values that will be recorded in a row, we need to tell TextFSM about the parsing states. Each state consists of some parsing rules that tell TextFSM what to do when it encounters a certain type of input. These rules are written using [RegEx](https://docs.python.org/3/howto/regex.html).
Each state definition must be separated by a blank line. A state is defined as such:

```
stateName
  ^rule
  ^rule
```

There are 3 reserved states:

- `Start`: This state is required to be defined in the template, else the template will not work.
- `End`: State to complete processing of incoming strings, tells TextFSM to stop here instead of going till the end of the text input file.
- `EOF`: Implicit state that is always executed when processing of the input text file reaches its end. This records the currently parsed values for the new record/row in the table, so if this behavior is not required, `EOF` must be explicitly stated at the end of the template.

TextFSM allows you to define custom states of your own.

#### Rules

In a state, you can define one or more rules that tell TextFSM what to do when it encounters certain input lines. TextFSM takes in incoming strings and compares them with the rules using RegEx patterns. If the input line matches with the rule, then the **action** specified in the rule is executed. Once the action completes, TextFSM moves to the next input line and starts comparing it with the rules starting from **the beginning of the state**.

Rules are written as such:

```
^regex [-> action]
```

The rule starts with a carat symbol `^`. This is followed with a RegEx pattern that is used to match with the input line. To capture a value when the rule is matched with the input line, a syntax of the form `${ValueName}` is used. To specify the end of the RegEx pattern, `$$` is used.

#### Actions

A rule can define what action needs to be executed when an input line matches with the RegEx pattern in the rule. An action is identified with the ` -> ` character after the RegEx pattern. There are 3 types of actions:

1. Line action: Actions that apply to an input string match.
2. Record action: Actions that apply to collected values.
3. State transition: Action to change to a different state.

The default action that takes place when a rule matches with the input line is `Next.NoRecord`, which means go to next input line, start attempting to match with rules from beginning of current state, and don't write a new row record.

Types of line actions:

- `Next`: Process line, go to next input line, start attempting to match with rules from beginning of current state. This line action is executed by default if nothing is specified.
- `Continue`: Continue to process rules with the current input line as if no match happened, while values matched here are still assigned.

Record options are optional actions that can be specified after a line action, separated by a fullstop `"."`. Types of record actions:

- `NoRecord`: Don't do anything, this happens by default if nothing else is specified.
- `Record`: Write a new row record in the TextFSM record table with variables that have been collected till now. Reset all variables except those with a Filldown option.
- `Clear`: Reset all variables collected except those with a Filldown option.
- `Clearall`: Reset all variables collected.

The fullstop `.` separator is mandatory only if both line and record actions are specified. If one or both are left as the implicit default then the dot is omitted. So, `Next`, `Next.NoRecord` and `NoRecord` do exactly the same thing.

A state transition allows the rule matching to be moved to another state from the current one. This state must be reserved, or defined in the template. In a rule that executes a state transition:

1. First all actions (line and record) are executed.
2. Then the next line is read.
3. Finally the current state changes to the new state, and rule matching with the newly read input line continues in the new state.

> If a rule uses `Continue` as a record option, it is not possible to change state in this rule.

An `Error` action is also allowed by TextFSM, in case you would like the template to fail if something unexpected shows up in the input lines. `Error` stops all line processing, discards everything collected so far, and throws an Exception.

```
^regex -> Error [word|"string"]
```

The `word` option for the error allows the throwing of a custom fail message.

## Example TextFSM template

A great example of a TextFSM template is one to parse the output from the command `show ip ospf database` on a Cisco router, as we saw an output for this above. Posting it again for convenience (edited it a bit).

```
Taken from https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/command/iro-cr-book/ospf-s1.html#wp1717098229

Router# show ip ospf database
OSPF Router with id(192.168.239.66) (Process ID 300)
                 Displaying Router Link States(Area 0.0.0.0)
  Link ID       ADV Router    Age        Seq#     Checksum  Link count
172.16.21.6   172.16.21.6    1731    0x80002CFB    0x69BC       8
172.16.21.5   172.16.21.5    1112    0x800009D2    0xA2B8       5
172.16.1.2    172.16.1.2     1662    0x80000A98    0x4CB6       9
172.16.1.1    172.16.1.1     1115    0x800009B6    0x5F2C       1
172.16.1.5    172.16.1.5     1691    0x80002BC     0x2A1A       5
172.16.65.6   172.16.65.6    1395    0x80001947    0xEEE1       4
172.16.241.5  172.16.241.5   1161    0x8000007C    0x7C70       1
172.16.27.6   172.16.27.6    1723    0x80000548    0x8641       4
172.16.70.6   172.16.70.6    1485    0x80000B97    0xEB84       6
                Displaying Net Link States(Area 0.0.0.0)
  Link ID       ADV Router      Age        Seq#        Checksum
172.16.1.3  192.168.239.66     1245    0x800000EC      0x82E
                Displaying Summary Net Link States(Area 0.0.0.0)
  Link ID       ADV Router       Age        Seq#        Checksum
172.16.240.0   172.16.241.5    1152      0x80000077      0x7A05
172.16.241.0   172.16.241.5    1152      0x80000070      0xAEB7
172.16.244.0   172.16.241.5    1152      0x80000071      0x95CB
```

The TextFSM template to parse this would be as such:

```
# Taken from ntc-templates, https://github.com/networktocode/ntc-templates/blob/master/templates/cisco_ios_show_ip_ospf_database.textfsm

Value Filldown ROUTER_ID (\d+\.\d+\.\d+\.\d+)
Value Filldown PROCESS_ID (\d+)
Value Filldown AREA (\d+\.\d+\.\d+\.\d+|\d+)
Value LINK_ID (\d+\.\d+\.\d+\.\d+)
Value ADV_ROUTER (\d+\.\d+\.\d+\.\d+)
Value AGE (\d+)
Value LINK_COUNT (\d+)
Value TAG (\d+)

Start
  ^.*\(${ROUTER_ID}\) \(.* ${PROCESS_ID}\)
  ^.*\(Area ${AREA}\)
  ^${LINK_ID}\s+${ADV_ROUTER}\s+${AGE}\s+\S+\s+\S+\s+${LINK_COUNT} -> Record
  ^${LINK_ID}\s+${ADV_ROUTER}\s+${AGE}\s+\S+\s+\S+ -> Record
  ^\s+Type-5 AS External Link States -> Tag
  # Capture time-stamp if vty line has command time-stamping turned on
  ^Load\s+for\s+
  ^Time\s+source\s+is

Tag
  ^Link ID\s+ADV Router\s+Age\s+Seq#\s+Checksum\s+Tag -> Next.Clearall
  ^${LINK_ID}\s+${ADV_ROUTER}\s+${AGE}\s+\S+\s+\S+\s+${TAG} -> Next
  ^\s -> Start

EOF
```

This template would have something of this sort to visualize the data collected.

```
| ROUTER_ID      | PROCESS_ID | AREA    | LINK_ID     | ADV_ROUTER  | AGE  | LINK_COUNT | TAG |
|----------------|------------|---------|-------------|-------------|------|------------|-----|
| 192.168.239.66 | 300        | 0.0.0.0 | 172.16.21.6 | 172.16.21.6 | 1731 | 8          |     |
| 192.168.239.66 | 300        | 0.0.0.0 | 172.16.21.5 | 172.16.21.5 | 1112 | 5          |     |
| 192.168.239.66 | 300        | 0.0.0.0 | 172.16.1.2  | 172.16.1.2  | 1662 | 9          |     |
... (Abridged)
```

## Using TextFSM with Python

Let's say, using [netmiko](https://github.com/ktbyers/netmiko), you send a show command to your network device to get back some string output. Using a TextFSM template to parse it, and the textfsm Python module, you can get a structured output for the data to be parsed.

```python
import textfsm
from netmiko import ConnectHandler

cisco_dev = {
  'device_type': 'cisco_ios',
  'host':   '10.10.10.10',
  'username': 'test',
  'password': 'password',
  'port' : 8022,          # optional, defaults to 22
  'secret': 'secret',     # optional, defaults to ''
}

net_connect = ConnectHandler(**cisco_dev)
output = net_connect.send_command("show ip ospf database")

template = open('template_name.textfsm') # open textfsm file with .textfsm extension, or .tpl also works.

re_table = textfsm.TextFSM(template) # initialize textfsm object
fsm_results = re_table.ParseText(output) # parse output text with textfsm object.

"""
ParseText output returns back a list of tuples.
First tuple is the header, every subsequent tuple is a row.
Let us make a dict with relevant key value pairs out of this!
"""

results = list()
for item in fsm_results:
    results.append(dict(zip(re_table.header, item)))

return results
```

And you get back a list of dictionaries like this:

```python
[
  {
    "ROUTER_ID": "192.168.239.66",
    "PROCESS_ID": "300",
    "AREA": "0.0.0.0",
    "LINK_ID": "172.16.21.6",
    "ADV_ROUTER": "172.16.21.6",
    "AGE": "1731",
    "LINK_COUNT": "8",
    "TAG": "",
  },
  {
    "ROUTER_ID": "192.168.239.66",
    "PROCESS_ID": "300",
    "AREA": "0.0.0.0",
    "LINK_ID": "172.16.21.5",
    "ADV_ROUTER": "172.16.21.5",
    "AGE": "1112",
    "LINK_COUNT": "5",
    "TAG": "",
  },
  ...
  # ABRIDGED
]
```
