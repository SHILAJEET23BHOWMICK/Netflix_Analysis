# **NETFLIX_nd_chill**

---

**OVERVIEW:**

---

This project focuses on a thorough analysis of Netflix’s movies and TV shows dataset using SQL, aiming to uncover meaningful insights and answer important business questions. The README provides a c[...]

---

**OBJECTIVE:**

---

The main objectives of the project include:  
- Understanding the distribution between movies and TV shows available on Netflix.  
- Identifying the most popular content ratings for both movies and TV shows to gain insight into the target audiences.  
- Examining content based on factors like release year, countries of origin, and duration to recognize trends and patterns.  
- Categorizing and analyzing content based on specific criteria and keywords to provide deeper understanding and support decision-making.  
- Overall, this project offers valuable perspectives on Netflix’s content library, helping guide strategies around content planning and audience engagement.

---

**SCHEMA:**

---

```sql
create database netflix;

use netflix;

create table movie_shows
(
    show_id VARCHAR(5),
    type VARCHAR(10),
    title VARCHAR(250),
    director VARCHAR(550),
    casts VARCHAR(1050),
    country VARCHAR(550),
    date_added VARCHAR(55),
    release_year INT,
    rating VARCHAR(15),
    duration VARCHAR(15),
    listed_in VARCHAR(250),
    description VARCHAR(550)
);
```

---

**IMPORTING:**

---

As we were facing problems to import csv file data in mysql workbench,  
so in workbench we used:

```sql
set global local_infile = ON;

LOAD DATA LOCAL INFILE 'D:/Projects DS/Netflix/netflix_titles.csv'
INTO TABLE movie_shows
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(show_id, type, title, director, casts, country, date_added, release_year, rating, duration, listed_in, description);
```
# Here type your own path.

---

**#1) Checking the number of movies and tv shows**

---

```sql
SELECT TYPE, COUNT(*) FROM MOVIE_SHOWS
GROUP BY TYPE;
```

---

**#2) Checking the most common type of rating**

---

```sql
SELECT TYPE,
       RATING
FROM (
    SELECT TYPE,
           RATING,
           COUNT(*) AS CNT,
           RANK() OVER (PARTITION BY TYPE ORDER BY COUNT(*) DESC) AS RANKING
    FROM MOVIE_SHOWS
    GROUP BY 1, 2
) AS T1
WHERE RANKING = 1;
```
# Here we cannot use max(rating) because rating is string here

---

**#3) Listing all the movies on a specific year**

---

We will ask the year from the user and then, it's not pre-specified:

```sql
DELIMITER //

CREATE PROCEDURE mov(IN input_year INT)
BEGIN
    SELECT * 
    FROM movie_shows
    WHERE type = 'Movie'
      AND release_year = input_year;
END //

DELIMITER ;

call mov(specific year);

delimiter;
```

---

**#4) Find the top 5 countries with most content on Netflix**

---

```sql
SELECT TRIM(jt.country) AS country,
       COUNT(ms.show_id) AS total_count
FROM movie_shows ms
JOIN JSON_TABLE(
        CONCAT('["', REPLACE(ms.country, ',', '","'), '"]'),
        '$[*]' COLUMNS(country VARCHAR(100) PATH '$')
     ) jt
ON TRUE
GROUP BY TRIM(jt.country)
ORDER BY total_count DESC
LIMIT 5;
```
# Here we will get count of rows where the country is null so we will add condition

```sql
SELECT TRIM(jt.country) AS country,
       COUNT(ms.show_id) AS total_count
FROM movie_shows ms
JOIN JSON_TABLE(
        CONCAT('["', REPLACE(ms.country, ',', '","'), '"]'),
        '$[*]' COLUMNS(country VARCHAR(100) PATH '$')
     ) jt
ON TRUE
WHERE jt.country IS NOT NULL AND TRIM(jt.country) <> ''
GROUP BY TRIM(jt.country)
ORDER BY total_count DESC
LIMIT 5;
```

---

**6) Identify the longest movie;**

---

```sql
SELECT 
    title,
    type,
    CAST(TRIM(REPLACE(duration, 'min', '')) AS UNSIGNED) AS minutes
FROM movie_shows
WHERE type = 'Movie'
  AND duration LIKE '%min%'
ORDER BY minutes DESC
LIMIT 1;
```

