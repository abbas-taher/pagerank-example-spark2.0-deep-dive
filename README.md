## Tutorial 101: PageRank Example in Spark 2.0
### Understanding the Algorithm & Spark Code Implementation
 
  The Apache [SparkPageRank.scala](/SparkPageRank.scala?raw=true "SparkPageRank") is a good example to learn how to program in Spark 2.0 using Scala. The sample program computes the PageRank of URLs from an input file that has the following format: <br>
  
 &nbsp; url_1 &nbsp;  url_4
 <br> &nbsp; url_2 &nbsp;  url_1
 <br> &nbsp; url_3 &nbsp;  url_2
 <br> &nbsp; url_3 &nbsp;  url_1
 <br> &nbsp; url_4 &nbsp;  url_3
 <br> &nbsp; url_4 &nbsp;  url_1  

Each URL and their neighbors are separated by space(s). The above input data can be represented by the graph below. For example, URL_3 references URL_1 & URL_2 while it is referenced by URL_4.  

<img src="/images/img-1.jpg" width="342" height="293">

The code looks deceivingly simple but to understand how things actually work requires a deeper understanding of Spark RDDs, Spark's Scala based functional API, as well as Page Ranking formula. In a previous article I have described the steps required to setup the project in Scala IDE for Eclipse and run the code on Hortonworks 2.5 Sandbox. Here are shall take a deep dive into how the algorithm works and try to uncover its implementation detail and how it actually runs. 

## Contents:
- How the Algorith Works
- Running the PageRank Program in Spark
- Part 1: Reading the Data File
- Part 2: Populating the Ranks Data - Initial Seeds
- Part 3: Looping and Calculating Contributions & Recalcualting Ranks

## How the Algorithm Works
The PageRank algorithm outputs a probability distribution that represents the likelihood that a person randomly clicking on web links will arrive at a particular web page. If we run the PageRank program with the input data file and indicate 20 iterations we shall get the following output: <br>

     url_4 has rank: 1.3705281840649928.
     url_2 has rank: 0.4613200524321036.
     url_3 has rank: 0.7323900229505396.
     url_1 has rank: 1.4357617405523626.
 
The results clearly indicates that URL_1 has the highest page rank followed by URL_4 and then URL_3 & last URL_2. The algorithm works in the following manner:

If a URL (page) is referenced by other URLs then its rank increases because being referenced means that it is important which is the case of URL_1. While if an important URL like URL_1 references other URLs this will increase the destination’s ranking which is the case of URL_4 that is referenced by URL_1; that is the reason why URL_4 ranking is higher than the other two URLs (URL_2 & URL_3). If we look at the various arrows in the above diagram we can see that URL_2 is referenced the least and that is the reason why it has the lowest ranking.

The rest of the article will take a deeper look at the Scala code that implements the algorithm in Spark 2.0. The code is made of 3 main parts as shown in the diagram below. The 1st part reads the data file then each URL is given a seed value in rank0. The third part of the code contains the main loop which calculates the contributions by joining the links and ranks data at each iteration and then recalculates the ranks based on that contribtion. 

<img src="/images/img-2.jpg" width="806" height="594">

## Running the PageRank Program in Spark
To run the PageRank program you need to pass the class name, jar location, input data file and number of iterations. The command looks like the following (please refer to the Project Setup article): 

      $ cd /usr/hdp/current/spark2-client
      $ export SPARK_MAJOR_VERSION=2
      $ ./bin/spark-submit --class com.scalaproj.SparkPageRank --master yarn --num-executors 1 --driver-memory 512m --executor-memory 512m --executor-cores 1 ~/testing/jars/page-rank.jar /input/urldata.txt 20  
 
In case you would like to use the bash command that comes with Spark you can run the example with the following:

      $ ./bin/run-example SparkPageRank /input/urldata.txt 20

## Part 1: Reading the Data File
The code for the 1st part of the program is as follows:

     (1)    val iters = if (args.length > 1) args(1).toInt else 10   // sets iteration from argument (in our case iter=20)
     (2)    val lines = spark.read.textFile(args(0)).rdd   // read text file into Dataset[String] -> RDD1
            val pairs = lines.map{ s =>
     (3)         val parts = s.split("\\s+")               // Splits a line into an array of 2 elements according space(s)
     (4)              (parts(0), parts(1))                 // create the tuple <url, url> for each line in the file
                  }
     (5)    val links = pairs.distinct().groupByKey().cache()   // RDD1 <string, string> -> RDD2<string, iterable>   

The 2nd line of the code reads the input data file and produce a Dataset of strings which are then transformed into an RDD with each line in the file being one whole string within the RDD. You can think of an RDD as a list that is special to Spark because the data within the RDD is distributed among the various nodes. <i> Please note that I have introduced a "pairs" variable into the original code to make the program more readable.<i>

In the 3rd line of the code, the split command generates for each line (whole string) an array with two elements. In 4th line each of the two elements of the array are accessed and then used to produce a key/value pair. The last line in the code applies the groupByKey command on the key/value pair RDD to produce the links RDD; which is also a key/value pair. Thus, the resultant links RDD for the input data file will be as follows:<br>

&nbsp; Key   &emsp;    Array (iterable)
<br> &nbsp; url_4  &emsp;   [url_3, url_1]
<br> &nbsp; url_3  &emsp;   [url_2, url_1]
<br> &nbsp; url_2   &emsp;  [url_1]
<br> &nbsp; url_1   &emsp;  [url_4]
 
Note that the Array in the above is not a true array it is actually an iterator on the resultant array of urls. This is what the groupByKey command produces when applied on an RDD. This is an important and powerful construct in Spark and every programmer needs to understand it well so that they can use it correctly in their code..

## Part 2: Populating the Ranks Data - Initial Seeds 
 
The code in this part is made of a single line:

    var ranks = links.mapValues(v => 1.0)    // create a new RDD <key,one>

which creates a third RDD that is also made of a key/value pair, but in this case a <string,double> pair. <br>

&nbsp;  Key  &emsp;  Value (Double) 
<br> &nbsp;  url_4 &emsp;  1.0
<br> &nbsp;  url_3 &emsp;  1.0
<br> &nbsp;  url_2 &emsp;  1.0
<br> &nbsp;  url_1 &emsp;  1.0
 
The ranks RDD is initially populated with a value=1.0 for all the URLs. In the next part of the code sample we shall see how this ranks RDD is recalcualted at each iteration to become eventually, after 20 iterations, the PageRank scores mentioned above.  

## Part 3: Looping and Calculating Contributions & Recalcualting Ranks
 

Here is the core of the algorithm and where 

