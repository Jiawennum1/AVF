Lane Detection

Region Detection
The first step involves cropping the region of interest (ROI) from the camera's image, focusing on the area above the hood and adjacent to both lanes, using cv::fillpoly for drawing and cv::bitwise_and() for cropping.

Color Detection
The second step involves color filtering within the ROI area, focusing on white or yellow lane lines using the HSV color space. White lines require high brightness values, while yellow lines need specific hue and saturation ranges.

It's important to note that reflections, snow, or white objects on the road can interfere with detection. Therefore, using the HSV color space is only suitable for simple road conditions; more complex scenarios may require advanced color filtering methods, such as incorporating deep learning techniques.

Edge Detection
After isolating white or yellow lane lines, edge detection is applied using  cv::Canny() or cv::Sobel() for edge and corner detection, with Canny generally providing better results.

Lane Detection
The final step in lane detection involves merging the previously identified edges to form the lanes on each side. There are two ways to fit the lane lines: straight line fitting and quadratic curve fitting.

Straight line fitting
Initially, cv::HoughLinesP() is utilized to detect line segments on edges by fine-tuning parameters within the function, which yields groups of line segments with varying lengths, widths, and distances. By adjusting the parameters, especially increasing the threshold for the length of the segments, it's possible to minimize the impact of small road debris or light reflections, enhancing the accuracy of the detection process.

After obtaining the appropriate group of line segments, it is necessary to distinguish between the left and right lanes, which can be done directly by the range of the slope. Positive slopes are categorized as the right lane, and negative slopes as the left lane. However, it's important to note that merely using the positive or negative distinction may not be sufficient. Lines with very small slopes, such as stop lines or crosswalk lines, could also be incorrectly classified into the left or right lanes, thus interfering with accurate lane detection.

Here's a useful trick: import the slopes of all detected raw lines into a CSV file, manually drive the vehicle through the entire course, and then analyze the slope range in the CSV. In my case, the slope range for the right lane was [0.3, 0.75], and for the left lane, it was [-0.75, -0.2].

After identifying the raw lines for each lane, these segments need to be merged into one. This is achieved using cv::fitLine(), which applies the principle of least squares to fit the points into a single line, representing each identified lane.

To enhance the robustness of the results, the historical data average of identified lines is used instead of the currently detected lines. This method significantly improves performance, as relying solely on current detection can lead to minor discrepancies due to the variation in detected line segments with each scan, manifesting as jitter in the recognized lane lines during practical application.

Finally, the identified lane lines are drawn onto the original image to complete the lane detection process.

quadratic curve fitting.
Curve fitting uses a sliding window approach.

Curve fitting does not require edge detection, but rather transforms the image after color detection into a bird's eye view. Then, by calculating the pixel values of each column, a histogram can be generated. By computing the maximum pixel values in the left and right halves of the image from this histogram, the approximate positions of the lane lines can be determined, preparing for the subsequent differentiation of the left and right lane positions.

Next, the sliding window method is used to find the pixels that meet the criteria for the left and right lane lines, and their coordinates are recorded. After obtaining the coordinates, PolynomialFit() is called to fit a quadratic curve, and the parameters a, b, c of the quadratic curve ax^2+bx+c are saved for the preparation of the next detection.

To improve efficiency, it is not necessary to transform the histogram and find new pixels each time. Instead, based on the previous detection, pixels that meet the criteria are searched for in the vicinity of the previous lane lines, specifically implemented in line_detection_by_previous().

The resulting curve is actually the curve in the bird's eye view and needs to be transformed back into the original view.

Steering Control
In steering control, a PID controller is used to determine the appropriate steering value. There are several reference values for defining the proportional error P:

By identifying the left and right lanes, a line connecting the lane center and vehicle center can be calculated. This line indicates the vehicle's direction, which should remain vertical during normal driving. Thus, the angle between this line and the vertical direction can serve as the proportional error P.
The deviation between the vehicle center and the road center can also be calculated based on the identified lanes. Normally, the vehicle should stay centered on the road.
Finally, the output range of the PID controller must be limited to between [-1,1].

Throttle Control
In throttle control, the approach differs based on whether there's a car ahead:

Without a car ahead, cruise control is used, with the PID error term being the difference between target and current speed. Adjusting the PID parameters yields the appropriate controller.
With a car ahead, the control becomes more complex, involving a cascaded PID controller. The first is a distance controller, calculating a target speed to maintain a distance of 0.6 times the current speed in meters. For instance, if the current speed is 50 km/h, the vehicle should maintain a distance of 30 meters from the car ahead, calculated as 50 * 0.6. This distance is dynamic, changing with the speed of the vehicle.The distance controller outputs a target velocity, which then determines the throttle value through a speed controller.
