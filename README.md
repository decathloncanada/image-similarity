# image-similarity
Library built to perform two tasks: (1) find the products in the catalog most similar to a given one based on their images, and (2) from the image taken by a user, find the products in the catalog which look the most similar. The application of (1) is the recommendation of similar products on Websites and applications. The application of (2) is to provide users the possibility to search for products based on images, similarly to https://images.google.com/.

The algorithm used to calculate image similarity is based on transfer learning. In a nutshell, we compute the features of each image using VGG19 and Inception_Resnet_V2 models. We then calculate the similarity of each image given the cosine distance between their feature vectors, and we rank the images with the highest similarity.

For more information, please contact aicanada@decathlon.net

## Getting started
Make sure you have python 3 and the required libraries properly installed (pip install requirements.txt). You can then git clone the project to the desired location:
```
git clone https://github.com/decathloncanada/image-classification.git
```
To be able to read files from the preprod.datamining bucket on S3, make sure that you have a config.py file in the root folder of the directory, with the following content:
```
aws_access_key_id={AWS_ACCESS_KEY_ID}
aws_secret_access_key={AWS_SECRET_ACCESS_KEY_ID}
```
where you replace {AWS_ACCESS_KEY_ID} and {AWS_SECRET_ACCESS_KEY_ID} with your own keys.

## Extraction of the images from the catalog
The images of the products in the catalog can be extracted by running the following command:
```
python main.py --task extract_images --dpt_number {DPT_NUMBER} --domain_id {DOMAIN_ID}
```
where you replace {DPT_NUMBER} with the number of the department (for instance, 371 for ice hockey) and {DOMAIN_ID} with the id of the domain (for instance, 0341 for decathlon.ca). When making this call, the library will load the following file on S3:
```
preprod.datamining/images/data/pixlIDs_domain_id_{DOMAIN_ID}_dpt_num_department_{DPT_NUMBER}000.gz'
```
This file must contain at least two columns, *product_id_model*, the id of the relevant products in the catalog, and *id*, the pixl ids of the images of these products.

The library will then run through all the pixl ids, and download the images from contents.mediadecathlon.com. The images will be saved in the data/dataset/dpt_number_department{DPT_NUMBER} subdirectory. The images are saved as a .jpg, the name of the image being the product and pixl ids separated by an underscore (for instance, 8156287_476862.jpg for image 476862 of product 8156287). 

## Calculation of the dictionary of most similar products 
To find, for each product, the list of the other products in the catalog which are most similar, run the following call:
```
python main.py --task fit --dpt_number {DPT_NUMBER} --domain_id {DOMAIN_ID} --transfer_model {TRANSFER_MODEL} --number {NUMBER} --data_augmentation {DATA_AUGMENTATION}
```
This will, as described in the previous section, read a file on S3 to extract the id of the relevant products and images. Then, the library runs through all the images, and calculate their feature vectors. These vectors are stored in an sqlite database located in the data/database directory. The name of the database is features.db, and the features are stored in a table named features_{DPT_NUMBER}. Note that the computation of the features can take hours if the dataset contains many (> 1000) images. However, the features can be extracted over multiple sessions as, at each call, only the features not found in the database are calculated.

Data augmentation can be enabled by passing 1 as the --data_augmentation argument - in this case, the features will be calculated for the original image, as well as the image fliped left-right and up-down, and the image rotated by angles of 90, 180 and 270 degrees. 

After the features have been calculated, the library will loop through all products, and find the products in the rest of the catalog which look the most similar. The similarity is based on the cosine distance between the feature vectors calculated using the model provided as the --transfer_model argument (two models are currently supported, VGG19 (VGG) and Inception_Resnet_V2 (Inception_Resnet)). The results are saved in a a file named similar_products_dpt_num_department_{DPT_NUMBER}\_model_\{TRANSFER_MODEL}.pickle, found in the data/trained_models directory. This file contains a dictionary {product: [list of most similar product]} providing, for each product, the list of products (the lenght of this list is provided as the --number argument) which are the most similar. 

## Show the most similar products to a given one
Once the dictionary of most similar products has been built (see the previous section), we can run the following call to print the most similar products to a given one.
```
python main.py --task show_similar_products --product_id {PRODUCT_ID} --dpt_number {DPT_NUMBER} --transfer_model {TRANSFER_MODEL}
```
When making this call, the library first opens the dictionary of most similar products. If the argument --product_id is not provided, one product id is picked randomly, and the most similar products are returned. 

For example, to get the most similar products to product 8401523 (a hockey stick), based on model Inception_Resnet_V2, you can run the following call:
```
python main.py --task show_similar_products --product_id 8401523 --dpt_number 371 --transfer_model Inception_Resnet
```
The response should look like:
```
Most similar product number 1 : 8401522
Most similar product number 2 : 8401525
Most similar product number 3 : 8401526
Most similar product number 4 : 8401482
Most similar product number 5 : 8397732
```
In this example, the algorithm indeed found that the most similar products to product 8401523 are the four other hockey sticks in the catalog (products 8401522, 8401525, 8401526 and 8401482). 

## Find the most similar products to an image
Once the features for all the images have been calculated (see the section *Calculation of the dictionary of most similar products*), the products most similar to a given image can be found by providing the path to the image:
```
python main.py --task search_products --img {IMG} --dpt_number {DPT_NUMBER} --transfer_model {TRANSFER_MODEL}
```
where {IMG} is the path to the given image.

For instance, let's say you have the following image of your used pair of hockey skates:
![Alt text](test/test_image.jpg?raw=true "Title")

And you want to find the products in the catalog with the most similar images (including the images produced by data augmentation). You can run the following command:
```
python main.py --task search_products --img {IMG} --transfer_model Inception_Resnet --dpt_number 371 --data_augmentation 1
```
The response could look like:
```
Most similar product number 1 : 8524152
Most similar product number 2 : 8156286
Most similar product number 3 : 8514405
Most similar product number 4 : 8156287
Most similar product number 5 : 8184005
Most similar product number 6 : 8524150
```
In this example, the library successfully found the hockey skates in the catalog as the most similar products.

## Roadmap
The roadmap for this project includes building a 100% open-source version of the library, as well as a blog post on developers.decathlon.com. Some other improvements which could be done include the possibility to remove the background color in the pictures taken by the user (when the catalog only contains products with a white background), and identifying the most similar products using an ensemble approach (when there is more than one picture per product in the catalog).
