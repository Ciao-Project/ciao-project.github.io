---
title: Command line scripting
permalink: scripting.html
keywords: ciao, command line, tool, scripting, templates
---

## Scripting with ciao-cli

Most of the ciao-cli commands contain a list or show subcommand, e.g.,

```shell
$ ciao-cli instance list
```

By default, these commands format their output in a style that is pleasing to
the human eye.  For example,

```shell
$ ciao-cli instance show --instance cef5b810-5ffb-4dee-ab95-29748869afb6

    UUID: cef5b810-5ffb-4dee-ab95-29748869afb6
    Status: active
    Private IP: 172.16.0.3
    MAC Address: 02:00:ac:10:00:03
    CN UUID: fe4fa7da-0c46-46cf-9205-28c9d675aa5a
    Image UUID: 73a86d7e-93c0-480e-9c41-ab42f69b7799
    Tenant UUID: f452bbc7-5076-44d5-922c-3b9d2ce1503f
    SSH IP: 198.51.100.75
    SSH Port: 33003
```

However, this is not always what we want, particularly if we are writing a
script to automate a set of ciao commands.  For example, say we wanted to
programmatically retrieve the ssh connection details for the above instance.
Using the command above we'd need to do some scripting to ignore the first 7
lines and extract the IP and port number from lines 8 and 9.  Nasty.

Luckily all the ciao-cli show and list commands accept a -f option which
is specified along with a [Go template](https://golang.org/pkg/text/template/).
These templates are little programs that can be used to extract the specific
data we are interested in.  For example, to extract the SSH IP and port numbers
we would issue the following command.

{% raw %}
```shell
$ ciao-cli instance show --instance cef5b810-5ffb-4dee-ab95-29748869afb6 -f '{{.SSHIP}}:{{.SSHPort}}'

198.51.100.75:33003
```
{% endraw %}

No parsing required.

Check the help for each individual show and list command to discover which
fields, e.g., SSHIP, are supported.  For example,

```shell

$ ciao-cli instance show --help

usage: ciao-cli [options] instance show [flags]

Print detailed information about an instance

The show flags are:

  -f string
    	Template used to format output
  -instance string
    	Instance UUID

The template passed to the -f option operates on a

struct {
	HostID   string                               // ID of the host node
	ID       string                               // Instance UUID
	TenantID string                               // Tenant UUID
	Flavor   struct {
		ID string                             // Workload UUID
	}
	Image struct {
		ID string                             // Backing image UUID
	}
	Status    string                              // Instance status
	Addresses struct {
		Private []struct {
			Addr               string     // Instance IP address
			OSEXTIPSMACMacAddr string     // Instance MAC address
		}
	}
	SSHIP   string                                // Instance SSH IP address
	SSHPort int                                   // Instance SSH Port
}
```

## Template Cheat Sheet

Let's look at a few more examples of how we can use templates to extract
information from the ciao-cli command.  Looking at the help of the ciao-cli
instance show command shown above we can see that ciao-cli passes a structure
to the template passed to the -f option.  The members of this structure can be
accessed inside template code by prefixing their name with '.'.  For example
the following command prints out the ID of an instance.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{.ID}}'
80efbb0a-23ae-4d47-8e74-39fb18497c85
```
{% endraw %}

Note the command prints out the instance without a newline which can be a bit
confusing.  We can fix this by including a newline character directly in the template.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{.ID}}
'
80efbb0a-23ae-4d47-8e74-39fb18497c85
```
{% endraw %}

or by using the println function

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{println .ID}}'
80efbb0a-23ae-4d47-8e74-39fb18497c85
```
{% endraw %}

Here's a more elaborate example in which we output the id, the status and ssh connection details
of the instance.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{.ID}} ({{.Status}}) {{.SSHIP}}:{{.SSHPort}}{{println}}'
80efbb0a-23ae-4d47-8e74-39fb18497c85 (active) 198.51.100.96:33002
```
{% endraw %}

Now let's take a look at the Addresses field.  This field is a structure that
contains a slice of structures.  We can gain access to this slice as follows
.Addresses.Private.  Let's output the slice to see what happens.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{println .Addresses.Private}}'
[{172.16.0.2 02:00:ac:10:00:02  0}]
```
{% endraw %}

We see what appears to be a slice of structures.  We can use the template range and index
commands to access the elements of this slice.  For example to output the MAC addresses
of each structure we would type:

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{range .Addresses.Private}}{{println .OSEXTIPSMACMacAddr}}{{end}}'
02:00:ac:10:00:02
```
{% endraw %}

