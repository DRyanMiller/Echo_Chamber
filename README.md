# Overcoming Echo Chambers in Recommendation Systems (Using Movie Ratings)
==============================
# Introduction
Automated recommendation systems have become a part of everyday life online. Whether its Amazon recommending products, Facebook recommending news articles, or Netflix recommending movies, we frequently interact with recommendation systems to the benefit of both consumers and businesses. A drawback of these systems is their potential, over time, to limit the diversity of items recommended as they narrow in on a user's preferences. This process results echo chambers (exposure only to recommendations from others like yourself), feedback loops (recommendations that reinforce your recorded preferences), and filter bubbles (exposure only to recommendations similar to items you have historically liked). These phenomenon result in users only being exposed to recommendations that reinforce their own biases or purchase patterns. 

In this project, I build a system that provides additional recommendations based on the preferences of others who are different, but not too different, from the user. These additional recommendations are identified by clustering users based on the latent user factors obtained from an alternating least squares (ALS) decomposition of the ratings matrix. Once clusters are created, the top-rated items for each cluster (based on the cluster centroid) are identified. The top-rated items in each cluster form the set of items from which recommendations may be extracted. A user's recommendations are extracted from the two clusters nearest to the user's cluster. The complete system provides a user two sets of recommendations. The first set comes from the traditional ALS model and the second set comes from the augmented model. 

# Business Understanding

Automated recommendations systems are commonplace among businesses with an online presence. They are used to recommend products, news articles, and even recipes.  These systems aim to increase consumer engagement and purchasing by recommending new items the consumer will presumably like. A common method for producing recommendations is collaborative filtering which generates recommendations based on the combined preferences of the consumer requesting recommendations and those of other consumers. The ability of collaborative filtering (CF) to make good recommendations (i.e., recommendations the consumer will like) improves as additional preferences are recorded. A limitation of collaborative filtering is that over time the recommendations become narrower in scope resulting in echo chambers, feedback loops, and filter bubbles. 

The negative effects of echo chambers, feedback loops, and filter bubbles are felt both socially and economically. Socially, these effects limit exposure to diverse and contrary ideas leading to a more divided and divisive society. Economically, these effects limit the variety of products to which consumers are exposed, limiting a business' potential to increase product demand.

The following sections describe a method for overcomming the limitations of CF recommendation systems by enhancing an ALS algorithm to provide more diverse recommendations. The goal is to provide additional recommendations that are qualitatively different from the items recommended by the ALS model, but not too different. Four metrics are used to evaluate the ability of the augmented model to provide diverse (but not too diverse) recommendations.

The first metric evaluates the overlap between the ALS recommendations and recommendations from the augmented system that are intended to match the ALS recommendations. A high degree of overlap indicates that the augmented model can correctly classify users into group's with similar item preferences. 

The second metric evaluates the overlap between the ALS recommendations and the set of diverse recommendations generated by the augmented model. A low degree of overlap indicates that the augmented model can identify recommendations that are different from the ALS recommendations. 

The third metric evaluates the whether the recommendations from the augmented model are qualitatively different from those of the ALS model. This metric utilizes a t-test for the difference in means. A negative and statistically significant t-statistic indicates items recommended by the augmented model are qualitatively different from those recommended by the ALS model. 

The final metric evaluates whether the items recommended by the augmented model are too qualitatively different from the ALS recommendations. This metric utilizes a t-test for the difference in differences. A positive and statistically significant t-statistic indicates the augmented recommendations are not as qualitatively different from the ALS recommendations as other potential recommendations. This metric is taken as a indicator that the augmented recommendations are not so different from the ALS recommendations as to be uninteresting to the user. Additional information about the evaluation metrics used in this project can be found in evaluation section below and in the evaluation_drm_ec notebook. 

# Data Understanding and Preparation

The project utilizes the <a href='https://grouplens.org/datasets/movielens/'>MovieLens</a> dataset (Harper & Konstan, 2015). The MovieLens dataset is an open source data set containing 27,753,444 movie ratings from 283,228 users for 58,098 movies. The ratings are on a five-star scale ranging from 0.5 stars to 5 stars in 0.5 star increments. The ratings include data from January 09, 1995 to September 26, 2018 for a random sample of users with at least 1 movie rating. An additional file contains data about the movies including their titles, release years, and genres. The MovieLens usage license prohibits the redistribution of the data without separate permission, but can be freely downloaded from the license owners at the above link.

