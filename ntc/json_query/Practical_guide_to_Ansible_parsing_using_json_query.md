# Practical guide to Ansible parsing using json_query

## Intro

Parsing structured JSON data in Ansible playbooks is a common task. Nowadays, the JSON format is heavily used by equipment vendors to represent complex objects in a structured way to allow programmatic interaction with devices. It utilises two main data structures:

- object - an unordered collection of key/value pairs (like Python `dict`)
- array - an ordered sequence of objects  (like Python `list`)

Ansible provides many built-in capabilities to consume JSON using Ansible specific filters or the jinja2 built-in filters. Sometimes the number of filters to learn and which filters to select for a given use case can be a little daunting when starting out. More often than not, multiple filters need to be chained together to achieve a desired result. This can lead to complex task definition, making playbook maintenance more difficult.

The built-in `json_query` filter provides the functionality for filtering, shaping and transforming JSON data. It uses the third-party `jmespath` library, which is a powerful JSON query language supporting the parsing pf complex structured data. 

This blog post will focus on giving practical examples of the `json_query` filter and some of the extended capabilities integrated in the `jmespath` library.   

### Setup

As mentioned, `json_query` filter uses a third-party library, so this must be installed on the host before the filter can be used in a play.

```python
pip install jmespath
```

### Data

To demonstrate how the filter can be used, we'll query against the following dataset. Those familiar with Palo Alto firewalls will recognise the data as a modified version of the `stdout` result of an application content update query.

```python
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

On Palo Alto devices the stdout response is returned as a JSON encoded string. For convenience, we'll set a variable to focus on the `json_query` string part of the filter in the examples. 

```python
- name: "SET FACT FOR DEVICE STDOUT RESPONSE"
  set_fact:
    result: "{{ (content_info['stdout'] | from_json)['response']['result'] }}"
```

## Practical examples

We'll demonstrate the use of some common operators used in the `jmespath` query language. By no means is this an exhaustive list. Please refer to references at the end of the blog to see the specification.

### JMESPath Operators
|Operator| Description |
| --- | --- |
|@|	The current node being evaluated.|
|*|	Wildcard. All elements.|
|.<key> |	Dot-notation to access a value of the given key.|
| [<index0>, <index1>, ..] |	Indexing array elements, like a list.|
| [?<expression>] |	Filter expression. Boolean evaluation|
| && |	AND expression.
| &#124; |	Pipe expression, like unix pipe.|
| &<expression> |	Using an expression evaluation as a data type.|


### Basic Filter

Using our result data, we pass the valid JSON into the filter with an additional query string argument. The query string will provide the expression that is evaluated against the data to return a result. The default data type returned by `json_query` filter is a list.

In the basic filter example, the query string will select the `version` key for each element in the `entry` array.

```python
- name: "BASIC FILTER"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[*].version') }}
```

```python
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

Returning a list is not always convenient. We can use the pipe expression to select the value of a array by specifying a certain index location. This operates in much the same way as the unix pipe, within the query string. 

Here, we select the first element. Array indexing begins at zero.

```python
- name: "ARRAY INDEX VALUE"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[*].version | [0]') }}"
```

The response output is a string value for the first element in the array.

```python
TASK [ARRAY INDEX VALUE] *********************************************************
ok: [demo-fw01] =>
  msg: 8368-6520
```

### Filter

Using a filter expression within the `json_query` string, we can filter the query result using a boolean expression.

If we only wanted to get the filenames of the software version that are actually downloaded on the device, we can an evaluate against the objects that have `downloaded=='yes'`. The ability to filter datasets, within Ansible playbooks, using a subset of attributes as the selection criteria and select another set of attribute values is extremely useful.

```python
- name: "FILTER EXACT MATCH"
  debug:
    msg: "{{ result['content-updates'] | json_query('entry[?downloaded==`yes`].filename') }}"
```

The output, shows the list of filenames returned.

```python
TASK [FILTER EXACT MATCH] ********************************************************
ok: [demo-fw01] =>
  msg:
  - panupv2-all-apps-8373-6537
  - panupv2-all-apps-8372-6534
```

### Function

The `jmespath` library provides built-in functions to assist in transformation and filtering tasks. Some are quite similar to existing filters in Ansible or jinja2 but we'll take a look at the `max_by` function and also how we can use unconventional variable names in our query.

Using a variable for the query string is in many ways a cleaner approach. How strings are quoted actually matters when the key name is hyphenated. The query string should include double quotes `"key-name"` around the key name, which is required by the `jmespath` specification 

In our example we're selecting the filename of the maximum `app-version` value from the `entry` array. The `&` allows us to define an expression which will be evaluated as a data type when processed by the function. 

```python
- name: "MAX BY APP-VERSION"
  set_fact:
    content_file: "{{ result['content-updates'] | json_query(querystr) }}"
  vars:
    querystr: 'max_by(entry, &"app-version").filename'
```

The resulting `filename` value is returned.

```python
TASK [JMESPATH FUNCTION] *********************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8373-6537
```

Using single quotes in the previous example instructs the filter to treat the expression as a string, returning the first element of the array.

```python
vars:
  querystr: "max_by(entry, &'app-version').filename"
```

```python
TASK [MAX BY APP-VERSION] *******************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8368-6520
```

### Query String with Dynamic Variable

You can also pass other ansible facts into the query string. Again, quoting is important, so it will be better to provide a separate string to pass into the filter. 

Here, we'll use the filter expression with another built-in function `contains` that provides a boolean result based on a match with a search string on any element with the `version` array. The search string in this case is an ansible variable passed into the query string using a jinja2 template. 

```python
- name: "FUNCTION WITH VARIABLE"
  debug:
    msg: "{{ result['content-updates'] | json_query(querystr) }}"
  vars:
    querystr: "entry[?contains(version, '{{ content_version }}')].filename | [0]"
    content_version: "8368-6520"
```

Again, we're selecting the first filename within the returned list using the indexing capability. 

```python
TASK [FUNCTION WITH VARIABLE] **************************************************
ok: [demo-fw01] =>
  msg: panupv2-all-apps-8368-6520
```

### Multiple Expressions

We can also evaluate multiple expressions using the logical AND operation. This follows the normal truth table rules. Along with the filter operator, we provide two expressions that will evaluate to a boolean value. This will filter the resulting data based on two selection criteria and provide a list of version values. 

```python
- name: "MULTIPLE FILTER EXPRESSIONS"
  debug:
    msg: "{{ result['content-updates']  | json_query(querystr) }}"
  vars:
    querystr: "entry[?contains(filename, '{{ version }}') && downloaded==`yes`].version"
    version: "8373-6537"
```

In this case, we only have one element in the list.

```python
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

