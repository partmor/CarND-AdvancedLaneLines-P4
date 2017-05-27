# **CarND: Advanced Lane Tracking**  [![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)
[//]: # (Image References)

[find_corners]: ./examples/find_corners.png
[test_undistort]: ./examples/test_undistort.png
[original_img]: ./examples/original_img.png
[frame_undistort]: ./examples/frame_undistort.png
[yuv]: ./examples/yuv.png
[luv]: ./examples/luv.png
[lab]: ./examples/lab.png
[L_thresh]: ./examples/L_thresh.png
[B_thresh]: ./examples/B_thresh.png
[L_sobelx]: ./examples/L_sobelx.png
[combined_thresh]: ./examples/combined_thresh.png
[src_dst_definition]: ./examples/src_dst_definition.png
[src_dst_masked]: ./examples/src_dst_masked.png
[sliding_window]: ./examples/sliding_window.png
[near_range]: ./examples/near_range.png
[pipeline_output]: ./examples/pipeline_output.png

The goal of this project is to build a software pipeline to identify the lane boundaries in a video stream generated by a front-facing camera mounted on a vehicle.

## Camera calibration

Calibrating for distortion is performed by taking pictures of known shapes: for instance a **chessboard**. With multiple pictures of a chessboard on a flat surface, it is possible to establish the transformation that maps the position of the corners in the distorted images and their theoretical known values. The parameters of this transformation are the **camera's distortion parameters**.

*[Cells 2 - 7 of the project notebook]*

The chessboard used here has 6 x 9 internal corners. The object points - theoretical real (x,y,z) coordinates in the world - are defined assuming the chessboard is fixed on the (x,y) plane with z = 0:
```python
# prepare object points, like (0,0,0), (1,0,0), (2,0,0) ....,(6,5,0)
objp = np.zeros((6*9,3), np.float32)
objp[:,:2] = np.mgrid[0:9,0:6].T.reshape(-1,2)
```
The calibration images are loaded, converted to grayscale and fed to the `cv2.findChessboardCorners()` method, that detects the internal vertices of the chessboard, and returns their (x,y) position in the image plane.

![find_corners]

Each group of image plane coordinates is paired with their theoretical position, which happens to always be the same `objp` since we are evaluating the same real-world object.

The collection of image points - object points pairs are passed to the `cv2.calibrateCamera()` method that returns the camera's distortion parameters.

Given the latter parameters `cv2.undistort()` allows to undistort any image taken by the camera. Applying this transform on the test image:

![test_undistort]

## Pipeline

Now that the camera has been calibrated, it is possible to start setting up the image pipeline for lane detection. The example worked through this pipeline will be the following image:

![original_img]

This image was chosen since it is a fairly complicated example, due to the irregular shading of the road, presence of white and yellow lines, and the side fence.

### Distortion correction

*[Cells 8 - 11 of the project notebook]*

The distortion correction step has been encapsulated in `aux_fun.undistort()`. This method applied to the test frame outputs:

![frame_undistort]

### Thresholding

*[Cells 12 - 22 of the project notebook]*

The objective is to find colorspaces that are robust *picking* white and yellow lines under changing lighting conditions and tarmac integrity.

The **Lab** colorspace behaved very well in this mission:

![lab]

+ A high-pass filter on the **L channel** picks white lane lines.

![L_thresh]

+ A high-pass filter on the **b channel** helps pick yellow lines, but cannot detect them in low-light conditions.

![B_thresh]

+ Absolute **sobel** thresholding in the **X direction** to the **L channel** picks very well edges of white and yellow lines where the basic color channels seem to fail.

![L_sobelx]

A combined filter using the above three individual filters yields the following mask:

![combined_thresh]

The combined thresholding pipeline has been encapsulated in `thresholding_pipeline()`, with the usage of `aux_fun.abs_sobel_thresh()` for the Sobel filter, and `aux_fun.color_channel_thresh()` for the color channel filters.

It is worth pointing out that the YUV channel provides very similar results to the Lab: 

![yuv]

In particular, the roles of L and b in the described pipeline could be exchanged by Y and V respectively.

### Perspective transform

*[Cells 23 - 30 of the project notebook]*

A perspective transformation of the image requires a prior definition of the so-called *source* and *destination* points, in order to compute the transformation matrix.

For this project, the source and destination points were hardcoded, using an image with a theoretically straight lane to perform the manual *calibration* of the vertices.

Imposing the following mapping:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 203, 720      | 320, 720        | 
| 585, 460      | 320, 0      |
| 700, 460     | 960, 0      |
| 1107, 720      | 960, 720        |

```python
src_vertices = np.float32(
    [
        [img_size[1] / 6 - 10, img_size[0]],
        [img_size[1] / 2 - 55, img_size[0] / 2 + 100],
        [img_size[1] / 2 + 60, img_size[0] / 2 + 100],
        [img_size[1] * 5 / 6 + 40, img_size[0]]
    ]
)

dst_vertices = np.float32(
    [
        [img_size[1] / 4, img_size[0]],
        [img_size[1] / 4, 0],
        [img_size[1] * 3 / 4, 0],
        [img_size[1] * 3 / 4, img_size[0]]
    ]
)
```
yielded the following result on the calibration image:

![src_dst_definition]

Back to the example image used to illustrate the pipeline, the application of the warp transform provides:

![src_dst_masked]

The *birds-eye* transformation is encapsulated in `aux_fun.warp_image()`.

### Lane detection

*[Cells 31 - 37 of the project notebook]*

The next step in the pipeline is to identify explicitly the pixels in the warped binary mask as belonging to the left or right lane.

The lanes are scanned using a **vertical-sliding window**. Consider a static window of the image that covers a vertical section of the image and spans along the whole horizontal axis. In this window, a histogram of the binary image along the horizontal direction will present peaks in aeras with a high density of highlighted pixels. It is reasonable to affirm that the 2 largest peaks are due to the lanes.

A bounding box with a given width (the height is equal to the window's) is defined around the peaks, and all the pixels that lie inside it are catalogued as belonging to the left or right lane, depending if the box is located on the left or right half of the image.

This process is repeted for several subsequent and non-overlapping vertical windows, ending up with a collection of pixels tagged as left/right.

With the coordinates of the pixels in the rectfied image, a second degree polynomial can be fit to each lane.

This methodology is encapsulated in `aux_fun.find_lanes_sliding_window_hist()` and visualized as follows:

![sliding_window]

The yellow line in the image above represents the fitted lane line.

Regarding the utility of this step when processing a video stream, once the lanes have been successfully located in previous frames, there is no need to start a new blind search with the sliding window approach. Instead, the lanes for the new incoming frame can be searched within a small range around the previous fitted detections. This approach is encapsulated in `aux_fun.find_lanes_near_previous()` and builds a search area as the following:

![near_range]

### Curvature and offset calculation

*[Cells 38 - 39 of the project notebook]*

The **radius of curvature** of a line is defined in terms of its first and second derivatives. Hence, the curvature of the fitted lines in the warped image can be immediately calculated since the  expression *x = f(y)* is known for both.

However, this radius is calculated based on pixel values in the warped space. The U.S. regulations for lane dimensions allow to establish an aproximate conversion factor from pixel space to real space: 30/720 [meters/pixel] in the *y* dimension, and 3.7 [meters/pixel] in the *x* dimension.

The method `aux_fun.calculate_radius_in_meters()` implements the form factor correctioni and calculates the radius of curvature in meters, receiving the fitted line coefficients from the pixel space.

The **offset** is defined as the horizontal distance between the lane midpoint and image midpoint, evaluated in the bottom edge of the image.

This calculation follows the same rationale as the curvature when it comes to mapping results from the warped pixel space to the physical space, and has been encapsulated in `aux_fun.calculate_offset_in_meters()`.

### Output display

*[Cells 38 - 39 of the project notebook]*

The last step of the pipeline is to print all the results back on the undistorted image.

The detected lane pixels and fitted lines on the warped space are transformed back to the physical space using the inverse perspective transform, simply built by calling `cv2.getPerspectiveTransform(dst, src)` rather than `cv2.getPerspectiveTransform(dst, src)`.

The detected lanes now in the real-world space are superimposed on the original undistorted image. The instantaneous radius of curvature and offset are also plotted on the image:

![pipeline_output]

This task is performed by `aux_fun.print_summary_on_original_image()`.
