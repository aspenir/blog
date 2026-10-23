+++
date = '2026-10-23T11:31:14+01:00'
draft = true
title = 'No-trust dependencies: how Zea could improve supply-chain security'
+++

Over the last few years, ecosystems such as Rust and JavaScript have increasingly embraced small, reusable "micropackages". While this approach encourages code reuse, it can also lead to surprisingly large dependency trees: `bat`, a Rust alternative to the Unix `cat` command, pulls in 249 dependencies during compilation.

Large dependency trees aren't inherently insecure, but each dependency expands the amount of code we implicitly trust. This can increase the attack surface of our software supply chain, particularly when dependencies or their maintainers are compromised.

Recent incidents illustrate how these attacks can happen:
- The account of a lead maintainer of the `axios` JavaScript HTTP client was compromised by attackers, who published a trojan-containing version of the library. This library has over 100 million weekly downloads on npm, illustrating the potential reach of an unnoticed compromised release. Fortunately, this was discovered within three hours, limiting the window of exposure and causing minimal impact.
- Microsoft's `durabletask` package on PyPI was compromised and had malicious code introduced that stole developer credentials.
- The Laravel PHP web framework had older version tags on GitHub silently changed to point to a malicious `autoload.php` file that harvested SSH keys and GitHub personal access tokens (PATs).

### So, how can we fix this?
As [purplesyringa (Alisa Sireneva) points out](https://purplesyringa.moe/blog/no-one-owes-you-supply-chain-security/) - maintainers do not necessarily owe users supply-chain security. The MIT license itself includes a disclaimer of responsibility; `THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND`.

In contemporary language ecosystems like Rust or Python, the developer takes the responsibility of auditing their dependencies, or hoping some thankless security analyst finds any malicious code before it reaches their projects.

This approach obviously does not scale well for large codebases.

Perhaps our compiler could help us avoid granting implicit permission to everything. What if a function had to declare the effects it performs in its signature?

At the moment, most ecosystems treat dependencies as trusted black boxes, even when implementations can access sensitive resources - sometimes build scripts can access resources beyond their project directories.

Zea takes a different approach: every function must declare its effects, and the compiler checks that callers account for the effects of the functions they invoke. These requirements propagate through the call chain to the program's entry point (`main()`).
```zea
const std = import("std");

fn write_foo_to_file()
-> void
: std.File.write, std.Raises(string)
{
	var file = std.File.open("file.txt");
	file.write("foo").expect("foo failed :(");
}

fn call_foo()
-> void
: std.File.write, std.Raises(string)
{
	write_foo_to_file();
}

fn fail_to_call_foo()
-> void
{
	write_foo_to_file();
	// COMPILER ERROR; fail_to_call_foo does not declare effects
	// std.File.write or std.Raises, which are used in
	// write_foo_to_file()
}
```
Here, `std.File.write` tells the compiler that `write_foo_to_file` performs a filesystem write. Any caller that invokes this function must also declare the effects required by that call, ensuring that these requirements propagate through the call stack.


### But... how does this resolve our issue?
Consider a compromised release of a library like `axios`. A developer importing an HTTP client reasonably expects it to perform network operations, not to harvest SSH keys or execute unrelated malicious code.

In a conventional language ecosystem, a dependency may execute with the same underlying permissions as the application that uses it. Its intended purpose does not necessarily constrain what its implementation can do.

Zea aims to make these requirements explicit. If a function performs filesystem operations, it must declare the corresponding effects, and those effects must propagate through its callers. Unexpected changes to a dependency's declared effects could therefore become visible during compilation or dependency review.

This is not a complete solution to supply-chain attacks: malicious code can still abuse capabilities it legitimately possesses, and code that executes outside Zea's effect-checking system requires separate consideration. However, explicit effects provide a foundation for identifying unexpected operations and restricting access to sensitive resources.

One possible future improvement is to introduce more granular effects. Rather than granting general filesystem-write access, Zea could allow a function to request write access only to a particular directory or resource. Enforcing such restrictions would help prevent malicious code from abusing a legitimate permission to access unrelated files.

However, there is a trade-off between security and usability. More granular effects can provide tighter control, but they can also make APIs more verbose. Zea aims to balance explicit security requirements with brief, readable code.

This principle also extends to Zea's build system, which uses a Zig-like `build.zea` file to describe how a project should be compiled and which libraries to link.

Build scripts are an important part of the supply-chain threat model because they may execute before the application itself is built. If Zea applies effect checking and appropriate restrictions to build scripts as well, it could help prevent build-time code from accessing resources it doesn't need.

The broader goal is to make the capabilities of dependencies explicit and, where possible, enforce the principle of least privilege throughout both the build process and program execution.

Zea cannot eliminate supply-chain attacks, but it aims to make dependencies easier to reason with and their required capabilities easier to audit.