To prepare the ratings data for analysis, I removed users with fewer than ten ratings and movies with fewer than 5 ratings. The processed dataset contains 27,510,397 ratings for 243,658 unique users and 28,755 movies. The movies data was processed to produce two new datasets. The first contains only movies with more than 50 ratings. These "most rated" movies are used as the basis for all recommendations. This is done to ensure that recommendations contain only movies with high average ratings from a large number of users preventing movies with high average ratings from only a few users from skewing the recommendation set. The second contains the top 100 most-rated movies (based on average rating). The top 100 movies are used to randomly select movies for a new user to rate. This is done to ensure a user is presented with familiar movies to rate. Detailed code for processing the raw data can be found in the clean_drm_ec notebook. 

After fitting the ALS model (see below) using Amazon Web Services (AWS) Elastic MapReduce (EMR), further data processing is required. The user and item factor outputs of the ALS model are saved as sets of files (a function of the MapReduce process). To work with the user and item factors outside of AWS EMR, each set of files needs to be combined into a single csv file. In addition, the user factors must be scaled for use in the clustering algorithm. Detailed code for processing the ALS output from AWS can be found in the AWS_data notebook.

# Modeling

![image](https://github.com/DRyanMiller/Overcoming-Echo-Chambers-in-Recommendation-Systems/blob/master/reports/figures/RecProcess.png)

The above diagram outlines the process the augmented recommendation system follows to generate recommendations. Throughout this process, the augmented recommendation system utilizes three machine learning algorithms to produce movie recommendations: Alternative Least Squares (ALS), KMeans, and a Gradient Boosting Machine. The following explains how the process works and how the machine learning algorithms play into the process. 

Before recommendations can be made, the ALS and KMeans algorithms are used to generate user and item factors (ALS) and then cluster users into groups (KMeans) using the user factors from the ALS model. To generate the user and item factors, an ALS model is fit to the processed ratings data (see the clean_drm_ec notebook for the code to process the raw data). The ALS model can be run on an AWS EMR cluster using the code found in SparkALS.py. Once the output is saved, the factors can be processed using the AWS_data notebook. 

The processed ALS factors are then fed into a KMeans algorithm on the AWS cluster to determine the optimal number of clusters. Code for completing the KMeans evaluation can be found in SparkALS.py. The errors from the KMeans models are then fed into Section 3 of the model_drm_ec notebook and the optimal number of clusters chosen using the elbow plots produced by the evaluate_spark_kmeans function in the same notebook.

The KMeans algorithm is then rerun on a local machine to produce the predicted clusters for all users in the ratings dataset and the cluster centroids for each cluster (see Section 4 of the models_drm_ec notebook). A gradient boosting machine is then trained using the clusters predicted by the KMeans model (see Section 5 of the models_drm_ec notebook). Lastly, the distances between the cluster centroids are calculated and saved for later use (see Section 6 of the models_drm_ec notebook). 

Once these models are trained, the process of making recommendations begins with a user ranking ten movies. These rankings are combined with the item factors from the ALS, using the process described in Zhou, Wilkinson, Schreiber, and Pan (n.d.), to predict user ratings for all movies in the processed dataset. The user's highest rated movies become the ALS recommendations. 

To generate the augmented recommendations, the user is classified into a cluster using the the user factors generated above and the trained gradient boosting machine. Augmented recommendations are then extracted from the top rated movies from the two clusters nearest to the user's predicted cluster. The recommendations are generated by randomly selecting movies from the 100 top rated movies in each of these two clusters. A total of ten recommendations are provided, six from the nearest cluster and four from the next nearest cluster. The number of recommendations provided from each cluster is weighted so that as the distance between clusters increases (i.e., the differences between the clusters increases), fewer recommendations are contributed to the final recommendations list. 

# Evaluation

As discussed above, the augmented recommendation model was evaluated using four metrics. The metrics and the model's performance on each is described below. The performance of the augmented model was evaluated by applying the metrics to a random sample of 1000 users from the MovieLens dataset. 

**The First Metric**

The first metric looks at a user's Top 100 ALS recommendations (based on predicted user ratings) and the Top 100 recommendations from the user's cluster (based on the cluster centroid's ratings). The proportion of the ALS recommended movies also found in the cluster recommendations should be high. A high proportion indicates that the user's cluster is representative of the user's movie preferences.

![image](https://github.com/DRyanMiller/Overcoming-Echo-Chambers-in-Recommendation-Systems/blob/master/reports/figures/Metric1.png)

The model did not perform as expected on this metric. As the graph above demonstrates, for most of the sample less than 10% of the ALS recommended movies were also recommended by the user's cluster. This may be a result of using the cluster centroid to identify recommendations. A different method of identifying the top rated movies in each cluster (e.g., average ratings of users in the cluster) may produce better results. 

**The Second Metric**

The second metric looks at a user's Top 100 ALS recommendations (based on predicted user ratings) and the top recommendations from the augmented model (based on the cluster centroid's ratings for the two clusters nearest to the user's cluster). The proportion of the ALS recommended movies also found in the augmented model recommendations should be small. A low proportion indicates that the recommendations from the augmented model differ from those produced by the ALS model.

![image](https://github.com/DRyanMiller/Overcoming-Echo-Chambers-in-Recommendation-Systems/blob/master/reports/figures/Metric2.png)

The model performed as expected on this metric. As demonstrated in the above graph, for most of the sample less than 10% of the ALS recommended movies were also recommended by the augmented model. 


**The Third Metric**

The third metric utilizes the distance between movies (based on the ALS item factors) to evaluate the extent to which the movies from the augmented model are qualitatively different from the movies from the ALS model. For this metric, the mean squared distance between the Top 100 movies from the ALS model (excluding the distance between a movie and itself) is calculated for each user in the sample. Likewise, the mean squared distance between each of the Top 100 ALS movies and each of the top movies from augmented model is calculated for each user in the sample. The difference between these mean squared distances for the sample are tested using a t-test. A negative and statistically significant t-statistic indicates the mean difference between the two sets of recommendations is greater than the mean difference within the ALS recommendations and, thus, the movies from the augmented model are qualitatively different from the movies from the ALS model.

As expected, the ALS recommendations were qualitatively different from the augmented recommendations (t-statistic = -66.2, p = 0.000).

**The Fourth Metric**

The final metric evaluates whether the movies recommended by the augmented model are too qualitatively different from the ALS recommendations. The metric tests the difference between two differences: the difference between the ALS recommendations and the augmented model's recommendations and the difference between the ALS recommendations and recommendations from the two clusters furthest away from the user's cluster. These differences are calculated in the same way as described above the for third metric. The difference in differences is tested using a t-test. A positive and statistically significant t-statistic indicates the difference between the ALS recommendations and those from the furthest clusters is greater than the distance between the ALS recommendations and the augmented model's recommendations. A greater distance is evidence that the augmented model recommendations are not too different from the ALS recommendations since the movies recommended by the furthest clusters are more different.

As expected, the augmented recommendations are less different from the ALS recommendations than recommendations generated from the furthest clusters (t-statistic 5.5, p = 0.000)

# Next Steps

Future work can improve upon this project by:
    1. Enhancing the ALS model through incorporation of more sophisticated algorithm features (e.g., a time 
       component),
    2. Improving the performance of the gradient boosting machine classification through parameter tuning, 
    3. Improving the performance of the augmented recommendation model by changing how 
       movie recommendations are generated from the clusters, and
    4. Deploying the augmented recommendation function as a web application.
    
# References

F. Maxwell Harper and Joseph A. Konstan. 2015. The MovieLens Datasets: History and Context. ACM Transactions on Interactive Intelligent Systems (TiiS) 5, 4, Article 19 (December 2015), 19 pages. DOI=http://dx.doi.org/10.1145/2827872.

Zhou, Y., Wilkinson, D., Schreiber, R. & Pan, R. (n.d.). Large-scale Parallel Collaborative Filtering for the Netflix Prize. Retrieved from https://endymecy.gitbooks.io/spark-ml-source-analysis/content/推荐/papers/Large-scale%20Parallel%20Collaborative%20Filtering%20the%20Netflix%20Prize.pdf.

==============================

Project Organization
------------
The directory structure for this projects is below. Brief descriptions follow the diagram.

```
Echo_Chamber
├── LICENSE
│
├── Makefile  : Makefile with commands to perform selected tasks, e.g. `make clean_data`
│
├── README.md : Project README file containing description provided for project.
│
├── .env      : file to hold environment variables (see below for instructions)
│
├── test_environment.py
│
├── data
│   ├── processed : directory to hold interim and final versions of processed data.
│   ├── raw : holds original data. This data should be read only.
│   └── test : holds a smaller dataset used to test the code and tune the ALS model.
│
├── models  : holds classification model object and .py file with code used to implement ALS and KMeans in PySpark on AWS.
│
├── notebooks : holds notebooks for eda (clean_drm_ec and AWS_data), modeling (model_drm_ec), evaluation (evaluation_drm_ec),
│               and presentation (drm_ec)
│
├── reports : slides for presentations and accompanying visualizations.
│   └── figures
│
├── requirements.txt
│
├── setup.py
│
├── src : local python scripts. Pre-supplied code stubs include clean_data, model and visualize.
    ├── recapp.py
    ├── __settings__.py
    ├── custom.py
    ├── make_data.py
    └── model.py
     

```
