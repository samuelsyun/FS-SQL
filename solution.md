CREATE TABLE movies (
  id INTEGER PRIMARY KEY,
  name TEXT DEFAULT NULL,
  year INTEGER DEFAULT NULL,
  rank REAL DEFAULT NULL
);

CREATE TABLE actors (
  id INTEGER PRIMARY KEY,
  first_name TEXT DEFAULT NULL,
  last_name TEXT DEFAULT NULL,
  gender TEXT DEFAULT NULL
);

CREATE TABLE roles (
  actor_id INTEGER,
  movie_id INTEGER,
  role_name TEXT DEFAULT NULL
);

-- Find all movies made in 1985
SELECT *
  FROM movies
  WHERE year = 1985;


-- How many movies does our dataset have for the year 1990 and 2000?
SELECT COUNT(*)
  FROM movies
  WHERE year = 1990 OR year = 2000
  GROUP BY year;


-- Find actors who have "stack" in their last name
SELECT *
  FROM actors
  WHERE last_name like '%stack%';


-- What are the 10 most popular first names and last names in the business?
-- And how many actors have each given first or last name?
SELECT first_name, COUNT(*) as occurrences
  FROM actors
  GROUP BY first_name
  ORDER BY occurrences DESC
  LIMIT 10;

SELECT last_name, COUNT(*) as actors_number
  FROM actors
  GROUP BY last_name
  ORDER BY actors_number DESC
  LIMIT 10;


-- 10 most popular full names and count
SELECT (first_name || " " || last_name) AS full_name, COUNT(*) AS occurrences
  FROM actors
  GROUP BY first_name, last_name
  ORDER BY occurrences DESC
  LIMIT 10;


-- 50 most active actors and number of roles
SELECT first_name, last_name, COUNT(*) as numberOfRoles
  FROM actors
  INNER JOIN  roles
    ON actors.id = roles.actor_id
  GROUP BY actors.id
  ORDER BY numberOfRoles DESC
  LIMIT 50;

OR

SELECT first_name, last_name, COUNT(*) AS num_roles
  FROM actors, roles, movies
  WHERE actors.id = roles.actor_id
    AND roles.movie_id = movies.id
  GROUP BY actors.id
  ORDER BY num_roles DESC
  LIMIT 50;


-- How many movies does IMDB have of each genre, ordered by least popular genre?
SELECT genre, COUNT(*) as num_movies_by_genres
  FROM movies_genres
  WHERE movie_id IS NOT NULL
  GROUP BY genre
  ORDER BY num_movies_by_genres ASC;

OR

SELECT genre, COUNT(*) AS num_movies_by_genres
  FROM movies_genres
  INNER JOIN movies
    ON movies_genres.movie_id = movies.id
  GROUP BY genre
  ORDER BY num_movies_by_genres ASC;


-- list first name and last name of all actors played in Braveheart in 1995
SELECT first_name, last_name
  FROM actors, roles, movies
  WHERE year = 1995
    AND name = 'Braveheart'
    AND movie_id = movies.id
    AND actor_id = actors.id
  ORDER BY last_name ASC;

OR

SELECT first_name, last_name
  FROM actors
    INNER JOIN roles
      ON actors.id = roles.actor_id
    INNER JOIN movies
      ON roles.movie_id = movies.id
    WHERE movies.name = 'Braveheart'
      AND movies.year = 1995
  ORDER BY actors.last_name ASC;


-- all directors who directed movies with 'Film-Noir' genre in leap year
SELECT
    d.first_name,
    d.last_name,
    m.name,
    m.year
  FROM directors d, movies_directors md, movies m, movies_genres mg
  WHERE d.id = md.director_id
    AND md.movie_id = m.id
    AND mg.movie_id = m.id
    AND mg.genre = 'Film-Noir'
    AND m.year % 4 = 0
  ORDER BY m.year ASC;

OR

SELECT
    d.first_name,
    d.last_name,
    m.name,
    m.year
  FROM directors AS d
  INNER JOIN movies_directors AS md
    ON d.id = md.director_id
  INNER JOIN movies AS m
    ON md.movie_id = m.id
  INNER JOIN movies_genres AS mg
    ON mg.movie_id = m.id
  WHERE mg.genre = 'Film-Noir'
    AND m.year % 4 = 0
  ORDER BY m.name ASC;


-- list all actors that have worke with Kevin Bacon in Drama movies

## with WHERE
SELECT m.name, costars.first_name || " " || costars.last_name AS full_name
  FROM
    actors AS bacon,
    roles AS r,
    movies AS m,
    movies_genres AS mg,
    roles AS costar_roles,
    actors AS costars
  WHERE bacon.first_name = 'Kevin'
    AND bacon.last_name = 'Bacon'
    AND mg.genre = 'Drama'
    AND costars.id <> bacon.id
    AND bacon.id = r.actor_id
    AND r.movie_id = m.id
    AND m.id = mg.movie_id
    AND m.id = costar_roles.movie_id
    AND costar_roles.actor_id = costars.id
  ORDER BY m.name ASC
  LIMIT 100;

