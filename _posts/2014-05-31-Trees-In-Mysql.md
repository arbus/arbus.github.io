---
layout: post_page
title:  "Trees in MySql"
date:   2014-05-31 22:04:06
categories: mysql
comments: true
---

As you develop more complex apps, the need to model hierarchal relationships between entities arises. This can be for things as simple as a threaded comment list to more complex commission calculations for managers with child agents etc. If we model these relationships according to our querying needs correctly, things will get much easier when the app scales and you have thousands of these entities to manage. In this article, we focus on modeling trees.

### What are trees?

In CS terms, a tree is defined as an undirected graph where each point is connected to every other point through a single path. In simpler terms, each data point we store can have either 0 or 1 parent with our entire set of data points having atleast 1 element with no parent(this is the root node). Throughout the rest of the article, we shall try and model a comment thread that looks like this:

  <pre>
              1
            /   \
           2     3
          /       \
         4         5
       /   \
      6     7
  </pre>  

### Modelling trees

When it comes to modelling trees in a database, the questions that we need to ask ourselves are

+ What is the most performance sensitive operation that I will be doing, inserting, updating or selecting?
+ Will I need to query for a nodes descendants without knowing the depth?
+ Will I need to know the line of ancestors of a node?
+ Will I need to detach a node and all its children and attach it to another node?

As you can imagine, each of the method below produces a very good answer for one of the questions above while being not so good at answering the others. Depending on our needs, we can analyze the trade offs in detail before making a choice. The most common application involves combining at least 2 of the methods described below to adequacy fulfill the needs of the application.

### Intro to stored procedures

Stored procedures are the user defined functions of MySQL. They allow you to perform more complex calculations by defining the procedure once and then just calling it when you need it. It also opens up more flexibility by adding in control flow statements along with loops. 

You can create a stored procedure like so: 
{% highlight sql %}
CREATE PROCEDURE helloWorld()
BEGIN
  SELECT "Hello World";
END;
{% endhighlight %}

Once you have created the procedure, you can use it like so: `CALL helloWorld()`. Using this, you can store complex actions in the database without shuttling the data back and forth between the app and the database. 

### Adjacency list

An adjacency list is the simplest method of storing hierarchal data in MySQL. The idea is that you store the ID of the parent of each element with the root element having just a NULL in the parent column.  So if we were to represent the tree shown above, it would look like this:

  <pre>
  +------+--------+
  | id   | parent |
  +------+--------+
  |    1 |   NULL |
  |    2 |      1 |
  |    3 |      1 |
  |    4 |      2 |
  |    5 |      3 |
  |    6 |      4 |
  |    7 |      4 |
  +------+--------+
  </pre>

Select the children of an element:

{% highlight sql %}
SELECT * FROM comments c WHERE c.parent = 1
{% endhighlight %}

Selecting the grandchildren of an element:

{% highlight sql %}
SELECT c2.* FROM comments c1 LEFT JOIN comments c2 ON c2.parent = c1.id WHERE c1.parent = 1;
{% endhighlight %}

An adjacency list is the easiest way of modeling trees. It requires very little code and less code means less things to go wrong. Also, as opposed to other models, insertion and deletion are very quick operations since inserting or deleting a node only affects the node itself and other nodes need not be updated. 

A major issue with this method is that there is no way for you to query items without knowing their depth. For example, you cannot query for all descendants of an element. You will need to know how many levels you want to go down. Finally, for every level that you want to traverse, you will need an extra join statement. This can bog down your query very quickly, especially with very large tables.

#### Calculating depth

You can enhance the usefulness of an adjacency list by specifying the depth of each element. This step will greatly reduce the complexity when you want to convert from adjacency list to other forms of storing hierarchical data.

This can be accomplished by a simple stored procedure:

{% highlight sql %}

