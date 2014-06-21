---
layout: post
title:  "Introduction to Elastic Map Reduce"
date:   2014-06-18
---


## Introduction to Elastic Map Reduce


### Download the dataset

First things first, you need to download the dataset. I was able to pull the entire dataset from the following [link](http://www.andresmh.com/nyctaxitrips/). I used `wget` to pull download the dataset to a running EC2 instance, and then saved the tar.gz to an S3 bucket for later use.

```
wget https://nyctaxitrips.blob.core.windows.net/data/trip_fare_4.csv.zip
```

I pulled down every file using `wget` before starting. After the entire data set was pulled down, I needed uncompress the fare files and the trip files using tar


	sudo apt-get install unzip
	unzip trip_fare_4.csv.zip

Finally, we need to upload the uncompressed files to an S3 buckets so they can be processed by Amazon EMR. To do this, I used [s3cmd](http://s3tools.org/s3cmd). Before using the command, you need to configure it to use your Amazon access/secret keys

    s3cmd --configure
	
Once the command has been configured, you can run the following commands to make new s3 buckets, and to copy the files from your local hard drive to those buckets:

### Data set format

```
ubuntu@/taxi-data$ cat trip_data_1.csv | head -n 2
medallion,hack_license,vendor_id,rate_code,store_and_fwd_flag,pickup_datetime,dropoff_datetime,passenger_count,trip_time_in_secs,trip_distance,pickup_longitude,pickup_latitude,dropoff_longitude,dropoff_latitude
89D227B655E5C82AECF13C3F540D4CF4,BA96DE419E711691B9445D6A6307C170,CMT,1,N,2013-01-01 15:11:48,2013-01-01 15:18:10,4,382,1.00,-73.978165,40.757977,-73.989838,40.751171
```

The fields in the trip data set are the following

	medallion
	hack_license
	vendor_id
	rate_code
	store_and_fwd_flag
	pickup_datetime
	dropoff_datetime
	passenger_count
	trip_time_in_secs
	trip_distance
	pickup_longitude
	pickup_latitude
	dropoff_longitude
	dropoff_latitude

The fields in the fare data set are the following

```
medallion, hack_license, vendor_id, pickup_datetime, payment_type, fare_amount, surcharge, mta_tax, tip_amount, tolls_amount, total_amount
89D227B655E5C82AECF13C3F540D4CF4,BA96DE419E711691B9445D6A6307C170,CMT,2013-01-01 15:11:48,CSH,6.5,0,0.5,0,0,7
```

	medallion
	hack_license
	vendor_id
	pickup_datetime
	payment_type
	fare_amount
	surcharge
	mta_tax
	tip_amount
	tolls_amount
	total_amount

### Writing the mapper and the reducer

We need a mapper that parses the trip file

{% highlight python %}
#!/usr/bin/env python
# encoding: utf-8

import sys, os, re, argparse

def main(options):
	line = sys.stdin.readline()	
	try:
		while line:
			
			line = line.strip().split(",")
			data = {
			'medallion' : line[0],
			'hack_license' : line[1],
			'vendor_id' : line[2],
			'rate_code' : line[3],
			'store_and_fwd_flag' : line[4],
			'pickup_datetime' : line[5],
			'dropoff_datetime' : line[6],
			'passenger_count' : line[7],
			'trip_time_in_secs' : line[8],
			'trip_distance' : line[9],
			'pickup_longitude': line[10],
			'pickup_latitude' :line[11],
			'dropoff_longitude': line[12],
			'dropoff_latitude' :line[13],
			}

			
			print "\t".join([data[f] for f in options.fields.split(",")])
			
			line =  sys.stdin.readline()
	except:
		return None

if __name__ == "__main__":
	parser = argparse.ArgumentParser(description='')
	parser.add_argument('--fields', dest='fields',type=str,default="")
	sys.exit(main(parser.parse_args()))
	
{% endhighlight %}

You can use this command as follows:

	cat trip_data_1_10K.csv | ./mapper.py  --fields medallion,dropoff_datetime

The reducer doesnt need to do much

{% highlight python %}

#!/usr/bin/env python
# encoding: utf-8

import sys, os, re, json

for line in sys.stdin:
	try:
		line = line.strip()
		respondent, likes = line.split('\t')
		likes = likes.split(",")
		for like in likes:
			print like
	except:
		pass

{% endhighlight %}

### Questions?

If you have any questions, or want to read more articles about machine learning, hadoop, pandas, python django, and statistics, following me [@josephmisiti](https://www.twitter.com/josephmisiti)