---

**7) List the movies that were added in the last 5 years**

---

```sql
SELECT COUNT(*) FROM movie_shows
WHERE STR_TO_DATE(date_added,'%M %d,%Y') >= CURDATE() - INTERVAL 5 YEAR;
```

---

**8) List the TV Shows with more than 5 Seasons**

---

```sql
SELECT * FROM MOVIE_SHOWS 
WHERE TYPE = 'TV SHOW'
AND CAST(SUBSTRING_INDEX(DURATION, '  ', 1 ) AS UNSIGNED) > 5;
```

---

**9) Count the Number of Content Items in Each Genre**

---

```sql
SELECT 
    TRIM(jt.genre) AS genre,
    COUNT(*) AS total_content
FROM movie_shows ms
JOIN JSON_TABLE(
        CONCAT('["', REPLACE(ms.listed_in, ',', '","'), '"]'),
        '$[*]' COLUMNS (genre VARCHAR(255) PATH '$')
     ) jt
ON TRUE
WHERE jt.genre IS NOT NULL 
  AND TRIM(jt.genre) <> ''
GROUP BY TRIM(jt.genre)
ORDER BY total_content DESC;
```

---

**10) Find each year and the average numbers of content release in India on Netflix (top 5 years with highest avg content release)**

---

```sql
SELECT
    country,
    release_year,
    COUNT(show_id) AS total,
    ROUND(
        COUNT(show_id) * 100.0 /
        (SELECT COUNT(show_id) FROM movie_shows WHERE country = 'India'),
        2
    ) AS avg_release
FROM movie_shows
WHERE country = 'India'
GROUP BY country, release_year
ORDER BY avg_release DESC
LIMIT 5;
```

---

**11) List all the shows with Documentary genre**

---

```sql
SELECT TITLE 
FROM MOVIE_SHOWS
WHERE LISTED_IN COLLATE utf8mb4_general_ci LIKE '%Documentaries%';
```

---

**12) Find how many movies actor 'Salman Khan' appeared in the last 10 years**

---

```sql
SELECT * 
FROM MOVIE_SHOWS 
WHERE casts COLLATE utf8mb4_general_ci LIKE '%Salman Khan%'
  AND release_year > YEAR(CURDATE()) - 10;
```

---

**13) Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India**

---

```sql
SELECT 
    TRIM(jt.casts) AS casts_,
    COUNT(*) AS appearance
FROM movie_shows ms
JOIN JSON_TABLE(
    CONCAT('["', REPLACE(COALESCE(ms.casts, ''), ',', '","'), '"]'),
    '$[*]' COLUMNS (
        casts VARCHAR(255) PATH '$'
    )
) AS jt ON TRUE
WHERE ms.country = 'India'
  AND jt.casts IS NOT NULL
  AND TRIM(jt.casts) <> ''
GROUP BY casts_
ORDER BY appearance DESC
LIMIT 10;
```

---

**14) Show Titles Containing 'Love' and Their Release Dates**

---

```sql
SELECT TITLE, RELEASE_YEAR
FROM MOVIE_SHOWS
WHERE TITLE LIKE '%Love%'
GROUP BY TITLE;
```

---

**15) Categorize Content Based on the User Presence of 2 words**

---

```sql
DELIMITER //

CREATE PROCEDURE categorize_content_counts_dynamic23(
    IN keyword1 VARCHAR(100),
    IN keyword2 VARCHAR(100),
    IN output1 VARCHAR(100),
    IN output2 VARCHAR(100)
)
BEGIN
    SELECT CATEGORY, COUNT(*) AS COUNT_OF_CATEGORY
    FROM (
        SELECT 
            CASE
                WHEN DESCRIPTION LIKE CONCAT('%', keyword1, '%') 
                     OR DESCRIPTION LIKE CONCAT('%', keyword2, '%') THEN output1
                ELSE output2
            END AS CATEGORY
        FROM MOVIE_SHOWS
    ) AS CATEGORISED_CONTENT
    GROUP BY CATEGORY;
END
//

DELIMITER ;

call categorize_content_count_dynamic23(keyword1, keyword2, output1, output2)
```
# Here when we call the