CREATE PROCEDURE calcDepthFromAdjacencyList()
BEGIN
  -- First we reset the depth column
  UPDATE comments SET depth = NULL;
  -- We set the depth to 1 for all root nodes(parent = NULL)
  UPDATE comments SET depth = 1 WHERE parent IS NULL;
  -- We set variable that lets us keep track of the current depth
  SET @depth = 1;
  
  -- This repeat statement will loop until every value has a depth parameter set
  REPEAT
    UPDATE 
      comments c1 
    -- We do this because you cannot update a table with data 
    -- that you select from that table itself.So we select the 
    -- data that we want separately and then reference it as parents
    JOIN(
      SELECT c2.id
      FROM comments c2
      WHERE c2.depth = @depth
    ) AS parents
    -- We set the depth of all the selected children to be 1 more than that of their parents
    SET c1.depth = (@depth + 1) 
    WHERE c1.parent = parents.id;
    
    -- finally, we increment the @depth variable
    SET @depth = @depth + 1;
  UNTIL (SELECT COUNT(1) FROM comments WHERE depth IS NULL) = 0 END REPEAT;
END;
{% endhighlight %}

Running the above code on our dataset will produce:

  <pre>
  +------+--------+--------+
  | id   | parent | depth  |
  +------+--------+--------+
  |    1 |   NULL |      1 |
  |    2 |      1 |      2 |
  |    3 |      1 |      2 |
  |    4 |      2 |      3 |
  |    5 |      3 |      3 |
  |    6 |      4 |      4 |
  |    7 |      4 |      4 |
  +------+--------+--------+
  </pre>

#### When to use it

You should use this if you want very fast inserts, updates and you only look up the immediate hierarchies of the nodes. If you need to look up the descendants of an element of an unknown depth, you will need to look at the 2 models listed below. However, this model is really useful as a complementary to the models listed below as it provides you with enough information to derive them while not needing to be derived in the first place

### Breadcrumb

The breadcrumb model is conceptually similar to the adjacency list model, except that instead of each node storing its direct parent, each node stores its entire list of ancestors up to its root. This can be useful if you need to get the full hierarchy of a node in a single query, especially when you don't know the depth of the node. If we were to represent the tree shown above, it would look like this:

  <pre>
  +------+------------+
  | id   | breadcrumb |
  +------+------------+
  |    1 |          1 |
  |    2 |        1/2 |
  |    3 |        1/3 |
  |    4 |      1/2/4 |
  |    5 |      1/3/5 |
  |    6 |    1/2/4/6 |
  |    7 |    1/2/4/7 |
  +------+------------+
  </pre>

The delimiter used to show the hierarchy is up to you, I have chosen '/' here. 

This query will get you all the descendants of a given node using the breadcrumb model:

{% highlight sql %}
SELECT c1.* FROM comments c1 WHERE breadcrumb LIKE CONCAT(
  (SELECT c2.breadcrumb FROM comments c2 WHERE c2.id = ?),
  '%'
);
{% endhighlight %}

This model is much better at dealing with hierarchies of variable or unknown depth as compared to the adjacency list. It is very useful when you have to get a list of all the parents to verify certain constraints as you can get that list in a single query.

However, when you have to update the position of a element in the hierarchy, it you will also have to update all the descendants of that element. Also, you cannot easily work with the breadcrumb string within MySQL as it does not provide you with an easy string splitting function.

We can calculate the breadcrumb from an adjacency list(w depth calculated) like so:

{% highlight sql %}

