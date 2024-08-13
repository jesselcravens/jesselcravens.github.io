---
layout: post
title:  BASH Data File Shrink and Repackage Script
date:   2024-08-13 11:19:37 -0500
categories: Linux
---
# BASH Data File Shrink and Repackage Script

This script was ran to break down large data files into smaller pieces as it was stopping our software from successfully processing them. Internally, the gzip files were composed of tarred csv files. This script would untar the files, then retar the individual files and preserve the filenames to better track them as belonging to the original files. 

This was written to give the user inputs and outputs for those less familiar with BASH. 

```BASH

read -p 'Data filename: ' filename

# for this specific file type, the timestamp is in positions 25 through 40 and will be used later to rename the files
timestamp=$(echo $filename | cut -c 25-40)

# being working in the tmp directory
cd /tmp/

# if there are any files in this directory remove them 
rm -rf /tmp/datafiles/*

echo "Untarring file begin"
echo ""
tar -C /tmp/datafiles/ -xvf $filename
echo ""
echo "Untarring complete"

# I wanted to add two digits to the end of the filename. 
# This starts the variable at 9, but the for loop adds 1 at the start. 
i=9


cd /tmp/datafiles/
echo "Begin Tarring smaller files:"
echo ""
for x in `find . -name "*.gz" -printf '%P\n'`
do
        ((i=i+1))
	cd /tmp/datafiles/
        tar -cf "datafile-$timestamp-$i.tar" $x

done

cd /tmp/datafiles/
echo "Tarring smaller files complete"
echo ""
rm *.csv.gz

echo ""
echo "Moving datafiles into local input directory /opt/output/"
echo ""
mv /tmp/datafiles/*.tar /opt/output/

echo "Process complete"

```
