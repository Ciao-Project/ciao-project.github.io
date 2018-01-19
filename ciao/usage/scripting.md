---
title: Command line scripting
permalink: scripting.html
keywords: CIAO, command line, tool, scripting, templates
---

## Scripting with ciao

The cli has two commands that present information to the user: show and list

```shell
$ ciao list instances
```

By default, these commands format their output in a style that is pleasing to
the human eye.  For example,

```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6
PrivateAddresses: [{172.16.0.2 02:00:ac:10:00:02}]        
Created:          2018-01-04 16:15:51.767182887 +0000 UTC 
WorkloadID:       22368826-b7ba-4f97-8400-57276b1c383c    
NodeID:           4879795e-18f0-4c18-9be3-eb21e5c6df12    
ID:               45e56df2-350b-49f0-b394-f949cdf6b3b6    
Name:                                                     
Volumes:          [af75d901-2dcd-41fc-9827-3b30d320577c]  
Status:           active                                  
TenantID:         b5df368f-c097-437c-90d2-620a5d1bb0b0    
SSHIP:            198.51.100.71                           
SSHPort:          33002    
```

However, this is not always what we want, particularly if we are writing a
script to automate a set of ciao commands.  For example, say we wanted to
programmatically retrieve the ssh connection details for the above instance.
Using the command above we'd need to do some scripting to ignore the first 7
lines and extract the IP and port number from lines 8 and 9.  Nasty.

Luckily the ciao show and list commands accept a -f option which
is specified along with a [Go template](https://golang.org/pkg/text/template/).
These templates are little programs that can be used to extract the specific
data we are interested in.  For example, to extract the SSH IP and port numbers
we would issue the following command.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{.SSHIP}}:{{.SSHPort}}'
198.51.100.71:33002
```
{% endraw %}

No parsing required.

Check the help for each individual show and list subcommand to discover which
fields, e.g., SSHIP, are supported.  For example,

```shell
$ ciao help show instance
Show information about an instance

Usage:
  ciao show instance ID [flags]

Flags:
  -h, --help   help for instance

Global Flags:
  -f, --template string   Template used to format output

When using the template flag the following structure is provided:

struct {
	PrivateAddresses []struct {
		Addr    string `json:"addr"`
		MacAddr string `json:"mac_addr"`
	} `json:"private_addresses"`
	Created    time.Time `json:"created"`
	WorkloadID string    `json:"workload_id"`
	NodeID     string    `json:"node_id"`
	ID         string    `json:"id"`
	Name       string    `json:"name"`
	Volumes    []string  `json:"volumes"`
	Status     string    `json:"status"`
	TenantID   string    `json:"tenant_id"`
	SSHIP      string    `json:"ssh_ip"`
	SSHPort    int       `json:"ssh_port"`
}
```

## Template Cheat Sheet

Let's look at a few more examples of how we can use templates to extract
information from the ciao command.  Looking at the help of the ciao
show instance command shown above we can see that ciao passes a structure
to the template passed to the -f option.  The members of this structure can be
accessed inside template code by prefixing their name with '.'.  For example
the following command prints out the ID of an instance.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{.ID}}'
45e56df2-350b-49f0-b394-f949cdf6b3b6
```
{% endraw %}

Note the command prints out the instance without a newline which can be a bit
confusing.  We can fix this by including a newline character directly in the template.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{.ID}}
'
45e56df2-350b-49f0-b394-f949cdf6b3b6
```
{% endraw %}

or by using the println function

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{println .ID}}'
45e56df2-350b-49f0-b394-f949cdf6b3b6
```
{% endraw %}

Here's a more elaborate example in which we output the id, the status and ssh connection details
of the instance.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{.ID}} ({{.Status}}) {{.SSHIP}}:{{.SSHPort}}{{println}}'
45e56df2-350b-49f0-b394-f949cdf6b3b6 (active) 198.51.100.71:33002

```
{% endraw %}

 Now let's take a look at the PrivateAddresses field. This field is a slice of
 structures. We can gain access to this slice as follows .PrivateAddresses
 Let's output the slice to see what happens.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{println .PrivateAddresses}}'
[{172.16.0.2 02:00:ac:10:00:02}]
```
{% endraw %}

