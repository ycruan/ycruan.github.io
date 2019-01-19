---
title: "[DL] Introduction to Deep Learning"
excerpt: "<i>This is a collection of homework and course projects for CMU 11785: Introduction to Deep Learning (Term: 18 Fall).
</i><br/><br/><img src='/images/projects_785_dl.png' height='300' width='500'>"
collection: portfolio
---

Introduction to Deep Learning
======

This is a collection of homework and course projects for CMU 11785: Introduction to Deep Learning (Term: 18 Fall).
* Homework 1: Frame Level Classification of Speech ([spec](https://www.kaggle.com/c/11-785hw1p2-f18))
  * Build from scratch a MLP class supporting `backprob`, `batchnorm`, `softmax` and `momentum`, using only Numpy.
  * Identify the phoneme state label for WSJ utterance frames using MLP.

* Homework 2: Speaker Veriﬁcation via Convolutional Neural Networks ([spec](https://ycruan.github.io/files/785_hw2_spec.pdf))
  * Determining whether two speech segments were uttered by the same speaker.
  * Extract speaker embeddings from utterances using `CNN`, followed by dense layers to train with` N-way classification`.

* Homework 3: Seq2Seq Phonemes Prediction ([spec](https://www.kaggle.com/c/Fall-11-785-homework-3-part-2))
  * Build a `Seq2Seq` model for phonemes prediction of unaligned utterance data.
  * Incorporate into the model the `CTC loss` and `beam search` decoder.

* Homework 4: Attention-based End-to-End Speech-to-Text Deep Neural Network ([spec](https://ycruan.github.io/files/785_hw4_spec.pdf))
  * Implement the character based Listen, Attend and Spell ([LAS](https://arxiv.org/abs/1508.01211)) model to translate utterances to corresponding text transcripts.
  * The listener consists of a `pyramidal bi-LSTM` network that produce attention keys and values. The decoder is an `LSTM` that 
    yields sequential outputs.

* Course Project: Dynamic fleet management with deep reinforcement learning ([report](https://ycruan.github.io/files/785_proj_report.pdf))
  * Apply online deep reinforcement learning models for dynamic taxi dispatch in NYC.
  * Implement a CNN-based `deep Q network` and a refined `diffusion-CNN` model.

Technical tools: Python, PyTorch, TensorFlow, AWS