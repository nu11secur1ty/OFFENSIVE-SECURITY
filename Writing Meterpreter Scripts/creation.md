**There are a few things you need to keep in mind when creating a new Meterpreter script.**

- Not all versions of Windows are the same
- Some versions of Windows have security countermeasures for some of the commands
- Not all command line tools are in all versions of Windows
- Some of the command line tools switches vary depending on the version of Windows

In short, the same constraints that you have when working with standard exploitation methods. MSF can be of great help, but it can’t change the fundamentals of that target. Keeping this in mind can save a lot of frustration down the road. So keep your target’s Windows version and service pack in mind, and build to it.

For our purposes, we are going to create a stand alone binary that will be run on the target system that will create a reverse Meterpreter shell back to us. This will rule out any problems with an exploit as we work through our script development.
