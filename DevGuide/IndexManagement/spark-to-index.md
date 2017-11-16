<!-- --- title: Spark to Index Conversion -->

#### Conversion of Strings

Conversion of Spark UTF8 Strings to Java Strings is computationally and Memory intensive. Since all Dimensions are maintained as Strings, this is a very common operation during Indexing. 

1. A Thread-Local ByteBuffer is used to copy UTF8 bytes into it, instead of the default behavior of creating new byte buffers.
2. TBD: avoid creation of `char[]` during String creation, but this requires rewriting Java String Decoding logic.

#### Timestamp Column that is a String Column in Spark

* We optimize the conversion, by using the Spark Column's DateTime Format to convert to a DateTime and passing it to the Index.
* As opposed to the default conversion pipeline of Converting the Spark Value -&gt; to a Spark Timestamp -&gt; converting to a DateTime -&gt; formatting as an ISO string



