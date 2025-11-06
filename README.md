# Spotify ETL Pipeline | AWS Data Engineering Project

## Introduction
This end-to-end data engineering project I have completed is part of the "Python for Data Engineering" course from DataVidhya.

## The Project Case
A business wants to get information on what kind of music is globally trending. As part of the music production industry it wants to focus on music that is being listened to by a wide audience.
Data from Spotify will be able to provide very useful insights for the business by finding the most played songs, artists and albums every week.
The data will be used long term for finding trends and insights.

The business has the following requirements for this project:
- Make services in AWS part of the pipeline solution as this is it's primary cloud provider and it's familiar to them.
- Minimum responsibility and work of managing the (infrastructure) for the pipeline.
- An ETL (Extract, Transform & Load) data pipeline.

## Data Pipeline Architecture
![Data pipeline architecture.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Data%20pipeline%20architecture.png)

## Technology Used
**Spotify API:** This is provided by Spotify and makes it possible to extract data from Spotify and make use of it.

**Python:** The code will be written in Python and deployed throughout the data pipeline.

**Amazon CloudWatch:** This will be used to apply a trigger that will make sure data is collected every week.

**AWS Lambda:** This is a serverless compute service deploying the Python code once the triggers from CloudWatch and Amazon S3 are executed. Being serverless means the business will not have to worry about managing the servers as this is handled automatically by AWS.
Airflow could be used instead of AWS Lambda but I chose to use Lambda as the business was already familiar with the different AWS services.

**AmazonS3:** This is where the actual data will be stored. One S3 bucket will be used to store the raw data and one for storing the transformed data. S3 is a highly scalable storage solution that automatically scales up or down to accomodate data volumes.

**AWS Glue Crawler:** This will automatically "crawl" through all the transformed data and make sense of rows, columns, data types etc. and infer schemas.

**AWS Glue Data Catalog:** This is a stored catalog containing metadata that makes it easy to discover and manage data in AWS.

**Amazon Athena:** This is used to run SQL queries directly in the S3 storage or in the Glue Data catalog to analyze data.

