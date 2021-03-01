# Practical guide to Ansible parsing using json_query

## Intro

Parsing structured JSON data in Ansible playbooks is a common task. Nowadays, the JSON format is heavily used by equipment vendors to represent complex objects in a structured way to allow programmatic interaction with devices. It utilises two main data structures:

- object - an unordered collection of key/value pairs (like Python `dict`)
- array - an ordered sequence of objects  (like Python `list`)

Ansible provides many built-in capabilities to consume JSON using Ansible specific filters or the jinja2 built-in filters. The extensive filters available, and what to use and when, can be overwhelming at first and the desired result can often require multiple filters chained together. This can lead to complex task definition, making playbook maintenance more difficult.

The built-in `json_query` filter provides the functionality for filtering, shaping and transforming JSON data. It uses the third-party `jmespath` library, a powerful JSON query language supporting the parsing pf complex structured data. 

### Setup

The `jmespath` third-party library must be installed on the host for the json_query filter to operate.

```
pip install jmespath
```

### Data

Those familiar with Palo Alto firewalls will recognise a modified version of an application content update query. This result dataset will be used to demonstrate how to manipulate JSON data using the `json_query` filter. 

```
response:
      result:
        content-updates:
          entry:
          - app-version: 8368-6520
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8368-6520
            version: 8368-6520
          - app-version: 8369-6522
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8369-6522
            version: 8369-6522
          - app-version: 8367-6513
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8367-6513
            version: 8367-6513
          - app-version: 8371-6531
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8371-6531
            version: 8371-6531
          - app-version: 8366-6503
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8366-6503
            version: 8366-6503
          - app-version: 8370-6526
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8370-6526
            version: 8370-6526
          - app-version: 8373-6537
            current: 'no'
            downloaded: 'yes'
            filename: panupv2-all-apps-8373-6537
            version: 8373-6537
          - app-version: 8365-6501
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8365-6501
            version: 8365-6501
          - app-version: 8364-6497
            current: 'no'
            downloaded: 'no'
            filename: panupv2-all-apps-8364-6497
            version: 8364-6497
          - app-version: 8372-6534
            current: 'yes'
            downloaded: 'yes'
            filename: panupv2-all-apps-8372-6534
            version: 8372-6534
```

On Palo Alto devices the `stdout` response is returned as a JSON encoded string. A variable was set to simplify the following examples. 

```
- name: "SET FACT FOR DEVICE STDOUT RESPONSE"
  set_fact:
    result: "{{ (content_info['stdout'] | from_json)['response']['result'] }}"
```

## Practical examples

The JMESPath Operators table below summarises some of the most common operators used in the jmespath query language. See references at the end for the jmespath specification.

### JMESPath Operators
| Operator | Description |
| --- | --- |
| @ |	The current node being evaluated.|
| * |	Wildcard. All elements.|
| .*key* |	Dot-notation to access a value of the given key. |
| [*index0*, *index1*, ..] |	Indexing array elements, like a list. |
| [?*expression*] | Filter expression. Boolean evaluation. |
| && |	AND expression.
| &#124; |	Pipe expression, like unix pipe. |
| &*expression* |	Using an expression evaluation as a data type. |


### Basic Filter

Using the result data, the filter is applied to the valid JSON result with an additional query string argument. The query string provides the expression that is evaluated against the data to return filtered output. The default data type returned by json_query filter is a list.

In the Basic Filter example below, the query string selects the version key for each element in the entry array.

```
- name: "BASIC FILTER"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[*].version') }}
```

```
TASK [BASIC FILTER - VERSION] ***************************************************
ok: [demo-fw01] =>
  msg:
  - 8368-6520
  - 8369-6522
  - 8367-6513
  - 8371-6531
  - 8366-6503
  - 8370-6526
  - 8373-6537
  - 8365-6501
  - 8364-6497
  - 8372-6534
```

### Return Array Element

A pipe expression can be used to select the value of an array by declaring the desired index location. This operates in much the same way as the unix pipe, within the query string.  

In the Array Index example below, the first element is selected. Array indexing begins at zero.

```
- name: "ARRAY INDEX VALUE"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[*].version | [0]') }}"
```

The response output is a string value for the first element in the array.

