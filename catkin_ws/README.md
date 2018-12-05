### Perception & object detection using 3D Point clouds

This project directory contains ROS perception implementation to recognition and classify the objects in noisy environment 
from a RGB-D camera from noisy table environment. This project is used for shorting object and pick that one and then place 
in basket. This project used python along with ROS in udactiy project 3 of the [![Udacity - Robotics NanoDegree Program] 
(https://s3-us-west-1.amazonaws.com/udacity-robotics/Extra+Images/RoboND_flag.png)](https://www.udacity.com/robotics)
The goal of this project is 
1.Have the PR2 image its world
2.Identify objects from the pick list
3.Manoeuvre the PR2 towards the objects
4.Grasp the object
5.Manoeuvre the PR2 towards the object's destination basket
6.Successfully place the object in the basket


### Prerequisites

1. [Ubuntu](https://www.ubuntu.com/) OS, as at the time of this writing, ROS only works on Ubuntu.
2. Python 2. Installation instructions can be found [here](https://www.python.org/downloads/).
3. Robot Operating System (ROS). Installation instructions can be found [here](http://wiki.ros.org/ROS/Installation).


### External repositories

This project have two directories ons is for train the classification for the objects and second is for implement the 
PR2 project for pick and place.


#### Training repository

This step can be skipped and is not required if the pre-trained `model.sav` is used. Otherwise, Download and setup 
the [Udacity Perception Exercises repository](https://github.com/udacity/RoboND-Perception-Exercises). If ROS is 
installed, follow the setup instructions outlined in the repositories README.

#### ### Environment Setup

Download and setup the [Udacity Perception Project repository](https://github.com/udacity/RoboND-Perception-Project.git). 
If ROS is installed, follow the setup instructions outlined in the repositories README.

Make th directory for project execution 
$ mkdir -p ~/catkin_ws/src
$ cd ~/catkin_ws/
$ catkin_make

Clone the PR2 project
$ cd ~/catkin_ws/src
$ git clone https://github.com/udacity/RoboND-Perception-Project.git

Install missing dependencies:
$ cd ~/catkin_ws
$ rosdep install --from-paths src --ignore-src --rosdistro=kinetic -y

Build it:
$ cd ~/catkin_ws
$ catkin_make

Add the following to your '.bashrc' file:
export GAZEBO_MODEL_PATH=~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/models:$GAZEBO_MODEL_PATH
source ~/catkin_ws/devel/setup.bash

[Filtering and Segmentation Exercises](https://github.com/udacity/RoboND-Perception-Exercises)
Using the 'python-pcl' library, which could be invoked using 'import pcl'
[Point Cloud Library](http://www.pointclouds.org/)

### Code Analysis ###

##### OpenCV
we won't be using OpenCV to calibrate our camera(we'll be using a ROS package), the process for doing so with OpenCV is outlined here:

1.Use 'cv2.findChessboardCorners()' to find corners in chessboard images and aggregate arrays of image points (2D image plane points) 
  and object points (3D world points) .
2.Use the OpenCV function 'cv2.calibrateCamera()' to compute the calibration matrices and distortion coefficients.
3.Use 'cv2.undistort()' to undistort a test image.
OpenCV functions for calibrating camera: 'findChessboardCorners()' and 'drawChessboardCorners()'

##### Filtering
First, we're going to walk through some filtering techniques that will help give us only the parts of the point cloud we're intrested in.
We'll harness some filters from the [Point Cloud Library](http://www.pointclouds.org/)

###### VoxelGrid Downsampling Filter
[image1]: ./src/images/VoxelGrid.png
Down sampling is used to decrease the density of the pointcloud that is output from the RGB-D camera. This is done because the very 
feature rich pointclouds can be quite computationally expensive.
*Downsampling* can be thought of decreasing the resolution of a three dimensional point cloud.
'RANSAC.py' contains the Voxel Downsampling Code.

###### Pass Through Filtering
We'll use Pass Through Filtering to trim down our point cloud space along specified axes, in order to decrease the sample size. 
We will allow a specific region to 'Pass Through'. This is called the 'Region of Interest'.
'Pass Through Filtering' can be thought of as cropping a three dimensional space across one or many of the dimensions.

###### RANSAC Plane Fitting

We can model the table in our dataset as a plane, and remove it from the pointcloud using 'Random Sample Consensus' or 'RANSAC' algorithm.
As you can see, we selected the horizontal plane below, which was the table our target objects were sitting on.
[image2]: ./src/images/RANSAC.png

###### Extracting Indices - Inliers and Outliers
Once we've determined where our table is located, we can create two separate point clouds: one for the table(inliers), and one for the objects beyond(outliers).
1.Inliers
extracted_inliers = cloud_filtered.extract(inliers, negative=False)
2.Outliers
extracted_outliers = cloud_filtered.extract(inliers, negative=True)

###### Outlier Removal Filter
[image3]: ./src/images/Outlier.png
This filter is used to statistically remove noise from the image.
Much like the previous filters, we start by creating a filter object: 
outlier_filter = cloud_filtered.make_statistical_outlier_filter()
Set the number of neighboring points to analyze for any given point:
outlier_filter.set_mean_k(50)
x = 1.0
Any point with a mean distance larger than global (mean distance+x*std_dev) will be considered an outlier
outlier_filter.set_std_dev_mul_thresh(x)
Finally call the filter function
cloud_filtered = outlier_filter.filter()

##### Clustering for Segmentation
Now that we've used shape attributes in the dataset to filter and segment the data, we'll move on to using other elements 
of our dataset such as: color and spatial properties

###### K-Means Clustering
K-Means Clustering is an appropriate clustering algorithm if you are aware of your data space and have a rough idea of the number of clusters.
K-Means used as followed:

1.Choose the number of k-means (the number of clusters to look for)
2.Define the convergence / termination criteria (stability of solution / number of iterations)
3.Select the initial centroid locations, or randomly generate them
4.Calculate the distance of each data point to each of the centroids
5.Assign each of the data points to one of the centroids(clusters) based upon closest proximity
6.Recompute the centroid based on the data points that belong to it
7.Loop back to Step 4 until convergence / termination criteria is met
[image4]: ./src/images/k-means.png
If you are unsure of the number of clusters, it is best to use a different clustering solution! Such as DBSCAN...

###### DBSCAN (Density-Based Spatial Clustering of Applications with Noise) Algorithm Sometimes called Euclidean Clustering
DBSCAN is a nice alternative to k-means when you don't know how many clusters to expect in your data, but you do know something 
about how the points should be clustered in terms of density (distance between points in a cluster).
[image5]: ./src/images/DBSCAN.png
Original data on the left and clusters identified by the DBSCAN algorithm on the right. For DBSCAN clusters, large coloured points 
represent core cluster members, small coloured points represent cluster edge members, and small black points represent outliers.
DBSCAN data points do not have to be spatial data; they can be color data, intensity values, or other numerical features!
This means we can cluster not only based upon proximity, but we can cluster similarly coloured objects!

##### Run the VM to test our filtering and segmentation code
Down sampling, Pass through, RANSAC plane fitting, extract inliers/outliers
$ roslaunch sensor_stick robot_spawn.launch
'segmentation.py' contains all of our filtering code. Run it with:
$ ./segmentation.py
When you click on topics in RViz, you should be able to only see this view when the 'pcl_objects' topic is selected:
[image6]: ./src/images/segmentation.png

### Object Recognition  ###

### HSV
HSV can help us identify objects when the lighting conditions change on an RGB image. 'Hue' is the colour, 'Saturation' 
is the color intensity, and 'Value' is the brightness. A conversion from RGB to HSV can be done with 'OpenCV':
hsv_image = cv2.cvtColor(rgb_image, cv2.COLOR_RGB2HSV)

### Color Histograms

Using color historgrams, we can identify the colour pattern of an object, and not be limited to spatial data. 
Let's use this example of the Udacity can:
[image7]: ./src/images/ColorHistograms.png
Using 'numpy', we can create a histogram for each one of the colour channels. See 'color_histogram.py' to see how we do this with code. 
The result is:
[image8]: ./src/images/ColorHistograms1.png
This is the RGB Signature of the blue Udacity can!

###  Surface Normals
Just as we did with plotting colours, we'll now want to plot shape. This can be done by looking at the 'surface normals' of a shape in aggregate. 
We'll also analyse them using histograms like the ones below:
[image9]: ./src/images/SurfaceNormals.png
These 'surface normal' histograms correspond to the following shapes:
[image10]: ./src/images/SurfaceNormals1.jpg

### Support Vector Machines (SVM)
'SVM' is just a funny name for a particular supervised machine learning algorithm that allows you to characterize the parameter space 
of your dataset into discrete classes. It is a classification technique that uses hyperplanes to delineate between discrete classes. 
The ideal hyperplane for each decision boundary is one that maximizes the margin, or space, from points.
[image11]: ./src/images/SVM.png
'Scikit-Learn' or 'sklearn.svm.SV'` will help us implement the SVM algorithm. 
[Check this link out for documentation on scikit-learn](http://scikit-learn.org/stable/modules/classes.html#module-sklearn.svm)
'svm.py' will house our SVM code and 'generate_clusters.py' will help us create a random dataset.
svc = svm.SVC(kernel='linear').fit(X, y)
The line above is the one doing the heavy lifting. The type of delineation can be changed. 
It must be one of 'linear', 'poly', 'rbf', 'sigmoid', 'precomputed' or a callable. 
[Read more here](http://scikit-learn.org/stable/modules/generated/sklearn.svm.SVC.html).

#### Recognition Exercise - Combining what we have learned!
This is a very good confusion matrix. Initially, ours will not come out this good.
[image12]: ./src/normalizedConfusionMatrix.png
$ cd ~/catkin_ws
$ roslaunch sensor_stick training.launch
Once the environment is up, then:
$ cd ~/catkin_ws
$ rosrun sensor_stick capture_features.py
The 'capture_features.py' script should randomly place the 7 objects infront of the RGB-D camera and capture features about the items, 
then output 'training_set.sav'
Then, to generate the confusion matrix:
$ rosrun sensor_stick train_svm.py
As you can see, this is not a good confusion matrix. That's because the functions 'compute_color_histograms()' and 
'compute_normal_histograms()', in the file 'features.py' is not appropriately filled out. 
You can find 'features.py' in the directory:
~/catkin_ws/src/sensor_stick/src/sensor_stick/
First up, we'll alter the 'compute_color_histograms()' function, which will do RGB color analysis on each point of the point cloud.
==================================================================
def compute_color_histograms(cloud, using_hsv=False):

    # Compute histograms for the clusters
    point_colors_list = []

    # Step through each point in the point cloud
    for point in pc2.read_points(cloud, skip_nans=True):
        rgb_list = float_to_rgb(point[3])
        if using_hsv:
            point_colors_list.append(rgb_to_hsv(rgb_list) * 255)
        else:
            point_colors_list.append(rgb_list)

    # Populate lists with color values
    channel_1_vals = []
    channel_2_vals = []
    channel_3_vals = []

    for color in point_colors_list:
        channel_1_vals.append(color[0])
        channel_2_vals.append(color[1])
        channel_3_vals.append(color[2])

    nbins=32
    bins_range=(0,256)
    
    # Compute histograms
    r_hist = np.histogram(channel_1_vals, bins=nbins, range=bins_range)
    g_hist = np.histogram(channel_2_vals, bins=nbins, range=bins_range)
    b_hist = np.histogram(channel_3_vals, bins=nbins, range=bins_range)
    
    # Extract the features
    # Concatenate and normalize the histograms
    hist_features = np.concatenate((r_hist[0], g_hist[0], b_hist[0])).astype(np.float64)
    normed_features = hist_features / np.sum(hist_features)  

    return normed_features 
========================================================================================	
Next, we'll add the histogram, compute features, concatenate them, and normalize them for the 'surface normals' 
function 'compute_normal_histograms'. Remember to aleter the range of the histograms here,as compared to the RGB 
histograms above. '0-255' was our range there, but here it will be from '-1 to 1':

def compute_normal_histograms(normal_cloud):
    norm_x_vals = []
    norm_y_vals = []
    norm_z_vals = []

    for norm_component in pc2.read_points(normal_cloud,
                                          field_names = ('normal_x', 'normal_y', 'normal_z'),
                                          skip_nans=True):
        norm_x_vals.append(norm_component[0])
        norm_y_vals.append(norm_component[1])
        norm_z_vals.append(norm_component[2])


    nbins=32
    bins_range=(-1,1)
    # Compute histograms of normal values (just like with color)

    # Compute histograms
    x_hist = np.histogram(norm_x_vals, bins=nbins, range=bins_range)
    y_hist = np.histogram(norm_y_vals, bins=nbins, range=bins_range)
    z_hist = np.histogram(norm_z_vals, bins=nbins, range=bins_range)
    
    # Concatenate and normalize the histograms
    hist_features = np.concatenate((x_hist[0], y_hist[0], z_hist[0])).astype(np.float64)
    normed_features = hist_features / np.sum(hist_features)  

    return normed_features
=======================================================================================
Now, we can relaunch the Gazebo environment, re-run the training and feature capture, then generate the confusion matrix:
$ cd ~/catkin_ws
$ roslaunch sensor_stick training.launch
Once the environment is up, then:
$ cd ~/catkin_ws
$ rosrun sensor_stick capture_features.py
Give it a minute or so to go through all 7 objects. Then: 
$ rosrun sensor_stick train_svm.py
The outputted confusion matrix should be better than the previous one. But there are still strategies to improve:
Convert RGB to HSV
Compute features for a larger set of random orientations of the objects
Try different binning schemes with the histogram(32,64, etc)
Modify the SVM parameters(kernel, regularization, etc)
To modify how many times each object is spawned randomly, look for the for loop in 'capture_features.py' that begins with
'for i in range(5):' Increase this loop to increase the number of times you capture random orientations for each object.
To use HSV, find the line in 'capture_features.py' where you're calling 'compute_color_histograms()' and change the flag to 'using_hsv=True'.
To mess with the SVM parameters, open up 'train_svm.py' and find where you're defining your classifier. Check out the sklearn.svm docs 
to see what your options are there.
After setting 'using_hsv=true' and 'bins=3'`(which seems to be a sweet spot), I began playing with the other two features, 
number of random orientations, and SVM kernel, in order to improve the accuracy of my object classifier.
My results were as follows, with 'using_hsv=true' and 'bins=32':
64% with SVM kernel=linear, orientations=7
64% with SVM kernel=rbf, orientations=10
74% with SVM kernel=linear, orientations=10
79% with SVM kernel=linear, orientations=20
82% with SVM kernel=sigmoid, orientations=20
84% with SVM kernel=rbf, orientations=20
It seems as though our accuracy is improving with orientations increasing--which makes logical sense as we increase the sample size 
of the training set, our algorithm gets better. Let's continue increasing orientations exposed to the camera, to '40':
88% with SVM kernel=sigmoid, orientations=40
89% with SVM kernel=linear, orientations=40
90% with SVM kernel=rbf, orientations=40
And at '80' orientations per object:
89% with SVM kernel=sigmoid, orientations=80
90% with SVM kernel=rbf, orientations=80
95% with SVM kernel=linear, orientations=80
Make sure you are in the directory where 'model.sav' is located!
$ roslaunch sensor_stick robot_spawn.launch
$ ./object_recognition.py
%%% Putt above all together for our Project's Perception Pipeline

In the previous exercises, we have built out much of our perception pipeline. Building on top of 'object_recognition.py' we'll just 
have to add a new publisher for the RGB-D camera data, add noise filtering, and a few other items. Let's get started!

###  Filter and Segment  ###
Since we have a new table environment, not all of our filters will be effective. I'm going to create a new file to replace 'segmentation.py' 
that will help us filter and segment the new environment I'll call it 'segmentation.py'. Most of the code from 'segmentaion.py' can just be 
copied over, but note the differences in the table height. This means we'll have to adjust the pass through filtering Z-axis minimum and 
maximum values. 
For 'axis-min' I'll try '0.6'(meters) and '1.1'(meters) for 'axis-max'.
We'll also want to publish our RGB-D camera data to a 'ROS-topic` named. Following the syntax of other publishers/subscribers should help us out with this.
Also, we'll have to deal with actual noise in this scene, so we'll apply what we learned earlier on and add in the statistical outlier remover 
to 'segmentation.py' with:
outlier_filter = cloud_filtered.make_statistical_outlier_filter()
outlier_filter.set_mean_k(50)
x = 1.0
outlier_filter.set_std_dev_mul_thresh(x)
cloud_filtered = outlier_filter.filter()

This should leave 'segmentation.py' just about ready to go, but to verify, we have the following items in the 'pcl_callback(pcl_msg)' function:
Convert ROS message to PCL (goes from PC2 data to PCL pointXYZRGB)
Voxel Grid Downsampling  (essentially reduce the resolution of our Point Cloud)
Statistical Outlier Removal Filter  (gets rid of points that aren't apart of a group, determined by proximity)
Pass Through Filtering (a vertical cropping along the Z-axis that cuts down our Point Cloud)
RANSAC Plane Segmentation (helps find a plane - in this case the table)
Extract Inliers and Outliers (Create two point clouds from the RANSAC analysis)
Euclidean Clustering/DBSCAN (Calculates the distance to other points, and based on thresholds of proximity, combines groups of points into clusters that we assume are objects)
Cluster Mask (color each object differently so they are easily distinguished from one another)

We'll then want to make sure this new point cloud is published with:
pcl_cluster_pub = rospy.Publisher("/pcl_world", PointCloud2, queue_size=1)
To test everything out, open a new terminal in the ROS system, and type:
$ cd ~/catkin_ws/
$ roslaunch pr2_robot pick_place_project.launch
Give the PR2 a moment to visualize and map its surroundings. Once it's finished, type:
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts/
$ ./pr2_segmentation.py
You should now be able to go into RViz and click on the new ROS topics:'pcl_world which is our filtered, segmented, and colored group of objects;
With this done, and once you are happy with the results, you can transfer the code. You'll find 'project_template.py' in the '/pr2_robot/scripts' directory. 
This will be our new location for our **primary project code. Port in the code from 'segmentation.py  and 'object_recognition.py' into the respective 'TO DOs'.

### Capture Features ###
Just like we did in the lecture exercise, we need to capture the features of the objects that we will need to find in the project 
(essentially photograph them with the RGB-D camera from many different angles). 
There will be 3 different "worlds". These can be found in the 'config`'directory of 'pr2_robot', and are named 'pick_list_1.yaml', 'pick_list_2.yaml', and 'pick_list_3.yaml'.
Pick out the models from there (should be 8 models)and put their names into the 'models' dictionary of 'capture_features.py'. 
Then, set the number of random orientations each model will be exposed to in the 'for loop'. I used '80' for higher accuracy. 
Lastly, change the very last line of the script to make sure my new feature set is not confused with previous ones. 
Set it to: 'training_set.sav'
We're now ready to capture our objects' features. Launch our handy 'sensor_stick' Gazebo environment with:
$ roslaunch sensor_stick training.launch
Now, change the directory to where you want the feature set 'training_set.sav' to save. I put mine here:
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts/
Then, run the feature capture script:
$ rosrun sensor_stick capture_features.py
This will take some time. Mine took ~10 minutes running on a VM. 
Once finished, you will have your features saved in that directory. We'll be using them in the next step when we train our classifier!

### Train the SVM Classifier  ###
We're now going to train our classifier on the features we've extracted from our objects. 'train_svm.py' is going to do this for us, 
and is located where your directory should still be pointing:
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts/
Open 'train_svm.py' and make sure it is loading the correct feature data we just produced:
training_set = pickle.load(open('training_set.sav', 'rb'))
Then, we're going to check our SVC kernel is set to 'linear':
clf = svm.SVC(kernel='linear')
'rbf' and 'sigmoid' also work well, but with 80 iterations, I've found linear to have the highest accuracy.
Now, from this same directory, run the script:
$ rosrunh sensor_stick train_svm.py
Two confusion matrices will be generated one will be a normalized version. Prior to any training, I had a very confused confusion matrix 
that looked like this:
![](https://github.com/chriswernst/Perception-Udacity-RoboticsND-Project3/blob/master/images/initial_confusion_matrix.png?raw=true)
If you did the above steps correctly, it should look closer to this:
![](https://github.com/chriswernst/Perception-Udacity-RoboticsND-Project3/blob/master/images/linear_conf_matrix_n80_accur93.png?raw=true)

### Check the Labeling in RViz  ###
Now, we can check the labeling of objects in RViz. Exit out of any existing Gazebo or RViz sessions, and type in a terminal:
$ roslaunch pr2_robot pick_place_project.launch
Give it a moment to boot up, then change to our favorite directory, and run our 'project_template.py' code:
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts/
$ rosrun pr2_robot project_template.py
It's important to run 'project_template.py' from this directory because that is where 'model.sav' lives; which is the output of our 
training of the Support Vector Machine classifier.

### Output yaml Files  ###
We now need to load each of the three worlds, and determine which objects to look for once we're there. The launch file 'pick_place_project.launch' 
in 'pr2_robot/launch' at lines '13' and '39' have world parameters you need to alter.
We now need to obtain our picklists from '.yaml' files that are located here: '/pr2_robot/config/'
They are called: 'pick_list_1.yaml', 'pick_list_2.yaml', and 'pick_list_3.yaml'. 
For instance, 'pick_list_1.yaml' looks like this:
object_list:
  - name: biscuits
    group: green
  - name: soap
    group: green
  - name: soap2
    group: red
We want to read from our 'pick_list_X.yaml' files, but then write back to output '.yaml' files that capture the objects we've encountered 
and where they're located.
########################################## Running the Project ################################################################################
runs our 'project_template.py' script. They can be invoked from the terminal
You'll be prompted in one of the terminal windows to click next in RViz.
# World 1 
[image13]: ./src/world1_objects_identified.png
[image14]: ./src/world1_complete.png
#World 2
[image15]: ./src/world2_objects_identified.png
#World 3
[image16]: ./src/world3_objects_identified.png
[image17]: ./src/world3_complete.png

### Debugging ###

### Improving Real-time Factor in Gazebo
In the directory 'pr2_robot/worlds/', you'll find the 'test#.world' files. Search for 'physics', and find the block of code below:
===========================================================
    <physics name='default_physics' default='0' type='ode'>
      <max_step_size>0.005</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>
===========================================================	
This controls the rate of the Gazebo world, and since I was experiencing run-time rates of about '0.2', I needed to increase it by a 
factor of 5 to get to 1.0; so I set the 'max_step_size' to '0.005' from '0.001'. The product of 'max_step_size' and 'real_time_update_rate'
gives you the expected real time factor.

### Improving Accuracy  ###
After some issues with object labeling and recognition, I decided to re-train on the 8 objects, but this time for **200** iterations each 
[image18]: ./src/normalizedConfusionMatrix.png

### Improving Grasping ###
Due to us running a Virtual instance of Linux Ubuntu, our PR2 doesn't always successfully grasp items. Let's put in a delay like we did in project2.
'/pr2_robot/src/' has a c++ file named 'pr2_pick_place_server.cpp'. Line '222' sets the delay of the 'right gripper' and line '311' sets the 
delay after closing the 'left gripper'. At default, these are set to '3.0' seconds, but let's bump that up to '8.0'.
Because we changed one of the build files, we have to re-make the project. Close all terminal windows, open a new one, and type: 
$ cd ~/catkin_ws/`
$ catkin_make

### Improving Classification  ###
Our classifier wasn't initially recognizing the books! We'll change a few features in filtering and clustering to make it perform better.
Voxel DownSampling
LEAF_SIZE = 0.003
Euclidean Clustering
    ec.set_ClusterTolerance(0.01)
    ec.set_MinClusterSize(200)
    # Refering to the minimum and maximum number of points that make up an object's cluster
    ec.set_MaxClusterSize(50000)
Logic for when Object Recognition has Completed Successfully
We'll write some quick logic for when we've done a good enough job to start moving the PR2:
===========================================================================================
    object_list_param = rospy.get_param('/object_list')
    pick_list_objects = []
    for i in range(len(object_list_param)):
        pick_list_objects.append(object_list_param[i]['name'])
        
    print "\nPick List includes: "
    print pick_list_objects
    print "\n"
    pick_set_objects = set(pick_list_objects)
    detected_set_objects = set(detected_objects_labels)

    if detected_set_objects <= pick_set_objects:
        try:
            pr2_mover(detected_objects)
        except rospy.ROSInterruptException:
            pass
===========================================================================================

###  Improving Re-Classification  ###
After much experience in the simulated robot world, I found that objects would get knocked around when the robot would plan an overly 
complex motion path. Therefore, objects would not be located at the initial centroids that were calculated. Although computationally 
more expensive, I decided to relocate the centroid 'for loop' to be inside of the primary loop:
=========================================================
for i in range(0, len(object_list_param)):
    for object in object_list:
        labels.append(object.label)
        points_arr = ros_to_pcl(object.cloud).to_array()
        temp = np.mean(points_arr, axis=0)[:3]
        centroids.append(temp)
=========================================================

### Conclusion  ###
This has been a very comprehensive project by far the most challenging in the course so far. To improve it further, it would be best to 
run the project in a native Linux enivronment, without a Virtual Machine due to much of the errors coming from lag, and the robot not fully 
grasping objects, or simply holding onto them for too long.
There could also be more development done in detecting the 'glue' object that appears in 'world2' and 'world3'. It is a small object, so many 
of the filters keeping the noise out would often exclude it from our analysis.
