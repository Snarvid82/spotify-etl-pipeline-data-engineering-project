# Spotify ETL Pipeline | AWS Data Engineering Project

## Introduction
This end-to-end data engineering project I have completed is part of the "Python for Data Engineering" course from DataVidhya.

### The Project Case
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



