# Porting AlmaLinux to RISC-V

First, a little background. I noticed the other day that there are now SBCs
with RISC-V processors. And since (to my knowledge) neither RedHat nor any
of the distributions downstream of it support RISC-V yet, I thought it would
be an interesting project to try porting AlmaLinux 9 to RISC-V.

I shared the idea with my manager and quickly got funding for a target that
I could use for porting and testing: a StarFive VisionFive2. Because StarFive
only provides a Debian image for this target, I frankensteined a Fedora 38
image for the board, which I use for native package builds. Details can be
found in the [Targets](#targets) section.

## Targets

* [StarFive VisionFive2](targets/visionfive2.md)
* Sipeed LicheePi A4 (coming soon)
