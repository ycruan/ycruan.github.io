---
title: "[DB] Introduction to Database Systems"
excerpt: "<i>This is a collection of course projects for Berkeley CS186/286: Introduction to Database Systems (Term: 16 Fall, Grade: A).
</i><br/><br/><img src='/images/projects_286_db.png', border=10>"
collection: portfolio
---

Introduction to Database Systems
======

This is a collection of course projects for Berkeley CS186/286: Introduction to Database Systems (Term: 16 Fall, Grade: A).
* Project 1: File Management and B+ Tree ([spec](https://ycruan.github.io/files/286_project1_spec.pdf))
  * Design and implement the `table` and `schema` class that supports addition, deletion, search, update and iteration of entries;
  * A B+ tree data structure for key-value storage of pages.

* Project 2: Join Algorithms and Query Optimization ([spec](https://ycruan.github.io/files/286_project2_spec.pdf))
  * Implement three join algorithms: page nested loop join, block nested loop join and grace hash join.
  * Implement query optimization by cost estimation, single table access selection and join selection.

* Project 3: Concurrency Control ([spec](https://ycruan.github.io/files/286_project3_spec.pdf))
  * Create a lock manager for multi-thread concurrent access.
  * Detect and handle potential deadlock.

Technical tools: Java, Maven, JUnit