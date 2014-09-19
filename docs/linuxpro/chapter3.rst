
Chapter 3 : Memory Management
################################################

3.1 Overview
==========

3.1 Organization in the (N)UMAModel
===========================================

3.2 Page Tables
=========================


Data Structures
------------------



Creating and Manipulating Entries
-----------------------------------


3.3 Initialization ofMemoryManagement
=======================================


Data Structure Setup
----------------------



Architecture-Specific Setup
----------------------------


3.4 Memory Management during the Boot Process
================================================



3.5 Management of Physical Memory
================================



Structure of the Buddy System
--------------------------------


Avoiding Fragmentation
---------------------------


Initializing the Zone and Node Data Structures
------------------------------------------------


Allocator API
------------------


Reserving Pages
---------------------


Freeing Pages
--------------------


Allocation of Discontiguous Pages in the Kernel
--------------------------------------------------


Kernel Mappings
---------------------



3.6 The Slab Allocator
=========================


Alternative Allocators
---------------------------


Memory Management in the Kernel
--------------------------------


The Principle of Slab Allocation
---------------------------------


Implementation
------------------------


General Caches
-----------------------


3.7 Processor Cache and TLB Control
==================================


Summary
===========