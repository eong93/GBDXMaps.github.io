---
layout: post
title: Building detection on mosaics using deep learning
---

Picture being able to select an arbitrary region of the world in your browser and within it rapidly locate all the objects of interest. The advancement of deep learning combined with easy access of high resolution satellite imagery enabled by platforms such as [GBDX](https://platform.digitalglobe.com/) are making accurate object detection at scale an attainable goal. However, the application of deep learning on satellite imagery is still in its infancy and many questions are open with regards to its efficacy at a global scale.

In a [recent experiment](http://gbdxstories.digitalglobe.com/building-detection/), we used the VGG-16 convolutional neural network (CNN) to detect settlements in north-eastern Nigeria. We trained the CNN using a few thousand labeled chips from a small collection of WorldView-2 strips. We then deployed the model on these same strips, as well as a different set of strips that the model 'had never seen'. We discovered that the model often performed much worse on the latter set, yielding many false positives. Simply put, the model could not **generalize** based on the data that it was trained on. The experiment raised many questions: How much training data is enough? How does terrain variability affect the performance of a model? How about differences in sensor, sun angle and capture time? Which type of image preprocessing is more conducive to deep learning?       

{:refdef: style="text-align: center;"}
![old-results.png]({{ site.baseurl }}/images/building-detection-large-scale/old-results.png){:height="600px"}  
{: refdef}
*Variability in accuracy when deploying VGG-16 on an image that it was trained on (left) vs an image that it was not trained on (right).*

## Setup

In an attempt to address some of these questions, we decided to conduct a larger experiment in the same part of the world. The region of interest is a square at the border of Nigeria and Cameroon, spanning an area just short of 20000 km2. We picked out 31 recent images captured by WorldView-2, WorldView-3 and GeoEye-1, and constructed a mosaic using our proprietary **Flexible Large Area Mosaic Engine (FLAME)** technology.

FLAME solves a rather complicated problem: it turns a hodgepodge of images that have been individually orthorectified, atmospherically compensated and pansharpened with our [image preprocessor](http://gbdxdocs.digitalglobe.com/docs/advanced-image-preprocessor), into a seamless mosaic. It is designed and implemented to take advantage of parallel computation, such that it can process millions of km2 in a matter of hours. FLAME constructs the mosaic in two main steps:

+ it adjusts the pixel intensities in the R,G,B bands to color-match a global base layer, a procedure that is called **Base Layer Matching (BLM)**;
+ it intelligently weaves the images across optimized boundaries called cutlines in order to maximize blending.

The end result is a collection of color-consistent tif image tiles that comprise the mosaic, delivered in an S3 bucket.
You can explore the mosaic in the map below or, for a full page view, you can click [here]({{ site.baseurl }}/pages/building-detection-large-scale/mosaic.html). This visualization was created by setting up a Web Map Service (WMS) with [MapServer](http://mapserver.org/).

{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/building-detection-large-scale/mosaic.html"></iframe>
{% endraw %}  

We collected a reference data set by creating a giant grid of step size 125m over the mosaic, and having our crowd label 845000 chips in this grid as 'Buildings' and 'No Buildings'. The number of chips with buildings was just short of 13000, i.e., about 1.5% of the reference data set, which illustrates how sparsely populated this region is.

{:refdef: style="text-align: center;"}
![training-data.png]({{ site.baseurl }}/images/building-detection-large-scale/training-data.png){:height="500px"}  
{: refdef}
*5000 samples from the reference data set with 'Buildings' and 'No Buildings' in equal parts.*


## Training and deploying the CNN

Here is the workflow that we implemented for training and deploying the CNN on the mosaic.

![workflow.png]({{ site.baseurl }}/images/building-detection-large-scale/workflow.png)  

The inputs to the workflow are:
+ the mosaic tiles;
+ train.geojson, which includes the geometry and class name of each training chip;
+ a collection of target.geojson files, each including the geometries of the chips covering part of the grid.

The task [chip-from-vrt](https://github.com/PlatformStories/chip-from-vrt) extracts chips from the mosaic for training and deploying. It does this by creating a [vrt file](http://www.gdal.org/gdal_vrttut.html) which specifies the full path to the S3 location of each tile in the mosaic using the GDAL virtual filesystem  ```/vsis3/```. It then extracts the chips defined by the geometries in the input geojson by calling [gdal_translate](http://www.gdal.org/gdal_translate.html) on the vrt, and saves then to a user-defined bucket.
This process allows the task to pull chips remotely from the mosaic tiles without actually mounting the tiles onto the GBDX worker executing the task.

![chip-from-vrt.png]({{ site.baseurl }}/images/building-detection-large-scale/chip-from-vrt.png)  
*chip-from-vrt extracts chips from the collection of image tiles comprising the mosaic, by treating it as a single image.*

The training chips are the input to [train-cnn-chip-classifier](https://github.com/PlatformStories/train-cnn-chip-classifier)
which produces a Keras model. For each part of the grid, the corresponding target.geojson and the model are passed to a separate [deploy-cnn-chip-classifier](https://github.com/PlatformStories/deploy-cnn-chip-classifier) task which produces classified.geojson. classified.geojson contains all the chip geometries in target.geojson, each appended with the model decision and a confidence score.

You can run the entire workflow with [gbdxtools](https://github.com/digitalglobe/gbdxtools) in [this Jupyter notebook](https://github.com/PlatformStories/notebooks/blob/master/Building%20detection%20on%20mosaics%20using%20deep%20learning.ipynb).

## Experiments

We ran a series of experiments to determine the effect of different factors on the precision/recall performance of the model. The performance was evaluated using the reference dataset that we collected previously.

For each data point in the plots that follow, we trained the CNN 10 different times and computed the mean and standard deviation of the precision in order to derive a confidence interval. Unless mentioned otherwise, the training set was 5000 labeled chips selected randomly from the reference data set. Each chip was downsampled from 260x260 to 150x150 in order to fit the CNN architecture into memory. For each training cycle, we trained the CNN on balanced classes for 50 epochs, and selected the model that resulted in the lowest validation loss.

Training with train-cnn-chip-classifier on 5000 chips took approximately 9 hours on a g2.2xlarge instance. For the deployment phase, we divided the grid into 13 roughly equal parts, each containing about 100000 chips, and ran deploy-cnn-chip-classifier on each part in parallel on 13 different g2.2xlarge instances. The deployment phase took about 4 hours, i.e., 0.72 sec/km2.  

### Dynamic Range Adjustment (DRA)

Dynamic Range Adjustment (DRA) converts pixel values of an orthorectified, atmospherically compensated, pansharpened image from 16 bits to 8 bits, so that the image is viewable by the human eye on a computer screen. DRA is available as an option on the GBDX [image preprocessor](http://gbdxdocs.digitalglobe.com/docs/advanced-image-preprocessor). In order to construct a FLAME mosaic, DRA is performed with BLM, in order to achieve color consistency across the mosaic.  

Our goal was to assess the effect of DRA on the model accuracy. Using the original orthorectified, atmospherically compensated and pansharpened 16-bit imagery, and the FLAME cutlines, we created two **pseudo-mosaics** by:

+ clipping the lowest 0.5% and highest 0.05% pixel intensities for each image tile **individually** and setting them to 0 and 255, respectively, and stitching these tiles;

+ not performing any DRA at all, i.e., directly stitching the 16-bit tiles.

Clipping the pixel intensities to create 8-bit imagery is a naive form of DRA; we refer to it as CLIP. The CLIP pseudo-mosaic is compared to the actual mosaic below. Not surprisingly, the colors are different across tiles.

![mosaics.png]({{ site.baseurl }}/images/building-detection-large-scale/mosaics.png)  
*CLIP vs BLM. CLIP is performed on a per-tile basis. The FLAME cutlines outlining the tile boundaries are shown in green. BLM is the result of adjusting the colors to match an underlying global base layer.*

16-bit imagery can not be displayed on 8-bit monitors so we can't really view the ACOMP pseudo-mosaic unless DRA is applied. Below, we've plotted the pixel intensity histograms of a single chip for BLM, CLIP and ACOMP. The histograms follow the same pattern. Note that the horizontal axis has a much larger range for ACOMP than for CLIP and BLM as intensity takes values from 0 to 65535.

![histograms.png]({{ site.baseurl }}/images/building-detection-large-scale/histograms.png)
*Pixel intensity histograms for a BLM, CLIP and ACOMP chip. The colors of the ACOMP chip are not 'true'; the chip has been DRA'd in QGIS so that it can be displayed on the monitor.*

We trained and deployed the CNN on the BLM, CLIP and ACOMP mosaics using 5000 and 7500 training samples selected randomly from the reference data set. The results are shown below.

![clip-blm-acomp.png]({{ site.baseurl }}/images/building-detection-large-scale/clip-blm-acomp.png)
*The performance on the BLM and CLIP mosaics is close to identical. Using the ACOMP mosaic incurs a performance loss which decreases with training sample size.*

There is no notable difference in performance for the BLM and CLIP mosaics. Our interpretation of this result is that using training data across the entire mosaic enables the model to understand the differences in color across the CLIP tiles. In contrast, using the ACOMP mosaic results in a rather surprising performance penalty, and wider confidence intervals than the other two cases. This result requires further investigation! Our guess at the moment is that the model requires more time and/or more data to learn, given that 16-bit imagery contains more information than its DRA'd counterpart.        


### Training data spatial distribution

In order to assess the effect of the training data spatial distribution on the model accuracy, we restricted training data selection to
a small part of the mosaic.  

{:refdef: style="text-align: center;"}
![spatial-dist.png]({{ site.baseurl }}/images/building-detection-large-scale/spatial-res.png){:height="400px"}  
{:refdef}
*5000 training samples selected from a small part of the mosaic.*

The PR curves for BLM and CLIP are shown in the following figure.

{:refdef: style="text-align: center;"}
![spatial-res-blm.png]({{ site.baseurl }}/images/building-detection-large-scale/spatial-res-blm.png){:height="400px"}  
{:refdef}
*The model can generalize better on the BLM mosaic.*

The BLM curve is (mostly) to the right of the CLIP curve. The results confirm our intuition that the model can generalize
better on the BLM mosaic since the colors are more consistent compared to the CLIP mosaic.

We also compared the model trained on the restricted training set to the model trained on the mosaic-wide training set on the CLIP mosaic.

![spatial-res-plots.png]({{ site.baseurl }}/images/building-detection-large-scale/spatial-res-plots.png)  
*Lack of mosaic-wide training data leads to decreased accuracy and more dollars spent on result validation.*

The PR curves on the left show that the performance penalty of collecting training data from one location only is significant.
On the right, we have plotted the approximate cost of validating the building detections of both models with a crowdsourcing campaign. Prices were calculated assuming 3 votes for each detection, at $0.02 per vote. Poor precision is manifested in more dollars spent on removing false positives, e.g, at a 90% recall, the price doubles for the poorly trained model.

### Compression

JPEG compression is typically 10:1, and is therefore an attractive solution for storing big image files.
Since it is lossy, we wanted to assess its impact on the performance.
We trained and deployed the CNN on the CLIP mosaic using both JPEG-compressed and uncompressed tiles. The results are shown below.

{:refdef: style="text-align: center;"}
![compression-clip.png]({{ site.baseurl }}/images/building-detection-large-scale/compression-clip.png){:height="400px"}  
{:refdef}
*JPEG compression does not incur an accuracy penalty.*

JPEG compression does not seem to affect the performance. On the contrary, it leads to tighter confidence intervals than uncompressed imagery. In other words, there is considerably larger variance in accuracy for the models we trained on the uncompressed imagery.


## Discussion

In summary, we found that:

+ DRA increases accuracy. This is rather surprising since DRA is an operation which reduces the image information content. Along this line of thought, we suspect that more training time and/or more training data are required to harness the full potential of 16-bit imagery.

+ The spatial distribution of the training data has a crucial impact on accuracy. If training data collection has to be restricted to one area using a mosaic can help a model generalize to other areas.

+ JPEG compression has no impact on the performance. This is good news; compressed imagery takes up much less space than uncompressed imagery and the cost-savings could be very significant at a global scale.

Keep in mind that these observations were derived for a particular use case in a specific part of the world, and are not meant to be used as guidelines. More investigation for diverse use cases and geographical locations is required to reach meaningful conclusions.

Perhaps more important than the accuracy viewpoint is the fact that the framework presented here can be used to achieve tremendous search area reduction when looking for sparse settlements over large areas. For a model trained on 5000 samples, the precision at 90% recall is about 30%. That sounds quite bad, yet, in reality, what this means is that the model can single out 27300 locations out of 845000 candidates; a 96.8% search area reduction. It is much faster and economic to run a crowdsourcing campaign to remove the false positives from 27300 detections, than to collect labels for 850000 chips.  

We deployed our best model on the entire mosaic and kept the detections with confidence higher than 97.5%.
You can explore the results, shown in green, below (full page view [here]({{ site.baseurl }}/pages/building-detection-large-scale/deploy-results.html)). To make this map, we [uploaded the geojson with all the detections to Mapbox](https://github.com/platformstories/upload-to-mapbox) and used the [Mapbox GL Javascript library](https://www.mapbox.com/mapbox-gl-js/api/) to reference the vector tile set.

{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="https://kostasthebarbarian.github.io/pages/building-detection-large-scale/deploy-results.html"></iframe>
{% endraw %}

For more information on machine learning research at DigitalGlobe and on GBDX in general, [get in touch](mailto:kostas.stamatiou@digitalglobe.com).
