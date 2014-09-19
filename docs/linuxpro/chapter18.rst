Chapter 18 : Page Reclaim and Swapping
###########################################################


18.1 Overview
===========================================


Swappable Pages
-------------------------------------


Page Thrashing
-------------------------------------


Page-Swapping Algorithms
-------------------------------------


18.2 Page Reclaim and Swapping in the Linux Kernel
=======================================================


Organization of the Swap Area
-------------------------------------


Checking Memory Utilization
-------------------------------------


Selecting Pages to Be Swapped Out
-------------------------------------


Handling Page Faults
-------------------------------------


Shrinking Kernel Caches
-------------------------------------


18.3 Managing Swap Areas
===========================================


Data Structures
-------------------------------------


Creating a Swap Area
-------------------------------------


Activating a Swap Area
-------------------------------------


18.4 The Swap Cache
===========================================


Identifying Swapped-Out Pages
-------------------------------------


Structure of the Cache
-------------------------------------


Adding New Pages
-------------------------------------


Searching for a Page
-------------------------------------


18.5 Writing Data Back
===========================================


18.6 Page Reclaim
===========================================


Overview
-------------------------------------


Data Structures
-------------------------------------


Determining Page Activity
-------------------------------------


Shrinking Zones
-------------------------------------


Isolating LRU Pages and Lumpy Reclaim
-------------------------------------


Shrinking the List of Active Pages
-------------------------------------


Reclaiming Inactive Pages
-------------------------------------


18.7 The Swap Token
===========================================


18.8 Handling Swap-Page Faults
===========================================


Swapping Pages in
-------------------------------------


Reading the Data
-------------------------------------


Swap Readahead
-------------------------------------


18.9 Initiating Memory Reclaim
===========================================


Periodic Reclaim with kswapd
-------------------------------------


Swap-out in the Event of Acute Memory Shortage
----------------------------------------------------


18.10 Shrinking Other Caches
===========================================


Data Structures
-------------------------------------


Registering and Removing Shrinkers
-------------------------------------


Shrinking Caches
-------------------------------------


Summary
===========================
