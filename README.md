## Advanced Lane Finding Project

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

[undistort_chessboard]: ./output_images/undistort_output.png "Undistorted"
[undistort_road]: ./output_images/undistort_road.png "Road Transformed"
[binary_threshold]: ./output_images/binary_threshold.png "Binary Threshold"
[perspective_transform]: ./output_images/perspective_transform.png "Perspective Transform"
[find_lanes]: ./output_images/find_lanes.png "Find Lanes"
[update_lanes]: ./output_images/update_lanes.png "Update Lanes"
[lane_marker]: ./output_images/lane_marker.png "Lane Marker"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.

---
### Writeup / README

#### Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.

You're reading it!
### Camera Calibration

#### Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first and second code cells of the IPython notebook located in "./examples/example.ipynb" (or in lines # through # of the file called `some_file.py`).

The camera is calibrated using a series of calibration (chessboard) images. The calibration process involves finding each chessboard corner and mapping those 2D image-space locations to the chessboard corners in 3D object space. 

The chessboard is assumed to be fixed on the (x, y) plain at z=0, such that the object points are the same for each calibration image. The object points, `objp`, are thus initialized with values like (0, 0, 0), (1, 0, 0), (2, 0, 0), ..., (9, 6, 0). For each calibrarion image, the corners found using `cv2.findChessboardCorners()` are added to `imgpoints` while the predefined `objp` array is appened to `ojbpoints`. The set of calibration images provides enough pixel-to-object mapping information to construct the camera matrix and distortion coefficients using `cv2.calibrateCamera()`. 

The `Camera` class is defined to encapsulate the calibration and image correction concepts.

Distortion correction is applied to a test image using the cv2.undistort() function and obtained this result in the third code cell of the notebook:

![alt text][undistort_chessboard]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
One test image was randomly selected and corrected in the fourth code cell of the notebook:
![alt text][undistort_road]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
A combination of color and gradient thresholds is used to generate a binary image (thresholding steps in the fifth code cell of the notebook).

The binary image is generated using a combination of horizontal gradient threshold with a saturation threshold (see the fifth code cell of the notebook). Each of these thresholds has different strenghts for different lighting conditions and together produce a better threshold than alone.

Here's an example output for this step:

![alt text][binary_threshold]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for the perspective transform includes a function called `warperbirds_eye_warp()`, which is in the 6th code cell of the notebook.  This function takes as inputs an image (`img`) and selects the source `src` and destination (`dst`) points for the transform.  The hardcoded the source and destination points were chosen experimentally using a test image and resulted in the following final values:

```
    h, w, d = img.shape
    src = np.float32(
        [[(w / 2) - 58, h / 2 + 100],
        [((w / 6) - 10), h],
        [(w * 5 / 6) + 40, h],
        [(w / 2 + 60), h / 2 + 100]])
    dst = np.float32(
        [[(w / 4), 0],
        [(w / 4), h],
        [(w * 3 / 4), h],
        [(w * 3 / 4), 0]])
```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 582, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1106, 720     | 960, 720      |
| 700, 460      | 960, 0        |

The prespective transform was verified to work as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][perspective_transform]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The lane finding function called `find_lanes()` uses the suggested algorithm whereby a histogram of the bottom half of the binary (corrected) image is analyzed to find peak values of each the left and right halves of the histogram. These values are used as the starting point for the lane search algorith (see the 7th code cell in the notebook).

Once the starting points at the bottom of the image have been located, the algorithm continues search upward with a series of windows (areas of interest) to narrow the search for the connecting segment of the lane line pixels. Once the lane is followed to the top of the image, a 2nd order polynomial is fit to the detected segments of the lane lines. 

The following image output image shows the locations of the windows drawn in green and the binary image pixels colored red and blue to indicate which pixels were selected as left or right lane pixels. The polynomials are also displayed in output:

![alt text][find_lanes]

Once the initial lane lines are located, subsequent frames use a simplified algorithm that leverages the coherence between frames. The function `update_lanes` implements this algorithm in the 8th code cell of the notebook. The algorithm simply begins where the previous frame's polynomials intersect the bottom of the image. It the proceeds by fitting a new 2nd order polynomial for each lane for pixels along the old curves plus some horizonal margin. 

The following image shows the area of the image used to fit the polynomials for each lane. Again pixles are highlight in red and blue for the left and right lanes as well as the polynomial curves draw in yellow:

![alt text][update_lanes]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The next code cell (9th) calculates the radius of curvature for each lane polynomial using the suggested method. The vehicles position within the lane is also calculated by assuming camera is positioned at the center of the vehicle and thus is at exactly the halfway point of the image. The offset of each lane line from the center of the image indicates whether the vehicle closer to one lane or the other and by how much. An offset of 0 indicates the vehicle is in exactly the center of the lane. Negative values indicate the vehicle is closer to the left lane and positive indicates closer to the right lane.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The lane curves are projected back onto the original (corrected) image in the 10th code cell of the notebook. The curves identified by the lane finding algorithm are draw as a filled polygon then warped back to the original perspective using the `M_inv` inverse projection matrix. The resulting warped polygon is then composited over the original (corrected) image to correctly identify the lane. The radius of curvature for each lane and the vehicle lane position is also displayed in a HUD overlay.

Here is an example of output on a test image:

![alt text][lane_marker]

---

### Pipeline (video)

#### Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The still image pipeline is finally assembled into a video pipeline for the given test video. The 11th code cell of the notebook dfines the class `LaneFinder` that is initialized with a calibrated camara. The first frame is processed using the full lane search `find_lanes()` to initially identify the lanes. Subsequent frames use the `update_lanes()` in order to leverage the coherency bewteen frames which stabilizes the lane tracking process.

The radius of curvature is calculated for each frame after lane detection and a simple sanity check is performed to filter out curves with an unreasonably small radius. This check is a bit naive but works well enough for this simplified project. If a frame's lane detection is rejected, it simply uses the curve from the previous frame. When the radius for each lane is displayed in the HUD, the text is shown in green when the current frame detected a "sane" curve, otherwise it is show in red.

Here's a [link to the video output](https://youtu.be/xGDEj1xNkgE)

---

### Discussion

#### Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

There were a few places in the video where the color of the pavement is much lighter which reduces the contrast between the road and the laner markers to become difficult to properly threshold. Improvments to further stabilize the thresholding process are an obvious place to start.

Inspecting some debug output of the lane finding algorithm shows that these challenging parts of the video result in blotchy areas that cause the polynomial fitting to include pixels that are not actually part of the lane markers. A few techniques were experimnented with to reject outlier pixels, but none of them worked well enough to bother including in the final submission. With more time, it might be nice to also try finding the horizontal center within each search window and fit *those* points instead of the fitting to every pixel within the window. It would be fun to continue experimenting and iterating to make this algorithm faster and more stable.
