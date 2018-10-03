---
title: "[Parallel] Parallel and Distributed Computing"
excerpt: "<i>This is a collection of parallel and distributed computing projects I did. Levels of parallelism vary from
data level SIMD to thread level OpenMP to Spark based map-reduce.
</i><br/><br/><img src='/images/projects_parallel_spark.png' height='300' width='500'>"
collection: portfolio
---

Parallel and Distributed Computing
======

This is a collection of parallel and distributed computing projects I did. Levels of parallelism vary from
data level SIMD to thread level OpenMP to Spark based map-reduce.

* Project 1: Homemade Numpy ([spec](https://ycruan.github.io/files/61c_project3_numc.htm))
  * Design and implement a slower version of numpy that supports cache-optimized parallel matrix computations.
  * Highlights: C, SIMD, OpenMP

* Project 2: Yelp Rating Prediction ([spec](https://ycruan.github.io/files/61c_project4_yelp.htm))
  * Use the MapReduce programming paradigm to parallelize a Naive Bayes classifier with a Bag of Words model in Spark to predict Yelp review ratings.
  * Highlights: C, Spark, Map Reduce

* Project 3: Parallel Huffman Coding ([report](https://ycruan.github.io/files/15853_project_report.pdf))
  * Implement a parallel algorithm to generate Huffman codes.
  * Highlights: Java multithreading
