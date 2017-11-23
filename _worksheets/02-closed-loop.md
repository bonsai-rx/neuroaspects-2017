---
layout: worksheet
title: Closed-Loop Systems
permalink: /worksheets/closed-loop/
---

In a closed-loop system, the results of data processing feedback into the external world, establishing a relationship where the output of the system depends on the sensory input. Many behavioural experiments in neuroscience require some kind of closed-loop interaction between the subject and the experimental setup. The exercises below will show you how to use the online data processing capabilities of Bonsai to create many different kinds of closed-loop systems.

### **Exercise 1:** Measuring closed-loop latency

The easiest way to measure the latency of a closed-loop system is to use a feedback test. In this test, we feed the output of the closed-loop system directly into the same sensor that was used to generate the output in the first place, and record all the sensor measurements. For example, to perform a simple audio feedback test you can connect the microphone directly to the speakers.

* Insert an `AudioCapture` source.
* Insert an `AudioPlayback` sink.
* Insert an `AudioWriter` sink and configure its `SamplingFrequency` and `FileName` properties.
* Run the workflow and try to produce a sharp loud sound a couple of times.
* Open the recorded WAV file in Audacity or other waveform analysis software.
* Measure the round-trip time between the sound onset and its feedback.

In PC sound systems, latency depends greatly on audio acquisition and generation hardware, so expect the numbers to vary greatly across different machines. You can use video feedback in a similar way to measure latency in video based closed-loop systems.

### **Exercise 2:** Triggering a digital line based on region of interest activity

* Insert a `CameraCapture` source.
* Insert a `Crop` transform.
* Run the workflow and use the `RegionOfInterest` property to specify the desired area (hint: you can use the visual editor for an easier calibration).
* Insert a `Grayscale` and a `Threshold (Vision)` transform (or your preferred segmentation pipeline).
* Insert a `Sum (Dsp)` transform. This operator sums all the pixel values across the entire input image.
* Select the `Scalar` > `Val0` field from the right-click context menu.

**Note:** The `Sum` operator sums the pixel values across all image colour channels. However, in the case of grayscale binary images, there is only one active channel and its sum is stored in the `Val0` field.
{: .notice--info}

* Insert a `GreaterThan` transform and configure the `Value` property to an appropriate threshold.
* Insert the Arduino `DigitalOutput` sink.
* Set the `Pin` property of the `DigitalOutput` operator to 13.
* Configure the `PortName` property.
* Run the workflow and verify that entering the region of interest triggers the Arduino LED.
* **Optional:** Measure the latency of this closed-loop system.
* **Optional:** Replace the `Crop` transform by a `CropPolygon` to use non-rectangular regions.

**Note:** The `CropPolygon` operator uses the `Regions` property to define multiple, possibly non-rectangular regions. The visual editor is similar to `Crop`, where you draw a rectangular box. However, in `CropPolygon` you can move the corners of the box by right-clicking *inside* the box and dragging the cursor to the new position. You can add new points by double-clicking with the left mouse button, and delete points by double-clicking with the right mouse button. You can delete regions by pressing the `Del` key and cycle through regions by pressing the `Tab` key.
{: .notice--info}

### **Exercise 3:** Centring the video on a tracked object

* Insert a `CameraCapture` source.
* Insert a `WarpAffine` transform. This node applies affine transformations on the input defined by the `Transform` matrix.
* Externalize the `Transform` property of the `WarpAffine` operator using the right-click context menu.
* Create an `AffineTransform` source and connect it to the externalized property.
* Run the workflow and change the values of the `Translation` property while visualizing the output of `WarpAffine`. Notice that the transformation induces a shift in the input image controlled by the values in the property.
* In a new branch, create an object tracking workflow using `FindContours` and `BinaryRegionAnalysis`.
* Insert a `LargestBinaryRegion` transform to extract the largest detected object in the image.

To centre the video on the tracked object, you can start by shifting each frame by the negative position of the object, such that its new position is now (0,0). However, because the centre of the image is actually not at (0,0), you need to add an additional fixed offset to position the object at the image centre. 

* Select the `ConnectedComponent` > `Centroid` field of the largest binary region using the context menu.
* Insert a `Negate` transform. This will make the X and Y coordinates of the centroid negative.
* Insert an `Add` transform. This will add a fixed offset to the point. Configure the offset to be the position of the image centre, e.g. (320,240).

