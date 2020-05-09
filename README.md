Author: Mitch Fairweather
Research Paper that was followed: https://arxiv.org/pdf/1812.07606.pdf

# Step 1: importing all of the necessary packages. 


```python

import os, math
from os import makedirs
from os.path import expanduser, exists, join

import argparse
import PIL
from PIL import Image
from queue import Queue
from threading import Thread

import keras
from keras.preprocessing.image import ImageDataGenerator
from keras.preprocessing.image import img_to_array
from keras.preprocessing.image import load_img
from keras.applications import inception_v3
from keras.applications.inception_v3 import InceptionV3
from keras.layers import GlobalAveragePooling2D
from keras.models import Model
from keras.models import Sequential
from keras.layers import Dense, Conv2D, MaxPool2D , Flatten
from keras.optimizers import Adam
from keras.metrics import categorical_crossentropy
from keras import optimizers
from keras.applications.vgg16 import VGG16
from keras.preprocessing import image
from keras.applications.vgg16 import preprocess_input
from keras.layers import Input, Flatten, Dense
from keras.callbacks import ModelCheckpoint, EarlyStopping

import sklearn
from sklearn.model_selection import train_test_split
from sklearn import metrics
from sklearn.metrics import classification_report


import warnings
warnings.filterwarnings('ignore')

import numpy as np 
from numpy import random

import pandas as pd

import matplotlib.pyplot as plt

import cv2

from tqdm import tqdm

import numpy as np

import itertools

```

    Using TensorFlow backend.
    

# Step 2: Converting the 10868 byte files into RGB images. 

I used the the github I found at https://github.com/ncarkaci/binary-to-image/blob/master/binary2image.py as a starting point, and slightly tweaked it for my needs.  

The function below will read in the binary data files, and basically converts the individual bytes into unicode format, and appends it to the "binary_values" array. 

#### Converting byte files binary

```python
def getBinaryData(filename):

    binary_values = []

    with open(filename, 'rb') as fileobject:

        # read file byte by byte
        data = fileobject.read(1) # initializes the reading byte values by reading the first

        while data != b'': # While the data object isn't empty 
            binary_values.append(ord(data)) #returns integer representing the Unicode code point of the character
            data = fileobject.read(1) # Reads the next line

    return binary_values # returning the binary values array 
```

#### Using Binary Arrays to Calculate Image Size

The function below sets up the data for converting to an image. Using the size of the binary file data, it calculates the necessary height and width of the image to be created. I used the same specifications found in the original paper. 


```python
def get_size(data_length, width=None):

    if width is None: # if the user doesn't pass in a hardcoded width value. 

        size = data_length

        if (size < 10240):
            width = 32
        elif (10240 <= size <= 10240 * 3):
            width = 64
        elif (10240 * 3 <= size <= 10240 * 6):
            width = 128
        elif (10240 * 6 <= size <= 10240 * 10):
            width = 256
        elif (10240 * 10 <= size <= 10240 * 20):
            width = 384
        elif (10240 * 20 <= size <= 10240 * 50):
            width = 512
        elif (10240 * 50 <= size <= 10240 * 100):
            width = 768
        else:
            width = 1024

        height = int(size / width) + 1

    else:
        width  = int(math.sqrt(data_length)) + 1
        height = width

    return (width, height)
```

#### Saving the greyscale or RGB image as a png file

The below function, when called, will save the converted binary to image file into a .png file. 


```python
def save_file(filename, data, size, image_type):

    try:
        image = Image.new(image_type, size)
        image.putdata(data)

        # setup output filename
        dirname     = os.path.dirname(filename)
        name, _     = os.path.splitext(filename)
        name        = os.path.basename(name)
        imagename   = dirname + os.sep + image_type + os.sep + name + '.png'
        os.makedirs(os.path.dirname(imagename), exist_ok=True)

        image.save(imagename)
        print('The file', imagename, 'saved.')
    except Exception as err:
        print(err)
```

#### Creating Grey Scale Image from binary values

The below function converts the binary files into a grescale image. I have commented out the save_file portion, as I will only be using the RGB png files for training and testing. 


```python
def createGreyScaleImage(filename, width=None):
    
    greyscale_data  = getBinaryData(filename)
    size            = get_size(len(greyscale_data), width)
    #save_file(filename, greyscale_data, size, 'L')
```
#### Creating RGB Image from binary values

The below function is the important one, converting the 24 bit binary data into RGB format, and then saving the file as a RGB png file. 


```python
def createRGBImage(filename, width=None):
    """
    Create RGB image from 24 bit binary data 8bit Red, 8 bit Green, 8bit Blue
    :param filename: image filename
    """
    index = 0
    rgb_data = []

    # Read binary file
    binary_data = getBinaryData(filename)

    # Create R,G,B pixels
    while (index + 3) < len(binary_data):
        R = binary_data[index]
        G = binary_data[index+1]
        B = binary_data[index+2]
        index += 3
        rgb_data.append((R, G, B))

    size = get_size(len(rgb_data), width)
    save_file(filename, rgb_data, size, 'RGB')
```

# Step 3: Creating my Data

#### Converting all of my byte files to RGB images

The below function is the wrapper for all of the above functions. It gets the binary data, finds the correct image size based on the file size, and then converts it to an RGB image. 


```python
def main(filename):
    Binary_Values = getBinaryData(filename)
    size = get_size(data_length = len(Binary_Values))
    
    rgbImage = createRGBImage(filename)

    return Binary_Values
```

With over 10868 binary files, I iterated over all of the files in the specified directory and called the main function at each iteration. The end result from this is a folder in the trainBytes directory called RGB holding 10868 total png files. I made the block below a markdown cell to prevent it from running again, as it took nearly 24 hours the initial pass through. 


```python
directory = r"C:\Users\mitch\ISA480FinalProject\trainBytes"
for filename in os.listdir(directory):
    main(filename = directory + os.sep + filename)

```

#### Bringing in Data Labels 

From the Kaggle Website, there is another csv file that holds each file's ID along with its malware class noted by a 1 through 9 integer. 


```python
train_malware = pd.read_csv(r"C:\Users\fairwemr\ISA 480\FinalProject\trainLabels.csv")
train_malware
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Id</th>
      <th>Class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>01kcPWA9K2BOxQeS5Rju</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>04EjIdbPV5e1XroFOpiN</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>05EeG39MTRrI6VY21DPd</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>05rJTUWYAKNegBk2wE8X</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0AnoOZDNbPXIr2MRBSCJ</td>
      <td>1</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>10863</th>
      <td>KFrZ0Lop1WDGwUtkusCi</td>
      <td>9</td>
    </tr>
    <tr>
      <th>10864</th>
      <td>kg24YRJTB8DNdKMXpwOH</td>
      <td>9</td>
    </tr>
    <tr>
      <th>10865</th>
      <td>kG29BLiFYPgWtpb350sO</td>
      <td>9</td>
    </tr>
    <tr>
      <th>10866</th>
      <td>kGITL4OJxYMWEQ1bKBiP</td>
      <td>9</td>
    </tr>
    <tr>
      <th>10867</th>
      <td>KGorN9J6XAC4bOEkmyup</td>
      <td>9</td>
    </tr>
  </tbody>
</table>
<p>10868 rows × 2 columns</p>
</div>



Below, I am matching up the ID's from kaggles label file with the PNG file names. The output is the original label file, with a third column that holds the image path for each observation. 


```python
train_folder = r"C:\Users\fairwemr\ISA 480\FinalProject\RGB"
train_malware['image_path'] = train_malware.apply( lambda x: (train_folder + os.sep +  x["Id"] + ".png"), axis=1)

pd.options.display.max_colwidth = 100
train_malware
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Id</th>
      <th>Class</th>
      <th>image_path</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>01kcPWA9K2BOxQeS5Rju</td>
      <td>1</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\01kcPWA9K2BOxQeS5Rju.png</td>
    </tr>
    <tr>
      <th>1</th>
      <td>04EjIdbPV5e1XroFOpiN</td>
      <td>1</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\04EjIdbPV5e1XroFOpiN.png</td>
    </tr>
    <tr>
      <th>2</th>
      <td>05EeG39MTRrI6VY21DPd</td>
      <td>1</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\05EeG39MTRrI6VY21DPd.png</td>
    </tr>
    <tr>
      <th>3</th>
      <td>05rJTUWYAKNegBk2wE8X</td>
      <td>1</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\05rJTUWYAKNegBk2wE8X.png</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0AnoOZDNbPXIr2MRBSCJ</td>
      <td>1</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\0AnoOZDNbPXIr2MRBSCJ.png</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>10863</th>
      <td>KFrZ0Lop1WDGwUtkusCi</td>
      <td>9</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\KFrZ0Lop1WDGwUtkusCi.png</td>
    </tr>
    <tr>
      <th>10864</th>
      <td>kg24YRJTB8DNdKMXpwOH</td>
      <td>9</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\kg24YRJTB8DNdKMXpwOH.png</td>
    </tr>
    <tr>
      <th>10865</th>
      <td>kG29BLiFYPgWtpb350sO</td>
      <td>9</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\kG29BLiFYPgWtpb350sO.png</td>
    </tr>
    <tr>
      <th>10866</th>
      <td>kGITL4OJxYMWEQ1bKBiP</td>
      <td>9</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\kGITL4OJxYMWEQ1bKBiP.png</td>
    </tr>
    <tr>
      <th>10867</th>
      <td>KGorN9J6XAC4bOEkmyup</td>
      <td>9</td>
      <td>C:\Users\fairwemr\ISA 480\FinalProject\RGB\KGorN9J6XAC4bOEkmyup.png</td>
    </tr>
  </tbody>
</table>
<p>10868 rows × 3 columns</p>
</div>



# Step 4: Preparing Data for Modeling

#### Data partitions 

I first need to create my "response" variable, which is the Class column. 

```python
target_labels = train_malware['Class']
```

I used numpys image to array function to convert my images to an array with the size 299x299. 

```python
train_data  = np.array([img_to_array(load_img(img, target_size=(299, 299))) for img in train_malware[r'image_path'].values.tolist()]).astype('float32')
```

```python
x_train, x_validation, y_train, y_validation = train_test_split(train_data, target_labels, test_size=0.2, stratify=np.array(target_labels), random_state=100)
```


In the original paper, their final model was based on the Inception V1 model, which used a resized image of 224x224. However, the inception V1 model is no longer in use from what I can find, so I started with the inception V3 model as my starting point. The target image size for this model is 299x299. 


```python
print ('x_train shape = ', x_train.shape)
print ('x_validation shape = ', x_validation.shape)
```

    x_train shape =  (8694, 299, 299, 3)
    x_validation shape =  (2174, 299, 299, 3)
    

#### Visualization of Training and testing data sets


```python
# Calculate the value counts for train and validation data and plot to show the partitioning of the data
data = y_train.value_counts().sort_index().to_frame()   # this creates the data frame with train numbers
data.columns = ['train']
data['validation'] = y_validation.value_counts().sort_index().to_frame()   # add the validation numbers
new_plot = data[['train','validation']].sort_values(['train']+['validation'], ascending=False)   # sort the data
new_plot.plot(kind='bar', stacked=True)
plt.show()

```