CREATE PROCEDURE generateBreadCrumbFromAdjacencyList()
BEGIN
  -- Clear any previously calculated values
  UPDATE comments SET breadcrumb = NULL;
  -- Set the initial crumb for all the root elements(root elements have a depth value of 1)
  UPDATE comments SET breadcrumb = id WHERE depth = 1;
  -- We start with a depth of 2
  SET @depth = 2;
  -- and choose a element which does not have a breadcrumb calculated already.
  SET @id = (SELECT comments.id FROM comments WHERE 
    depth = @depth AND 
    breadcrumb IS NULL 
    ORDER BY comments.id LIMIT 1);
  -- We loop until all the nodes have a breadcrumb value
  REPEAT
    -- If there are any nodes in this depth with an calculated breadcrumb
    IF((SELECT COUNT(1) FROM comments WHERE depth = @depth AND breadcrumb IS NULL) > 0) THEN
      -- Set the breadcrumb of the current element as the 
      -- breadcrumb of the parent along with this elements' id
      SET @crumb = (SELECT CONCAT_WS('/', p.breadcrumb, u.id) 
        FROM comments u, comments p
        WHERE u.parent = p.id AND u.id = @id);

      UPDATE comments u1 SET u1.breadcrumb = @crumb WHERE u1.id = @id;
    -- If all the nodes of the current depth have been calculated, we move to the next level
    ELSE
      SET @depth = @depth + 1;
    END IF;
    -- Select the next element to be updated
    SET @id = (SELECT comments.id FROM comments WHERE depth = @depth AND breadcrumb IS NULL ORDER BY comments.id LIMIT 1);
  UNTIL (SELECT COUNT(1) FROM comments WHERE breadcrumb IS NULL) = 0 END REPEAT;
END;

{% endhighlight %}

#### When to use it

We should use this when we need to get the entire ancestry line of an element in 1 shot. This is useful for testing a particular element to see if it is part of a particular line for filtering. We can also get all the children of the given node regardless of the depth in a single query, which makes this particularly useful.

If your workload involves frequent updating of the hierarchy of an element, you should be aware that you need to recompute the breadcrumb of all the children of the element that you repositioned.

### Nested Sets

Nested sets are the most complex of the 3 models discussed here to implement. To use this model, you need to store 2 values per node, a left and right value. Visualize each node as a circle, with all the children of that node as smaller circles within that node. Once you have all the circles drawn, draw a line across through all the circles. While keeping count of the number of times you cross a border of a circle, set the left value of the node to the count when you enter it and the right value of the node to the count when you exit it. This will look something like this:

  <pre>
                              1                           
   +-----------------------------------------------------+
   |                2                         3          |
   |  +---------------------------+   +---------------+  |
   |  |             4             |   |       5       |  |
   |  |  +---------------------+  |   |  +---------+  |  |
   |  |  |     6        7      |  |   |  |         |  |  |
   |  |  |  +-----+  +-----+   |  |   |  |         |  |  |
  1| 2| 3| 4|     |56|     |7  |8 |910|11|         |12|13|14
  +-----------------------------------------------------------+
   |  |  |  |     |  |     |   |  |   |  |         |  |  |
   |  |  |  +-----+  +-----+   |  |   |  |         |  |  |
   |  |  |                     |  |   |  |         |  |  |
   |  |  +---------------------+  |   |  +---------+  |  |
   |  |                           |   |               |  |
   |  +---------------------------+   +---------------+  |
   |                                                     |
   +-----------------------------------------------------+
  </pre>

We can then store this on the table like so:

  <pre>
  +------+-----+-----+
  | id   | lft | rgt |
  +------+-----+-----+
  |    1 |   1 |  14 |
  |    2 |   2 |   9 |
  |    3 |  10 |  13 |
  |    4 |   3 |   8 |
  |    5 |  11 |  12 |
  |    6 |   4 |   5 |
  |    7 |   6 |   7 |
  +------+-----+-----+
  </pre>

These calculated left and right numbers have a few nice properties

Get all decendants:

{% highlight sql %}
SELECT c2.* FROM (
  SELECT c1.lft, c1.rgt FROM comments c1 WHERE c1.id = ?
) AS root, comments c2
WHERE c2.lft > root.lft AND c2.rgt < root.rgt;
{% endhighlight %}

Get all ancestors:

{% highlight sql %}
SELECT c2.* FROM (
  SELECT c1.lft, c1.rgt FROM comments c1 WHERE c1.id = ?
) AS root, comments c2
WHERE c2.lft < root.lft AND c2.rgt > root.rgt;
{% endhighlight %}

We can also use the way that left and right are calculated to figure out things about a node without making any other queries.