The next step is to modify the output of `AffineTransform` dynamically each time a new image shift is calculated. You can do this by using property mapping operators, which are described in more detail at [http://bonsai-rx.org/docs/property-mapping](http://bonsai-rx.org/docs/property-mapping).

* Insert an `InputMapping` operator.
* Connect the `InputMapping` to the `AffineTransform` operator.
* Open the `PropertyMappings` editor and add a new mapping to the `Translation` property.
* Run the workflow and verify that the output of `WarpAffine` is a video which is always centred on the tracked object.
* **Optional**: Insert a `Crop` transform after `WarpAffine` to select a bounded region around the object.

### **Exercise 4:** Modulating stimulus intensity based on distance to a point

* Create an object tracking workflow using `FindContours` and `BinaryRegionAnalysis`.
* Insert a `LargestBinaryRegion` transform.
* Select the `ConnectedComponent` > `Centroid` field of the largest binary region using the context menu.
* Insert a `Subtract` transform and configure the `Value` property to be some target coordinate in the image. 

The result of the `Subtract` operator will be a vector pointing from the target to the centroid of the largest object. The distance of the centroid to the target would be the length of that vector. There is currently no built-in node in Bonsai to extract this specific quantity, but you can make our own using the `Scripting` package.

* Insert an `ExpressionTransform` operator. This node allows you to write small mathematical and logical expressions to transform input values.
* Change the `Expression` property to `Math.Sqrt(X*X + Y*Y)`.

**Note:** Inside the `Expression` property you can access any field of the input by name. In this case `X` and `Y` represent the corresponding fields of the `Point2f` data type. You can check which fields are available by right-clicking the previous node. You can use all the normal arithmetical and logical operators as well as the mathematical functions available in the [`Math`](https://msdn.microsoft.com/en-us/library/system.math(v=vs.110).aspx) type. The default expression `it` means "input" and represents the input value itself.
{: .notice--info}

* Insert a `FunctionGenerator` source and set the `Frequency` property to 10.
* Insert a `ConvertScale` transform.
* Insert an `AudioPlayback` sink.

The `FunctionGenerator` periodically emits buffered waveforms with values ranging between 0 and 1. You can increase the `Scale` property of the `ConvertScale` operator in order to bring them to an audible range, which should happen for values above 100. The next step is to modulate the `Scale` property dynamically based on the distance of the object to the target.

* Externalize the `Scale` property of the `ConvertScale` operator using the right-click context menu.
* Insert a `Rescale` transform after the `ExpressionTransform` operator.
* Set the `RangeMax` property of the `Rescale` operator to 500.
* Set the `RescaleType` property to `Clamp`. This forces the output value of `Rescale` to keep between `RangeMin` and `RangeMax`.
* Finally, configure the `Min` and `Max` properties of `Rescale` to set the allowed input range.

**Note:** You can specify inverse relationships if you set the *maximum* input value to the `Min` property, and the *minimum* input value to the `Max` property. In this case, a small distance will generate a large output, and a large distance will produce a small output.
{: .notice--info}

* Connect the `Rescale` operator to the externalized `Scale` property.
* Run the workflow and verify that stimulus intensity is modulated by the distance of the object to the target point.

### **Exercise 5:** Triggering a digital line based on distance between objects

* Create an object tracking workflow using `FindContours` and `BinaryRegionAnalysis`.
* Insert a `SortBinaryRegions` transform. This operator will sort the list of objects by area, in order of largest to smallest.

To calculate the distance between the two largest objects in every frame you will need to take into account some special cases. Specifically, there is the possibility that no object is detected, or that the two objects may be touching each other and will be detected as a single object. You can develop a new operator in order to perform this specific calculation.

* Insert a `PythonTransform` operator. Change the `Script` property to the following code:

```python
from math import sqrt

@returns(float)
def process(value):

  # no objects were detected
  if value.Count == 0:
    return float.NaN

  # only one object was detected, assume objects are touching
  elif value.Count == 1:
    return 0

  # two or more objects were detected, compute distance
  else:
    # d: displacement between two largest objects
    d = value[0].Centroid - value[1].Centroid
    return sqrt(d.X * d.X + d.Y * d.Y)
```

* Insert a `LessThan` transform and configure the `Value` property to an appropriate threshold.
* Connect the boolean output to Arduino pin 13 using a `DigitalOutput` sink.
* Run the workflow and verify that the Arduino LED is triggered when the two objects are close together.