Note that inside the {{range}}{{end}} tags the meaning of the . cursor changes.  Rather than
referring to the entire structure passed to template it refers to an individual element
of the .Addresses.Private slice.  If you want to access a field of the top level structure
inside the range statement you need to use the $ operator.  For example, the following
command prints the HostID of the instance before it prints each MAC address.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{range .Addresses.Private}}{{$.HostID}} : {{println .OSEXTIPSMACMacAddr}}{{end}}'
c483c178-2109-4a54-bf00-98cbf4bfa58b : 02:00:ac:10:00:02
```
{% endraw %}

If we are only interested in the MAC address of a specific element of the slice
we can access it directly.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{println (index .Addresses.Private 0).OSEXTIPSMACMacAddr}}'
02:00:ac:10:00:02
```
{% endraw %}

If you find the expression (index .Addresses.Private 0).OSEXTIPSMACMacAddr a little confusing
you can split things up by introducing a new variable.

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{$addr := index .Addresses.Private 0}}{{println $addr.OSEXTIPSMACMacAddr}}'
02:00:ac:10:00:02
```
{% endraw %}

Here the $addr variable becomes the first element of the slice.  $addr itself is a structure and
so we can use the . operator to access its fields.

However, there's a problem with this approach.  We don't know in advance how many entries
are present in the .Addresses.Private slice.  If we try to index an element that doesn't
exist we'll get an error.  For example,

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{$addr := index .Addresses.Private 1}}{{println $addr.OSEXTIPSMACMacAddr}}'
F1117 10:54:29.246108    8776 template.go:30] ciao-cli FATAL: template: instance-show:1:11: executing "instance-show" at <index .Addresses.Pri...>: error calling index: index out of range: 1
goroutine 1 [running]:
```
{% endraw %}

We can use the if statement to prevent us from accessing non-existing elements, e.g.,

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{if gt (len .Addresses.Private) 1}}{{$addr := index .Addresses.Private 1}}{{println $addr.OSEXTIPSMACMacAddr}}{{end}}'
```
{% endraw %}

The command prints nothing as our if statement evaluates to false.  There's only one element in
.Addresses.Private slice.

Note in the previous example we use the rather unwieldy .Addresses.Private twice.  We can eliminate
the repetition using the with statement, e.g.,

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{with .Addresses.Private}}{{if gt (len .) 1}}{{$addr := index . 1}}{{println $addr.OSEXTIPSMACMacAddr}}{{end}}{{end}}'
```
{% endraw %}

Inside the with statement the . cursor takes on a new meaning.  It becomes assigned to the
value of .Addresses.Private.

The example above is getting a little big to fit onto one line.  We might find it easier to
read if it were split onto multiple lines, e.g.,

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{with .Addresses.Private}}
   {{if gt (len .) 1}}
     {{$addr := index . 1}}{{println $addr.OSEXTIPSMACMacAddr}}
   {{end}}
 {{end}}
'
```
{% endraw %}

This is easier to read but unfortunately the newlines in the template that we added to improve
readability get copied to the output.  We can fix this by appending a '-' after the {% raw %}{{s and before
the }}s{% endraw %}, e.g.,

{% raw %}
```shell
$ ciao-cli instance show --instance 80efbb0a-23ae-4d47-8e74-39fb18497c85 --f '{{with .Addresses.Private}}
  {{- if gt (len .) 1}}
    {{- $addr := index . 1}}{{println $addr.OSEXTIPSMACMacAddr}}
  {{- end}}
{{- end -}}
'
```
{% endraw %}

The '-'s gobble up the white space that occurs before the {% raw %}{{- and after the -}}{% endraw %}.

Here's one final example.  Let's take a look at the help for the ciao-cli workload list
command.

```shell
$ ciao-cli workload list --help
usage: ciao-cli [options] workload list

List all workloads

  -f string
    	Template used to format output

The template passed to the -f option operates on a

[]struct {
	OSFLVDISABLEDDisabled  bool    // Not used
	Disk                   string  // Backing images associated with workload
	OSFLVEXTDATAEphemeral  int     // Not currently used
	OsFlavorAccessIsPublic bool    // Indicates whether the workload is available to all tenants
	ID                     string  // ID of the workload
	Links                  []Link  // Not currently used
	Name                   string  // Name of the workload
	RAM                    int     // Amount of RAM allocated to instances of this workload
	Swap                   string  // Not currently used
	Vcpus                  int     // Number of Vcpus allocated to instances of this workload
}
```

The important thing to note here is that the template is passed a slice of structures.
That means we need to use template function that can handle slices, e.g., len, index or range.
So to determine the number of workloads available we would type.

{% raw %}
```shell
$ ciao-cli workload list -f '{{println (len .)}}'
5
```
{% endraw %}

To output the names of each workload we might do

{% raw %}
```shell
$ ciao-cli workload list -f '{{range .}}{{println .Name}}{{end}}'
Fedora 24 Cloud
Clear Cloud
Docker Debian latest
Docker Iperf
Boot Fedora24 from created volume based on image
```
{% endraw %}
