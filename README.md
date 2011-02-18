RbVmomi
=======

Introduction
------------

RbVmomi is a Ruby interface to the vSphere API. Like the Perl and Java SDKs,
you can use it to manage ESX and VirtualCenter servers. The current release
supports the vSphere 4.1 API.

Usage
-----

A simple example of turning on a VM:

    require 'rbvmomi'
    conn = RbVmomi.connect host: 'foo', user: 'bar', password: 'baz'
    dc = conn.serviceInstance.find_datacenter("mydatacenter") or fail "datacenter not found"
    vm = dc.find_vm("myvm") or fail "VM not found"
    vm.PowerOn_Task.wait_for_completion

This code uses several RbVmomi extensions to the VI API for concision. The
expanded snippet below uses only standard API calls and should be familiar to
users of the Java SDK:

    require 'rbvmomi'
    conn = RbVmomi.connect host: 'foo', user: 'bar', password: 'baz'
    rootFolder = conn.serviceInstance.content.rootFolder
    dc = rootFolder.childEntity.grep(RbVmomi::VIM::Datacenter).find { |x| x.name == "mydatacenter" } or fail "datacenter not found"
    vm = dc.vmFolder.childEntity.grep(RbVmomi::VIM::VirtualMachine).find { |x| x.name == "myvm" } or fail "VM not found"
    task = vm.PowerOn_Task
    filter = conn.propertyCollector.CreateFilter(
      spec: {
        propSet: [{ type => 'Task', all: false, pathSet: ['info.state']}],
        objectSet: [{ obj: task }]
      },
      partialUpdates: false
    )
    ver = ''
    while true
      result = conn.propertyCollector.WaitForUpdates(version: ver)
      ver = result.version
      break if ['success', ['error'].member? task.info.state
    end
    filter.DestroyPropertyFilter
    raise task.info.error if task.info.state == 'error'

As you can see, the extensions RbVmomi adds can dramatically decrease the code
needed to perform simple tasks while still letting you use the full power of
the API when necessary. RbVmomi extensions are often more efficient than a
naive implementation; for example, the find_vm method on VIM::Datacenter used
in the first example uses the SearchIndex for fast lookups.

A few important points:
 * Ruby 1.9 is required.
 * Properties are exposed as methods: vm.summary
 * All class, method, parameter, and property names match the [official documentation](http://www.vmware.com/support/developer/vc-sdk/visdk41pubs/ApiReference/index.html).
 * Data object types can usually be inferred from context, so you may simply use a hash instead.
 * Enumeration values are simply strings.
 * Example code is included in the examples/ directory.
 * A set of helper methods for Trollop is included to speed up development of
   command line apps. See the included examples for usage.
 * This is a side project of a VMware developer and is entirely unsupported by VMware.

Built-in extensions are in lib/rbvmomi/extensions.rb. You are encouraged to
reopen VIM classes in your applications and add extensions of your own. If you
write something generally useful please send it to me and I'll add it in. One
important point about extensions is that since VIM classes are lazily loaded,
you need to trigger this loading before you can reopen the class. Putting the
class name on a line by itself before reopening is enough.

Development
-----------

Send patches to rlane@vmware.com.
