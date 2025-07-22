# Google Cloud Snaphot

## Pre-amble

Unlike with other cloud providers it seems that Google’s separates the compute and storage layers, at least where snapshots are concerned. This means that it’s not possible to do a snapshot like in vmware / Xen where you can snapshot the servers memory state at the same time. As such snapshotting is stored in the “Storage → Disks” section.

## Notes

Snapshots will incur a charge based under disk usage per GB [https://cloud.google.com/compute/all-pricing#disk](https://cloud.google.com/compute/all-pricing#disk)

## Process

1. Navigate to the “Compute Engine” for the compute project that holds the server
2. Click on “Disks” under “Storage” in the left side bar
3. Locate the disk of the server that you wish to snapshot.
   1. NB: at the top of the page under the “Disks” header there’s a search / filter box. This can eitehr be a free text filter or you can specify which type of data you would like to search under i.e
      1. “Name” - the of the disk
      2. “In use by” - the name of the server using the disk
4. Click into the disk
5. At the top of the details page click “CREATE SNAPSHOT”
6. Enter a sensible name or description for the snapshot so that it can easily be understood why it was taken, and likely the required time for keeping it.
7. Amend other fields as required however it should be safe to leave them as default with name being the only required filed that needs filling in.
8. click create