+ A node is a leaf(no children) if `lft + 1 === rgt`
+ A node has `((rgt - lft + 1) / 2) - 1` children
+ The first direct child of a node has a `lft` value of the nodes' `lft` + 1
+ The last direct child of a node has a `rgt` value of the nodes' `rgt` - 1

We can calculate the left and right values assuming we have the data from an adjacency list with the following stored procedure:

{% highlight sql %}
CREATE PROCEDURE calcNestedSetFromAdjacencyList()
BEGIN
  -- Start by setting the root elements' parent to -1
  UPDATE comments SET parent = -1 WHERE parent IS NULL;
  -- Clear out the old lft and rgt values
  UPDATE comments SET lft = NULL, rgt = NULL;
  
  -- Start our counter at 1
  SET @n = 1;
  -- Choose the id of one root element to start with
  SET @id = (SELECT id FROM comments WHERE parent = -1 ORDER BY id DESC LIMIT 1);

  -- Repeat until we have set rgt values for every node
  REPEAT
    -- Set the initial left value
    UPDATE comments SET lft = @n WHERE id = @id AND lft IS NULL;
    -- Increment the counter
    SET @n = @n + 1;
    -- If we have children for the current node, point to them
    IF ((SELECT COUNT(1) FROM comments WHERE parent = @id AND rgt IS NULL) > 0) THEN
     SET @id = (SELECT id FROM comments WHERE 
        parent = @id AND 
        lft IS NULL 
        ORDER BY id DESC LIMIT 1);
    -- If we don't have children, set the right value and point back to the parent
    ELSE
     UPDATE comments SET rgt = @n WHERE id = @id;
      SET @id = (SELECT parent FROM comments WHERE id = @id);
    END IF;
  UNTIL (SELECT COUNT(1) FROM comments WHERE rgt IS NULL) = 0 END REPEAT;
  
  -- Reset back the value of root nodes
  UPDATE comments SET parent = NULL WHERE parent = -1;
END;
{% endhighlight %}

#### When to use it

You should use this when you want to get both the ancestors and the descendants of an element of unknown depth while with the breadcrumb, you could only get the descendants. The nested sets method of querying is also much faster then the breadcrumbs(even if you have a index on the breadcrumb column).

Unlike the breadcrumb, when a position of a node changes, the entire set of lft and rgt values need to be recalculated instead just the children of the node that was modified. This applies to insertion as well. 


### Triggers

Triggers are similar to procedures in that you define a set of instructions which are then executed on the server. Unlike procedures, triggers execute given a set of conditions match. We can use these triggers to make sure that any of the updates that we do to the system do not invalidate tree model assumption

{% highlight sql %}

CREATE TRIGGER `treeModelInsertCheck` BEFORE INSERT ON `comments`
FOR EACH ROW
BEGIN
  -- Make sure that the parent of the element that you are trying to insert exists
  IF (SELECT COUNT(1) FROM comments WHERE id = NEW.parent) != 1 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'You are trying to insert an item pointing to a parent that does not exist!';
  END IF;
END;

{% endhighlight %}

{% highlight sql %}

CREATE TRIGGER `treeModelUpdateCheck` BEFORE UPDATE ON `comments`
FOR EACH ROW
BEGIN
  -- Check that the parent referred to in updated row exists
  IF (SELECT COUNT(1) FROM comments WHERE id = NEW.parent) != 1 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'You are trying to update an item to point to a parent that does not exist!';
  END IF;
  -- Make sure that atleast 1 root element exists
  IF (SELECT COUNT(1) FROM comments WHERE parent IS NULL) = 0 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'You need atleast 1 root element';
  END IF;  
END;

{% endhighlight %}

### Conclusion

+ If you care a lot about select and not much about insertion and updating, go with nested sets.
+ If you care about selecting but also about insertion, go with breadcrumb.
+ If you care about insertion and updating and not much about selection, go with adjacency list.

There is no reason that you cannot implement all 3 models on the same table and take advantage of each for the different workloads that you have. 
