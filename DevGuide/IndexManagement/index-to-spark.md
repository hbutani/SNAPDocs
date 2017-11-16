<!-- --- title: Index to Spark Conversion -->

#### Timestamp Column that is a String Column in Spark

* We optimize the conversion, by using the Spark Column's DateTime Format to convert from an Index provided DateTime to a String
* As opposed to the default conversion pipeline of Converting the ISO String -&gt; DateTime -&gt; Spark Timestamp value -&gt; value in Spark Column's DataType



