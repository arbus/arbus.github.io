---
layout: post_page
title:  "Graphs in MySql"
date:   2014-06-09 22:04:06
categories: mysql
comments: true
---

This is a follow up to the post on modeling [trees in mysql]({% post_url 2014-05-31-Trees-In-Mysql %}).

Some of the restrictions that we were able to assume with trees no longer apply to graphs. This can be in the form of there being circular paths(ie no one main root element) or one node having more than one parent.

For graphs, we are concerned with 2 major classes of questions, reachability(ie can I get to a node from another node) and path finding(what is the shortest path from one node to another node). To try and answer those questions, we need some data to model so we can better visualize the problem. Let us try and model the following section from the London railway system:

<p align="center"><img src="{{site.url}}/img/london_tube.png"></p>

For trees, parent relationships were a 1 to many relationship, ie each node has only 1 parent but many children. However with graphs, this become a many to many relationship, which means we can no longer store the relationship information in the same table as the node itself. So lets create 2 tables, 1 to store the stations and another to store the tracks which connect the stations. 

{% highlight sql %}
CREATE TABLE stations(
  name varchar(255) NOT NULL,
  PRIMARY KEY (name)
);

INSERT INTO stations VALUES ('Notting Hill Gate');
INSERT INTO stations VALUES ('Lancaster Gate');
INSERT INTO stations VALUES ('Bond Street');
INSERT INTO stations VALUES ('Oxford Circus');
INSERT INTO stations VALUES ('Tottenham Court Road');
INSERT INTO stations VALUES ('Goodge Street');
INSERT INTO stations VALUES ('Holborn');
INSERT INTO stations VALUES ('Covent Garden');
INSERT INTO stations VALUES ('Leicester Square');
INSERT INTO stations VALUES ('Charing Cross');
INSERT INTO stations VALUES ('Embankment');
INSERT INTO stations VALUES ('Westminster');
INSERT INTO stations VALUES ('St. James Park');
INSERT INTO stations VALUES ('Victoria');
INSERT INTO stations VALUES ('Sloane Square');
INSERT INTO stations VALUES ('South Kensington');
INSERT INTO stations VALUES ('Gloucester Road');
INSERT INTO stations VALUES ('High Street Kensington');
INSERT INTO stations VALUES ('Knightsbridge');
INSERT INTO stations VALUES ('Hyde Park Corner');
INSERT INTO stations VALUES ('Green Park');
INSERT INTO stations VALUES ('Piccadilly Circus');
{% endhighlight %}

Now to create the table to hold the tracks. Here, we are modeling the directional information as well. This allows us to also account for 1 way routes although they are not part of this particular data set.

{% highlight sql %}
CREATE TABLE tracks(
  depart varchar(255),
  arrive varchar(255),
  PRIMARY KEY (depart, arrive)
);

INSERT INTO tracks VALUES('Notting Hill Gate', 'Queensway');
INSERT INTO tracks VALUES('Notting Hill Gate', 'High Street Kensington');
INSERT INTO tracks VALUES('Queensway', 'Notting Hill Gate');
INSERT INTO tracks VALUES('Queensway', 'Lancaster Gate');
INSERT INTO tracks VALUES('Lancaster Gate', 'Queensway');
INSERT INTO tracks VALUES('Lancaster Gate', 'Marble Arch');
INSERT INTO tracks VALUES('Marble Arch', 'Lancaster Gate');
INSERT INTO tracks VALUES('Marble Arch', 'Bond Street');
INSERT INTO tracks VALUES('Bond Street', 'Marble Arch');
INSERT INTO tracks VALUES('Bond Street', 'Oxford Circus');
INSERT INTO tracks VALUES('Bond Street', 'Green Park');
INSERT INTO tracks VALUES('Oxford Circus', 'Bond Street');
INSERT INTO tracks VALUES('Oxford Circus', 'Green Park');
INSERT INTO tracks VALUES('Oxford Circus', 'Piccadilly Circus');
INSERT INTO tracks VALUES('Oxford Circus', 'Tottenham Court Road');
INSERT INTO tracks VALUES('Tottenham Court Road', 'Oxford Circus');
INSERT INTO tracks VALUES('Tottenham Court Road', 'Leicester Square');
INSERT INTO tracks VALUES('Tottenham Court Road', 'Holborn');
INSERT INTO tracks VALUES('Tottenham Court Road', 'Goodge Street');
INSERT INTO tracks VALUES('Goodge Street', 'Tottenham Court Road');
INSERT INTO tracks VALUES('Holborn', 'Tottenham Court Road');
INSERT INTO tracks VALUES('Holborn', 'Covent Garden');
INSERT INTO tracks VALUES('Covent Garden', 'Holborn');
INSERT INTO tracks VALUES('Covent Garden', 'Leicester Square');
INSERT INTO tracks VALUES('Leicester Square', 'Covent Garden');
INSERT INTO tracks VALUES('Leicester Square', 'Tottenham Court Road');
INSERT INTO tracks VALUES('Leicester Square', 'Piccadilly Circus');
INSERT INTO tracks VALUES('Leicester Square', 'Charing Cross');
INSERT INTO tracks VALUES('Charing Cross', 'Leicester Square');
INSERT INTO tracks VALUES('Charing Cross', 'Embankment');
INSERT INTO tracks VALUES('Charing Cross', 'Piccadilly Circus');
INSERT INTO tracks VALUES('Embankment', 'Charing Cross');
INSERT INTO tracks VALUES('Embankment', 'Westminster');
INSERT INTO tracks VALUES('Westminster', 'Embankment');
INSERT INTO tracks VALUES('Westminster', 'St. James Park');
INSERT INTO tracks VALUES('St. James Park', 'Westminster');
INSERT INTO tracks VALUES('St. James Park', 'Victoria');
INSERT INTO tracks VALUES('Victoria', 'St. James Park');
INSERT INTO tracks VALUES('Victoria', 'Green Park');
INSERT INTO tracks VALUES('Victoria', 'Sloane Square');
INSERT INTO tracks VALUES('Sloane Square', 'Victoria');
INSERT INTO tracks VALUES('Sloane Square', 'South Kensington');
INSERT INTO tracks VALUES('South Kensington', 'Sloane Square');
INSERT INTO tracks VALUES('South Kensington', 'Knightsbridge');
INSERT INTO tracks VALUES('South Kensington', 'Gloucester Road');
INSERT INTO tracks VALUES('Gloucester Road', 'South Kensington');
INSERT INTO tracks VALUES('Gloucester Road', 'High Street Kensington');
INSERT INTO tracks VALUES('High Street Kensington', 'Gloucester Road');
INSERT INTO tracks VALUES('High Street Kensington', 'Notting Hill Gate');
INSERT INTO tracks VALUES('Knightsbridge', 'South Kensington');
INSERT INTO tracks VALUES('Knightsbridge', 'Hyde Park Corner');
INSERT INTO tracks VALUES('Hyde Park Corner', 'Knightsbridge');
INSERT INTO tracks VALUES('Hyde Park Corner', 'Green Park');
INSERT INTO tracks VALUES('Green Park', 'Hyde Park Corner');
INSERT INTO tracks VALUES('Green Park', 'Bond Street');
INSERT INTO tracks VALUES('Green Park', 'Oxford Circus');
INSERT INTO tracks VALUES('Green Park', 'Westminster');
INSERT INTO tracks VALUES('Green Park', 'Victoria');
INSERT INTO tracks VALUES('Green Park', 'Piccadilly Circus');
INSERT INTO tracks VALUES('Piccadilly Circus', 'Green Park');
INSERT INTO tracks VALUES('Piccadilly Circus', 'Oxford Circus');
INSERT INTO tracks VALUES('Piccadilly Circus', 'Leicester Square');
INSERT INTO tracks VALUES('Piccadilly Circus', 'Charing Cross');
{% endhighlight %}

