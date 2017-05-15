# **Advanced Lane Finding Project**

**Advanced Lane Finding Project**

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

[image1]: ./output_images/calibration1.png "Calibration1"
[image2]: ./output_images/calibration2.png "Calibration2"
[image3]: ./output_images/binary_combo.png "Binary Example"
[image4]: ./output_images/warped_straight_lines.png "Warp Example"
[image5]: ./output_images/histogram_analysis.png "Histogram analysis"
[image6]: ./output_images/formula1.png "Polynomial function formula"
[image7]: ./output_images/formula2.png "Radius of curvature formula"
[image8]: ./output_images/pipeline.png "result of full pipeline"
[video1]: ./project_video_out.mp4 "Processed Video"


## [Rubric] (https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 3rd code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. Also, I used several chessboard images for calibration to get a more consistant and precise calibration results. 

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function. 

Here's one example before and after camera calibration :

![alt text][image1]

One can see the distorted edge regions are successfully calibrated.


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Using the calibration parameters `mtx` and `dist` got from previous step, I apply the distortion correction to one of the test image using `cv2.undistort()` function. Here's the result before and after distortion correction :

![alt text][image2]

The image is successfully undistorted after distortion correction.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps are in the 12th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb").  

For robust edge detection of lane line finding, I also cropped lane line region using `region_of_interest()` function at the end of thresholded binary image generation pipeline (in the 8th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb")

Here's several test examples of my output : 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `get_perspective()`, which appears in the 7th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb").  The `get_perspective()` function takes as inputs an image (`img`), calibration parameters `mtx` , `dist`, source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
#source vertices of region for perspective transform
src_left_top = [(img_size[0] / 2) - 58, img_size[1] / 2 + 100]
src_right_top = [(img_size[0] / 2 + 65), img_size[1] / 2 + 100]
src_right_bottom = [(img_size[0] * 5 / 6) + 35, img_size[1]]
src_left_bottom = [((img_size[0] / 6) ), img_size[1]]

perspective_src_vertices = np.float32([src_left_top, src_right_top, src_right_bottom, src_left_bottom])      

#destincation vertices of region for perspective transform
dst_left_top = [(img_size[0] / 4), 0]
dst_right_top = [(img_size[0] * 3 / 4), 0]
dst_right_bottom = [(img_size[0] * 3 / 4), img_size[1]]
dst_left_bottom = [(img_size[0] / 4), img_size[1]]

perspective_dst_vertices = np.float32([dst_left_top, dst_right_top, dst_right_bottom, dst_left_bottom])  
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 582, 460      | 320, 0        | 
| 705, 460      | 960, 0        |
| 1101, 720     | 960, 720      |
| 213, 720      | 320, 720      |

I used a straight lane line image to verify perspective transform. If the lane lines appear parallel in the warped images, the perspective transform is succsessfully performed.

Here's my pipeline for verification :
1. Load original image
2. Camera calibration to undistort imgae
3. Crop region of interest
4. Perform perspective transform

Here's the result of the pipeline :

![alt text][image4]

One can see the lane lines in the final warped image is parallel that verify the success of perspective transformation.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

First, I took histogram lower half of binary perspective image and used sliding window technique to locate left and right lane line points and wraped those steps in `histogram_analysis()` function in the 14th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb")

After utilize `histogram_analysis()` to get left and right lane line points, I use `np.polyfit()` to fit left and right lane line points to get 2nd order polynomial parameters `left_fit` and `right_fit` for left and right lane line, respectively. These steps are coded in line 6~14 in the 16th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb")

Here's one example of histogram analysis with sliding window found lane line segments visualization :

![alt text][image5]


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

To calculated the radius of curvature, I first convert image space (pixel number unit) to world space (meter unit), use `np.polyfit()` to fit converted left and right lane line points to get 2nd order polinomial parameters `left_fit_cr` and `right_fit_cr` , then calculate radius of curvature of left and right lane using the following formula :

Fitted polynomial function :

![alt text][image6]

Calculate radius of curvature using fitted parameters A,B,C from polynomial function :

![alt text][image7]


To calculate position of the vehicel with respect to center, I first use `np.mean()` to estimate mean x values of left and right lane points and get the lane center. Afterwards, I use half x value of warped image shape as vehicle center and minus the previous lane center to get the position of the vehicle with respect to center.

These steps are coded in lines 16~38 in the 16th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb")


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines 42~56 in the 16th code cell of the IPython notebook located in "./advanced_lane_lines_final.ipynb")

The full image process pipeline as described in step 1 to 6 is wraped in `pipeline()` function and verified in 17th code cell. Here is one example of my full pipeline result on test image :

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_out.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My pipeline takes hybrid combination of several filtering techniques and the filtering parameters are fixed according to specific type of scenes. However, it may fail in some cases when pixel value distribution is out of those scene types.

To make it more robust, I'd try to adaptively tune the paramters according to scenes or record lane line location history and use it to track lane line position for the next image.




