---
title: "Hive UDFs in Ruby and Other Languages"
date: 2014-02-13T00:00:00-00:00
aliases:
    - /blog/2014/2/13/hive-udfs-in-ruby-and-other-languages
---

Apache Hive is a very powerful tool for processing data stored in Apache Hadoop. Structured and unstructured data can be accessed, processed, and manipulated using a SQL-like query language. This architecture allows anyone with reasonable SQL knowledge to write complex jobs with little to no knowledge of Hadoop, HDFS, and Hive.

```
-- Create a summary of purchases by product and day
SELECT day, product, count(*) as purchases, sum(revenue) as revenue, avg(revenue) as average_revenue_by_purchase
FROM purchases
WHERE day between '2014-01-01' and '2014-01-31'
GROUP BY day, product
ORDER BY product, day;
```

With only SQL knowledge very powerful data extractions can be performed, but SQL can be limiting.

### Enter UDFs

Hive supports User-Defined Functions as a way to add more complex capabilities than are available in SQL. There is a good list of included UDFs. Here at LivingSocial we have a few of our own in our HiveSwarm project. There are a few other bundles we like Klout Brickhouse and stewi2's UDFs. These UDFs greatly expand the capabilities of Hive by including features not available in standard SQL.

Developing new UDFs requires knowledge of the Hive API and Java. This is fine for Hadoop developers, but is a significant barrier for non-Java developers.

### Enter Scripted UDFs

Java supports running code in other languages through the javax.script API. Our HiveSwarm project has a new scripted UDF that makes use of this so UDFs can be written in languages other than Java. As a bonus this allows scripts to be directly in the Hive SQL instead of being compiled and jared before they can be run.

This shows how it can be used to compute data that would normally be difficult to generate via SQL.

```
create temporary function scriptedUDF as 'com.livingsocial.hive.udf.ScriptedUDF';
-- Gather complex data combining groups and individual rows without joins
 select person_id, purchase_data['time'], purchase_data['diff'],
   purchase_data['product'], purchase_data['purchase_count'] as pc,
   purchase_data['id_json']
 from (
   select person_id, scriptedUDF('
 require "json"
 def evaluate(data)
   # This gathers all the data about purchases by person in one place so 
   # complex infromation can be gathered while avoiding complex joins
   # Note:  In order for this to work all the data passed into 
   # scriptedUDF for a row needs to fit into memory
   tmp = []  # convert things over to a ruby array
   tmp.concat(data)
   tmp.sort_by! { |a| a.get("time") } # for the time differences
   last=0
   tmp.map{ |row|
     # Compute the time difference between purchases and add the total 
     # purchase count per person
     t = row["time"]
     # The parts that would be much more difficult to generate with SQL
     row["diff"] = t - last
     row["purchase_count"] = tmp.length
     row["first_purchase"] = tmp[0]["time"]
     row["last_purchase"] = tmp[-1]["time"]
     # This shows that built-in libraries are available
     row["id_json"] = JSON.generate({"id" => row["id"]})
     last = t
     row
   }
 end', 'ruby', 'array<map<string,string data-preserve-html-node="true">>',
        -- gather all the data about purchases by people so it can all be 
        -- passed into the evaluate function
        bh_collect(map(   -- Note, bh_collect is from Klouts Brickhouse 
                          -- and allows collecting any type, 
                          -- see https://github.com/klout/brickhouse/
           'time', purchase_time,
           'product', product_id)) ) as all_data
     from purchases
    group by person_id
 ) foo
 -- explode the data back out so it is available in flattened form
 lateral view explode(all_data) bar as purchase_data
 ```