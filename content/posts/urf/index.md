---
title: "Detecting Similar Malicious Payloads over Encrypted Data"
date: 2023-05-05
description: Detecting Similar Malicious Payloads over Encrypted Data
menu:
  sidebar:
    name: Detecting Malicious PDFs
    identifier: urf
    weight: 10
# tags: []
# categories: []
---

I worked on this project at the Cyptography, Security and Privacy Lab at the University of Waterloo under supervision of Professors Florian Kerschbaum and Ehsan Amjadian. With the goal of creating a shared anomaly detection system over confidential PDF files, I designed a custom PyTorch autoencoder to convert features from PDF files to a format suitable for encryption. Then, I designed a clustering algorithm over the encrypted information, which identifies classes of similar PDFs based on distances between the encrypted feature vectors.

We are currently working on publishing the work as a conference paper. The current draft of the publication can be found [here](urf.pdf).