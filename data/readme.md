## Data Files

For this workshop, we're using a standard demo data set from Cockroach Labs, slightly modified.  You can follow along with the workshop instructions right here in github - start at the readme in the root of the repository - but you'll need to download the data and perform a couple pre-processing steps on it to actually do the exercises.

Unfortunately, the data file is much too large to upload to github.  Here's what you need to do:

### Fetch and process

1. Click this URL, or copy and paste it into your browser, to download the CSV file to your computer:
  
https://cockroach-university-public.s3.amazonaws.com/10000000_books.csv

2. In the directory you downloaded the file to, run this command to drop the first data column:
```
awk -F, 'BEGIN {OFS=","} { $1=""; sub(/^,/, ""); print }' 10000000_books.csv > 10M_books.csv
```

3. Then run this command to strip off the header row:
```
tail -n +2 10M_books.csv > library.csv
```

4. Verify the results with `head +5 library.csv`. The output should look like this:
```
% head -n 5 library.csv
these thing clap,Jaqueline Sauer,47.65,hardcover,2020-01-01
wandering child stack,Waldo Raynor,31.86,hardcover,2020-01-01
too way sit,Jennings Carter,85.46,paperback,2020-01-01
beautiful company listen,Thea McLaughlin,58.84,audio,2020-01-01
which thing dig,Cathy Heathcote,23.16,hardcover,2020-01-01
```

5. You'll use this resulting `library.csv` file to set up your workshop environment, as described in the introduction.  

### Clean up

The other 2 files (`10000000_books.csv` and `10M_books.csv`) are together occuping about 1.5GB of your disk space.  We're not going to need them again.  You can delete them.
