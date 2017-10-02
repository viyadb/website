Clustering
===========

ViyaDB implements shared nothing articecture, which means that workers hold partitions
of data independently from each other. This allows for many possible implementations of
clustering depending on business needs.

One possible implementation looks like follows:

![Clustering Example](img/clustering1.png)

The work is still in progress. To get involved into design considerations please start
a new discussion on the [user group](https://groups.google.com/forum/#!forum/viyadb),
or drop me a line at [michael@viyadb.com](mailto:michael@viyadb.com).