## Data Used
The data in this project is collected from [this Spotify list](https://open.spotify.com/playlist/6UeSakyzhiEt4NB3UAd6NQ).

## Project Preparation
At first I created a new app at the Spotify for developers website. This was quite straight-forward…simply log in, click "Create app" and follow the steps.

*I also had to provide a Redirect URI but I found detailed instructions on the [Spotify for developers documentation page](https://developer.spotify.com/documentation/web-api/concepts/apps).*

Next step was to create a new Notebook in Jupyter and install "Spotipy", a lightweight Python library for the Spotify Web API.
Then I imported Spotipy and wrote the code necessary for credentials (authentication and authorization).

```
!pip install spotipy
```

```
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import pandas as pd
```

```
client_credentials_manager = SpotifyClientCredentials(client_id="XXXX", client_secret="XXXX")
```
```
sp = spotipy.Spotify(client_credentials_manager = client_credentials_manager)
```


My "client_id" and "client_secret" were found by clicking on my app on my Spotify for developers dashboard.

![Spotify app dashboard.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Spotify%20app%20dashboard.png)

Now the setup is ready to start extracting data from the Spotify API.

I used the Python `.split` command to get the exact information I needed and defined it as a variable called "Playlist_URI".
I stored the data as a variable called "data" and had a thorough look through the data I was dealing with.

```
playlist_link = "https://open.spotify.com/playlist/6UeSakyzhiEt4NB3UAd6NQ"
```

```
playlist_URI = playlist_link.split("/")[-1]
```

```
data = sp.playlist_tracks(playlist_URI)
```


## Extraction and Transformation
Then I would start the work of extracting and transforming the data. The business needed information regarding songs, artists and albums.

The data was quite messy so I needed to properly structure the data and extract the information that the business actually needs.

First I extracted all the relevant data related to album information, artist information and song information and created different dictionaries for them.

```
album_list = []
for row in data['items']:
    album_id = row['track']['album']['id']
    album_name = row['track']['album']['name']
    album_release_date = row['track']['album']['release_date']
    album_total_tracks = row['track']['album']['total_tracks']
    album_url = row['track']['album']['external_urls']['spotify']
    album_element = {'album_id':album_id,'name':album_name, 'release_date':album_release_date,'total_tracks':album_total_tracks,'url':album_url}
    album_list.append(album_element)
```

```
artist_list = []
for row in data['items']:
    for key, value in row.items():
        if key == "track":
            for artist in value['artists']:
                artist_dict = {'artist_id':artist['id'], 'artist_name':artist['name'], 'external_url':artist['href']}
                artist_list.append(artist_dict)
```

```
song_list = []
for row in data['items']:
    song_id = row['track']['id']
    song_name = row['track']['name']
    song_duration = row['track']['duration_ms']
    song_url = row['track']['external_urls']['spotify']
    song_popularity = row['track']['popularity']
    song_added = row['added_at']
    album_id = row['track']['album']['id']
    artist_id = row['track']['album']['artists'][0]['id']
    song_element = {'song_id':song_id,'song_name':song_name,'duration_ms':song_duration,'url':song_url,
                    'popularity':song_popularity,'song_added':song_added,'album_id':album_id,'artist_id':artist_id
                   }
    song_list.append(song_element)
```

Next I wanted to convert the extractions into a proper dataframe to make the data more organized and readable.

```
album_df = pd.DataFrame.from_dict(album_list)
```

```
artist_df = pd.DataFrame.from_dict(artist_list)
```

```
song_df = pd.DataFrame.from_dict(song_list)
```

I went on to drop duplicates in the data and changed data type for date/time where necessary.

```
album_df = album_df.drop_duplicates(subset=['album_id'])
```

```
album_df['release_date'] = pd.to_datetime(album_df['release_date'])
```

## Deployment to Cloud
So far I have extracted and transformed the data needed by the business.
Next step is using AWS to set things up in the cloud.

![Data pipeline architecture - Status 1.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Data%20pipeline%20architecture%20-%20Status%201.png)

I already have an AWS account and started by creating a S3 bucket with 2 folders: "raw_data" and "transformed_data". 
The sub-folders inside these 2 folders can be seen below:

![Folders inside raw_data folder.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Folders%20inside%20raw_data%20folder.png)

![Folders inside transformed_data folder.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Folders%20inside%20transformed_data%20folder.png)


I created a Lambda function in AWS, updated the necessary code and ran a test to see that the api was working as expected.

I also needed to create a layer for Lambda to be able to use Spotify. This was done by creating the layer in AWS and uploading a spotify_layer.zip file. 
Then I applied the layer to the Lambda function.

Next step is to store this data in the S3 bucket. For that I will use the command `import boso3` in the lambda function code. 
This function package was created by AWS in order to communicate properly with AWS services.

I also needed to attach a permission policy giving the necessary permissions for S3 actions.

Additionally I needed to adjust the "timeout" setting in the Lambda function. This was initially set to only 3 seconds and I changed it to 90 seconds.


*I encountered some errors while running the Lambda tests. These errors were connected to small typos, some missing code and the timeout setting.
Errors are part of the learning process and fixing errors is a great skill to have.*

Code for the data extraction Lambda function:
```
import json
import os
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import boto3
from datetime import datetime

def lambda_handler(event, context):

    client_id = os.environ.get('client_id')
    client_secret = os.environ.get('client_secret')


    client_credentials_manager = SpotifyClientCredentials(client_id=client_id, client_secret=client_secret)
    sp = spotipy.Spotify(client_credentials_manager = client_credentials_manager)
    playlists = sp.user_playlists('spotify')

    playlist_link = "https://open.spotify.com/playlist/6UeSakyzhiEt4NB3UAd6NQ"
    playlist_URI = playlist_link.split("/")[-1].split("?")[0]

    spotify_data = sp.playlist_tracks(playlist_URI)

    client = boto3.client('s3')

    filename = "spotify_raw_" + str(datetime.now()) + ".json"

    client.put_object(
        Bucket='spotify-etl-project-arvid',
        Key='raw_data/to_processed/' + filename,
        Body=json.dumps(spotify_data)
    )
```


Data had now been stored in the S3 raw_data/to_processed folder.

Status on the data pipeline so far:
![Data pipeline architecture - Status 2.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Data%20pipeline%20architecture%20-%20Status%202.png)


Next I will set up the transformation by creating a new Lambda function in AWS. This function will transform the raw data and put it into the "raw_data/processed" folder.
From there it will be moved to the "transformed_data" folder in S3, and the data in "to_processed" will be automatically deleted.

Code for the data transformation and load Lambda function:
```
import json
import boto3
from datetime import datetime
from io import StringIO
import pandas as pd

def album(data):
    album_list = []
    for row in data['items']:
        album_id = row['track']['album']['id']
        album_name = row['track']['album']['name']
        album_release_date = row['track']['album']['release_date']
        album_total_tracks = row['track']['album']['total_tracks']
        album_url = row['track']['album']['external_urls']['spotify']
        album_element = {'album_id':album_id,'name':album_name, 'release_date':album_release_date,'total_tracks':album_total_tracks,'url':album_url}
        album_list.append(album_element)
    return album_list

def artist(data):
    artist_list = []
    for row in data['items']:
        for key, value in row.items():
            if key == "track":
                for artist in value['artists']:
                    artist_dict = {'artist_id':artist['id'], 'artist_name':artist['name'], 'external_url':artist['href']}
                    artist_list.append(artist_dict)
    return artist_list

def songs(data):
    song_list = []
    for row in data['items']:
        song_id = row['track']['id']
        song_name = row['track']['name']
        song_duration = row['track']['duration_ms']
        song_url = row['track']['external_urls']['spotify']
        song_popularity = row['track']['popularity']
        song_added = row['added_at']
        album_id = row['track']['album']['id']
        artist_id = row['track']['album']['artists'][0]['id']
        song_element = {'song_id':song_id,'song_name':song_name,'duration_ms':song_duration,'url':song_url,
                        'popularity':song_popularity,'song_added':song_added,'album_id':album_id,'artist_id':artist_id
                 }
        song_list.append(song_element)

    return song_list
    

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = 'spotify-etl-project-arvid'
    key = 'raw_data/to_processed/'

    spotify_data = []
    spotify_keys = []
    for file in s3.list_objects(Bucket=bucket, Prefix=key)['Contents']:
        file_key = file['Key']
        if file_key.split('.')[-1] == 'json':
            response = s3.get_object(Bucket=bucket, Key=file_key)
            content = response['Body']
            jsonObject = json.loads(content.read())
            spotify_data.append(jsonObject)
            spotify_keys.append(file_key)
            
    for data in spotify_data:
        song_list = songs(data)
        artist_list = artist(data)
        album_list = album(data)

        album_df = pd.DataFrame.from_dict(album_list)
        album_df = album_df.drop_duplicates(subset=['album_id'])

        artist_df = pd.DataFrame.from_dict(artist_list)
        artist_df = artist_df.drop_duplicates(subset=['artist_id'])

        song_df = pd.DataFrame.from_dict(song_list)

        album_df['release_date'] = pd.to_datetime(album_df['release_date'])
        song_df['song_added'] = pd.to_datetime(song_df['song_added'])

        songs_key = "transformed_data/songs_data/songs_transformed_" + str(datetime.now()) + ".csv"
        song_buffer = StringIO()
        song_df.to_csv(song_buffer, index=False)
        song_content = song_buffer.getvalue()
        s3.put_object(Bucket=bucket, Key=songs_key, Body=song_content)

        album_key = "transformed_data/album_data/album_transformed_" + str(datetime.now()) + ".csv"
        album_buffer = StringIO()
        album_df.to_csv(album_buffer, index=False)
        album_content = album_buffer.getvalue()
        s3.put_object(Bucket=bucket, Key=album_key, Body=album_content)

        artist_key = "transformed_data/artist_data/artist_transformed_" + str(datetime.now()) + ".csv"
        artist_buffer = StringIO()
        artist_df.to_csv(artist_buffer, index=False)
        artist_content = artist_buffer.getvalue()
        s3.put_object(Bucket=bucket, Key=artist_key, Body=artist_content)

    s3_resource = boto3.resource('s3')
    for key in spotify_keys:
        copy_source = {
            'Bucket': bucket,
            'Key': key
        }
        s3_resource.meta.client.copy(copy_source, bucket, 'raw_data/processed/' + key.split('/')[-1])
        s3_resource.Object(bucket, key).delete()
```

Next I needed to automate the process by creating triggers for the two Lambda functions.

For the Lambda function for extraction of data ("spotify_api_data_extract") I set up an EventBridge (CloudWatch Events) 
trigger to run every 1 minute (for testing purposes).

*I deleted this trigger after all the testing was complete in order to avoid any possible AWS charges.*

![EventBridge trigger for data extraction.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/EventBridge%20trigger%20for%20data%20extraction.png)

For the Lambda function for transformation of data (spotify_transformation_load_function) I created a S3 trigger. 
Here I also needed to update the permissions by including the AWSLambdaRole.

![S3 trigger for data transformation.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/S3%20trigger%20for%20data%20transformation.png)

Further, I needed to make sure the two Lambda functions were allowed to communicate with each other.

## Loading
I have now reached the Load part of the ETL pipeline.

![Data pipeline architecture - Status 3.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Data%20pipeline%20architecture%20-%20Status%203.png)

First I created the crawlers which will go through all the processed data and make sense of rows, columns, data types etc. and infer schemas.
I made one crawler for songs, one for artist and one for album.

![Created 3 crawlers.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Created%203%20crawlers.png)

When running the crawlers the schema for artist data was not displaying column name properly. 
I manually edited the schema (JSON code) to correct this.

![Column name not displayed properly.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Column%20name%20not%20displayed%20properly.png)

![JSON code edited manually.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/JSON%20code%20edited%20manually.png)

The crawlers created data catalogs and provided tables for each category (song, artist and albums).

![Tables created by the crawlers.png](https://github.com/Snarvid82/spotify-etl-pipeline-data-engineering-project/blob/main/Tables%20created%20by%20the%20crawlers.png)

By using the Query Editor in Amazon Athena I could now perform SQL queries on the tables made by the crawler.
By completing the ETL data pipeline the business could now take full advantage of the data.