```
TASK [ARRAY INDEX VALUE] *********************************************************
ok: [demo-fw01] =>
  msg: 8368-6520
```

### Filter

Using a filter expression within the `json_query` string, the query result can be filtered using standard comparison operators. A filter expression allows each element of the array to be evaluated against the expression. If it evaluates to true, the element is included in the returned result.

The equality comparision operator is used in the following example to retrieve the filename(s) of the software version(s) downloaded on the device. Elements in the array that have `downloaded=='yes'` will have the `filename` included in the returned result. This provides the ability to use certain attributes for the selection criteria, and return the value(s) of other attributes for the result.

```
- name: "FILTER EXACT MATCH"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[?downloaded==`yes`].filename') }}"
```

The output, shows the list of filenames returned.

```
TASK [FILTER EXACT MATCH] ********************************************************
ok: [demo-fw01] =>
  msg:
  - panupv2-all-apps-8373-6537
  - panupv2-all-apps-8372-6534
```

### Function

The `jmespath` library provides built-in functions to assist in transformation and filtering tasks, for example the `max_by` function that returns the maximum element in an array. In the following task selects the filename of the maximum `app-version` value from the `entry` array. The `&` provides the ability to define an expression which will be evaluated as a data type value when processed by the function. 

Using a variable for the query string is in many ways a cleaner approach to the query definition. How strings are quoted matters when the key name is hyphenated. in this case the query string should include double quotes `"key-name"` around the key name, which is required by the `jmespath` specification. None hypenathed keys do not need double quotes.


```
- name: "MAX BY APP-VERSION"
  set_fact:
    content_file: "{{ result['content-updates'] | json_query(querystr) }}"
  vars:
    querystr: 'max_by(entry, &"app-version").filename'
```

The resulting `filename` value is returned.

```
TASK [JMESPATH FUNCTION] *********************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8373-6537
```

Using single quotes in the previous example instructs the filter to treat the expression as a string, returning the first element of the array.

```
vars:
  querystr: "max_by(entry, &'app-version').filename"
```

```
TASK [MAX BY APP-VERSION] *******************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8368-6520
```

### Query String with Dynamic Variable

Ansible facts can be substituted into the query string. Again, quoting is important, so it will be better to provide a separate string to pass into the filter. 

In this example the filter expression is used with another built-in function `contains`. This function provides a boolean result based on a match with a search string, on any element within the `version` array. The search string in this case is an Ansible variable passed into the query using a jinja2 template. 

```
- name: "FUNCTION WITH VARIABLE"
  debug:
    msg: "{{ result['content-updates'] | json_query(querystr) }}"
  vars:
    querystr: "entry[?contains(version, '{{ content_version }}')].filename | [0]"
    content_version: "8368-6520"
```

Again, we're selecting the first filename within the returned list using the indexing capability. 

```
TASK [FUNCTION WITH VARIABLE] **************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8368-6520
```

### Multiple Expressions

Multiple expressions can be evaluated using the logical AND operation. This follows the normal truth table rules. Along with the filter operator, we provide two expressions that will evaluate to a boolean value. This will filter the resulting data based on two selection criteria and provide a list of version values. 

```
- name: "MULTIPLE FILTER EXPRESSIONS"
  debug:
    msg: "{{ result['content-updates']  | json_query(querystr) }}"
  vars:
    querystr: "entry[?contains(filename, '{{ version }}') && downloaded==`yes`].version"
    version: "8373-6537"
```

In this case, we only have one element in the list.

```
TASK [MULTIPLE FILTER EXPRESSIONS] ***********************************************
ok: [demo-fw01] =>
  msg:
  - 8373-6537
```

### Conclusion

The `jmespath` specification has some very good documentation and support for numerous languages. It does require learning a new query language but hopefully this guide will help you get started with some common use cases. 

The `json_query` filter is a powerful tool to have at your disposal. It can solve many JSON parsing tasks within your playbook, avoiding the need to write custom filters. If you'd like to see more please let us know.

### References

- [https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#selecting-json-data-json-queries](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#selecting-json-data-json-queries)
- [https://jmespath.org/specification.html](https://jmespath.org/specification.html)
- [https://jmespath.org/tutorial.html](https://jmespath.org/tutorial.html)