![png](output_31_0.png)


#### Creating the Response variable as one-hot dummy variables 

```python
y_train = pd.get_dummies(y_train.reset_index(drop=True))
y_validation = pd.get_dummies(y_validation.reset_index(drop=True))
```

#### Creating image generators for training the keras models. 

For most applications of image classification, many use a variety of resizing, zoom, rotation, etc. of the image to make the neural network able to pick up more on the individual pixel patterns. However, because the original paper made no mention of their resizing technique, I was unsure of whether to follow that path. Additionally, I used the logic that because our images are created from binary code, and not humans taking pictures of objects at different angles for example, all of those image manipulations aren't as applicable to classifying RGB images of malware binary code as other image classification techniques. 


```python
# Create train generator.
train_datagen = ImageDataGenerator()
train_generator = train_datagen.flow(x_train, y_train, shuffle=True, batch_size=100, seed=10)
```


```python
# Create validation generator
val_datagen = ImageDataGenerator()
val_generator = train_datagen.flow(x_validation, y_validation, shuffle=True, batch_size=10, seed=10)
```

# Inception V3 Model 

In the paper, they claimed that their technique could be applied to a variety of pre-trained neural networks including the VGG model, inception model, ResNet, among others. Their "final" model as presented in the paper, however, was based on the pre-trained inception V1. This model is no longer in use, however the inception V3 is. Thus, this is the first model I tried to retrain. 

#### Loading in the Base Inception V3 Model

```python
# Get the InceptionV3 model
#include_top = false to allow me to create my own pooling and output layer 
base_inception_model = InceptionV3(weights = 'imagenet', include_top = False, input_shape=(299, 299, 3))
```
#### Adding layers to the pre-trained model

From my limited, but growing knowledge, global average pooling layers are used to help prevent overfitting. Nearly every resource I found to retrain inception models include adding this layer, and it aligned with my goals, so I did as well. 


```python
# Add a global spatial average pooling layer.
x = base_inception_model.output
x = GlobalAveragePooling2D()(x)
```

Many of the resources I found were adding a dense layer with far more nodes than the 512 value I ended on, some upwards of 5000. I tried this at first, realized it was taking far too long to retrain, and was seeing abismal results all around on both the training and testing data. After a lot of research, many people recommended lowering this value to between 400-800 to improve performance on both partitions and speed up training.  


```python
# Add a fully-connected layer and a logistic layer with 9 outputs, one for each class of malware
x = Dense(512, activation='relu')(x)
predictions = Dense(9, activation='softmax')(x)
```


```python
# The model we will train
inceptionV3Model = Model(inputs = base_inception_model.input, outputs = predictions)

```

#### Freezing base layers from being trained

The original paper was not very clear on which layers of the neural network they decided to train and which they kept the pre-trained weights. They mentioned that they froze everything but the last few layers, and explicitly said they trained the final pooling layer. Because the final pooling layer was one that I added myself, I froze all of the base model layers, and only trained the pooling layer I created, the dense layer, and the output layer. 


```python
# first: train only the top layers i.e. freeze all convolutional InceptionV3 layers
for layer in base_inception_model.layers:
    layer.trainable = False

```


