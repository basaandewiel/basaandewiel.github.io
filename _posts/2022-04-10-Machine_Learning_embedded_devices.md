---
layout: post
title: TinyML - Machine learning on embedded devices, possibilities
---

Machine learning is very interesting and also can have several applications on embedded devices. 

Appications usually have to do with all kind of different sensors on these embedded devices. For instance 
* categorising movement of objects during transport - are they handled with care? Is the box (on which the embedded device is present never flipped/turned upside down?
* monitor animal health by using various sensors (movement, temperature)
* activity recognition is sport trackers or mobile phones.
This are only a few examples; the possibilities are endless.

But how can we implement applications with machine learning on embedded devices, and what scenario should be applied in which situation?
* transfer the signals to a cloud service that uses machine learning and proces the results (requires reliable internet connection with sufficient bandwidth, and latency should nog be a problem)
* train a model on a host/in the cloud, and then migrate the trained model to the embedded device? (low latency and embedded device does not need internet connection)
* use special AI-hardware, low latency and no internet connection is necessary
* etc.

# Train model on HOST and transfer trained model to embedded device
TinyML is a machine learning technique that integrates compressed and optimized machine learning to suit very low-power MCUs
There are different TimyML frameworks, for instance:
* TensorFlow Lite (ARM Cortex-M, developed by Google)
* CMSIS-NN (ARM Cortex-M, developed by AMR)
* TinyMLgen (ARM Cortex-M and ESP32)




# Train model in CLOUD and transfer trained model to embedded device
There are several cloud services that can be used, of instance:
## Edgeimpulse
The cloud service edgeimpulse.com supports several embedded devices (among them the popular Arduino nano 33 BLE sense, but also the Nordic Thingy:91). 
With edgeimpulse you can train a model with their cloud service, and export for instance a portable C++ library of the trained model, that can be used in programming a application for the embedded device. 
They have a very good tutorial on this.
A sample appication combines signal processing (to extract features), neural networks (for classification) and clustering algorithms(for anomaly detection) to classify the output of a movement sensor.

I tried to get this running with Zephyr on an Arduino nano 33 ble sense (using the input of https://docs.edgeimpulse.com/docs/tutorials/running-your-impulse-locally/running-your-impulse-locally-zephyr, but did not succeed yet to build this succesfully. 