We see what appears to be a slice of structures.  We can use the template range and index
commands to access the elements of this slice.  For example to output the MAC addresses
of each structure we would type:

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{range .PrivateAddresses}}{{println .MacAddr}}{{end}}'
02:00:ac:10:00:02
```
{% endraw %}

Note that inside the {{range}}{{end}} tags the meaning of the . cursor changes.  Rather than
referring to the entire structure passed to template it refers to an individual element
of the .PrivateAddresses slice.  If you want to access a field of the top level structure
inside the range statement you need to use the $ operator.  For example, the following
command prints the NodeID of the instance before it prints each MAC address.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{range .PrivateAddresses}}{{$.NodeID}} : {{println .MacAddr}}{{end}}'
4879795e-18f0-4c18-9be3-eb21e5c6df12 : 02:00:ac:10:00:02
```
{% endraw %}

If we are only interested in the MAC address of a specific element of the slice
we can access it directly.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{println (index .PrivateAddresses 0).MacAddr}}'
02:00:ac:10:00:02
```
{% endraw %}

If you find the expression (index .PrivateAddresses 0).MacAddr a little confusing
you can split things up by introducing a new variable.

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{$addr := index .PrivateAddresses 0}}{{println $addr.MacAddr}}'
02:00:ac:10:00:02
```
{% endraw %}

Here the $addr variable becomes the first element of the slice.  $addr itself is a structure and
so we can use the . operator to access its fields.

However, there's a problem with this approach.  We don't know in advance how many entries
are present in the .PrivateAddresses slice.  If we try to index an element that doesn't
exist we'll get an error.  For example,

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{$addr := index .PrivateAddresses 1}}{{println $addr.MacAddr}}'
Error: Error generating template output: template: :1:11: executing "" at <index .PrivateAddres...>: error calling index: index out of range: 1
```
{% endraw %}

We can use the if statement to prevent us from accessing non-existing elements, e.g.,

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{if gt (len .PrivateAddresses) 1}}{{$addr := index .PrivateAddresses 1}}{{println $addr.MacAddr}}{{end}}'
```
{% endraw %}

The command prints nothing as our if statement evaluates to false.  There's only one element in
.PrivateAddresses slice.

Note in the previous example we use the rather unwieldy .PrivateAddresses twice.  We can eliminate
the repetition using the with statement, e.g.,

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '{{with .PrivateAddresses}}{{if gt (len .) 1}}{{$addr := index . 1}}{{println $addr.MacAddr}}{{end}}{{end}}'
```
{% endraw %}

Inside the with statement the . cursor takes on a new meaning.  It becomes assigned to the
value of .PrivateAddresses.

The example above is getting a little big to fit onto one line.  We might find it easier to
read if it were split onto multiple lines, e.g.,

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '
{{with .PrivateAddresses}}
	{{if gt (len .) 1}}
		{{$addr := index . 1}}
		{{println $addr.MacAddr}}
	{{end}}
{{end}}'
```
{% endraw %}

This is easier to read but unfortunately the newlines in the template that we added to improve
readability get copied to the output.  We can fix this by appending a '-' after the {% raw %}{{s and before
the }}s{% endraw %}, e.g.,

{% raw %}
```shell
$ ciao show instance 45e56df2-350b-49f0-b394-f949cdf6b3b6 -f '
{{- with .PrivateAddresses}}
	{{- if gt (len .) 1}}
		{{- $addr := index . 1}}
		{{- println $addr.MacAddr}}
	{{- end}}
{{- end -}}'
```
{% endraw %}

The '-'s gobble up the white space that occurs before the {% raw %}{{- and after the -}}{% endraw %}.

Here's one final example.  Let's take a look at the help for the ciao list workloads 
command.

```shell
$ ciao help list workloads
List workloads.

Usage:
  ciao list workloads [flags]

Flags:
  -h, --help   help for workloads

Global Flags:
  -f, --template string   Template used to format output

When using the template flag the following structure is provided:

[]struct {
	ID   string `json:"id"`
	Name string `json:"name"`
	CPUs int    `json:"vcpus"`
	Mem  int    `json:"ram"`
}
```

The important thing to note here is that the template is passed a slice of structures.
That means we need to use template function that can handle slices, e.g., len, index or range.
So to determine the number of workloads available we would type.

{% raw %}
```shell
$ ciao list workloads -f '{{println (len .)}}'
4
```
{% endraw %}

To output the names of each workload we might do

{% raw %}
```shell
$ ciao list workloads -f '{{range .}}{{println .Name}}{{end}}'
Clear Linux test VM
Debian latest test container
Ubuntu latest test container
Ubuntu test VM
```
{% endraw %}
