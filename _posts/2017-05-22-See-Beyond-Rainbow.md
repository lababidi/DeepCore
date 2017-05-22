---
layout: post
title: Seeing Beyond the Rainbow
author: Mahmoud Lababidi
desc: Image based ConvNets go beyond RGB to 8-band multispectral sources to optimize for identification of land use and objects. We demonstrate that the Conv layers filter out patterns on the near infrared multispectral bands in satellite imagery.
keywords: multispectral, image segmentation, infrared, RGB
published: True
---

Image based ConvNets go beyond RGB to 8-band multispectral sources to optimize for identification of land use and objects. 
We demonstrate that the Conv layers filter out patterns on the near infrared multispectral bands in satellite imagery.

# Seeing Beyond the Rainbow
Neural Network based image recognition, identification, classification, detection, and segmentation have been very successful 
in their tasks in the traditional RGB image bands that are hosted in JPEG, PNG, RAW, and regular TIFFs.
While the networks have been growing and getting more complex to gain performance improvements, 
one aspect that's left out is starting with "better data". In this case, it would be using data that contains more information.
If you've never experimented with multispectral data, give this [tool a whirl](http://www.geocarto.com.hk/edu/PJ-BCMBWV3G/main_BCW3.html).
It allows you to visualize the different bands in an 8-band multispectral image. 
Since computer screens can only show three channels (RGB), the tool allows you to pick of the 3 bands and map them to one of RGB. 
Two examples below are natural color using bands 5-3-1 as RGB and infrared false color using bands 8-3-1 as RGB. 
<p align="center">
<img src="/assets/images/531.jpg" width="40%" />
    <figcaption>531 - True Color</figcaption>

<img src="/assets/images/831.jpg" width="40%" />
    <figcaption>831 - nIR False Color</figcaption>

</p>

As you can see, the 831 image shows a strong red color for the trees. 
Clearly, this is more information than the human eye sees, so how can a ConvNet leverage this information?

If we look at an architecture of a ConvNet, we can see that the first Conv layer is where the image gets passed in.
All three channels (RGB) have a convolutional filter applied to them simultaneously. 
We must perform some surgery on our network to handle 8 channels instead of just 3. 
After this is done, it's not clear whether the filter actually pulls out features from the images from certain bands.
One way to verify is to actually inspect the output of the filter for each channel on an image.
Below are a few images that are output from a convolution with some of the filters of Conv layer 1.
You can see that some of the texture that shows up in channels 7,8 (near IR) do not show up in the other channel outputs.
