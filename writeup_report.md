## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

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

[//]: # "Image References"

[image1]: ./output_images/calibration-ex.png "Undistorted"
[image2]: ./output_images/undistorted_ex.png "Road Transformed"
[image3]: ./output_images/find_lane_good.png "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./output_images/fit-poly.png "Fit Visual"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "[part1_cam_calibration.ipynb](part1_cam_calibration.ipynb)".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

The main parameters for image undistortion are stored in the pickle file "./camera_cal/wide_dist_pickle.p" which is loaded in the pipleline. The pipeline (for both image and video) can be found in [part2_pipeline.ipynb](part2_pipeline.ipynb)

### Pipeline (single images)

The image pipeline `img_pipeline` function invokes the following functions:

* `remove_distortion` 
* `threshold_binary`
* `prespective_transform`
* `find_line_pixels`: applies the rolling window approach, calling two subroutines 
  * `_find_window_centroids`
  * `_window_mask`
* `overlay_image`: calculates radius of curvature, position, and draws the final output image

The figure below shows the main stages of the pipeline, referred below as the pipeline figure.

![alt text][image3]

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

The code is provided in the second cell of [part2_pipeline.ipynb](part2_pipeline.ipynb), which consists of loading the pickle stored from the previous python notebook, and then calling function `remove_distortion(img)` which invokes `cv2.undistort(img, mtx, dist, None, mtx)`.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in the cell following title Color Threshold). The gradient (sobelx) with threshold `sx_thresh=(10,150)`seems quite robust in most situations except when the line is blurry, shadows on the roads, and nearby curbs with sharp edges. The outputs of the threshold operation is shown in the pipeline figure, third column as follows:

* Red color is Saturation channel in HLS color space
* Blue color is gradient threshold (sobel in x direction)
* Green is Lightness channel of LAB color space

The output binary is blue + yellow (which is the combination of red and green). According to Wikipidia "*Lab* color is designed to approximate human vision" and the lightness channel of LAB produces more accurate results than that of HLS.

The reason why I am not solely relaying gradient threshold is seen the following examples:

* start.jpg (shown row 1, columns 3,4 in the pipeline figure): without the yellow color, the rolling window algorithm picks the blue line and starts on the far left (curb line). 
* As seen in test4.jpg and test5.jpg, lightness and saturation cannot produce satisfactory results alone on blurry lines, but together they provide fairy accurate estimation of the lines, which otherwise can't be found using the gradient technique.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `prespective_transform()`, which appears in the cell below "Perspective Transform" section.  The `prespective_transform()` function takes as inputs an image (`img`), and uses global variables  (`src`) and destination (`dst`).  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32([(678,443),(605,443),(285,665),(1019,665)]) 
dst = np.float32([(1019-100,0),(285,0),(285,665),(1019-100,665)])
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto test images and their warped counterparts to verify that the lines appear parallel in the warped image. See the pipeline figure.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

This is found in function `overlay_image` . It basically uses the same approach as the lecture, and mostly same code snippets except for adding blue and red lines.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in `overlay_image` function which follows the following main steps:

```python
vehicle_loc = w/2
left_lane_loc = left_fit[0]*h**2 + left_fit[1]*h + left_fit[2]
right_lane_loc = right_fit[0]*h**2 + right_fit[1]*h + right_fit[2]
lane_center_loc = (left_lane_loc + right_lane_loc) /2
shift = (vehicle_loc - lane_center_loc) * xm_per_pix
```
#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in  function `overlay_image()`.  The results is plotted above. The complete image pipleline is found in function `img_pipeline(img)`

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

The video pipeline function `video_pipeline`(see code cells bellow section " Video Pipeline" in [part2_pipeline.ipynb](part2_pipeline.ipynb))  uses two instances of class `Line`, `left_line, right_line`. 

`video_pipeline` invokes the following functions:

- `remove_distortion` 
- `threshold_binary`
- `prespective_transform`
- `find_line_pixels`: applies the rolling window approach. It calls two subroutines 
  - `_find_window_centroids`
  - `_window_mask`
- `draw_lane(img,left_line,right_line)` draws  the final image.

Class `Line` has the following functions (short descriptions are provided in the code) :

- `update(self,x,y, force_append=False)`
- `revert_to_previous_fit`
- `update_lane_data`
- `find_line_pixels_by_best_fit`

I also added `self.queue_length=5` to denote the window size for smoothing lines over time. The code in `video_pipeline` is self explanatory.



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

Thresholding is a bit tricky as each threshold technique performs well in certain scenarios and bad in other scenarios. One needs to get the right thresholds and channels as well as a combination and AND/OR rules between them. I found Sobelx | (S channel of HLS  && L channel of LAB) performed well with all stationary images (./test_images/). Another challenge is to detect the right histogram peaks at each frame. Reusing previous polynomial fits and performing sanity check on the distance between lanes are two good approaches to mitigate any issues related to wrong peak detection for the lanes. For example image `test1.jpg` didn't perform well, but when I reuse the previous fit, it fixes the issue.

###### Limitation and Future work

The pipeline is still sensitive to the start frame of the video. Consider the scenario when the histogram peak occur on the left curb boundary instead of the left lane (its possible to construct a case where lane width and lane curvature are not sufficient to detect). Further sanity checks should be performed for this case. I also didn't deal with the case when no peak is detected for a lane.

One way to improve thresholding is by setting thresholds dynamically. For example, if the number of pixels detected by a threshold rule is too high, then it should be lowered automatically for that frame. Also a weighted average approach could be useful, say, by putting more emphasis on the intersection between different threshold rules.

An interesting future work would be to consider the scenario of changing lanes, and how track to lanes simultaneously in order to achieve (horizontal) localization with respect to the lanes.

