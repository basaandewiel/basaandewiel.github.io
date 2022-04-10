---
layout: post
title: TinyML - Machine learning on embedded devices, possibilities
---
Used sources: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8271867/

Machine learning is very interesting and also can have several applications on embedded devices. 

Appications usually have to do with all kind of different sensors on these embedded devices. For instance categorising movement of objects during transport - are they handled with care? Is the box (on which the embedded device is present never flipped/turned upside down?
This are only a few examples; the possibilities are endless.

But how can we implement applications with machine learning on embedded devices, and what scenario should be applied in which situation?
* transfer the signals to a cloud service that uses machine learning and proces the results (requires reliable internet connection with sufficient bandwidth, and latency should nog be a problem)
* train a model on a host/in the cloud, and then migrate the trained model to the embedded device? (low latency and embedded device does not need internet connection)
* use special AI-hardware, low latency and no internet connection is necessary
* etc.

# Train model in cloud and transfer trained model to embedded device
There are several cloud services that can be used, of instance:
## Edgeimpulse
The cloud service edgeimpulse.com supports several embedded devices (among them the popular Arduino nano 33 BLE sense, but also the Nordic Thingy:91). 
With edgeimpulse you can train a model with their cloud service, and export for instance a portable C++ library of the trained model, that can be used in programming a application for the embedded device. 
They have a very good tutorial on this.
A sample appication combines signal processing (to extract features), neural networks (for classification) and clustering algorithms(for anomaly detection) to classify the output of a movement sensor.


