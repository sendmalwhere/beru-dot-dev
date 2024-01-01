---
title: 'So You\'ve Decided To Access Serialized Hashes In Ansible'
date: 2019-07-25 00:00:00
---

I just had to solve a challenge with Ansible configuration which is not well-documented anywhere, and it’s always my ambition to share solutions to problems like that when I’m able to find them.

A task that’s very simple in any scripting language, but which is hilariously complex in Ansible. is the problem of pulling JSON from a REST API endpoint, deserializing that JSON, and accessing values based on a given key.

Specifically, I have a data structure that looks **roughly** like this (which I grabbed from a REST API using Ansible’s [uri module](https://docs.ansible.com/ansible/latest/modules/uri_module.html):

```json
{
  "cache_control": "no-cache, no-store, max-age=0, must-revalidate",
  "changed": false,
  "connection": "close",
  "content_type": "application/json; charset=utf-8",
  "cookies": {},
  "cookies_string": "",
  "date": "Thu, 25 Jul 2019 19:29:41 GMT",
  "elapsed": 1,
  "expires": "Fri, 01 Jan 1990 00:00:00 GMT",
  "json": {
      "applianceIp": "192.168.1.1",
      "dhcpBootFilename": "not_the_real_filename.kpxe",
      "dhcpBootNextServer": "192.168.1.254",
      "dhcpBootOptionsEnabled": false,
      "dhcpHandling": "Run a DHCP server",
      "dhcpLeaseTime": "1 week",
      "dnsNameservers": "opendns",
      "fixedIpAssignments": {
        "de:ad:be:ef:ca:fe": {
          "ip": "192.168.1.2",
          "name": "test_client_2"
        },
        "de:ad:be:ef:ca:ff": {
          "ip": "192.168.1.3",
          "name": "test_client_3"
        },
        "de:ad:be:ef:ca:f0": {
          "ip": "192.168.1.4",
          "name": "test_client_4"
        },
        "de:ad:be:ef:ca:f1": {
          "ip": "192.168.1.5",
          "name": "test_client_5"
        },
      },
      "id": 1,
      "name": "Default",
      "networkId": "L_1234567891234567891234",
      "subnet": "192.168.1.0/24"
    },
    "msg": "OK (unknown bytes)",
    "pragma": "no-cache",
    "redirected": true,
    "server": "nginx",
    "status": 200
  }
```

## The thing I’m trying to do
Simply: I have the MAC address on my client, as discovered in ansible_facts. I have the above json output from my DHCP server’s REST API, showing a list of Fixed IP Assignments (from MAC addresses, as keys, to IP addresses, as values). Given a host’s MAC, I want to retrieve the IP address it should have from this big json structure.

As you can see in the `json` section of that data structure, my REST API endpoint returns a section called `fixedIpAssignments`, in which MAC addresses of clients are keys, and a dict (containing the IP address and hostname of the client) is the value on that key. If I had this data structure in a Python program, I would use some code that looks like the following to get the IP address for a particular MAC:

```python
import json
from assume_I_have_this_information import get_client_mac_address

my_dict = {}
with open('json_datastruct.json', 'r') as f:
  my_dict = json.loads(f.read())

client_mac_address = get_client_mac_address(host_facts, 'en1')

ip_addr = my_dict['json']['fixedIpAssignments'][client_mac_address]['ip']
```

As you can see, this is hilariously trivial for Python or Ruby, etc. But in an Ansible playbook, the Jinja2 templating makes it very difficult to run a lookup on the contents of a dictionary or json object where a key is a variable.

## Why it’s harder than it has to be in Ansible
Jinja2 is a problem. Its variable expansion allows for the evaluation of simple expressions only, so while the "mustaches" ( `\{\{` and `\}\}` ) allow for pretty reasonable text interpolation and variable expansion, they run into heavy limitations when we try to access dictionaries with variable keys. Take for example the expression above—the last line of Python pseudocode, where we grab nested keys from my_dict and call a variable in as one of those keys. It would look like this if we translated it directly into Ansible syntax:

```
- set_fact:
    client_mac_address: ansible_facts['en1']['macaddress']
- debug:
    msg: "\{\{ json_result_from_uri['json']['fixedIpAssignments'][client_mac_address]['ip'] \}\}"
```

…But this turns out to be completely invalid! Jinja2 won’t expand the variable `client_mac_address` in the hash access part of the debug task. So your debug msg will throw a KeyError or something similar, indicating that there’s not a `client_mac_address` key in `json_result_from_uri['json']['fixedIpAssignments']`. So much for “radically simple” configuration management.

But ok, let’s take a step back here, because the mustaches do variable expansion, right? So why not just nest them, like this?

```
- set_fact:
    client_mac_address: ansible_facts['en1']['macaddress']
- debug:
    msg: "\{\{ json_result_from_uri['json']['fixedIpAssignments'][\{\{ client_mac_address \}\}]['ip'] \}\}"
```

This turns out to be illegal. Interpolation in Jinja2 is flat, and can’t handle a nested call to interpolate in a variable inside an interpolation, so this will just interpret the internal mustaches as literal curly-braces, and you’ll probably get a SyntaxError on this to the effect of “expected eof but instead got \{“. Bad news bears. So how the heck are we supposed to access values stored at dynamic keys? This shouldn’t be as exotic a request as it apparently is, but I haven’t seen anyone else address this specific question before.

I came up with an answer that’s fundamentally really, really silly, but as best I can tell this is the way Ansible wants us to do it.

The Ansible way to pull complex chains of keys from a json object or dictionary is basically to use [the Jinja2/Ansible json_query filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#json-query-filter). But you can’t just pipe the json data structure into json_query with the client_mac_address variable in the query string, because it’s _still going to fail to expand in there_. It’s just going to be interpreted as a string literal:

```
- set_fact:
    client_mac_address: ansible_facts['en1']['macaddress']
- debug:
    msg: "\{\{ json_result_from_uri.json.fixedIpAssignments | json_query('client_mac_address.ip') \}\}"
```

Now, one can _partially_ resolve this by interrupting Jinja2’s Python string interpolation and just forcibly concatenating in client_mac_address, which will be available to Jinja2 as an environment variable once we set it as a fact:

```
- set_fact:
    client_mac_address: ansible_facts['en1']['macaddress']
- debug:
    msg: "\{\{ json_result_from_uri.json.fixedIpAssignments | json_query('\" + client_mac_address + \".ip') \}\}"
```

This would probably work just fine if `client_mac_address` were just an alpha String variable. But you and I know that MAC addresses contain numbers and (in this case) colons, and that means that this interpolation is going to fail in this specific case, because the json_query filter doesn’t allow queries that start with numbers or queries that contain colons—unless we escape the MAC address by enclosing it in double-quotes. But we can’t do that (at least not in-place), because we’ve already used escaped quotes to hack our way into variable interpolation inside of Jinja2 mustaches.

It’s enough to make a grown man blow up his own TV.

## How to solve it for real
This is the solution I came up with—again, bearing in mind that there’s probably a better way to handle this kind of problem, but StackOverflow doesn’t know what that better way is:

```
  - set_fact:
      my_query: "\"\{\{ ansible_facts['en1']['macaddress'] \}\}\".ip"
  - debug:
      msg: "\{\{ json_result_from_uri.json.fixedIpAssignments | json_query(my_query) \}\}"
```

Let’s break down what’s happening here:

1. First we expand the `my_query` var so that it literally resolves to "de:ad:be:ef:ca:fe".ip (i.e., one of the MACs in the JSON, escaped by double-quotes, followed by access on the "ip" key of the dict at that MAC key).
2. Then, we do normal access to the dict structure we got from the uri query (syntax before the pipe would be equally valid if it were spelled out as `json_result_from_uri['json']['fixedIpAssignments'])`
3. Finally, pipe the output of that access into the input of `json_query`. Voila! You now have access to the IP assigned as a fixed value for this MAC address on the DHCP server that gave us the API response.

That’s it, and that’s why simply accessing a hash value with a key name stored in a variable took me a full business day. Thanks a lot, Ansible!
