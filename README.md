# Leveraging the Linode API

This is a place to document complex (and not-so-complex) solutions for leveraging the Linode API. This is *not* the place
to learn how to use the API. Please see the API [documentation](https://developers.linode.com/api/v4) if you're just getting started.

## Getting Started

Many solutions in this repo require `jq` to be [installed](https://stedolan.github.io/jq/) locally.

The [Linode CLI](https://github.com/linode/linode-cli) can be used in place of the `curl` used in this guide.

## A jq primer

<details><summary>Expand</summary>

`jq` is a very high-level functional programming language best used for managing JSON data. It should be thought of as a program that takes a JSON input and produces a filtered output. It is used similar to how you'd use `sed`, `awk`, or `grep` with plain text.

#### Prettify JSON

```bash
# Doesn't change data, only reformats it.
...stdout | jq '.'
```


```bash
# Without jq formatting
$ curl -H "Authorization: Bearer $TOKEN" \
https://api.linode.com/v4/linode/instances/5365909

{"region": "us-east", "updated": "2018-06-01T21:05:17", "created": "2018-01-19T12:11:42", "ipv6": "2600:3c03::f03c:91ff:fe0c:1b48/64", "backups": {"enabled": true, "schedule": {"window": "Scheduling", "day": "Scheduling"}}, "id": 5365909, "alerts": {"transfer_quota": 80, "network_in": 10, "network_out": 10, "cpu": 90, "io": 10000}, "group": "Test Linodes", "type": "g5-nanode-1", "specs": {"memory": 1024, "disk": 20480, "transfer": 1000, "vcpus": 1}, "image": "linode/centos6.8", "hypervisor": "kvm", "ipv4": ["172.104.16.252"], "label": "CentOS-7", "watchdog_enabled": true, "status": "running"}
```

```bash
# With jq formatting
curl -H "Authorization: Bearer $TOKEN" \
https://api.linode.com/v4/linode/instances/5365909 | jq '.'

{
  "region": "us-east",
  "updated": "2018-06-01T21:05:17",
  "created": "2018-01-19T12:11:42",
  "ipv6": "2600:3c03::f03c:91ff:fe0c:1b48/64",
  "backups": {
    "enabled": true,
    "schedule": {
      "window": "Scheduling",
      "day": "Scheduling"
    }
  },
  "id": 5365909,
  "alerts": {
    "transfer_quota": 80,
    "network_in": 10,
    "network_out": 10,
    "cpu": 90,
    "io": 10000
  },
  "group": "Test Linodes",
  "type": "g5-nanode-1",
  "specs": {
    "memory": 1024,
    "disk": 20480,
    "transfer": 1000,
    "vcpus": 1
  },
  "image": "linode/centos6.8",
  "hypervisor": "kvm",
  "ipv4": [
    "172.104.16.252"
  ],
  "label": "CentOS-7",
  "watchdog_enabled": true,
  "status": "running"
}
```

#### Filtering JSON data

```bash
# A simple filter. Returns the value of key .foo and can be chained for nested objects.
$ curl -sH "Authorization: Bearer $TOKEN" \ 
https://api.linode.com/v4/linode/instances/5365909 | jq '.specs'

{
  "memory": 1024,
  "transfer": 1000,
  "vcpus": 1,
  "disk": 20480
}
```

```bash
# Chaining filters for nested objects
$ curl -sH "Authorization: Bearer $TOKEN" \ 
https://api.linode.com/v4/linode/instances/5365909 | jq '.specs | .memory'

1024
```

#### Handling Arrays

```bash
# A JSON value can be a string, number, boolean, null, an object, and even an array. Arrays are
# handled with a slight change in syntax:

...stdout | jq '.foo[]'
```

```bash
# You can select a specific element of an array by providing the index:

# First element
...stdout | jq '.foo[0]'

# Third element
...stdout | jq '.foo[2]'

# Last element
...stdout | jq '.foo[-1]'

# Second to last element
...stdout | jq '.foo[-2]'
```
</details>

## Solution Patterns with Examples

### :warning: Deleting services or disks
I hesitate to offer actual commands for removing services since an error here could be catastrophic.

### --- List all Linode IDs ---
```bash
# This call is the starting point for more complex calls where we'll
# pass this list to another function for further processing.

$ curl -sH "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances | jq '.data[] | .id'
```

### --- Perform action on all Linodes ---

##### List network helper status for all Linodes:
```bash
# First, grab all Linode IDs. Then, make a second API call to print the status of
# 'Auto-configure networking' for each Linode's configuration profile(s).

$ curl -sH "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances | jq '.data[] | .id' | while read line; do echo "===$line==="; \
  curl -sH "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances/$line/configs | egrep -o '"network": [a-z]*'; done;
```

### --- Filter Linodes based on some quality ---
Using [X-Filter](https://developers.linode.com/api/v4#section/Filtering-and-Sorting) is a convenient way to do simple filtering, but not all keys are filterable. The examples below filter with `jq`.

##### Show all Linode IDs for Linodes with backups disabled:
```bash
# Here, we print the Linode IDs for all Linodes with backups disabled. 
# This takes advantage of the select function to filter for enabled == true.

$ curl -H "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances | jq '.data[] | select(.backups.enabled == false) | .id'
```

### --- Perform action on filtered list of Linodes ---

##### Boot all powered off Linodes:
```bash
# First, create filtered list of IDs for all powered off Linodes. Then, iterate over
# filtered list to boot Linodes.

$ curl -sH "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances | jq '.data[] | select(.status == "offline") | .id ' | while read line; do echo "Booting $line"; \
  curl -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -X POST \
    https://api.linode.com/v4/linode/instances/$line/boot; done;
```

##### Upgrade (mutate) all legacy (< g6) Linodes:
```bash
# First, create filtered list of IDs for all legacy Linodes. Then, iterate over
# filtered list to mutate.

$ curl -sH "Authorization: Bearer $TOKEN" \
    https://api.linode.com/v4/linode/instances | jq '.data[] | select(.type|test("^g6.")|not) | .id' | while read line; do echo "Mutating $line"; \
  curl -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -X POST \
    https://api.linode.com/v4/linode/instances/$line/mutate; done;
```