OR

## use JOIN and AND and '||' and IN
SELECT m.name, a.first_name || " " || a.last_name AS full_name
  FROM actors AS a
  INNER JOIN roles AS r
    ON r.actor_id = a.id
  INNER JOIN movies AS m
    ON r.movie_id = m.id
  INNER JOIN movies_genres AS mg
    ON mg.movie_id = m.id
    AND mg.genre = 'Drama'
  WHERE m.id IN (
    SELECT bacon_m.id
    FROM movies AS bacon_m
    INNER JOIN roles AS bacon_r
      ON bacon_r.movie_id = bacon_m.id
    INNER JOIN actors AS bacon_a
      ON bacon_r.actor_id = bacon_a.id
    WHERE bacon_a.first_name = 'Kevin'
      AND bacon_a.last_name = 'Bacon'
    )
    AND full_name != 'Kevin Bacon'
  ORDER BY m.name ASC
  LIMIT 100;


-- Which actors have acted in a film before 1900 and also in a film after 2000?

## Using `INTERSECT`
SELECT actors.id, actors.first_name, actors.last_name
  FROM actors
  INNER JOIN roles
    ON roles.actor_id = actors.id
  INNER JOIN movies
    ON movies.id = roles.movie_id
  WHERE movies.year < 1900
  INTERSECT
    SELECT actors.id, actors.first_name, actors.last_name
    FROM actors
    INNER JOIN roles
      ON roles.actor_id = actors.id
    INNER JOIN movies
      ON movies.id = roles.movie_id
    WHERE movies.year > 2000;

OR

## Using `WHERE` and `IN` with a SubQuery
SELECT first_name, last_name
  FROM actors, roles, movies
  WHERE movies.year > 2000
    AND roles.movie_id = movies.id
    AND actors.id = roles.actor_id
    AND roles.actor_id
    IN (
      SELECT roles.actor_id
      FROM roles, movies
      WHERE movies.year < 1900
        AND roles.movie_id=movies.id
    )
  GROUP BY actors.id
  ORDER BY actors.last_name;


-- BUSY FILMING
-- # Find actors that played five or more roles in the same movie after the year 1990.
-- # Notice that ROLES may have occasional duplicates, but we are not interested in these:
-- # we want actors that had five or more distinct roles in the same movie.
-- # Write a query that returns the actors' names, the movie name,
-- # and the number of distinct roles that they played in that movie (which will be ≥ 5).

## Using JOINs
SELECT
    actors.first_name,
    actors.last_name,
    movies.name,
    movies.year,
  COUNT (DISTINCT roles.role) AS num_roles_in_movies -- roles with DIFFERENT role names!
  FROM actors
  INNER JOIN roles
    ON roles.actor_id = actors.id
  INNER JOIN movies
    ON roles.movie_id = movies.id
  WHERE movies.year > 1990
  GROUP BY actors.id, movies.id
  HAVING num_roles_in_movies > 4;

OR

## Using WHERE
SELECT
    actors.first_name,
    actors.last_name,
    movies.name,
    movies.year,
    COUNT (DISTINCT roles.role) AS num_roles_in_movies
  FROM actors, roles, movies
  WHERE movies.year > 1990
    AND actors.id = roles.actor_id
    AND roles.movie_id = movies.id
  GROUP BY roles.actor_id, roles.movie_id
  HAVING num_roles_in_movies > 4;


-- FEMALE ACTORS ONLY
--# For each year, count the number of movies in that year that had only female actors.
--# For movies where no one was casted, you can decide whether to consider them female-only.

## Includes movies with no actors by looking at all movies withOUT male actors. Using `NOT EXISTS`
SELECT m.year, COUNT(*) femaleOnly
  FROM movies m
  WHERE NOT EXISTS (
    SELECT *
    FROM roles r, actors a
    WHERE a.id = r.actor_id
      AND r.movie_id = m.id
      AND a.gender = 'M'
  )
  GROUP BY m.year;

OR

## Includes movies with no actors by looking at all movies withOUT male actors. Using `NOT IN`
SELECT year, COUNT(*)
  FROM movies m
  WHERE id NOT IN (
    SELECT movie_id
    FROM roles r, actors a
    WHERE gender = 'M'
    AND a.id = actor_id
  )
  GROUP BY m.year;

OR

## Excludes movies with NO actors
SELECT m.year, count(*) femaleOnly
  FROM movies m
  WHERE NOT EXISTS (SELECT *
    FROM roles AS ma, actors AS a
    WHERE a.id = ma.actor_id
      AND ma.movie_id = m.id
      AND a.gender = 'M')
  AND EXISTS (SELECT *
    FROM roles AS ma, actors AS a
    WHERE a.id = ma.actor_id
      AND ma.movie_id = m.id
      AND a.gender = 'F')
  GROUP BY m.year;

