
# Simple Netflix EDA

### Database source: https://www.kaggle.com/datasets/shivamb/netflix-shows
### [Dashboard Link(PDF)](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Netflix_EDA.pdf)

## Data Overview
After importing the dataset into Excel I performed basic cleaning operations replacing all the empty cells with "Null" and splitting the columns that contains more than one parameter. 

![netflix_1](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Pictures/netflix1.png)

I then proceded to create the following csv files:
- Netflix_titles
- Listed in
- Directors
- Description
- Cast
- Countries 

## Description of each column 
- **show_id**: A unique identifier assigned to each element.
- **title**: The title of the show.
- **type**: Specifies the type of content, either "Movie" or "TV Show".
- **director**: The director of the show.
- **cast**: A list of actors in the show.
- **country**: The country where the show was produced.
- **release_year**: The year the show was originally released.
- **date_added**: The date when the show was added to Netflix, formatted as "Month, Day, Year".
- **rating**: The content rating of the show.
- **duration**: The length of the show. For movies, this is the number of minutes. For TV shows, indicates the number of seasons.
- **listed_in**: The genres or categories of the show.
- **description**: A summary or description of the show.

With the following relations: 

![netflix_2](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Pictures/netflix2.png)

I added a `continent` row to both `netflix_data_cast` and `netflix_data_director` to generate a simple view to filter the shows by continent of production in **MySQL**.

```
create or replace view netflix_data.directors_by_coutry  as

select d.*, co.continent
from netflix_directors d
left join countries_released co
on d.show_id = co.show_id and d.show_id is not null
```
## Dashboard Snapshot

![netflix_3](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Pictures/snap1.png)

![netflix_4](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Pictures/Screenshot_6.png)

![netflix_4](https://github.com/ThisIsMirquez/Portfolio/blob/main/Simple%20Netflix%20EDA/Pictures/snap2.png)
