---
layout: post
title: Flood mapping in Madagascar
---

Tropical cyclone Enawo hit northeastern Madagascar on 03/07/17 with winds of 145 mph.
It is reportedly the [third strongest tropical cyclone on record to strike the island](https://www.wunderground.com/blog/JeffMasters/category-4-tropical-cyclone-enawo-hits-madagascar).
As of 03/13, at least [100000 people have been directly affected by the cyclone](http://reliefweb.int/disaster/tc-2017-000023-mdg) and at least 50 people have been killed.
DigitalGlobe has [released imagery](http://blog.digitalglobe.com/industry/imagery-released-for-cyclone-enawo-to-support-mapping-activities/?utm_source=linkedin&utm_campaign=MadagascarCyclone&utm_medium=social) for mapping activities in support of the humanitarian relief efforts.

Recently, we implemented an algorithm which uses 8-band, atmospherically compensated imagery in order to detect **impure water**, and added the new capability to our proprietary image analysis and processing software Protogen. Flooding events
involve an overflowing of large amounts of water over what normally constitutes dry land, consisting of soil and materials used in built-up areas. The resulting flood water is 'dirty' and as such does not have the spectral signature of normal, 'clean' water. The algorithm takes into account the characteristics of impure water to produce a mask, i.e., a binary image where white corresponds to impure water and black to the background.

We ran the new algorithm on GBDX, on a small collection of images captured by GeoEye-1 a few days after the cyclone made landfall over Maroantsetra and Ambohitralanana.

| **location** | **catalog id** | **sensor** | **date** |
| :-------- |:-------- | :-------- | :-------- |
| Maroantsetra | 1050010008989C00 | GeoEye-1 | 03/11/17 |
| Ambohitralanana coast | 1050010008989A00 | GeoEye-1 | 03/11/17 |
| Ambohitralanana inland | 1050010008989B00 | GeoEye-1 | 03/11/17 |

*Imagery specifications.*

You can explore the mask, the post-cyclone image and a WorldView-3 pre-cyclone image captured on 09/12/15 (1040010011C75800) for Maroantsetra in the map below. Toggle between layers by clicking on the labels on the right.

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/enawo-madagascar/maroantsetra.html"></iframe>
{% endraw %}

*Pre- and post- cyclone images and flood water mask over Maroantsetra.*
<br><br>

A quick preview of the results reveals the extensive degree of flooding in this area, and that the roads between Maroantsetra and Andranofotsy are most likely inaccessible due to water or other deposited materials.    

The images and corresponding flood water masks for the coast and inland of Ambohitralanana are shown below.
The masks illustrate the devastating impact of the cyclone in this area as well.

<br>
{% raw %}
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width="800" height="800" src="../pages/enawo-madagascar/ambohitralanana.html"></iframe>
{% endraw %}

*Post-cyclone coast and inland images and corresponding flood water masks over Ambohitralanana.*
<br><br>

Note that the impure water mask includes muddy water, all water-saturated soil (mud), as well as impure water concentrations over man-made materials such as roads, paved surfaces and roofs. The mask does not include water which rests under dense vegetation or cloud cover.

We are currently looking into evaluating the performance of the algorithm in a variety of flooding incidents which have occurred in the past. The speed of the algorithm and the fact that it is **unsupervised** render it an attractive mapping solution for crisis-response situations: it literally takes minutes to process an entire strip on an r4.2xlarge instance.

Stay tuned for updates on improvements to the algorithm and combining the masks with other data sets in order to assess damage from flooding.
