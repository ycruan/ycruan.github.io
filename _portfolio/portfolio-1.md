---
title: "Resource Constrained Edge Machine Learning"
excerpt: "Scalable distributed neural nets running on resource constrained IoT devices. <br/><img src='/images/projects_10701_pi.png', width='500', height='300'>"
collection: portfolio
---

Traditionally, deploying machine learning models, especially deep learning models, 
requires a significant amount of computing power. However, it is not always viable 
to use such powerful computing devices. For example, it is very expensive to rent 
a server and people may have concerns over security when sending sensible information 
across the net. In our project, we propose an **edge computing solution** to this problem, 
which fully utilizes the IoT devices around us to deploy machine learning tasks without 
sending data to the powerful server. We **propose a tree-based hierarchical classification 
model and design a pruning algorithm** for these resource-constrained IoT devices. We 
**implement our algorithm on 8 Raspberry Pis for image classification**. 
Results show that compared to using a complicated model to classify all the data, 
it achieves faster image classification through parallelization.

Technical tools: Python/Keras<br />
Download links: [full report](https://ycruan.github.io/files/10701_final_report.pdf), [poster](https://ycruan.github.io/files/10701_final_poster.pdf).