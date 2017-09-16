# **Finding Lane Lines on the Road** 

### by Arjun Chandra
---

[//]: # (Image References)

[image1]: ./reflection/challenge-spurious-hough.png "Line detection with spurious artefacts and bonnet"
[image2]: ./reflection/challenge-spurious-solid.png "Solid lines with spurious artefacts and bonnet"
[image3]: ./reflection/challenge-spurious-cleaner-hough.png "Line detection with spurious artefacts and bonnet with different params and region"
[image4]: ./reflection/challenge-spurious-cleaner-solid.png "Solid lines with spurious artefacts and bonnet with different params and region"


## The current pipeline

The function `draw_lane_lines()` contains the entire pipeline and is called by functions `process_image_hough()` and `process_image_solid()` to find and mark hough segments and solid lines respectively. The following steps constitute my pipeline:

1. **Grayscaling**: the lane lines are brighter than the road, and grayscaling helps 
focus on brightness of pixels, aiding the computation of gradients as part of Canny 
edge detection.

2. **Blurring**: blurring the grayscaled image helps smooth out the gradients computed 
by Canny edge detection. I have considered different kernels for blurring the 
test and challenge images. The challenge video has spurious artefacts like tree 
shadows on the road, which required more smoothing, thereby making the shadow 
areas less prominent for Canny edge detection.   

3. **Canny edge detection**: detects pixels which are likely to be edges or boundaries 
around homogenous (in terms of brightness) parts of the image. The boundaties of 
the lane lines are marked on the grayscaled and blurred image. Different 
thresholds were found fitting for the test and challenge images/videos. A 
higher low_threshold and more blurring for the challenge video helps filter out 
spurious artefacts better.

4. **Hough line detection in region of interest**: Line detection is performed on 
a limited region of interest in the edge detected image. The region is an 
isoseles trapezium towards the bottom of the image covering the most relevant 
(lane line) part of the image. Note that the region vertices are different for 
the challenge video, as compared to the test images and videos. The lower part 
of the challenge video has a bonnet of the car throughout. To prevent it from 
contributing to line averaging (step 6), it is masked out.

Below are images after Step 4 and the full pipeline to show how spurious 
artefacts and bonnet necessitate different parameters for blurring and edge 
detectoin, and a different region of interest.

![alt text][image1]
![alt text][image2]

Now, with **different parameters and region**.

![alt text][image3]
![alt text][image4]

5. **Separating left and right lane lines**: To aid line averaging, the hough lines 
segments are are grouped by slope into left and right side segments.

6. **Ordering lines before averaging**: The detected and grouped lines are ordered 
by proximity to windshield (bottom of images). This is done in order to do a 
weighted averaging of detected lines, putting more weight on lines closer to the 
windshield.

7. **Averaging**: A weighted average of the ordered lines is performed. Three types 
of averaging were tried, namely, simple, linearly increasing weights, and 
exponentially increasing weights. Weights increase with proximity to windshield, 
as previously mentioned. It was observed that linear increasing weights lead to 
smoother lane tracking (step 9) for test videos. Exponentially increasing 
weights helped in the case of the challenge video.

8. **Extrapolation**: Both left and right average lines are extended along the 
y-axis between fixed x-coordinates.

9. **Tracking lane lines in video**: Averaging is stateful. Average computed in the 
new frame is fused in with the old via a linear combination if this average does 
not deviate beyond a threshold. The difference between average lines is currently 
computed as the eucliden distance between average line end points. This can be 
bettered by considering other alternatives (see point 6 below in shortcomings, and 
comments in notebook).

Test video `solidYellowLeft.mp4` results with different ways of averaging:

[Linear (better)](./test_videos_output/solidYellowLeft.mp4) and [Exponential](./reflection/solidYellowLeft-exp.mp4)

Challenge video results with different ways of averaging:

[Linear](./reflection/challenge-lin.mp4) and [Exponential (better)](./test_videos_output/challenge.mp4)



## Shortcomings and improvements with current pipeline

There are many shortcomings with the current pipeline. Following are some of
these with suggeted improvements:

1. There is currently **no explicit way of handling obstacles** within region of interest. Adding object detection followed by masking these objects could be an improvement, although no test images/videos had any obstacles.

2. **Blurring is fixed**, which otherwise could help deal with complex patterns on the road like tree shadows and make pipeline more robust across scenarios. Keeping trak of the distribution of lines returned after applying hough line detection can indicate whether noise in current frame has increased, thereby helping adapt blurring.

3. The averaging method consideres **weights over ordered hough segments**. Segments closer to the car contribute more to the average. However, these **weights are relative to detected segments**. Gaps in lanes closer towards windshield can thus cause the average to be concentrated at a distance. This can cause curved lane lines at a distance to influence the slope of the actual (invisible) lane. Making averages stateful and linearly fusing average lines between frames helps with this to some extent. 

4. If **no hough segment is detected, the previous average line is assumed** to 
represent the average line in the frame. A better approach or adaptive edge and 
line detection may prevent assuming lines and make tracking lanes smoother.

5. The **averaging step is stateful**. The state helps keep a watch on big 
deviations in detected average lines between frames. A **threshold is used** to 
track (adapt to) the detected average lines in the new frame. This extra 
parameter potentially makes the pipeline less robust, although it helps 
make the transition between detected lanes between frames smoother.

6. Tracking lane lines **fuses** old and new **average lines**. Fusing the extrapolated lines instead may make tracking smoother and more accurate, since the x-coordinates between all extrapolated drawn lines are shared. This provides a static reference to base differences on.
