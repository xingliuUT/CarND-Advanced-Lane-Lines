## Advanced Lane Finding project
---


The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/camera_calibration.jpg "Undistorted"
[image2]: ./output_images/test1_undist.jpg "Road Transformed"
[image3]: ./output_images/test2_binary.jpg "Binary Example"
[image4]: ./output_images/straight_lines1_warped.jpg "Warp Example"
[image5]: ./output_images/test2_fit.jpg "Fit Visual"
[image6]: ./output_images/test2_final.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup

#### 1. Provide a Writeup that includes all the rubric points and how you addressed each one.

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second and third code cells of the IPython notebook located in `./Lane_Finding_Pipeline.ipynb`.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in 3D space. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I applied the camera calibration matrix and distortion coefficients computed in the previous step to the `./test_images/test1.jpg` test image using the `cv2.undistort()` funtion and obtained the following result:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  

I used a combination of color and gradient thresholds to generate a binary image. Function `get_binary()` in the code cell of the IPython notebook under the title `2.4 Binary Image` contains the steps I took. First, I convert the image from BGR color space into HLS color space using the `hls_select()` function. For the H-channel, I applied a threshold of `(20, 100)`. For the L- and S-channel, I didn't apply a threshold but used its raw value as input to the gradient thresholding for the next step. Next, I apply perspective transform to focus on the region of the graph where the road is. I will describe the details of this step in the next point. Lastly, I apply Sobel gradient thresholding in the x-direction using the `abs_sobel_thresh()` function. The two images that I apply grad-x thresholding on are the L- and S- channel layers and I use a kernel size of 21 and threshold values `(15, 100)`. 

The output binary image is a combination of the pixels found in H-channel color thresholding, the S- and L-channel Sobel grad-x thresholding. Here's an example of my output for this step.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in the code cell of the IPython notebook under title `2.3 Perspective Transform`.  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 60, img_size[1] / 2 + 100],
    [(img_size[0] / 6), img_size[1] - 10],
    [(img_size[0] * 5 / 6), img_size[1] - 10],
    [(img_size[0] / 2 + 62), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 320, 0        | 
| 213, 710      | 320, 720      |
| 1067, 710     | 960, 720      |
| 702, 460      | 960, 0        |

The transform matrix (M) and inverse transform matrix (Minv) are computed by calling `cv2.getPerspectiveTransform()` from `src` to `dst`, and from `dst` to `src`. The warped image is generated by calling `cv2.warpPerspective()` using the transform matrix.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image `./test_images/straight_lines1.jpg` and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial.

The code for identifying lane lines and fit a polynomial is a function called `fit_poly()`, which appears in the code cell of the IPython notebook under title `2.5 Fit a Polynominal`.  The `fit_poly()` function has two mode depending on the input `cold_start`: when `cold_start = True`, it identifies lane line centers by sliding window from bottom up and identify nonzero pixels around two peaks the lane line pixels; when `cold_start = False`, it needs polynominal coefficients as input and looks for lane lines in the margins of the input lane lines. After lane line pixels are identified with either method, I use `np.polyfit()` function to fit individually to left and right lanes a quadratic function.
Fitting lane lines with a 2nd order polynomial of the `./test_images/test2.jpg` looks like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of curvature of the lane and the position of the vehicle with respect to the lane center in the function `get_curvature()` in the IPython notebook under the title `2.6 Calculate Curvature and Center`. I made use of the pixel to meters conversion in x and y (3.7 meters per 700 pixels in x and 30 meters per 720 pixels in y). After converting the polynomial into the unit of meters, I use the formula to compute the curvature radius according to [this blog](https://www.intmath.com/applications-differentiation/8-radius-curvature.php). I take the mean of the radius independently computed from left and right lane as the output curvature radius.

For the position of the vehicle with respect to the lane center, I assume that the camera is in the middle of an image. I find the average position of the two lanes when they cross the bottom of the image and take that as the center of the lane. Finally, I took the difference in pixels of the position of the camera and the lane center and convert into meters.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in `pipeline_function()` in the IPython notebook under the title `2.7 Lane Finder Class`. I create an image `color_warp` and use `cv2.fillPoly()` function to draw green shade on the region between the two lines I identified as lane lines. Then I use `cv2.warpPerspective()` function with the inverse perspective transform matrix `Minv` to cast the image back to the view from a camera. Finally, I combine this image with the original image using `cv2.addWaighted()` function and give the lane line shades a weight of 0.3 compared to 1. which is the weight of original image. Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
