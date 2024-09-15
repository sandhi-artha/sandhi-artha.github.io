---
title: "Self Organizing Map for Analysis of Geotechnical Data"
description: "SOM is one the biology inspired algorithms that can reduce the high dimensional data into a 2D map"
date: 2021-04-15T08:00:00
draft: False
tags: ["projects"]
---

# Summary

Self-organizing maps are trained in an unsupervised way aimed at dimensionality reduction. It utilizes a neighborhood function that preserves the topological properties of the input space. The models are automatically organized into a meaningful two-dimensional order in which similar models are closer to each other in the grid than the more dissimilar ones. Its computation is a nonparametric, recursive regression process [1].

We'll use SOM to classify soil type at certain locations and depth, based on measurement from a geotechnical probes. The unit used is called a Dilatometer of Marchetti Test (DMT), which is non-invasive. The probe with sensors attached is inserted into the ground, and various physical peroperties were measured at predefined depths [2].


# Training

When loading the data, all measurements are normalized (mean of 0 and std of 1). We'll use the [miniSom library](https://github.com/JustGlowing/minisom) for SOM implementation. Possible hyperparameter tunings are:
- **som_shape**: size of the lattice/map, m x n, the product is the num of nodes
- **sigma**: radius of neighbors. default=1.0
- **l_rate**: how much weights are adjusted each `epoch`
- **neighbor**: function to weight the neighborhood: `gaussian`, `mexican_hat`, `bubble`, `triangle`
- **topo**: the lattice topology structure: `rectangular` or `hexagonal`
- **epoch**: num of iteration
- **weight_init**: `pca` (populate weight to span 1st two principal componenets, faster convergence) or `random` (pick random samples from data)

A review of these parameters can be found in my [Colab notebook](https://colab.research.google.com/drive/1pZp0mUOpYqyrxf5iZEGZlQzu74Un3TYr?usp=sharing)

# Results
- The y-axis is depth, while the x-axis is the sensor value.
- Each depth observation belongs to certain cluster, which represents a soil type.
- The 2 sensors A and B were features used to determine clusters based on the topologies.

![som results](/projects/self-organizing-map/som_results.png)


# References
[1] T. Kohonen, “The self-organizing map,” Neurocomputing, vol. 21, no. 1–3, pp. 1–6, Nov. 1998, doi: 10.1016/S0925-2312(98)00030-7.

[2] Bilski, Piotr & Rabarijoely, Simon. (2009). Automated soil categorization using CPT and DMT investigations. 
