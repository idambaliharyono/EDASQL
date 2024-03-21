# SQL x Python Exploratory Data Analysis: Painter Artists' Works from 1395 to 1903 and Their Exhibiting Museums

![dataset-cover](https://github.com/Idam-Bali-Haryono/EDASQL/assets/115137963/02f0f6ad-4cc1-4fe9-aae5-390781781246)

## Introduction
In this project, we embark on an Exploratory Data Analysis (EDA) journey utilizing PostgreSQL as our primary database management system. Leveraging the versatility of Python scripting, we load the Painter Artists' Works and Exhibiting Museums dataset into the PostgreSQL database, laying the groundwork for our analytical exploration. Subsequently, we employ the user-friendly interface of pgAdmin 4 to execute SQL queries, enabling us to formulate and answer pertinent questions about the dataset.

Furthermore, this integrated approach harnesses the power of both Python and PostgreSQL, facilitating seamless data manipulation and analysis. By leveraging Python scripts for data loading and pgAdmin 4 for querying, we aim to streamline the EDA process, unraveling insights into painter artists, their works, and the museums exhibiting them.

Through this methodology, we endeavor to unlock valuable insights and deepen our understanding of the intricate relationships within the art world, spanning from the 14th to the 19th centuries.
### Problem Formulation
1. Formulate insightful questions based on the dataset.
2. Utilize SQL queries to answer the formulated questions.
3. Gain valuable insights into painter artists, their works, and the exhibiting museums.
## Dataset 

This dataset comprises eight .csv files intricately linked through various unique identifiers. It provides a comprehensive overview of renowned painter artists born between 1395 and 1903, along with the museums where their works are exhibited. Each entry includes details on the artist's background, notable works, and the respective museums housing their creations. Moreover, it offers information on museum operating hours and website links for virtual access to the displayed artworks. [access the dataset](https://data.world/atlas-query/paintings)

![dataset](https://github.com/Idam-Bali-Haryono/EDASQL/assets/115137963/a3cccfee-1a1f-46ea-b240-02eb28e23aba)

## Load Dataset to Database Using Python

To streamline preprocessing, we employed Python scripts to swiftly convert .csv files into pandas DataFrames, subsequently facilitating their seamless integration into a PostgreSQL database via SQLAlchemy library.
```python
import pandas as pd
from sqlalchemy import create_engine

conn_string = 'postgresql://postgres:anyut@localhost/Paintings'
db = create_engine(conn_string)
conn = db.connect()


files = ['artist', 'canvas_size', 'image_link', 'museum_hours', 'museum', 'product_size', 'subject', 'work']

for i in files:
    df = pd.read_csv(f'{i}.csv')
    df.to_sql(i, con=conn, if_exists='replace', index=False)

```

Upon examination of the dataset in pgAdmin 4, it was confirmed that the code executed flawlessly.

![dataset2](https://github.com/Idam-Bali-Haryono/EDASQL/assets/115137963/13d43ba5-33bf-4647-b052-4a2818330851)


## EDA
Now that the dataset is loaded into our PostgreSQL database, we proceed with conducting Exploratory Data Analysis (EDA) to extract meaningful insights and patterns from the data.
### Which are the top 5 most popular museum? (Popularity is defined based on most no of paintings in a museum)
```Sql
select m.name as museum, m.city,m.country,x.no_of_painintgs
	from (	select m.museum_id, count(1) as no_of_painintgs
			, rank() over(order by count(1) desc) as rnk
			from work w
			join museum m on m.museum_id=w.museum_id
			group by m.museum_id) x
	join museum m on m.museum_id=x.museum_id
	where x.rnk<=5;
```
Output:
museum   |   city   |   country   |  no_of_painintgs
--- | --- | --- | ---
"The Metropolitan Museum of Art"  | 	"New York"   |	  "USA" |	939

### Who are the top 5 most popular artist? (Popularity is defined based on most no of paintings done by an artist)
 ```sql
select a.full_name as artist, a.nationality,x.no_of_painintgs
	from (	select a.artist_id, count(1) as no_of_painintgs
			, rank() over(order by count(1) desc) as rnk
			from work w
			join artist a on a.artist_id=w.artist_id
			group by a.artist_id) x
	join artist a on a.artist_id=x.artist_id
	where x.rnk<=5;
```
Output:
artist   |   nationality   |   no_of_painintgs
--- | --- | --- 
"Pierre-Auguste Renoir"	| "French"	| 469
"Claude Monet"	| "French"	| 378
"Albert Marquet" |	"French"	| 233
"Maurice Utrillo" |	"French"	| 253
"Vincent Van Gogh" |	"Dutch"	| 308

### Which are the 3 most popular and 3 least popular painting styles?
```sql
with cte as 
		(select style, count(1) as cnt
		, rank() over(order by count(1) desc) rnk
		, count(1) over() as no_of_records
		from work
		where style is not null
		group by style)
	select style
	, case when rnk <=3 then 'Most Popular' else 'Least Popular' end as remarks 
	from cte
	where rnk <=3
	or rnk > no_of_records - 3;
```
Output:
artist   |   nationality 
--- | ---
"Impressionism" |	"Most Popular"
"Post-Impressionism" |	"Most Popular"
"Realism" |	"Most Popular"
"Avant-Garde" |	"Least Popular"
"Art Nouveau" |	"Least Popular"
"Japanese Art" |	"Least Popular"

### Are there museuems without any paintings?
```sql
select name, country from museum m
	where not exists (select 1 from work w
					 where w.museum_id=m.museum_id)
```
Output
name   |   country 
--- | ---

### Which museum has the most no of most popular painting style?
```sql
with pop_style as 
			(select style
			,rank() over(order by count(1) desc) as rnk
			from work
			group by style),
		cte as
			(select w.museum_id,m.name as museum_name,ps.style, count(1) as no_of_paintings
			,rank() over(order by count(1) desc) as rnk
			from work w
			join museum m on m.museum_id=w.museum_id
			join pop_style ps on ps.style = w.style
			where w.museum_id is not null
			and ps.rnk=1
			group by w.museum_id, m.name,ps.style)
	select museum_name,style,no_of_paintings
	from cte 
	where rnk=1;
```
Output:
museum_name   |   style | no_of_paintings 
--- | --- | ---
"The Metropolitan Museum of Art" |	"Impressionism"	| 244
### Which country has the 5th highest no of paintings?
Output:
```sql
with cte as 
		(select m.country, count(1) as no_of_Paintings
		, rank() over(order by count(1) desc) as rnk
		from work w
		join museum m on m.museum_id=w.museum_id
		group by m.country)
	select country, no_of_Paintings
	from cte 
	where rnk=5;
```
country   |   no_of_Paintiings 
--- | ---
"Spain" |	196

## Conclusion

In our comprehensive analysis of the Painter Artists' Works and Exhibiting Museums dataset, we've uncovered key insights into the art world's intricacies. Leveraging PostgreSQL and SQL queries, ***we identified the top five most popular museums, with "The Metropolitan Museum of Art" leading the pack. We also explored the most renowned artists, such as Pierre-Auguste Renoir and Claude Monet, and discerned prevalent painting styles like Impressionism and Realism.** Remarkably, **all museums were found to exhibit paintings,** reflecting a vibrant cultural landscape. Furthermore, our investigation into museum-painting style associations unveiled intriguing patterns, while globally, **Spain emerged as the fifth-largest contributor to the art domain.** This analysis not only deepens our understanding of the dataset but also sheds light on the diverse and captivating world of art history and museum curation practices.
