## Tutorial 101: PageRank Example in Spark 2.0
### Understanding the Algorithm & Spark Code Implementation
 
  The Apache Spark PageRank is a good example for learning how to use Spark. The sample program computes the PageRank of URLs from an input file which should be in format of:
   url_1   url_4
   url_2   url_1
   url_3   url_2
   url_3   url_1
   url_4   url_3
   url_4   url_1  

where each URL and their neighbors are separated by space(s). The above input data can be represented by the following graph. For example, URL_3 references URL_1 & URL_2 while it is referenced by URL_4.  

<img src="/images/img-1.jpg" width="447" height="383">
