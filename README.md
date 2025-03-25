# Logo Scraper & Clustering

A complete pipeline for **collecting, processing and clustering logos** using **Web Scrapping, PyTorch(ResNet50) and FAISS Clustering**.

## Features
- Extracts a list of domains from a Parquet file
- Downloads logos using Clearbit API
- Retries failed downloads using Web Scraping
- Extract visual features using ResNet50
- Applies clustering using FAISS
- Visualizes and saves results in organized files

## Dependencies
- torch, torchvision - ResNet50(feature extraction)
- requests -> Image downloading
- beautiful4soup -> Web Scraping
- faiss-cpu -> Efficient image clustering
- cairo-svg -> SVG to PNG conversion

## Step by Step Process

### 1. Extracting Domains from Parquet File
In the read_file() function, my script reads logos.snappy.parquet file and puts the domains in a set, so we will iterate only through unique sites, for better perfomance. The list of websites is saved in *logos.txt* to prevent excessive memory usage and allow batch processing without keeping all domains in RAM. Storing them in a file ensures scalability, especially when dealing with a large number of domains.

### 2. Downloading using Clearbit
Downloading over 3400 unique websites sequentially would be highly time-consuming(about 1 hour), so I decided to take a multi-threaded approach to download logos using Clearbit API. I've chosen to use this API in the first part instead of just web scraping because is faster, reliable, minimizes error and also reduces the cost of processing, where there isn't the need for parsing HTML structures no more. I was able to extract 2900+ logos using this method in a about 1 minute and 30 seconds. We will see later why this approach was better not just in theory but also in practice than web scraping. I saved the logos in a directory using the names of the domains they were extracted from. I'm also saving the domains where Clearbit wasn't able to extract anything.

### 3. Downloading using Web Scraping
Clearbit is not able to extract logos that have the .svg extension so I was able to extract the logos from thos domains that use svg for logos using web scraping. I'm using a multi-threaded approach as well. I've extracted the html source of the each domain and when I'm parsing them, I'm looking to locate **img** tags that contain **logo** in their **src, alt, or class** attributes. If a SVG logo is found, it is converted to PNG using **cairosvg**. If a logo is successfully found and downloaded, it is saved in **logos/**. It takes around 1 minute and 45 seconds to download less than 300 logos, so it's way slower than using Clearbit.

### 4. Extracting Logo Features
Each downloaded logo is processed through a pre-trained ResNet50 model (the final layer is removed) for extracting deep feature embeddings. The images are resized to 224x224, to fit the need of ResNet50 and after that they are normalized and transformed to tensors. The output is a feature vector represnting the logo's unique characteriscs. Also, if there's available an Nvidia GPU, we are gonna use it to extract the embeddings so the processing time is significantly reduced.

### 5. Clustering with FAISS
To organize the logos in cluster, the script uses **FAISS (Facebook AI Similarity Search)**. After the features are obtained using ResNet50, I'm indexing them using IndexFlat32. This index is a flat index that computes the Euclidean distance to measure the similarity between the vectors. Next, I set up the clustering process by defining the number of clusters to create (num_clusters). The script then calls the **FAISS Clustering API**, which uses a **k-means** algorithm to assign each feature vector to the nearest cluster center. The k-means algorithm initially selects random cluster centers and iteratively updates them based on the data points (feature vectors/embeddings). This process continues until the centroids stabilize, meaning that assigning feature vectors to clusters no longer results in significant changes.

### 6. Saving and Visualizing Clusters
The logos in each cluster are stored in separate directories under **output_cluster/**. I'm using Matplotlib for visualizing cluster images. For reference, up to 80 images per cluster are saved.