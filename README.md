[![Udacity - Robotics NanoDegree Program](https://s3-us-west-1.amazonaws.com/udacity-robotics/Extra+Images/RoboND_flag.png)](https://www.udacity.com/robotics)

# Project: 3D Perception with PR2 Robot

### Writeup by Muthanna A. Attyah
### Feb 2018
<p align="center"> <img src="./misc/pr2.png"> </p>


## [Rubric](https://review.udacity.com/#!/rubrics/1067/view) Points



# The Perception Pipeline

Following sections will explain the different stages of the percption pipeline used to detect objects before starting the pick and place robot movement.


## Select topic and convert ROS msg to PCL data

The first step in the perception pipeline is to subscribe to the the camera data (point cloud) topic `/pr2/world/points` from which we will get a point cloud with noise as seen below:

<p align="center"> <img src="./misc/rviz_world_points.png"> </p>

before we can process the data we need to convert it from **ROS PointCloud2** message to a **PCL PointXYZRGB** format using the following code:

```python
cloud_filtered = ros_to_pcl(ros_pcl_msg)
```

## Statistical Outlier Filtering

First filter is the  **PCL’s Statistical Outlier Removal** filter. in this filter for each point in the point cloud, it computes the distance to all of its neighbors, and then calculates a mean distance. By assuming a Gaussian distribution, all points whose mean distances are outside of an interval defined by the global distances mean+standard deviation are considered to be outliers and removed from the point cloud.

Code is as following:

```python
    # Create a statistical filter object: 
    outlier_filter = cloud_filtered.make_statistical_outlier_filter()
    # Set the number of neighboring points to analyze for any given point
    outlier_filter.set_mean_k(3)
    # Set threshold scale factor
    x = 0.00001
    # Any point with a mean distance larger than global (mean distance+x*std_dev)
    # will be considered outlier
    outlier_filter.set_std_dev_mul_thresh(x)
    # Call the filter function
    cloud_filtered = outlier_filter.filter()
```
Mean K = 3 was the best value I found to almost remove all noise pixels. any value higher than 3 was leaving some of the noise pixels behind. x was selcted to be 0.00001.

following image is showing result after removal of noise:

<p align="center"> <img src="./misc/rviz_statstical_filter.png"> </p>

## Voxel Grid Downsampling

2nd stage is **Voxel Grid Downsampling** filter to derive a point cloud that has fewer points but should still do a good job of representing the input point cloud as a whole. This is done to reduce required computation power without impacting the final results. Code is as following:

```python
    # Create a VoxelGrid filter object for our input point cloud
    vox = cloud_filtered.make_voxel_grid_filter()
    # Choose a voxel (also known as leaf) size
    # 1 means 1mx1mx1m leaf size   
    LEAF_SIZE = 0.005  
    # Set the voxel (or leaf) size  
    vox.set_leaf_size(LEAF_SIZE, LEAF_SIZE, LEAF_SIZE)
    # Call the filter function to obtain the resultant downsampled point cloud
    cloud_filtered = vox.filter()
```
After trying diffrent sizes I have selected leaf size 0.005 to avoid any impact on point cloud details.

result is as shown in below image:

<p align="center"> <img src="./misc/rviz_voxel_filter.png"> </p>

## PassThrough Filter

3rd stage is PassThrough filter which works much like a cropping tool allowing to crop any given 3D point cloud by specifying an axis with cut-off values along that axis. The region you allow to pass through, is often referred to as region of interest.

In our case I have applyed the filter two times, 1st one along **Z axis** to select only the table top and objects on it as shown in below code:

```python
    # Create a PassThrough filter object.
    passthrough = cloud_filtered.make_passthrough_filter()
    # Assign axis and range to the passthrough filter object.
    filter_axis = 'z'
    passthrough.set_filter_field_name(filter_axis)
    axis_min = 0.6095
    axis_max = 1.1
    passthrough.set_filter_limits(axis_min, axis_max)
    # Use the filter function to obtain the resultant point cloud. 
    cloud_filtered = passthrough.filter()
```
axis_min and axis_max was picked from RViz directly by reading the edge pixels values.

<p align="center"> <img src="./misc/rviz_passthrough_z_filter.png"> </p>

2nd one is along **Y axis** to remove the unwanted left and right edges of the table. Code is as following:

```python
    # Create a PassThrough filter object.
    passthrough = cloud_filtered.make_passthrough_filter()
    # Assign axis and range to the passthrough filter object.
    filter_axis = 'y'
    passthrough.set_filter_field_name(filter_axis)
    axis_min = -0.456
    axis_max =  0.456
    passthrough.set_filter_limits(axis_min, axis_max)
    # Use the filter function to obtain the resultant point cloud. 
    cloud_filtered = passthrough.filter()
```
again axis_min and axis_max was selected from RViz by reading the values of the edge pixels.

<p align="center"> <img src="./misc/rviz_passthrough_y_filter.png"> </p>


## RANSAC Plane Segmentation

Next we need to remove the table itself from the scene. To do this we will use a popular technique known as Random Sample Consensus or "RANSAC". RANSAC is an algorithm, that we can use to identify points in our dataset that belong to a particular model. In the case of the 3D scene we are working with here, the model we choose could be a plane, a cylinder, a box, or any other common shape. Since the top of the table in the scene is the single most prominent plane, after ground removal, we can effectively use RANSAC to identify points that belong to the table and discard/filter out those points.

code is as following:

```python
    # Create the segmentation object
    seg = cloud_filtered.make_segmenter()
    # Set the model you wish to fit 
    seg.set_model_type(pcl.SACMODEL_PLANE)
    seg.set_method_type(pcl.SAC_RANSAC)
    # Max distance for a point to be considered fitting the model
    # Experiment with different values for max_distance 
    # for segmenting the table
    max_distance = 0.006
    seg.set_distance_threshold(max_distance)
    # Call the segment function to obtain set of inlier indices and model coefficients
    inliers, coefficients = seg.segment()
    # Extract inliers (Table)
    extracted_table   = cloud_filtered.extract(inliers, negative=False)
    # Extract outliers (Tabletop Objects)
    extracted_objects = cloud_filtered.extract(inliers, negative=True)
```

Image of the objects:
<p align="center"> <img src="./misc/rviz_RANSAC_objects.png"> </p>

Image of the table:
<p align="center"> <img src="./misc/rviz_RANSAC_table.png"> </p>

## Euclidean Clustering

Last filtering step is to use **PCL's Euclidean Clustering** algorithm to segment the remaining points into individual objects. code is as following:

```python
    white_cloud = XYZRGB_to_XYZ(extracted_objects)
    tree = white_cloud.make_kdtree()
    # Create a cluster extraction object
    ec = white_cloud.make_EuclideanClusterExtraction()
    # Set tolerances for distance threshold 
    # as well as minimum and maximum cluster size (in points)
    ec.set_ClusterTolerance(0.03)
    ec.set_MinClusterSize(10)
    ec.set_MaxClusterSize(9000)
    # Search the k-d tree for clusters
    ec.set_SearchMethod(tree)
    # Extract indices for each of the discovered clusters
    cluster_indices = ec.Extract()
```

## Create Cluster-Mask Point Cloud to visualize each cluster separately

Then we use the following code to add a color for each segmented object:

```python
    # Assign a color corresponding to each segmented object in scene
    cluster_color = get_color_list(len(cluster_indices))
    color_cluster_point_list = []
    for j, indices in enumerate(cluster_indices):
        for i, indice in enumerate(indices):
            color_cluster_point_list.append([white_cloud[indice][0],
                                             white_cloud[indice][1],
                                             white_cloud[indice][2],
                                             rgb_to_float(cluster_color[j])])
    # Create new cloud containing all clusters, each with unique color
    cluster_cloud = pcl.PointCloud_PointXYZRGB()
    cluster_cloud.from_list(color_cluster_point_list)
 ```

resulting objects image:

<p align="center"> <img src="./misc/rviz_euclidean_cluster.png"> </p>


## Converts a pcl PointXYZRGB to a ROS PointCloud2 message

Befor we can publish the processed point clouds we need to convert the format back from **PCL PointXYZRGB** to **ROS PointCloud2** message:

```python
    ros_cloud_objects = pcl_to_ros(extracted_objects)
    ros_cloud_table   = pcl_to_ros(extracted_table)
    ros_cluster_cloud = pcl_to_ros(cluster_cloud)
```

## Publish ROS messages

finally we publish to required topics:

```python
    pcl_objects_pub.publish(ros_cloud_objects)
    pcl_table_pub.publish(ros_cloud_table)
    pcl_cluster_pub.publish(ros_cluster_cloud)
```


## Object Prediction

    Having the segmented objects, now we can use the SVM algorithm to predict each object:

```python
    detected_objects_labels = []
    detected_objects = []

    for index, pts_list in enumerate(cluster_indices):

        #----------------------------------------------------------------------------------
        # Grab the points for the cluster from the extracted_objects
        #----------------------------------------------------------------------------------
        pcl_cluster = extracted_objects.extract(pts_list)
        # Convert the cluster from pcl to ROS using helper function
        ros_cluster = pcl_to_ros(pcl_cluster)

        #----------------------------------------------------------------------------------
        # Generate Histograms
        #----------------------------------------------------------------------------------
        # Color Histogram
        c_hists = compute_color_histograms(ros_cluster, using_hsv=True)
        # Normals Histogram
        normals = get_normals(ros_cluster)
        n_hists = compute_normal_histograms(normals)
        
        #----------------------------------------------------------------------------------
        # Generate feature by concatenate of color and normals.
        #----------------------------------------------------------------------------------
        feature = np.concatenate((c_hists, n_hists))
        
        #----------------------------------------------------------------------------------
        # Make the prediction
        #----------------------------------------------------------------------------------
        # Retrieve the label for the result and add it to detected_objects_labels list
        prediction = clf.predict(scaler.transform(feature.reshape(1,-1)))
        label = encoder.inverse_transform(prediction)[0]
        detected_objects_labels.append(label)

        #----------------------------------------------------------------------------------
        # Publish a label into RViz
        #----------------------------------------------------------------------------------
        label_pos = list(white_cloud[pts_list[0]])
        label_pos[2] += .2
        object_markers_pub.publish(make_label(label,label_pos, index))


        # Add the detected object to the list of detected objects.
        #----------------------------------------------------------------------------------
        do = DetectedObject()
        do.label = label
        do.cloud = ros_cluster
        detected_objects.append(do)
        
    rospy.loginfo('Detected {} objects: {}'.format(len(detected_objects_labels), detected_objects_labels))

    #----------------------------------------------------------------------------------
    # Publish the list of detected objects
    #----------------------------------------------------------------------------------
    detected_objects_pub.publish(detected_objects)
  ```

Following image showing the objects with predicted names:

<p align="center"> <img src="./misc/rviz_predicted_cluster.png"> </p>


## Running the 3 worlds tests

Next we will be using the above mentioned pipeline to test all of the three worlds. We select the required test world by changing the following lines in `pick_place_project.launch`
    
```xml
    <!--TODO:Change the world name to load different tabletop setup-->
    <arg name="world_name" value="$(find pr2_robot)/worlds/test1.world"/>
```
and

```xml
    <!--TODO:Change the list name based on the scene you have loaded-->
    <rosparam command="load" file="$(find pr2_robot)/config/pick_list_1.yaml"/>

```

Results are as following:

## Test 1 - Training
| Test 1 | Values |
|-|-|
| Features in Training Set: | **51** |
| Invalid Features in Training set: | **0** |
| Scores: | **[1.  0.8 0.9 0.9 0.9]** |
| Accuracy: | **0.90 (+/- 0.13)** |
| accuracy score: | **0.9019607843137255** |


<p align="center"> <img src="./misc/Figure_1_test_1.png"> </p>

<p align="center"> <img src="./misc/Figure_2_test_1.png"> </p>

image of predicted objects:

<p align="center"> <img src="./misc/rviz_predicted_objects_1.png"> </p>

## Test 2 - Training
| Test 2 | Values |
|-|-|
| Features in Training Set: | **85** |
| Invalid Features in Training set: | **0** |
| Scores: | **[0.82352941 0.82352941 0.82352941 0.64705882 0.94117647]** |
| Accuracy: | **0.81 (+/- 0.19)** |
| accuracy score: | **0.8117647058823529** |


<p align="center"> <img src="./misc/Figure_1_test_2.png"> </p>

<p align="center"> <img src="./misc/Figure_2_test_2.png"> </p>

image of predicted objects:

<p align="center"> <img src="./misc/rviz_predicted_objects_2.png"> </p>

## Test 3 - Training
| Test 3 | Values |
|-|-|
| Features in Training Set: | **136** |
| Invalid Features in Training set: | **0** |
| Scores: | **[0.78571429 0.77777778 0.81481481 0.77777778 0.81481481]** |
| Accuracy: | **0.79 (+/- 0.03)** |
| accuracy score: | **0.7941176470588235** |



<p align="center"> <img src="./misc/Figure_1_test_3.png"> </p>

<p align="center"> <img src="./misc/Figure_2_test_3.png"> </p>

image of predicted objects:

<p align="center"> <img src="./misc/rviz_predicted_objects_3.png"> </p>

## 
## Results yaml files:

The output yaml files are on the following links:

[**output_1.yaml**](./misc/output_1.yaml)

[**output_2.yaml**](./misc/output_2.yaml)

[**output_3.yaml**](./misc/output_3.yaml)


### Issues faced during project

<p align="center"> <img src="./misc/gazebo.png"> </p>


* When compliling using `catkin_make` I used to get error "cannot convert to bool". I resolved it by adding `static_cast<bool>()`. [see this ](https://robotics.stackexchange.com/questions/14801/catkin-make-unable-to-build-and-throws-makefile138-recipe-for-target-all-fa)

### Future improvements

* 