```python
print(inceptionV3Model.summary())
```

    Model: "model_1"
    __________________________________________________________________________________________________
    Layer (type)                    Output Shape         Param #     Connected to                     
    ==================================================================================================
    input_1 (InputLayer)            (None, 299, 299, 3)  0                                            
    __________________________________________________________________________________________________
    conv2d_1 (Conv2D)               (None, 149, 149, 32) 864         input_1[0][0]                    
    __________________________________________________________________________________________________
    batch_normalization_1 (BatchNor (None, 149, 149, 32) 96          conv2d_1[0][0]                   
    __________________________________________________________________________________________________
    activation_1 (Activation)       (None, 149, 149, 32) 0           batch_normalization_1[0][0]      
    __________________________________________________________________________________________________
    conv2d_2 (Conv2D)               (None, 147, 147, 32) 9216        activation_1[0][0]               
    __________________________________________________________________________________________________
    batch_normalization_2 (BatchNor (None, 147, 147, 32) 96          conv2d_2[0][0]                   
    __________________________________________________________________________________________________
    activation_2 (Activation)       (None, 147, 147, 32) 0           batch_normalization_2[0][0]      
    __________________________________________________________________________________________________
    conv2d_3 (Conv2D)               (None, 147, 147, 64) 18432       activation_2[0][0]               
    __________________________________________________________________________________________________
    batch_normalization_3 (BatchNor (None, 147, 147, 64) 192         conv2d_3[0][0]                   
    __________________________________________________________________________________________________
    activation_3 (Activation)       (None, 147, 147, 64) 0           batch_normalization_3[0][0]      
    __________________________________________________________________________________________________
    max_pooling2d_1 (MaxPooling2D)  (None, 73, 73, 64)   0           activation_3[0][0]               
    __________________________________________________________________________________________________
    conv2d_4 (Conv2D)               (None, 73, 73, 80)   5120        max_pooling2d_1[0][0]            
    __________________________________________________________________________________________________
    batch_normalization_4 (BatchNor (None, 73, 73, 80)   240         conv2d_4[0][0]                   
    __________________________________________________________________________________________________
    activation_4 (Activation)       (None, 73, 73, 80)   0           batch_normalization_4[0][0]      
    __________________________________________________________________________________________________
    conv2d_5 (Conv2D)               (None, 71, 71, 192)  138240      activation_4[0][0]               
    __________________________________________________________________________________________________
    batch_normalization_5 (BatchNor (None, 71, 71, 192)  576         conv2d_5[0][0]                   
    __________________________________________________________________________________________________
    activation_5 (Activation)       (None, 71, 71, 192)  0           batch_normalization_5[0][0]      
    __________________________________________________________________________________________________
    max_pooling2d_2 (MaxPooling2D)  (None, 35, 35, 192)  0           activation_5[0][0]               
    __________________________________________________________________________________________________
    conv2d_9 (Conv2D)               (None, 35, 35, 64)   12288       max_pooling2d_2[0][0]            
    __________________________________________________________________________________________________
    batch_normalization_9 (BatchNor (None, 35, 35, 64)   192         conv2d_9[0][0]                   
    __________________________________________________________________________________________________
    activation_9 (Activation)       (None, 35, 35, 64)   0           batch_normalization_9[0][0]      
    __________________________________________________________________________________________________
    conv2d_7 (Conv2D)               (None, 35, 35, 48)   9216        max_pooling2d_2[0][0]            
    __________________________________________________________________________________________________
    conv2d_10 (Conv2D)              (None, 35, 35, 96)   55296       activation_9[0][0]               
    __________________________________________________________________________________________________
    batch_normalization_7 (BatchNor (None, 35, 35, 48)   144         conv2d_7[0][0]                   
    __________________________________________________________________________________________________
    batch_normalization_10 (BatchNo (None, 35, 35, 96)   288         conv2d_10[0][0]                  
    __________________________________________________________________________________________________
    activation_7 (Activation)       (None, 35, 35, 48)   0           batch_normalization_7[0][0]      
    __________________________________________________________________________________________________
    activation_10 (Activation)      (None, 35, 35, 96)   0           batch_normalization_10[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_1 (AveragePoo (None, 35, 35, 192)  0           max_pooling2d_2[0][0]            
    __________________________________________________________________________________________________
    conv2d_6 (Conv2D)               (None, 35, 35, 64)   12288       max_pooling2d_2[0][0]            
    __________________________________________________________________________________________________
    conv2d_8 (Conv2D)               (None, 35, 35, 64)   76800       activation_7[0][0]               
    __________________________________________________________________________________________________
    conv2d_11 (Conv2D)              (None, 35, 35, 96)   82944       activation_10[0][0]              
    __________________________________________________________________________________________________
    conv2d_12 (Conv2D)              (None, 35, 35, 32)   6144        average_pooling2d_1[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_6 (BatchNor (None, 35, 35, 64)   192         conv2d_6[0][0]                   
    __________________________________________________________________________________________________
    batch_normalization_8 (BatchNor (None, 35, 35, 64)   192         conv2d_8[0][0]                   
    __________________________________________________________________________________________________
    batch_normalization_11 (BatchNo (None, 35, 35, 96)   288         conv2d_11[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_12 (BatchNo (None, 35, 35, 32)   96          conv2d_12[0][0]                  
    __________________________________________________________________________________________________
    activation_6 (Activation)       (None, 35, 35, 64)   0           batch_normalization_6[0][0]      
    __________________________________________________________________________________________________
    activation_8 (Activation)       (None, 35, 35, 64)   0           batch_normalization_8[0][0]      
    __________________________________________________________________________________________________
    activation_11 (Activation)      (None, 35, 35, 96)   0           batch_normalization_11[0][0]     
    __________________________________________________________________________________________________
    activation_12 (Activation)      (None, 35, 35, 32)   0           batch_normalization_12[0][0]     
    __________________________________________________________________________________________________
    mixed0 (Concatenate)            (None, 35, 35, 256)  0           activation_6[0][0]               
                                                                     activation_8[0][0]               
                                                                     activation_11[0][0]              
                                                                     activation_12[0][0]              
    __________________________________________________________________________________________________
    conv2d_16 (Conv2D)              (None, 35, 35, 64)   16384       mixed0[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_16 (BatchNo (None, 35, 35, 64)   192         conv2d_16[0][0]                  
    __________________________________________________________________________________________________
    activation_16 (Activation)      (None, 35, 35, 64)   0           batch_normalization_16[0][0]     
    __________________________________________________________________________________________________
    conv2d_14 (Conv2D)              (None, 35, 35, 48)   12288       mixed0[0][0]                     
    __________________________________________________________________________________________________
    conv2d_17 (Conv2D)              (None, 35, 35, 96)   55296       activation_16[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_14 (BatchNo (None, 35, 35, 48)   144         conv2d_14[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_17 (BatchNo (None, 35, 35, 96)   288         conv2d_17[0][0]                  
    __________________________________________________________________________________________________
    activation_14 (Activation)      (None, 35, 35, 48)   0           batch_normalization_14[0][0]     
    __________________________________________________________________________________________________
    activation_17 (Activation)      (None, 35, 35, 96)   0           batch_normalization_17[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_2 (AveragePoo (None, 35, 35, 256)  0           mixed0[0][0]                     
    __________________________________________________________________________________________________
    conv2d_13 (Conv2D)              (None, 35, 35, 64)   16384       mixed0[0][0]                     
    __________________________________________________________________________________________________
    conv2d_15 (Conv2D)              (None, 35, 35, 64)   76800       activation_14[0][0]              
    __________________________________________________________________________________________________
    conv2d_18 (Conv2D)              (None, 35, 35, 96)   82944       activation_17[0][0]              
    __________________________________________________________________________________________________
    conv2d_19 (Conv2D)              (None, 35, 35, 64)   16384       average_pooling2d_2[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_13 (BatchNo (None, 35, 35, 64)   192         conv2d_13[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_15 (BatchNo (None, 35, 35, 64)   192         conv2d_15[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_18 (BatchNo (None, 35, 35, 96)   288         conv2d_18[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_19 (BatchNo (None, 35, 35, 64)   192         conv2d_19[0][0]                  
    __________________________________________________________________________________________________
    activation_13 (Activation)      (None, 35, 35, 64)   0           batch_normalization_13[0][0]     
    __________________________________________________________________________________________________
    activation_15 (Activation)      (None, 35, 35, 64)   0           batch_normalization_15[0][0]     
    __________________________________________________________________________________________________
    activation_18 (Activation)      (None, 35, 35, 96)   0           batch_normalization_18[0][0]     
    __________________________________________________________________________________________________
    activation_19 (Activation)      (None, 35, 35, 64)   0           batch_normalization_19[0][0]     
    __________________________________________________________________________________________________
    mixed1 (Concatenate)            (None, 35, 35, 288)  0           activation_13[0][0]              
                                                                     activation_15[0][0]              
                                                                     activation_18[0][0]              
                                                                     activation_19[0][0]              
    __________________________________________________________________________________________________
    conv2d_23 (Conv2D)              (None, 35, 35, 64)   18432       mixed1[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_23 (BatchNo (None, 35, 35, 64)   192         conv2d_23[0][0]                  
    __________________________________________________________________________________________________
    activation_23 (Activation)      (None, 35, 35, 64)   0           batch_normalization_23[0][0]     
    __________________________________________________________________________________________________
    conv2d_21 (Conv2D)              (None, 35, 35, 48)   13824       mixed1[0][0]                     
    __________________________________________________________________________________________________
    conv2d_24 (Conv2D)              (None, 35, 35, 96)   55296       activation_23[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_21 (BatchNo (None, 35, 35, 48)   144         conv2d_21[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_24 (BatchNo (None, 35, 35, 96)   288         conv2d_24[0][0]                  
    __________________________________________________________________________________________________
    activation_21 (Activation)      (None, 35, 35, 48)   0           batch_normalization_21[0][0]     
    __________________________________________________________________________________________________
    activation_24 (Activation)      (None, 35, 35, 96)   0           batch_normalization_24[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_3 (AveragePoo (None, 35, 35, 288)  0           mixed1[0][0]                     
    __________________________________________________________________________________________________
    conv2d_20 (Conv2D)              (None, 35, 35, 64)   18432       mixed1[0][0]                     
    __________________________________________________________________________________________________
    conv2d_22 (Conv2D)              (None, 35, 35, 64)   76800       activation_21[0][0]              
    __________________________________________________________________________________________________
    conv2d_25 (Conv2D)              (None, 35, 35, 96)   82944       activation_24[0][0]              
    __________________________________________________________________________________________________
    conv2d_26 (Conv2D)              (None, 35, 35, 64)   18432       average_pooling2d_3[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_20 (BatchNo (None, 35, 35, 64)   192         conv2d_20[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_22 (BatchNo (None, 35, 35, 64)   192         conv2d_22[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_25 (BatchNo (None, 35, 35, 96)   288         conv2d_25[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_26 (BatchNo (None, 35, 35, 64)   192         conv2d_26[0][0]                  
    __________________________________________________________________________________________________
    activation_20 (Activation)      (None, 35, 35, 64)   0           batch_normalization_20[0][0]     
    __________________________________________________________________________________________________
    activation_22 (Activation)      (None, 35, 35, 64)   0           batch_normalization_22[0][0]     
    __________________________________________________________________________________________________
    activation_25 (Activation)      (None, 35, 35, 96)   0           batch_normalization_25[0][0]     
    __________________________________________________________________________________________________
    activation_26 (Activation)      (None, 35, 35, 64)   0           batch_normalization_26[0][0]     
    __________________________________________________________________________________________________
    mixed2 (Concatenate)            (None, 35, 35, 288)  0           activation_20[0][0]              
                                                                     activation_22[0][0]              
                                                                     activation_25[0][0]              
                                                                     activation_26[0][0]              
    __________________________________________________________________________________________________
    conv2d_28 (Conv2D)              (None, 35, 35, 64)   18432       mixed2[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_28 (BatchNo (None, 35, 35, 64)   192         conv2d_28[0][0]                  
    __________________________________________________________________________________________________
    activation_28 (Activation)      (None, 35, 35, 64)   0           batch_normalization_28[0][0]     
    __________________________________________________________________________________________________
    conv2d_29 (Conv2D)              (None, 35, 35, 96)   55296       activation_28[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_29 (BatchNo (None, 35, 35, 96)   288         conv2d_29[0][0]                  
    __________________________________________________________________________________________________
    activation_29 (Activation)      (None, 35, 35, 96)   0           batch_normalization_29[0][0]     
    __________________________________________________________________________________________________
    conv2d_27 (Conv2D)              (None, 17, 17, 384)  995328      mixed2[0][0]                     
    __________________________________________________________________________________________________
    conv2d_30 (Conv2D)              (None, 17, 17, 96)   82944       activation_29[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_27 (BatchNo (None, 17, 17, 384)  1152        conv2d_27[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_30 (BatchNo (None, 17, 17, 96)   288         conv2d_30[0][0]                  
    __________________________________________________________________________________________________
    activation_27 (Activation)      (None, 17, 17, 384)  0           batch_normalization_27[0][0]     
    __________________________________________________________________________________________________
    activation_30 (Activation)      (None, 17, 17, 96)   0           batch_normalization_30[0][0]     
    __________________________________________________________________________________________________
    max_pooling2d_3 (MaxPooling2D)  (None, 17, 17, 288)  0           mixed2[0][0]                     
    __________________________________________________________________________________________________
    mixed3 (Concatenate)            (None, 17, 17, 768)  0           activation_27[0][0]              
                                                                     activation_30[0][0]              
                                                                     max_pooling2d_3[0][0]            
    __________________________________________________________________________________________________
    conv2d_35 (Conv2D)              (None, 17, 17, 128)  98304       mixed3[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_35 (BatchNo (None, 17, 17, 128)  384         conv2d_35[0][0]                  
    __________________________________________________________________________________________________
    activation_35 (Activation)      (None, 17, 17, 128)  0           batch_normalization_35[0][0]     
    __________________________________________________________________________________________________
    conv2d_36 (Conv2D)              (None, 17, 17, 128)  114688      activation_35[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_36 (BatchNo (None, 17, 17, 128)  384         conv2d_36[0][0]                  
    __________________________________________________________________________________________________
    activation_36 (Activation)      (None, 17, 17, 128)  0           batch_normalization_36[0][0]     
    __________________________________________________________________________________________________
    conv2d_32 (Conv2D)              (None, 17, 17, 128)  98304       mixed3[0][0]                     
    __________________________________________________________________________________________________
    conv2d_37 (Conv2D)              (None, 17, 17, 128)  114688      activation_36[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_32 (BatchNo (None, 17, 17, 128)  384         conv2d_32[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_37 (BatchNo (None, 17, 17, 128)  384         conv2d_37[0][0]                  
    __________________________________________________________________________________________________
    activation_32 (Activation)      (None, 17, 17, 128)  0           batch_normalization_32[0][0]     
    __________________________________________________________________________________________________
    activation_37 (Activation)      (None, 17, 17, 128)  0           batch_normalization_37[0][0]     
    __________________________________________________________________________________________________
    conv2d_33 (Conv2D)              (None, 17, 17, 128)  114688      activation_32[0][0]              
    __________________________________________________________________________________________________
    conv2d_38 (Conv2D)              (None, 17, 17, 128)  114688      activation_37[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_33 (BatchNo (None, 17, 17, 128)  384         conv2d_33[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_38 (BatchNo (None, 17, 17, 128)  384         conv2d_38[0][0]                  
    __________________________________________________________________________________________________
    activation_33 (Activation)      (None, 17, 17, 128)  0           batch_normalization_33[0][0]     
    __________________________________________________________________________________________________
    activation_38 (Activation)      (None, 17, 17, 128)  0           batch_normalization_38[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_4 (AveragePoo (None, 17, 17, 768)  0           mixed3[0][0]                     
    __________________________________________________________________________________________________
    conv2d_31 (Conv2D)              (None, 17, 17, 192)  147456      mixed3[0][0]                     
    __________________________________________________________________________________________________
    conv2d_34 (Conv2D)              (None, 17, 17, 192)  172032      activation_33[0][0]              
    __________________________________________________________________________________________________
    conv2d_39 (Conv2D)              (None, 17, 17, 192)  172032      activation_38[0][0]              
    __________________________________________________________________________________________________
    conv2d_40 (Conv2D)              (None, 17, 17, 192)  147456      average_pooling2d_4[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_31 (BatchNo (None, 17, 17, 192)  576         conv2d_31[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_34 (BatchNo (None, 17, 17, 192)  576         conv2d_34[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_39 (BatchNo (None, 17, 17, 192)  576         conv2d_39[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_40 (BatchNo (None, 17, 17, 192)  576         conv2d_40[0][0]                  
    __________________________________________________________________________________________________
    activation_31 (Activation)      (None, 17, 17, 192)  0           batch_normalization_31[0][0]     
    __________________________________________________________________________________________________
    activation_34 (Activation)      (None, 17, 17, 192)  0           batch_normalization_34[0][0]     
    __________________________________________________________________________________________________
    activation_39 (Activation)      (None, 17, 17, 192)  0           batch_normalization_39[0][0]     
    __________________________________________________________________________________________________
    activation_40 (Activation)      (None, 17, 17, 192)  0           batch_normalization_40[0][0]     
    __________________________________________________________________________________________________
    mixed4 (Concatenate)            (None, 17, 17, 768)  0           activation_31[0][0]              
                                                                     activation_34[0][0]              
                                                                     activation_39[0][0]              
                                                                     activation_40[0][0]              
    __________________________________________________________________________________________________
    conv2d_45 (Conv2D)              (None, 17, 17, 160)  122880      mixed4[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_45 (BatchNo (None, 17, 17, 160)  480         conv2d_45[0][0]                  
    __________________________________________________________________________________________________
    activation_45 (Activation)      (None, 17, 17, 160)  0           batch_normalization_45[0][0]     
    __________________________________________________________________________________________________
    conv2d_46 (Conv2D)              (None, 17, 17, 160)  179200      activation_45[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_46 (BatchNo (None, 17, 17, 160)  480         conv2d_46[0][0]                  
    __________________________________________________________________________________________________
    activation_46 (Activation)      (None, 17, 17, 160)  0           batch_normalization_46[0][0]     
    __________________________________________________________________________________________________
    conv2d_42 (Conv2D)              (None, 17, 17, 160)  122880      mixed4[0][0]                     
    __________________________________________________________________________________________________
    conv2d_47 (Conv2D)              (None, 17, 17, 160)  179200      activation_46[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_42 (BatchNo (None, 17, 17, 160)  480         conv2d_42[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_47 (BatchNo (None, 17, 17, 160)  480         conv2d_47[0][0]                  
    __________________________________________________________________________________________________
    activation_42 (Activation)      (None, 17, 17, 160)  0           batch_normalization_42[0][0]     
    __________________________________________________________________________________________________
    activation_47 (Activation)      (None, 17, 17, 160)  0           batch_normalization_47[0][0]     
    __________________________________________________________________________________________________
    conv2d_43 (Conv2D)              (None, 17, 17, 160)  179200      activation_42[0][0]              
    __________________________________________________________________________________________________
    conv2d_48 (Conv2D)              (None, 17, 17, 160)  179200      activation_47[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_43 (BatchNo (None, 17, 17, 160)  480         conv2d_43[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_48 (BatchNo (None, 17, 17, 160)  480         conv2d_48[0][0]                  
    __________________________________________________________________________________________________
    activation_43 (Activation)      (None, 17, 17, 160)  0           batch_normalization_43[0][0]     
    __________________________________________________________________________________________________
    activation_48 (Activation)      (None, 17, 17, 160)  0           batch_normalization_48[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_5 (AveragePoo (None, 17, 17, 768)  0           mixed4[0][0]                     
    __________________________________________________________________________________________________
    conv2d_41 (Conv2D)              (None, 17, 17, 192)  147456      mixed4[0][0]                     
    __________________________________________________________________________________________________
    conv2d_44 (Conv2D)              (None, 17, 17, 192)  215040      activation_43[0][0]              
    __________________________________________________________________________________________________
    conv2d_49 (Conv2D)              (None, 17, 17, 192)  215040      activation_48[0][0]              
    __________________________________________________________________________________________________
    conv2d_50 (Conv2D)              (None, 17, 17, 192)  147456      average_pooling2d_5[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_41 (BatchNo (None, 17, 17, 192)  576         conv2d_41[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_44 (BatchNo (None, 17, 17, 192)  576         conv2d_44[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_49 (BatchNo (None, 17, 17, 192)  576         conv2d_49[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_50 (BatchNo (None, 17, 17, 192)  576         conv2d_50[0][0]                  
    __________________________________________________________________________________________________
    activation_41 (Activation)      (None, 17, 17, 192)  0           batch_normalization_41[0][0]     
    __________________________________________________________________________________________________
    activation_44 (Activation)      (None, 17, 17, 192)  0           batch_normalization_44[0][0]     
    __________________________________________________________________________________________________
    activation_49 (Activation)      (None, 17, 17, 192)  0           batch_normalization_49[0][0]     
    __________________________________________________________________________________________________
    activation_50 (Activation)      (None, 17, 17, 192)  0           batch_normalization_50[0][0]     
    __________________________________________________________________________________________________
    mixed5 (Concatenate)            (None, 17, 17, 768)  0           activation_41[0][0]              
                                                                     activation_44[0][0]              
                                                                     activation_49[0][0]              
                                                                     activation_50[0][0]              
    __________________________________________________________________________________________________
    conv2d_55 (Conv2D)              (None, 17, 17, 160)  122880      mixed5[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_55 (BatchNo (None, 17, 17, 160)  480         conv2d_55[0][0]                  
    __________________________________________________________________________________________________
    activation_55 (Activation)      (None, 17, 17, 160)  0           batch_normalization_55[0][0]     
    __________________________________________________________________________________________________
    conv2d_56 (Conv2D)              (None, 17, 17, 160)  179200      activation_55[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_56 (BatchNo (None, 17, 17, 160)  480         conv2d_56[0][0]                  
    __________________________________________________________________________________________________
    activation_56 (Activation)      (None, 17, 17, 160)  0           batch_normalization_56[0][0]     
    __________________________________________________________________________________________________
    conv2d_52 (Conv2D)              (None, 17, 17, 160)  122880      mixed5[0][0]                     
    __________________________________________________________________________________________________
    conv2d_57 (Conv2D)              (None, 17, 17, 160)  179200      activation_56[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_52 (BatchNo (None, 17, 17, 160)  480         conv2d_52[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_57 (BatchNo (None, 17, 17, 160)  480         conv2d_57[0][0]                  
    __________________________________________________________________________________________________
    activation_52 (Activation)      (None, 17, 17, 160)  0           batch_normalization_52[0][0]     
    __________________________________________________________________________________________________
    activation_57 (Activation)      (None, 17, 17, 160)  0           batch_normalization_57[0][0]     
    __________________________________________________________________________________________________
    conv2d_53 (Conv2D)              (None, 17, 17, 160)  179200      activation_52[0][0]              
    __________________________________________________________________________________________________
    conv2d_58 (Conv2D)              (None, 17, 17, 160)  179200      activation_57[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_53 (BatchNo (None, 17, 17, 160)  480         conv2d_53[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_58 (BatchNo (None, 17, 17, 160)  480         conv2d_58[0][0]                  
    __________________________________________________________________________________________________
    activation_53 (Activation)      (None, 17, 17, 160)  0           batch_normalization_53[0][0]     
    __________________________________________________________________________________________________
    activation_58 (Activation)      (None, 17, 17, 160)  0           batch_normalization_58[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_6 (AveragePoo (None, 17, 17, 768)  0           mixed5[0][0]                     
    __________________________________________________________________________________________________
    conv2d_51 (Conv2D)              (None, 17, 17, 192)  147456      mixed5[0][0]                     
    __________________________________________________________________________________________________
    conv2d_54 (Conv2D)              (None, 17, 17, 192)  215040      activation_53[0][0]              
    __________________________________________________________________________________________________
    conv2d_59 (Conv2D)              (None, 17, 17, 192)  215040      activation_58[0][0]              
    __________________________________________________________________________________________________
    conv2d_60 (Conv2D)              (None, 17, 17, 192)  147456      average_pooling2d_6[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_51 (BatchNo (None, 17, 17, 192)  576         conv2d_51[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_54 (BatchNo (None, 17, 17, 192)  576         conv2d_54[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_59 (BatchNo (None, 17, 17, 192)  576         conv2d_59[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_60 (BatchNo (None, 17, 17, 192)  576         conv2d_60[0][0]                  
    __________________________________________________________________________________________________
    activation_51 (Activation)      (None, 17, 17, 192)  0           batch_normalization_51[0][0]     
    __________________________________________________________________________________________________
    activation_54 (Activation)      (None, 17, 17, 192)  0           batch_normalization_54[0][0]     
    __________________________________________________________________________________________________
    activation_59 (Activation)      (None, 17, 17, 192)  0           batch_normalization_59[0][0]     
    __________________________________________________________________________________________________
    activation_60 (Activation)      (None, 17, 17, 192)  0           batch_normalization_60[0][0]     
    __________________________________________________________________________________________________
    mixed6 (Concatenate)            (None, 17, 17, 768)  0           activation_51[0][0]              
                                                                     activation_54[0][0]              
                                                                     activation_59[0][0]              
                                                                     activation_60[0][0]              
    __________________________________________________________________________________________________
    conv2d_65 (Conv2D)              (None, 17, 17, 192)  147456      mixed6[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_65 (BatchNo (None, 17, 17, 192)  576         conv2d_65[0][0]                  
    __________________________________________________________________________________________________
    activation_65 (Activation)      (None, 17, 17, 192)  0           batch_normalization_65[0][0]     
    __________________________________________________________________________________________________
    conv2d_66 (Conv2D)              (None, 17, 17, 192)  258048      activation_65[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_66 (BatchNo (None, 17, 17, 192)  576         conv2d_66[0][0]                  
    __________________________________________________________________________________________________
    activation_66 (Activation)      (None, 17, 17, 192)  0           batch_normalization_66[0][0]     
    __________________________________________________________________________________________________
    conv2d_62 (Conv2D)              (None, 17, 17, 192)  147456      mixed6[0][0]                     
    __________________________________________________________________________________________________
    conv2d_67 (Conv2D)              (None, 17, 17, 192)  258048      activation_66[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_62 (BatchNo (None, 17, 17, 192)  576         conv2d_62[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_67 (BatchNo (None, 17, 17, 192)  576         conv2d_67[0][0]                  
    __________________________________________________________________________________________________
    activation_62 (Activation)      (None, 17, 17, 192)  0           batch_normalization_62[0][0]     
    __________________________________________________________________________________________________
    activation_67 (Activation)      (None, 17, 17, 192)  0           batch_normalization_67[0][0]     
    __________________________________________________________________________________________________
    conv2d_63 (Conv2D)              (None, 17, 17, 192)  258048      activation_62[0][0]              
    __________________________________________________________________________________________________
    conv2d_68 (Conv2D)              (None, 17, 17, 192)  258048      activation_67[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_63 (BatchNo (None, 17, 17, 192)  576         conv2d_63[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_68 (BatchNo (None, 17, 17, 192)  576         conv2d_68[0][0]                  
    __________________________________________________________________________________________________
    activation_63 (Activation)      (None, 17, 17, 192)  0           batch_normalization_63[0][0]     
    __________________________________________________________________________________________________
    activation_68 (Activation)      (None, 17, 17, 192)  0           batch_normalization_68[0][0]     
    __________________________________________________________________________________________________
    average_pooling2d_7 (AveragePoo (None, 17, 17, 768)  0           mixed6[0][0]                     
    __________________________________________________________________________________________________
    conv2d_61 (Conv2D)              (None, 17, 17, 192)  147456      mixed6[0][0]                     
    __________________________________________________________________________________________________
    conv2d_64 (Conv2D)              (None, 17, 17, 192)  258048      activation_63[0][0]              
    __________________________________________________________________________________________________
    conv2d_69 (Conv2D)              (None, 17, 17, 192)  258048      activation_68[0][0]              
    __________________________________________________________________________________________________
    conv2d_70 (Conv2D)              (None, 17, 17, 192)  147456      average_pooling2d_7[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_61 (BatchNo (None, 17, 17, 192)  576         conv2d_61[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_64 (BatchNo (None, 17, 17, 192)  576         conv2d_64[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_69 (BatchNo (None, 17, 17, 192)  576         conv2d_69[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_70 (BatchNo (None, 17, 17, 192)  576         conv2d_70[0][0]                  
    __________________________________________________________________________________________________
    activation_61 (Activation)      (None, 17, 17, 192)  0           batch_normalization_61[0][0]     
    __________________________________________________________________________________________________
    activation_64 (Activation)      (None, 17, 17, 192)  0           batch_normalization_64[0][0]     
    __________________________________________________________________________________________________
    activation_69 (Activation)      (None, 17, 17, 192)  0           batch_normalization_69[0][0]     
    __________________________________________________________________________________________________
    activation_70 (Activation)      (None, 17, 17, 192)  0           batch_normalization_70[0][0]     
    __________________________________________________________________________________________________
    mixed7 (Concatenate)            (None, 17, 17, 768)  0           activation_61[0][0]              
                                                                     activation_64[0][0]              
                                                                     activation_69[0][0]              
                                                                     activation_70[0][0]              
    __________________________________________________________________________________________________
    conv2d_73 (Conv2D)              (None, 17, 17, 192)  147456      mixed7[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_73 (BatchNo (None, 17, 17, 192)  576         conv2d_73[0][0]                  
    __________________________________________________________________________________________________
    activation_73 (Activation)      (None, 17, 17, 192)  0           batch_normalization_73[0][0]     
    __________________________________________________________________________________________________
    conv2d_74 (Conv2D)              (None, 17, 17, 192)  258048      activation_73[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_74 (BatchNo (None, 17, 17, 192)  576         conv2d_74[0][0]                  
    __________________________________________________________________________________________________
    activation_74 (Activation)      (None, 17, 17, 192)  0           batch_normalization_74[0][0]     
    __________________________________________________________________________________________________
    conv2d_71 (Conv2D)              (None, 17, 17, 192)  147456      mixed7[0][0]                     
    __________________________________________________________________________________________________
    conv2d_75 (Conv2D)              (None, 17, 17, 192)  258048      activation_74[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_71 (BatchNo (None, 17, 17, 192)  576         conv2d_71[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_75 (BatchNo (None, 17, 17, 192)  576         conv2d_75[0][0]                  
    __________________________________________________________________________________________________
    activation_71 (Activation)      (None, 17, 17, 192)  0           batch_normalization_71[0][0]     
    __________________________________________________________________________________________________
    activation_75 (Activation)      (None, 17, 17, 192)  0           batch_normalization_75[0][0]     
    __________________________________________________________________________________________________
    conv2d_72 (Conv2D)              (None, 8, 8, 320)    552960      activation_71[0][0]              
    __________________________________________________________________________________________________
    conv2d_76 (Conv2D)              (None, 8, 8, 192)    331776      activation_75[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_72 (BatchNo (None, 8, 8, 320)    960         conv2d_72[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_76 (BatchNo (None, 8, 8, 192)    576         conv2d_76[0][0]                  
    __________________________________________________________________________________________________
    activation_72 (Activation)      (None, 8, 8, 320)    0           batch_normalization_72[0][0]     
    __________________________________________________________________________________________________
    activation_76 (Activation)      (None, 8, 8, 192)    0           batch_normalization_76[0][0]     
    __________________________________________________________________________________________________
    max_pooling2d_4 (MaxPooling2D)  (None, 8, 8, 768)    0           mixed7[0][0]                     
    __________________________________________________________________________________________________
    mixed8 (Concatenate)            (None, 8, 8, 1280)   0           activation_72[0][0]              
                                                                     activation_76[0][0]              
                                                                     max_pooling2d_4[0][0]            
    __________________________________________________________________________________________________
    conv2d_81 (Conv2D)              (None, 8, 8, 448)    573440      mixed8[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_81 (BatchNo (None, 8, 8, 448)    1344        conv2d_81[0][0]                  
    __________________________________________________________________________________________________
    activation_81 (Activation)      (None, 8, 8, 448)    0           batch_normalization_81[0][0]     
    __________________________________________________________________________________________________
    conv2d_78 (Conv2D)              (None, 8, 8, 384)    491520      mixed8[0][0]                     
    __________________________________________________________________________________________________
    conv2d_82 (Conv2D)              (None, 8, 8, 384)    1548288     activation_81[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_78 (BatchNo (None, 8, 8, 384)    1152        conv2d_78[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_82 (BatchNo (None, 8, 8, 384)    1152        conv2d_82[0][0]                  
    __________________________________________________________________________________________________
    activation_78 (Activation)      (None, 8, 8, 384)    0           batch_normalization_78[0][0]     
    __________________________________________________________________________________________________
    activation_82 (Activation)      (None, 8, 8, 384)    0           batch_normalization_82[0][0]     
    __________________________________________________________________________________________________
    conv2d_79 (Conv2D)              (None, 8, 8, 384)    442368      activation_78[0][0]              
    __________________________________________________________________________________________________
    conv2d_80 (Conv2D)              (None, 8, 8, 384)    442368      activation_78[0][0]              
    __________________________________________________________________________________________________
    conv2d_83 (Conv2D)              (None, 8, 8, 384)    442368      activation_82[0][0]              
    __________________________________________________________________________________________________
    conv2d_84 (Conv2D)              (None, 8, 8, 384)    442368      activation_82[0][0]              
    __________________________________________________________________________________________________
    average_pooling2d_8 (AveragePoo (None, 8, 8, 1280)   0           mixed8[0][0]                     
    __________________________________________________________________________________________________
    conv2d_77 (Conv2D)              (None, 8, 8, 320)    409600      mixed8[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_79 (BatchNo (None, 8, 8, 384)    1152        conv2d_79[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_80 (BatchNo (None, 8, 8, 384)    1152        conv2d_80[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_83 (BatchNo (None, 8, 8, 384)    1152        conv2d_83[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_84 (BatchNo (None, 8, 8, 384)    1152        conv2d_84[0][0]                  
    __________________________________________________________________________________________________
    conv2d_85 (Conv2D)              (None, 8, 8, 192)    245760      average_pooling2d_8[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_77 (BatchNo (None, 8, 8, 320)    960         conv2d_77[0][0]                  
    __________________________________________________________________________________________________
    activation_79 (Activation)      (None, 8, 8, 384)    0           batch_normalization_79[0][0]     
    __________________________________________________________________________________________________
    activation_80 (Activation)      (None, 8, 8, 384)    0           batch_normalization_80[0][0]     
    __________________________________________________________________________________________________
    activation_83 (Activation)      (None, 8, 8, 384)    0           batch_normalization_83[0][0]     
    __________________________________________________________________________________________________
    activation_84 (Activation)      (None, 8, 8, 384)    0           batch_normalization_84[0][0]     
    __________________________________________________________________________________________________
    batch_normalization_85 (BatchNo (None, 8, 8, 192)    576         conv2d_85[0][0]                  
    __________________________________________________________________________________________________
    activation_77 (Activation)      (None, 8, 8, 320)    0           batch_normalization_77[0][0]     
    __________________________________________________________________________________________________
    mixed9_0 (Concatenate)          (None, 8, 8, 768)    0           activation_79[0][0]              
                                                                     activation_80[0][0]              
    __________________________________________________________________________________________________
    concatenate_1 (Concatenate)     (None, 8, 8, 768)    0           activation_83[0][0]              
                                                                     activation_84[0][0]              
    __________________________________________________________________________________________________
    activation_85 (Activation)      (None, 8, 8, 192)    0           batch_normalization_85[0][0]     
    __________________________________________________________________________________________________
    mixed9 (Concatenate)            (None, 8, 8, 2048)   0           activation_77[0][0]              
                                                                     mixed9_0[0][0]                   
                                                                     concatenate_1[0][0]              
                                                                     activation_85[0][0]              
    __________________________________________________________________________________________________
    conv2d_90 (Conv2D)              (None, 8, 8, 448)    917504      mixed9[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_90 (BatchNo (None, 8, 8, 448)    1344        conv2d_90[0][0]                  
    __________________________________________________________________________________________________
    activation_90 (Activation)      (None, 8, 8, 448)    0           batch_normalization_90[0][0]     
    __________________________________________________________________________________________________
    conv2d_87 (Conv2D)              (None, 8, 8, 384)    786432      mixed9[0][0]                     
    __________________________________________________________________________________________________
    conv2d_91 (Conv2D)              (None, 8, 8, 384)    1548288     activation_90[0][0]              
    __________________________________________________________________________________________________
    batch_normalization_87 (BatchNo (None, 8, 8, 384)    1152        conv2d_87[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_91 (BatchNo (None, 8, 8, 384)    1152        conv2d_91[0][0]                  
    __________________________________________________________________________________________________
    activation_87 (Activation)      (None, 8, 8, 384)    0           batch_normalization_87[0][0]     
    __________________________________________________________________________________________________
    activation_91 (Activation)      (None, 8, 8, 384)    0           batch_normalization_91[0][0]     
    __________________________________________________________________________________________________
    conv2d_88 (Conv2D)              (None, 8, 8, 384)    442368      activation_87[0][0]              
    __________________________________________________________________________________________________
    conv2d_89 (Conv2D)              (None, 8, 8, 384)    442368      activation_87[0][0]              
    __________________________________________________________________________________________________
    conv2d_92 (Conv2D)              (None, 8, 8, 384)    442368      activation_91[0][0]              
    __________________________________________________________________________________________________
    conv2d_93 (Conv2D)              (None, 8, 8, 384)    442368      activation_91[0][0]              
    __________________________________________________________________________________________________
    average_pooling2d_9 (AveragePoo (None, 8, 8, 2048)   0           mixed9[0][0]                     
    __________________________________________________________________________________________________
    conv2d_86 (Conv2D)              (None, 8, 8, 320)    655360      mixed9[0][0]                     
    __________________________________________________________________________________________________
    batch_normalization_88 (BatchNo (None, 8, 8, 384)    1152        conv2d_88[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_89 (BatchNo (None, 8, 8, 384)    1152        conv2d_89[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_92 (BatchNo (None, 8, 8, 384)    1152        conv2d_92[0][0]                  
    __________________________________________________________________________________________________
    batch_normalization_93 (BatchNo (None, 8, 8, 384)    1152        conv2d_93[0][0]                  
    __________________________________________________________________________________________________
    conv2d_94 (Conv2D)              (None, 8, 8, 192)    393216      average_pooling2d_9[0][0]        
    __________________________________________________________________________________________________
    batch_normalization_86 (BatchNo (None, 8, 8, 320)    960         conv2d_86[0][0]                  
    __________________________________________________________________________________________________
    activation_88 (Activation)      (None, 8, 8, 384)    0           batch_normalization_88[0][0]     
    __________________________________________________________________________________________________
    activation_89 (Activation)      (None, 8, 8, 384)    0           batch_normalization_89[0][0]     
    __________________________________________________________________________________________________
    activation_92 (Activation)      (None, 8, 8, 384)    0           batch_normalization_92[0][0]     
    __________________________________________________________________________________________________
    activation_93 (Activation)      (None, 8, 8, 384)    0           batch_normalization_93[0][0]     
    __________________________________________________________________________________________________
    batch_normalization_94 (BatchNo (None, 8, 8, 192)    576         conv2d_94[0][0]                  
    __________________________________________________________________________________________________
    activation_86 (Activation)      (None, 8, 8, 320)    0           batch_normalization_86[0][0]     
    __________________________________________________________________________________________________
    mixed9_1 (Concatenate)          (None, 8, 8, 768)    0           activation_88[0][0]              
                                                                     activation_89[0][0]              
    __________________________________________________________________________________________________
    concatenate_2 (Concatenate)     (None, 8, 8, 768)    0           activation_92[0][0]              
                                                                     activation_93[0][0]              
    __________________________________________________________________________________________________
    activation_94 (Activation)      (None, 8, 8, 192)    0           batch_normalization_94[0][0]     
    __________________________________________________________________________________________________
    mixed10 (Concatenate)           (None, 8, 8, 2048)   0           activation_86[0][0]              
                                                                     mixed9_1[0][0]                   
                                                                     concatenate_2[0][0]              
                                                                     activation_94[0][0]              
    __________________________________________________________________________________________________
    global_average_pooling2d_1 (Glo (None, 2048)         0           mixed10[0][0]                    
    __________________________________________________________________________________________________
    dense_1 (Dense)                 (None, 512)          1049088     global_average_pooling2d_1[0][0] 
    __________________________________________________________________________________________________
    dense_2 (Dense)                 (None, 9)            4617        dense_1[0][0]                    
    ==================================================================================================
    Total params: 22,856,489
    Trainable params: 1,053,705
    Non-trainable params: 21,802,784
    __________________________________________________________________________________________________
    None
    

#### Training the Model

```python
# Compile with Adam
inceptionV3Model.compile(Adam(lr=.001), loss='categorical_crossentropy', metrics=['accuracy'])

```

I initially set the learning rate to be 0.0001, a standard from what I could find. 

After seeing how terrible the performance was, I tried increasing my learning rate to the current 0.001, but saw no major impact. Performance was slightly better, but still far from satisfactory as you will see below. 


```python
# Save the model according to the conditions  

checkpoint = ModelCheckpoint("inception_v3_1.h5", monitor='val_acc', verbose=2, save_best_only=True, save_weights_only=False, mode='auto', period=1)
early = EarlyStopping(monitor='val_acc', min_delta=0, patience=3, verbose=2, mode='auto')

inceptionV3Hist = inceptionV3Model.fit_generator(generator= train_generator,
                                     validation_data = val_generator,
                                     epochs = 10,
                                     callbacks=[checkpoint,early])
```

    Epoch 1/10
    87/87 [==============================] - 1201s 14s/step - loss: 0.7889 - accuracy: 0.7473 - val_loss: 15.0749 - val_accuracy: 0.0396
    Epoch 2/10
    87/87 [==============================] - 1198s 14s/step - loss: 0.5040 - accuracy: 0.8430 - val_loss: 13.7417 - val_accuracy: 0.0313
    Epoch 3/10
    87/87 [==============================] - 1197s 14s/step - loss: 0.3945 - accuracy: 0.8749 - val_loss: 12.5104 - val_accuracy: 0.0695
    Epoch 4/10
    87/87 [==============================] - 1198s 14s/step - loss: 0.3710 - accuracy: 0.8827 - val_loss: 20.5829 - val_accuracy: 0.0690
    Epoch 5/10
    87/87 [==============================] - 1197s 14s/step - loss: 0.3303 - accuracy: 0.8929 - val_loss: 23.1406 - val_accuracy: 0.0672
    Epoch 6/10
    87/87 [==============================] - 1212s 14s/step - loss: 0.2823 - accuracy: 0.9086 - val_loss: 34.0384 - val_accuracy: 0.0690
    Epoch 7/10
    87/87 [==============================] - 1200s 14s/step - loss: 0.2553 - accuracy: 0.9167 - val_loss: 29.0741 - val_accuracy: 0.0699
    Epoch 8/10
    87/87 [==============================] - 1206s 14s/step - loss: 0.2426 - accuracy: 0.9189 - val_loss: 12.7246 - val_accuracy: 0.0681
    Epoch 9/10
    87/87 [==============================] - 1220s 14s/step - loss: 0.2283 - accuracy: 0.9241 - val_loss: 19.8445 - val_accuracy: 0.0672
    Epoch 10/10
    87/87 [==============================] - 1201s 14s/step - loss: 0.2182 - accuracy: 0.9237 - val_loss: 28.0628 - val_accuracy: 0.0672
    


```python
inceptionV3Model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\inceptionV3Model.h5")
inceptionV3Hist.model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\inceptionV3Hist.h5")
```

## Results from the Inception V3 Model


```python
inceptionV3Hist_acc = inceptionV3Hist.history['accuracy']
inceptionV3Hist_val_acc = inceptionV3Hist.history['val_accuracy']
inceptionV3Hist_loss = inceptionV3Hist.history['loss']
inceptionV3Hist_val_loss = inceptionV3Hist.history['val_loss']

inceptionV3Hist_epochs = range(len(inceptionV3Hist_acc))

plt.axis([0, 10, 0, 1])


plt.plot(inceptionV3Hist_epochs, inceptionV3Hist_acc, 'b', label='Training acc')
plt.plot(inceptionV3Hist_epochs, inceptionV3Hist_val_acc, 'r', label='Validation acc')
plt.title('Training and validation accuracy')
plt.legend()

plt.figure()

plt.plot(inceptionV3Hist_epochs, inceptionV3Hist_loss, 'b', label='Training loss')
plt.plot(inceptionV3Hist_epochs, inceptionV3Hist_val_loss, 'r', label='Validation loss')
plt.title('Training and validation loss')
plt.legend()

plt.axis([0, 10, 0, 35])


plt.show()
```


![png](output_53_0.png)



![png](output_53_1.png)


#### Discussion of Results

As you can see, the performance on the inception V3 model is absolutely terrible. I was very perplexed as to how the model could overfit the training data so much to the point where accuracy on the testing data was basically zero. The two data sets shouldn't be that different, where even if it is overfit I would expect halfway decent performance on the testing. Obviously not the case.

After hours of searching, rerunning the model with different image augmentations, learning rates, even different architectures and layers, this was the BEST performance I achieved. Many of the preliminary models had training accuracies of 20%. The only solution I could find from other people having similar issues was that the model itself is just not well suited to what I need. Many encouraged trying the VGG16 model, which was also mentioned in the original paper, as even without fine tuning parameters, performance was much better.

# VGG16 Model - Attempt 1

#### Reading in the base VGG16 model

```python
#Get  the convolutional part of a VGG network trained on ImageNet
model_vgg16_conv = VGG16(weights='imagenet', include_top=False)
model_vgg16_conv.summary()
```

    Model: "vgg16"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    input_2 (InputLayer)         (None, None, None, 3)     0         
    _________________________________________________________________
    block1_conv1 (Conv2D)        (None, None, None, 64)    1792      
    _________________________________________________________________
    block1_conv2 (Conv2D)        (None, None, None, 64)    36928     
    _________________________________________________________________
    block1_pool (MaxPooling2D)   (None, None, None, 64)    0         
    _________________________________________________________________
    block2_conv1 (Conv2D)        (None, None, None, 128)   73856     
    _________________________________________________________________
    block2_conv2 (Conv2D)        (None, None, None, 128)   147584    
    _________________________________________________________________
    block2_pool (MaxPooling2D)   (None, None, None, 128)   0         
    _________________________________________________________________
    block3_conv1 (Conv2D)        (None, None, None, 256)   295168    
    _________________________________________________________________
    block3_conv2 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_conv3 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_pool (MaxPooling2D)   (None, None, None, 256)   0         
    _________________________________________________________________
    block4_conv1 (Conv2D)        (None, None, None, 512)   1180160   
    _________________________________________________________________
    block4_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_pool (MaxPooling2D)   (None, None, None, 512)   0         
    _________________________________________________________________
    block5_conv1 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_pool (MaxPooling2D)   (None, None, None, 512)   0         
    =================================================================
    Total params: 14,714,688
    Trainable params: 14,714,688
    Non-trainable params: 0
    _________________________________________________________________
    

#### Adding layers to the base model


To prevent the need to create an entirely new data set of resized images, I created a new input layer for the VGG16 model to feed in images with the shape 299x299. The normal model uses an image size of 224x224. I attempted this at first when experimenting with it, but saw no impact on final results by using the same data sets used to retrain the Inception V3 model for retraining the VGG16 model. 


```python
#Creating my own input layer (299x299x3)
input = Input(shape=(299,299,3),name = 'image_input')

#Use the generated model 
output_vgg16_conv = model_vgg16_conv(input)
```


```python
#Add the fully-connected layers 
y = Flatten(name='flatten')(output_vgg16_conv)
y = Dense(512, activation='relu', name='fc1')(y)
y = Dense(512, activation='relu', name='fc2')(y)
y = Dense(9, activation='softmax', name='predictions')(y)
```


```python
#Create your own model 
vgg16model = Model(input=input, output=y)

vgg16model.summary()
```

    Model: "model_2"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 36,216,137
    Non-trainable params: 0
    _________________________________________________________________
    

#### Freezing the base layers from being trained

Similar to retraining the inception V3 model, I also froze all of the base layers and chose to only train the final few layers that I created. 


```python
for layer in vgg16model.layers[:-4]:
    layer.trainable = False
    
for layer in vgg16model.layers:
    print(layer, layer.trainable)
```

    <keras.engine.input_layer.InputLayer object at 0x000000B71367A648> False
    <keras.engine.training.Model object at 0x000000B713669908> False
    <keras.layers.core.Flatten object at 0x000000B71369F208> True
    <keras.layers.core.Dense object at 0x000000B71369F1C8> True
    <keras.layers.core.Dense object at 0x000000B71369F388> True
    <keras.layers.core.Dense object at 0x000000B71369FB88> True
    


```python
vgg16model.summary()
```

    Model: "model_2"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 21,501,449
    Non-trainable params: 14,714,688
    _________________________________________________________________
    

#### Training the Model

```python
opt = Adam(lr=0.001)
vgg16model.compile(optimizer=opt, loss=keras.losses.categorical_crossentropy, metrics=['accuracy'])
```


```python
from keras.callbacks import ModelCheckpoint, EarlyStopping
checkpoint = ModelCheckpoint("vgg16_1.h5", monitor='val_acc', verbose=2, save_best_only=True, save_weights_only=False, mode='auto', period=1)
early = EarlyStopping(monitor='val_acc', min_delta=0, patience=3, verbose=2, mode='auto')
vgg16modelHist = vgg16model.fit_generator(
                           generator=train_generator, 
                           validation_data= val_generator, 
                           epochs=10,
                           callbacks=[checkpoint,early], 
                           )
```

    Epoch 1/10
    87/87 [==============================] - 1562s 18s/step - loss: 0.9233 - accuracy: 0.8007 - val_loss: 0.3266 - val_accuracy: 0.8855
    Epoch 2/10
    87/87 [==============================] - 1540s 18s/step - loss: 0.2224 - accuracy: 0.9340 - val_loss: 0.0208 - val_accuracy: 0.9154
    Epoch 3/10
    87/87 [==============================] - 1543s 18s/step - loss: 0.1327 - accuracy: 0.9599 - val_loss: 0.0397 - val_accuracy: 0.9264
    Epoch 4/10
    87/87 [==============================] - 1529s 18s/step - loss: 0.0753 - accuracy: 0.9770 - val_loss: 0.3092 - val_accuracy: 0.9319
    Epoch 5/10
    87/87 [==============================] - 1525s 18s/step - loss: 0.0468 - accuracy: 0.9867 - val_loss: 0.0017 - val_accuracy: 0.9319
    Epoch 6/10
    87/87 [==============================] - 1541s 18s/step - loss: 0.0419 - accuracy: 0.9879 - val_loss: 8.6230e-04 - val_accuracy: 0.9333
    Epoch 7/10
    87/87 [==============================] - 1549s 18s/step - loss: 0.0331 - accuracy: 0.9901 - val_loss: 0.0013 - val_accuracy: 0.9181
    Epoch 8/10
    87/87 [==============================] - 1538s 18s/step - loss: 0.0520 - accuracy: 0.9860 - val_loss: 1.4007e-06 - val_accuracy: 0.9416
    Epoch 9/10
    87/87 [==============================] - 1541s 18s/step - loss: 0.0454 - accuracy: 0.9868 - val_loss: 3.4152 - val_accuracy: 0.9301
    Epoch 10/10
    87/87 [==============================] - 1532s 18s/step - loss: 0.0300 - accuracy: 0.9901 - val_loss: 7.7557e-04 - val_accuracy: 0.9338
    


```python
vgg16model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16model.h5")
vgg16modelHist.model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16modelHist.h5")
```

## Results from VGG16 - Attempt 1


```python
vgg16modelHist_acc = vgg16modelHist.history['accuracy']
vgg16modelHist_val_acc = vgg16modelHist.history['val_accuracy']
vgg16modelHist_loss = vgg16modelHist.history['loss']
vgg16modelHist_val_loss = vgg16modelHist.history['val_loss']

vgg16modelHist_epochs = range(len(vgg16modelHist_acc))

plt.axis([0, 10, 0, 1])

plt.plot(vgg16modelHist_epochs, vgg16modelHist_acc, 'b', label='Training acc')
plt.plot(vgg16modelHist_epochs, vgg16modelHist_val_acc, 'r', label='Validation acc')
plt.title('Training and validation accuracy')
plt.legend()

plt.figure()

plt.plot(vgg16modelHist_epochs, vgg16modelHist_loss, 'b', label='Training loss')
plt.plot(vgg16modelHist_epochs, vgg16modelHist_val_loss, 'r', label='Validation loss')
plt.title('Training and validation loss')
plt.legend()

plt.axis([0, 10, 0, 35])


plt.show()
```


![png](output_70_0.png)



![png](output_70_1.png)


#### Analysis of VGG16 Model Results

As you can see, the model quickly overfit the training data, reaching an overall accuracy of essentially 1 by the end of training. However, unlike the inception V3 model, I was still able to obtain fairly decent performance on the validation data. Ovearll, I am pleased that I am able to achieve somewhat close to similar results as the original paper, which claimed accuracies of 99% on testing data. I do believe there is room for improvement. To prevent such overfitting, I want to retrain the model using a lower learning rate. 

# VGG Model - Attempt 2

#### Reading in the base VGG16 model

```python
#Get back the convolutional part of a VGG network trained on ImageNet
model_vgg16_conv = VGG16(weights='imagenet', include_top=False)
model_vgg16_conv.summary()
```

    Model: "vgg16"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    input_3 (InputLayer)         (None, None, None, 3)     0         
    _________________________________________________________________
    block1_conv1 (Conv2D)        (None, None, None, 64)    1792      
    _________________________________________________________________
    block1_conv2 (Conv2D)        (None, None, None, 64)    36928     
    _________________________________________________________________
    block1_pool (MaxPooling2D)   (None, None, None, 64)    0         
    _________________________________________________________________
    block2_conv1 (Conv2D)        (None, None, None, 128)   73856     
    _________________________________________________________________
    block2_conv2 (Conv2D)        (None, None, None, 128)   147584    
    _________________________________________________________________
    block2_pool (MaxPooling2D)   (None, None, None, 128)   0         
    _________________________________________________________________
    block3_conv1 (Conv2D)        (None, None, None, 256)   295168    
    _________________________________________________________________
    block3_conv2 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_conv3 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_pool (MaxPooling2D)   (None, None, None, 256)   0         
    _________________________________________________________________
    block4_conv1 (Conv2D)        (None, None, None, 512)   1180160   
    _________________________________________________________________
    block4_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_pool (MaxPooling2D)   (None, None, None, 512)   0         
    _________________________________________________________________
    block5_conv1 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_pool (MaxPooling2D)   (None, None, None, 512)   0         
    =================================================================
    Total params: 14,714,688
    Trainable params: 14,714,688
    Non-trainable params: 0
    _________________________________________________________________
    


#### Adding layers to the base model

```python
#Create your own input format (here 299x299x3)
input = Input(shape=(299,299,3),name = 'image_input')

#Use the generated model 
output_vgg16_conv = model_vgg16_conv(input)
```


```python
#Add the fully-connected layers 
y = Flatten(name='flatten')(output_vgg16_conv)
y = Dense(512, activation='relu', name='fc1')(y)
y = Dense(512, activation='relu', name='fc2')(y)
y = Dense(9, activation='softmax', name='predictions')(y)
```


```python
#Create your own model 
vgg16modelV2 = Model(input=input, output=y)

#In the summary, weights and layers from VGG part will be hidden, but they will be fit during the training
vgg16modelV2.summary()
```

    Model: "model_3"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 36,216,137
    Non-trainable params: 0
    _________________________________________________________________
    

#### Freezing the base layers from being trained

```python
for layer in vgg16modelV2.layers[:-4]:
    layer.trainable = False
    
for layer in vgg16modelV2.layers:
    print(layer, layer.trainable)
```

    <keras.engine.input_layer.InputLayer object at 0x000000B7138E9C08> False
    <keras.engine.training.Model object at 0x000000B7138D3E08> False
    <keras.layers.core.Flatten object at 0x000000B71391B248> True
    <keras.layers.core.Dense object at 0x000000B71391B208> True
    <keras.layers.core.Dense object at 0x000000B71391B3C8> True
    <keras.layers.core.Dense object at 0x000000B71391B588> True
    


```python
vgg16modelV2.summary()
```

    Model: "model_3"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 21,501,449
    Non-trainable params: 14,714,688
    _________________________________________________________________
    

#### Training the Model

```python
opt = Adam(lr=0.0001)
vgg16modelV2.compile(optimizer=opt, loss=keras.losses.categorical_crossentropy, metrics=['accuracy'])
```


```python
checkpoint = ModelCheckpoint("vgg16_1.h5", monitor='val_acc', verbose=2, save_best_only=True, save_weights_only=False, mode='auto', period=1)
early = EarlyStopping(monitor='val_acc', min_delta=0, patience=3, verbose=2, mode='auto')
vgg16modelV2Hist = vgg16modelV2.fit_generator(
                           generator=train_generator, 
                           validation_data= val_generator, 
                           epochs=10,
                           callbacks=[checkpoint,early], 
                           )
```

    Epoch 1/10
    87/87 [==============================] - 1529s 18s/step - loss: 0.5072 - accuracy: 0.8482 - val_loss: 0.4219 - val_accuracy: 0.8983
    Epoch 2/10
    87/87 [==============================] - 1540s 18s/step - loss: 0.1959 - accuracy: 0.9418 - val_loss: 0.0049 - val_accuracy: 0.9213
    Epoch 3/10
    87/87 [==============================] - 1538s 18s/step - loss: 0.1024 - accuracy: 0.9723 - val_loss: 1.2242 - val_accuracy: 0.9080
    Epoch 4/10
    87/87 [==============================] - 1518s 17s/step - loss: 0.0640 - accuracy: 0.9831 - val_loss: 0.0441 - val_accuracy: 0.9370
    Epoch 5/10
    87/87 [==============================] - 1525s 18s/step - loss: 0.0339 - accuracy: 0.9924 - val_loss: 0.0053 - val_accuracy: 0.9420
    Epoch 6/10
    87/87 [==============================] - 1540s 18s/step - loss: 0.0249 - accuracy: 0.9946 - val_loss: 0.0013 - val_accuracy: 0.9430
    Epoch 7/10
    87/87 [==============================] - 1528s 18s/step - loss: 0.0114 - accuracy: 0.9986 - val_loss: 0.2620 - val_accuracy: 0.9443
    Epoch 8/10
    87/87 [==============================] - 1532s 18s/step - loss: 0.0105 - accuracy: 0.9984 - val_loss: 3.6149e-05 - val_accuracy: 0.9407
    Epoch 9/10
    87/87 [==============================] - 1526s 18s/step - loss: 0.0062 - accuracy: 0.9993 - val_loss: 0.0144 - val_accuracy: 0.9434
    Epoch 10/10
    87/87 [==============================] - 1549s 18s/step - loss: 0.0037 - accuracy: 0.9998 - val_loss: 5.7883e-04 - val_accuracy: 0.9462
    


```python
vgg16modelV2.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16modelV2.h5")
vgg16modelV2Hist.model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16modelV2Hist.h5")
```

## Results from VGG16 Model attempt 2


```python
vgg16modelV2Hist_acc = vgg16modelV2Hist.history['accuracy']
vgg16modelV2Hist_val_acc = vgg16modelV2Hist.history['val_accuracy']
vgg16modelV2Hist_loss = vgg16modelV2Hist.history['loss']
vgg16modelV2Hist_val_loss = vgg16modelV2Hist.history['val_loss']

vgg16modelV2Hist_epochs = range(len(vgg16modelV2Hist_acc))

plt.axis([0, 10, 0, 1])

plt.plot(vgg16modelV2Hist_epochs, vgg16modelV2Hist_acc, 'b', label='Training acc')
plt.plot(vgg16modelV2Hist_epochs, vgg16modelV2Hist_val_acc, 'r', label='Validation acc')
plt.title('Training and validation accuracy')
plt.legend()

plt.figure()

plt.plot(vgg16modelV2Hist_epochs, vgg16modelV2Hist_loss, 'b', label='Training loss')
plt.plot(vgg16modelV2Hist_epochs, vgg16modelV2Hist_val_loss, 'r', label='Validation loss')
plt.title('Training and validation loss')
plt.legend()

plt.axis([0, 10, 0, 35])


plt.show()
```


![png](output_85_0.png)



![png](output_85_1.png)


#### Analysis of results 

To my surprise, lowering the learning rate had only a minimal impact on the overfitting of the training data. Looking at the top graph plotting the training and validation accuracy, the training accuracy was "smoother" ie it took longer to reach an accuracy of 1. Based on the two attempts to retrain the model, it looks like an epoch of 1 might actually provide the best and most stable performance across the two data sets. 

# Final VGG16 Model 

#### Reading in the Base Model
```python
#Get back the convolutional part of a VGG network trained on ImageNet
model_vgg16_conv = VGG16(weights='imagenet', include_top=False)
model_vgg16_conv.summary()
```

    Model: "vgg16"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    input_4 (InputLayer)         (None, None, None, 3)     0         
    _________________________________________________________________
    block1_conv1 (Conv2D)        (None, None, None, 64)    1792      
    _________________________________________________________________
    block1_conv2 (Conv2D)        (None, None, None, 64)    36928     
    _________________________________________________________________
    block1_pool (MaxPooling2D)   (None, None, None, 64)    0         
    _________________________________________________________________
    block2_conv1 (Conv2D)        (None, None, None, 128)   73856     
    _________________________________________________________________
    block2_conv2 (Conv2D)        (None, None, None, 128)   147584    
    _________________________________________________________________
    block2_pool (MaxPooling2D)   (None, None, None, 128)   0         
    _________________________________________________________________
    block3_conv1 (Conv2D)        (None, None, None, 256)   295168    
    _________________________________________________________________
    block3_conv2 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_conv3 (Conv2D)        (None, None, None, 256)   590080    
    _________________________________________________________________
    block3_pool (MaxPooling2D)   (None, None, None, 256)   0         
    _________________________________________________________________
    block4_conv1 (Conv2D)        (None, None, None, 512)   1180160   
    _________________________________________________________________
    block4_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block4_pool (MaxPooling2D)   (None, None, None, 512)   0         
    _________________________________________________________________
    block5_conv1 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv2 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_conv3 (Conv2D)        (None, None, None, 512)   2359808   
    _________________________________________________________________
    block5_pool (MaxPooling2D)   (None, None, None, 512)   0         
    =================================================================
    Total params: 14,714,688
    Trainable params: 14,714,688
    Non-trainable params: 0
    _________________________________________________________________
    


#### Adding layers to the base model 

```python
#Create your own input format (here 299x299x3)
input = Input(shape=(299,299,3),name = 'image_input')

#Use the generated model 
output_vgg16_conv = model_vgg16_conv(input)
```


```python
#Add the fully-connected layers 
y = Flatten(name='flatten')(output_vgg16_conv)
y = Dense(512, activation='relu', name='fc1')(y)
y = Dense(512, activation='relu', name='fc2')(y)
y = Dense(9, activation='softmax', name='predictions')(y)
```


```python
#Create your own model 
vgg16modelFinal = Model(input=input, output=y)

vgg16modelFinal.summary()
```

    Model: "model_4"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 36,216,137
    Non-trainable params: 0
    _________________________________________________________________
    
#### Freezing the base layers from being trained

```python
for layer in vgg16modelFinal.layers[:-4]:
    layer.trainable = False
    
for layer in vgg16modelFinal.layers:
    print(layer, layer.trainable)
```

    <keras.engine.input_layer.InputLayer object at 0x000000B73F40F8C8> False
    <keras.engine.training.Model object at 0x000000B73F3F9D88> False
    <keras.layers.core.Flatten object at 0x000000B73F556948> True
    <keras.layers.core.Dense object at 0x000000B73F556888> True
    <keras.layers.core.Dense object at 0x000000B73F556C48> True
    <keras.layers.core.Dense object at 0x000000B73F55D448> True
    


```python
vgg16modelFinal.summary()
```

    Model: "model_4"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    image_input (InputLayer)     (None, 299, 299, 3)       0         
    _________________________________________________________________
    vgg16 (Model)                multiple                  14714688  
    _________________________________________________________________
    flatten (Flatten)            (None, 41472)             0         
    _________________________________________________________________
    fc1 (Dense)                  (None, 512)               21234176  
    _________________________________________________________________
    fc2 (Dense)                  (None, 512)               262656    
    _________________________________________________________________
    predictions (Dense)          (None, 9)                 4617      
    =================================================================
    Total params: 36,216,137
    Trainable params: 21,501,449
    Non-trainable params: 14,714,688
    _________________________________________________________________
    
#### Training the model

```python
opt = Adam(lr=0.0001)
vgg16modelFinal.compile(optimizer=opt, loss=keras.losses.categorical_crossentropy, metrics=['accuracy'])
```


```python
checkpoint = ModelCheckpoint("vgg16_1.h5", monitor='val_acc', verbose=2, save_best_only=True, save_weights_only=False, mode='auto', period=1)
early = EarlyStopping(monitor='val_acc', min_delta=0, patience=3, verbose=2, mode='auto')
vgg16modelFinalHist = vgg16modelFinal.fit_generator(
                           generator=train_generator, 
                           validation_data= val_generator, 
                           epochs=1,
                           callbacks=[checkpoint,early], 
                           )
```

    Epoch 1/1
    87/87 [==============================] - 1582s 18s/step - loss: 0.5388 - accuracy: 0.8393 - val_loss: 1.2721 - val_accuracy: 0.8997
    


```python
vgg16modelFinal.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16modelFinal.h5")
vgg16modelFinalHist.model.save(r"C:\Users\fairwemr\ISA 480\FinalProject\vgg16modelFinalHist.h5")
```

## Results from Final Model

#### Getting predictions from Final Model

```python
predictions =  vgg16modelFinal.predict(x_validation)
```


```python
y_validation_np =  y_validation.to_numpy()
```

The function below was found from: https://stackoverflow.com/questions/44054534/confusion-matrix-error-when-array-dimensions-are-of-size-3

#### Confusion matrix of the prediction errors


```python

def pretty_print_conf_matrix(y_true, y_pred, 
                             classes,
                             normalize=False,
                             title='Confusion matrix',
                             cmap=plt.cm.Blues):

    cm = confusion_matrix(y_true, y_pred)

    # Configure Confusion Matrix Plot Aesthetics (no text yet) 
    plt.imshow(cm, interpolation='nearest', cmap=cmap)
    plt.title(title, fontsize=14)
    tick_marks = np.arange(len(classes))
    plt.xticks(tick_marks, classes, rotation=45)
    plt.yticks(tick_marks, classes)
    plt.ylabel('True label', fontsize=12)
    plt.xlabel('Predicted label', fontsize=12)

    # Calculate normalized values (so all cells sum to 1) if desired
    if normalize:
        cm = np.round(cm.astype('float') / cm.sum(),2) #(axis=1)[:, np.newaxis]

    # Place Numbers as Text on Confusion Matrix Plot
    thresh = cm.max() / 2.
    for i, j in itertools.product(range(cm.shape[0]), range(cm.shape[1])):
        plt.text(j, i, cm[i, j],
                 horizontalalignment="center",
                 color="white" if cm[i, j] > thresh else "black",
                 fontsize=12)



    plt.tight_layout()

```


```python
# Plot Confusion Matrix
plt.style.use('classic')
plt.figure(figsize=(15,15))
pretty_print_conf_matrix(y_validation_np.argmax(axis=1), predictions.argmax(axis=1), 
                         classes= ['0',' 1', '2', '3', '4', '5', '6', '7', '8'],
                         normalize=False, 
                         title='Confusion Matrix')
```


![png](output_104_0.png)


As you can see, the final model did a very good job at classifying the validation images in to their proper category. As expected there is some error in this, but overall I am pleased. I found it interesting that the model had trouble with predicting the label 1 category. As you can see, it predicted "1" for quite a few wrong categories, especially when the ground truth was a label of 5.

#### Metrics on prediction performance

```python
 # Show Precision, Recall, F-1 Score
rpt = classification_report(y_validation_np.argmax(axis=1), predictions.argmax(axis=1))
rpt = rpt.replace('avg / total', '      avg')
rpt = rpt.replace('support', 'N Obs')
print(rpt)
```

                  precision    recall  f1-score   N Obs
    
               0       0.90      0.79      0.84       308
               1       0.81      0.95      0.88       496
               2       1.00      0.99      1.00       588
               3       0.90      0.92      0.91        95
               4       0.00      0.00      0.00         8
               5       0.74      0.61      0.67       150
               6       1.00      0.95      0.97        80
               7       0.90      0.89      0.90       246
               8       0.93      0.91      0.92       203
    
        accuracy                           0.90      2174
       macro avg       0.80      0.78      0.79      2174
    weighted avg       0.90      0.90      0.90      2174
    
    

# Next Steps

There are several other areas the paper explored that I have been unable to mainly because of computational ability. I'll walk through several below. 

1. The paper explored the process of training a smaller neural network from scratch to compare it to the pre-trained model from keras. I do not have the computational ability to explore this unfortunately. However, the paper was unable to reach a level of performance from the model trained from scratch that compared to the pre-trained keras model. Because of this, I am not too worried that not building one from scratch will change my final conclusions. 

2. The original paper also explored instead of classifying malware based on their family, but instead classify them either as a benign file or a malicious file. Unfortunately this isn't possible witht the data that I have. The binary files that I have access to are only malicious files, and thus I am unable to explore this. I do think this area would be more interesting and particularly useful as a whole. At least for me, I care more about determining whether or not file is malicious than just determining what family of malware it falls in. 

3. The original paper also explored several other models, including SVM's with a radial kernel and linear kernel. I explored this area, and how I could apply the type of data that I had to these types of classification models. Unfortunately, I wasn't quite able to figure out how this could be done in the time that I have to work and my current level of knowledge. I will continue to try and figure this problem out, as the area of image classification is extremely fascinating to me. 

4. I recognize that the validation performance of my data is not quite on par with the reported performance of the paper (accuracy of 90% for my final model versus 99% in the paper). Additionally, I recognize that the discrepency between training performance at 99% and validation at 90% would certainly indicate overfitting. If I had the knowledge on what layers to add to the VGG16 model to help prevent this, I think I could boost overall performance.

# Final Thoughts

Overall, I am very pleased with how much I was able to accomplish in working through this paper. I have never touched image classification before in any class or personal project, so I had to start from scratch in terms of learning how this is done. 

Additionally, this is my first time using Keras pre-trained models for transfer learning, and I am incredibly impressed at how simple the interface is as well as how accurate one can get a pre-trained model. Most of these models were trained on images like dogs, cats, cereal boxes, etc., so the fact that they can perform so well with just a little fine tuning on RGB images of malware byte code is very cool. 

I recognize there are parts of the paper I was unable to touch. Although I tried to hit the major components, there were a lot of smaller details that were very fascinating and certainly worth exploring further on my own. If I can find a good data set that has both malicious and benign byte code files, then I will certainly retrain these models for binary malware classification. 

Finally, I would love to explore some of the other machine learning models on this data, particularly their use of the XGBTree algorithm combined with PCA. I find their decision to keep only 80 components out of the original 50176 predictors to be concerning, especially with no discussion as to how much variation was kept in those 80. An overall accuracy of 96% on the XGBTree compared to their reported accuracy of 99% on Keras models I believe could probably be boosted further by more fine tuning, and potentially getting the XGBTree to perform on par with the keras neural networks.

# Resources

https://towardsdatascience.com/step-by-step-vgg16-implementation-in-keras-for-beginners-a833c686ae6c
https://stackoverflow.com/questions/45943675/meaning-of-validation-steps-in-keras-sequential-fit-generator-parameter-list
https://datascience.stackexchange.com/questions/29893/high-model-accuracy-vs-very-low-validation-accuarcy
https://www.kaggle.com/fujisan/use-keras-pre-trained-vgg16-acc-98
https://github.com/keras-team/keras/issues/4465
https://www.kaggle.com/careyai/inceptionv3-full-pretrained-model-instructions
https://stackoverflow.com/questions/44054534/confusion-matrix-error-when-array-dimensions-are-of-size-3
