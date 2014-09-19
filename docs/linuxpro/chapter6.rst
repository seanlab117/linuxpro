
Chapter 6 : Device Drivers
####################################################


6.1 I/O Architecture
==================================


Expansion Hardware
--------------------------------------


6.2 Access to Devices
==================================


Device Files
--------------------------------------


Character, Block, and Other Devices
--------------------------------------


Device Addressing Using Ioctls
--------------------------------------


Representation of Major and Minor Numbers
--------------------------------------


Registration
--------------------------------------


6.3 Association with the Filesystem
==================================



Device File Elements in Inodes
--------------------------------------


Standard File Operations
--------------------------------------


Standard Operations for Character Devices
----------------------------------------------


Standard Operations for Block Devices
--------------------------------------


6.4 Character Device Operations
==================================





Representing Character Devices
--------------------------------------


Opening Device Files
--------------------------------------


Reading and Writing
--------------------------------------


6.5 Block Device Operations
==================================



Representation of Block Devices
--------------------------------------


Data Structures
--------------------------------------


Adding Disks and Partitions to the System
----------------------------------------------


Opening Block Device Files
--------------------------------------


Request Structure
--------------------------------------


BIOs
--------------------------------------


Submitting Requests
--------------------------------------


I/O Scheduling
--------------------------------------


Implementation of Ioctls
--------------------------------------


6.6 Resource Reservation
==================================



Resource Management
--------------------------------------


I/O Memory
--------------------------------------


I/O Ports
--------------------------------------


6.7 Bus Systems
==================================


The Generic Driver Model
--------------------------------------


The PCI Bus
--------------------------------------


USB
--------------------------------------


Summary
==================================




