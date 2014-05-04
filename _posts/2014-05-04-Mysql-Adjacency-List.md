---
layout: post_page
title:  "Trees in MySql - Adjacency List"
date:   2014-05-04 22:04:06
categories: mysql
---

In this series of posts, we shall consider the problem of modeling trees in MySQL.

To give us some data to work with, let us consider trying to represent a comment thread with the following structure:

<pre>
            1
          /   \
         2     3
        /       \
       4         5
     /   \
    6     7
</pre>  

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

### Pros

An adjacency list is the easiest way of modeling trees. It requires very little code and less code means less things to go wrong. Also, as opposed to other models, insertion and deletion are very quick operations since inserting or deleting a node only affects the node itself and other nodes need not be updated. 

### Cons

A major issue with this method is that there is no way for you to query items without knowing their depth. For example, you cannot query for all descendants of an element. You will need to know how many levels you want to go down. Finally, for every level that you want to traverse, you will need an extra join statement. This can bog down your query very quickly, especially with very large tables. 

### Depth

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
    JOIN(
      SELECT c2.id
      FROM comments c2
      WHERE c2.depth = @depth
    ) AS parents
    SET c1.depth = (@depth + 1) 
    WHERE c1.parent = parents.id;
    SET @depth = @depth + 1;
  UNTIL (SELECT COUNT(1) FROM comments WHERE depth IS NULL) = 0 END REPEAT;
END;
{% endhighlight %}

The above procedure only works if there is a root(ie no cyclic references) and each element is connected to a parent(no orphaned elements). You can ensure this by adding a trigger on the table like so:

{% highlight sql %}

CREATE TRIGGER `adjacencyListSanityCheck` BEFORE INSERT ON `comments`
FOR EACH ROW
BEGIN
  IF (SELECT COUNT(1) FROM comments WHERE id = NEW.parent) != 1 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'You are trying to insert an item pointing to a parent that does not exist!';
  ELSE
    -- If the validation passes, you can recalculate the depth
    CALL calcDepthFromAdjacencyList();
  END IF;
END;

{% endhighlight %}


You can recreate the trigger for updates as well by replacing `BEFORE INSERT` to `BEFORE UPDATE`

You should consider adjacency list only if the data that you are trying to model will have frequent updates and you don't need to query anything of indeterminate depth or anything that is too deep. For cases that you do need to do that, we can consider other models like nested sets and breadcrumbs.