Now, to be able to answer our initial questions, we need to construct a 3rd final table that represents the possible trips that once can make now that we have defined all the stations as well as the direction of the tracks that connect these stations. This table will keep track of the trips' departure station, arrival station, the number of stations that you have to pass thought to get to your destination as well as the route that you had to take.

{% highlight sql %}
CREATE TABLE trips(
  depart VARCHAR(255) NOT NULL,
  arrive VARCHAR(255) NOT NULL,
  hops SMALLINT,
  route TEXT
);
{% endhighlight %}

<br>

> Views in MySQL are SELECT statements that can be queried like a table

Now that we have a initial set of possible journeys, we can build upon these journeys using a intermediate VIEW to assist us.

{% highlight sql %}
CREATE VIEW routebuilder AS
  SELECT
    trips.depart AS depart,
    tracks.arrive AS arrive,
    trips.hops + 1 AS hops,
    CONCAT(trips.route, ',', tracks.arrive) AS route
  FROM trips INNER JOIN tracks
    ON trips.arrive = tracks.depart
    AND LOCATE(tracks.arrive, trips.route) = 0
  GROUP BY depart, arrive;
{% endhighlight %}

Finally, we create a stored procedure that will seed the `trips` table with all the possible trips using the `routebuilder` view
{% highlight sql %}
CREATE PROCEDURE buildTrips()
BEGIN
  DECLARE row_count INT DEFAULT 0;
  TRUNCATE TABLE trips;
  -- First, we seed the trips table with all the 1 hop routes that are possible. This is basically just duplicating the data in the tracks table.
  INSERT INTO trips
    SELECT
      tracks.depart, tracks.arrive, 1, CONCAT(tracks.depart, ',', tracks.arrive)
    FROM tracks;

  SET row_count = ROW_COUNT();
  WHILE(row_count > 0) DO
    -- Next, for each of the possible journeys, we build upon it using the routebuilder view.
    INSERT INTO trips
      SELECT
        routebuilder.depart,
        routebuilder.arrive,
        routebuilder.hops,
        routebuilder.route
      FROM routebuilder
      LEFT JOIN trips ON 
        routebuilder.depart = trips.depart AND routebuilder.arrive = trips.arrive
      WHERE trips.depart IS NULL AND trips.arrive IS NULL;
    SET row_count = ROW_COUNT();
  END WHILE;
END;
{% endhighlight %}

Once the `buildTrips` procedure is called, the `trips` table will be seeded with the shortest possible routes from all possible start and end points. So using this method, we can answer our initial 2 questions in one shot.

{% highlight sql %}
SELECT * FROM trips WHERE depart = ? AND arrive = ?;
{% endhighlight %}

If the query returns a row, then the 2 points are reachable and the `route` column of the query will show the shortest route possible.

#### Conclusions

This approach works really well when your nodes and edges don't change a lot. In this case, we can always be sure that recalculating data will be significantly faster than building new stations and laying down new track! This precalculation step provides extremely fast lookup times for routes. The tradeoff is that you have to recalculate the `trips` table every time any new stations or tracks are laid.
